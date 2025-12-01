# Final-Project
# -*- coding: utf-8 -*-
import time
import requests
import BlynkLib
from BlynkTimer import BlynkTimer
from gpiozero import PWMLED, MotionSensor, DigitalOutputDevice, MCP3008
from picamera2 import Picamera2
from datetime import datetime
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email.mime.text import MIMEText
from email import encoders
from mydht import MyDHT11   # your custom DHT11 class

# -----------------------------
# Configuration
# -----------------------------
PIR_PIN = 23
LED_PIN = 18
DHT_PIN = 17
LDR_CHANNEL = 0  
FAN_IN1 = 27  
FAN_IN2 = 22  
TEMP_THRESHOLD = 28.0   
HUM_THRESHOLD = 60.0    

SENDER_EMAIL = "smroberts2004@gmail.com"
APP_PASSWORD = "cqlt prtu zbml ytqe"
RECEIVER_EMAIL = "robertssm@g.cofc.edu"

BLYNK_AUTH = "ppOrmgDCYMuAQWq8HchEcYVQfUonO24m"

V_LED_SWITCH = "V1"
V_LED_SLIDER = "V2"
V_TEMP = "V5"

# -----------------------------
# Hardware Setup
# -----------------------------
pir = MotionSensor(PIR_PIN)
led = PWMLED(LED_PIN)
ldr = MCP3008(channel=LDR_CHANNEL)

fan_in1 = DigitalOutputDevice(FAN_IN1)
fan_in2 = DigitalOutputDevice(FAN_IN2)

camera = Picamera2()
camera.configure(camera.create_still_configuration())

sensor = MyDHT11(DHT_PIN)

# -----------------------------
# Blynk Setup
# -----------------------------
blynk = BlynkLib.Blynk(BLYNK_AUTH)
timer = BlynkTimer()

# -----------------------------
# Functions
# -----------------------------
def send_email_with_attachment(image_path):
    msg = MIMEMultipart()
    msg["From"] = SENDER_EMAIL
    msg["To"] = RECEIVER_EMAIL
    msg["Subject"] = "Motion Detected - Photo Attached"
    msg.attach(MIMEText("Motion detected. See attached image.", "plain"))
    with open(image_path, "rb") as attachment:
        mime = MIMEBase("application", "octet-stream")
        mime.set_payload(attachment.read())
    encoders.encode_base64(mime)
    mime.add_header("Content-Disposition", f"attachment; filename={image_path}")
    msg.attach(mime)
    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
        smtp.login(SENDER_EMAIL, APP_PASSWORD)
        smtp.send_message(msg)
    print("Email sent!")

def get_temp_hum():
    humidity, temperature = sensor.read_data()
    if humidity is not None and temperature is not None:
        print(f"Temp: {temperature:.1f} C   Humidity: {humidity:.1f}%")
        return temperature, humidity
    else:
        print("Failed to read sensor")
        return None, None

def update_fan(temp, hum):
    if temp is None or hum is None:
        return
    if temp >= TEMP_THRESHOLD or hum >= HUM_THRESHOLD:
        fan_in1.on()
        fan_in2.off()
        print("Fan ON")
    else:
        fan_in1.off()
        fan_in2.off()
        print("Fan OFF")

def check_motion():
    if pir.motion_detected:
        print("Motion detected!")
        timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
        filename = f"motion_{timestamp}.jpg"
        camera.start()
        time.sleep(1)
        camera.capture_file(filename)
        camera.stop()
        print(f"Image saved as {filename}")
        send_email_with_attachment(filename)

def adjust_led():
    ambient = ldr.value
    if ambient < 0.3:
        led.value = 1.0
    elif ambient < 0.6:
        led.value = 0.5
    else:
        led.value = 0.0
    print(f"Ambient light: {ambient:.2f}, LED PWM: {led.value:.2f}")

def publish_data():
    temp, hum = get_temp_hum()
    if temp is not None:
        blynk.virtual_write(V_TEMP, temp)
    update_fan(temp, hum)
    adjust_led()
    check_motion()

# -----------------------------
# Blynk Event Handlers
# -----------------------------
@blynk.on("connected")
def blynk_connected(ping):
    print("Blynk ready. Ping:", ping, "ms")

@blynk.on(V_LED_SWITCH)
def v1_write_handler(value):
    if int(value[0]) == 1:
        led.on()
        print("LED turned ON via Blynk switch")
    else:
        led.off()
        print("LED turned OFF via Blynk switch")

@blynk.on(V_LED_SLIDER)
def v2_write_handler(value):
    brightness = int(value[0])
    led.value = brightness / 100
    print(f"Blynk slider value: {brightness}, LED PWM: {led.value:.2f}")

# -----------------------------
# Timer
# -----------------------------
timer.set_interval(3, publish_data)

# -----------------------------
# Main Loop
# -----------------------------
print("System ready. Monitoring environment and motion...")
try:
    while True:
        blynk.run()
        timer.run()
except KeyboardInterrupt:
    led.off()
    fan_in1.off()
    fan_in2.off()
    print("System stopped")
