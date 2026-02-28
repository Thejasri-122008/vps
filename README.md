import time
import random
from datetime import datetime
from twilio.rest import Client

# ==========================================
# CONFIGURATION (REPLACE WITH YOUR DETAILS)
# ==========================================
TWILIO_SID = "your_twilio_sid_here"
TWILIO_AUTH_TOKEN = "your_twilio_auth_token_here"
TWILIO_PHONE_NUMBER = "+1234567890" # Your Twilio number
PATIENT_PHONE_NUMBER = "+0987654321" # The mobile number to alert

# Thresholds for Shunt Failure
# ICP: Intracranial Pressure (mmHg)
# Flow: Cerebrospinal Fluid flow (ml/min)
NORMAL_MAX_PRESSURE = 15.0 
MIN_NORMAL_FLOW = 0.5 

class VPShuntMonitor:
    def __init__(self):
        self.client = Client(TWILIO_SID, TWILIO_AUTH_TOKEN)
        self.last_alert_time = 0
        self.alert_cooldown = 3600  # Only send alert once per hour (to prevent spam)
        self.system_failure_detected = False

    def get_sensor_readings(self):
        """
        In a real application, this function would read from 
        GPIO pins or a serial connection to a pressure/flow sensor.
        Here, we simulate random sensor data.
        """
        # Simulating: Normal state mostly, but occasionally failing
        if random.random() > 0.9:
            # Simulating Failure: High Pressure + Zero/Low Flow
            return 25.0, 0.0 
        else:
            # Simulating Normal: Low pressure, normal flow
            return 10.0, 1.5

    def send_sms_alert(self, message):
        current_time = time.time()
        
        # Check cooldown to prevent spamming the phone every second
        if current_time - self.last_alert_time > self.alert_cooldown:
            try:
                print(f"--- ALERT TRIGGERED: {message} ---")
                message = self.client.messages.create(
                    body=f"VP SHUNT ALERT: {message} at {datetime.now().strftime('%H:%M:%S')}",
                    from_=TWILIO_PHONE_NUMBER,
                    to=PATIENT_PHONE_NUMBER
                )
                print(f" SMS sent successfully! SID: {message.sid}")
                self.last_alert_time = current_time
            except Exception as e:
                print(f"Error sending SMS: {e}")
        else:
            print(f"Alert condition met, but waiting for cooldown timer.")

    def monitor_loop(self):
        print("VP Shunt Monitoring System Started...")
        
        while True:
            try:
                # 1. Read Data
                current_pressure, current_flow = self.get_sensor_readings()
                
                print(f"Reading -> Pressure: {current_pressure} mmHg | Flow: {current_flow} ml/min")

                # 2. Analyze Logic for VP Shunt Failure
                # Failure Scenario: Pressure is High (Blockage/Overdrainage) 
                # BUT Flow is Zero or Low (Shunt not working)
                
                if current_pressure > NORMAL_MAX_PRESSURE and current_flow < MIN_NORMAL_FLOW:
                    status = "CRITICAL FAILURE: High Pressure with No Flow Detected"
                    self.send_sms_alert(status)
                
                elif current_pressure > (NORMAL_MAX_PRESSURE + 5):
                     # Backup check: Just extremely high pressure
                    status = "WARNING: Extremely High Intracranial Pressure"
                    self.send_sms_alert(status)

                # Wait 5 seconds before next reading
                time.sleep(5)

            except KeyboardInterrupt:
                print("Monitoring stopped by user.")
                break
            except Exception as e:
                print(f"System Error: {e}")
                time.sleep(5)

if __name__ == "__main__":
    # Initialize and run
    monitor = VPShuntMonitor()
    monitor.monitor_loop()
