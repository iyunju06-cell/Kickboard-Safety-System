# Kickboard-Safety-System

// --- Python code ---


import sys
import time
import serial
from pynput import keyboard

ARDUINO_PORT = "COM4"
BAUD_RATE = 9600

    ser = serial.Serial(ARDUINO_PORT, BAUD_RATE, timeout=0.1)
    time.sleep(2)  

w_pressed = False


def on_press(key):
    global w_pressed
    try:
        if key.char in ["w", "W"] or key == keyboard.Key.up:
            if not w_pressed:
                ser.write(b"1")
                w_pressed = True
    except AttributeError:
        if key == keyboard.Key.up and not w_pressed:
            ser.write(b"1")
            w_pressed = True


def on_release(key):
    global w_pressed
    try:
        if key.char in ["w", "W"] or key == keyboard.Key.up:
            ser.write(b"0")
            w_pressed = False
    except AttributeError:
        if key == keyboard.Key.up:
            ser.write(b"0")
            w_pressed = False


with keyboard.Listener(on_press=on_press, on_release=on_release) as listener:
    listener.join()



// --- Arduino code ---


#include <Wire.h>

// --- Multiplexer and Buzzer Pins ---
const int S0 = 2, S1 = 3, S2 = 4, S3 = 5;
const int MUX_SIG = A0;
const int BUZZER = 8; 

// --- Motor Control Pins (6, 9, 10, 11 are PWM pins) ---
const int IN1 = 6;   // Left Motor Forward (PWM)
const int IN2 = 9;   // Left Motor Backward (Fixed LOW)
const int IN3 = 10;  // Right Motor Forward (PWM)
const int IN4 = 11;  // Right Motor Backward (Fixed LOW)

// --- Tuning Parameters ---
const int FSR_THRESHOLD = 500;     // Threshold for FSR pressure sensor activation
const float SHAKE_LIMIT = 5000.0;  // Sensitivity threshold for shake detection 
const int NORMAL_SPEED = 200;      // Cruising speed under normal conditions
const int REDUCED_SPEED = 100;     // Reduced speed when shaking is detected

bool isWPressed = false;
unsigned long previousMillis = 0;
bool buzzerState = false;
const long buzzerInterval = 250; 

void setup() {
  Serial.begin(9600);
  Wire.begin(); 
  
  // Initialize MPU6050 sensor
  Wire.beginTransmission(0x68);
  Wire.write(0x6B); 
  Wire.write(0);    
  Wire.endTransmission(true);

  pinMode(S0, OUTPUT); pinMode(S1, OUTPUT);
  pinMode(S2, OUTPUT); pinMode(S3, OUTPUT);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(BUZZER, OUTPUT);
}

int readMux(int channel) {
  digitalWrite(S0, bitRead(channel, 0));
  digitalWrite(S1, bitRead(channel, 1));
  digitalWrite(S2, bitRead(channel, 2));
  digitalWrite(S3, bitRead(channel, 3));
  delayMicroseconds(10); 
  return analogRead(MUX_SIG);
}

void loop() {
  // ---- 1. Tandem Riding (Multi-passenger) Detection System ----
  int pressedCount = 0;
  for (int i = 0; i < 8; i++) {
    if (readMux(i) > FSR_THRESHOLD) {
      pressedCount++;
    }
  }

  if (pressedCount >= 4) {
    // Stop all motors immediately for safety
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);

    // Non-blocking buzzer alert toggle (blinking sound)
    unsigned long currentMillis = millis();
    if (currentMillis - previousMillis >= buzzerInterval) {
      previousMillis = currentMillis;
      buzzerState = !buzzerState;
      if (buzzerState) tone(BUZZER, 1000); 
      else noTone(BUZZER);
    }
    return; 
  } else {
    noTone(BUZZER);
    buzzerState = false;
  }

  // ---- 2. Python Command ('W' Key) Reception via Serial ----
  if (Serial.available() > 0) {
    char cmd = Serial.read();
    if (cmd == '1') isWPressed = true;
    else if (cmd == '0') isWPressed = false;
  }

  if (!isWPressed) {
    // Stop motors if the 'W' key is not pressed
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
    return;
  }

  // ---- 3. MPU6050 Dangerous Ride (Shake) Detection System ----
  Wire.beginTransmission(0x68);
  Wire.write(0x43); 
  Wire.endTransmission(false);
  Wire.requestFrom(0x68, 6, true);
  
  int16_t GyX = Wire.read() << 8 | Wire.read();
  int16_t GyY = Wire.read() << 8 | Wire.read();
  int16_t GyZ = Wire.read() << 8 | Wire.read();

  // Calculate total shake magnitude using Euclidean norm
  float shakeMagnitude = sqrt((long)GyX*GyX + (long)GyY*GyY + (long)GyZ*GyZ);
  int finalSpeed = NORMAL_SPEED;

  if (shakeMagnitude > SHAKE_LIMIT) {
    finalSpeed = REDUCED_SPEED; // Trigger speed reduction on high shake levels
  }

  // ---- 4. Final Motor Output Control ----
  analogWrite(IN1, finalSpeed);
  digitalWrite(IN2, LOW);
  analogWrite(IN3, finalSpeed);
  digitalWrite(IN4, LOW);
}
