# Kickboard-Safety-System
#include <Wire.h>

// --- 먹스 및 부저 핀 ---
const int S0 = 2, S1 = 3, S2 = 4, S3 = 5;
const int MUX_SIG = A0;
const int BUZZER = 8; 

// --- 모터 제어 핀 (6, 9, 10, 11번 PWM) ---
const int IN1 = 6;   // 좌측 전진 (PWM)
const int IN2 = 9;   // 좌측 후진 (LOW 고정)
const int IN3 = 10;  // 우측 전진 (PWM)
const int IN4 = 11;  // 우측 후진 (LOW 고정)

// --- 튜닝 파라미터 ---
const int FSR_THRESHOLD = 200;     // 압력센서 눌림 기준
const float SHAKE_LIMIT = 5000.0;  // 흔들림 감지 민감도 
const int NORMAL_SPEED = 200;      // 평상시 속도
const int REDUCED_SPEED = 100;     // 흔들림 감지 시 속도

bool isWPressed = false;
unsigned long previousMillis = 0;
bool buzzerState = false;
const long buzzerInterval = 250; 

void setup() {
  Serial.begin(9600);
  Wire.begin(); 
  
  // MPU6050 센서 입력
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
  // 2인 이상 탑승 감지 시스템
  int pressedCount = 0;
  for (int i = 0; i < 8; i++) {
    if (readMux(i) > FSR_THRESHOLD) {
      pressedCount++;
    }
  }

  if (pressedCount >= 4) {
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);

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

  // 파이썬 W키 수신
  if (Serial.available() > 0) {
    char cmd = Serial.read();
    if (cmd == '1') isWPressed = true;
    else if (cmd == '0') isWPressed = false;
  }

  if (!isWPressed) {
    digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
    return;
  }

  // MPU6050 흔들림 감지 시스템
  Wire.beginTransmission(0x68);
  Wire.write(0x43); 
  Wire.endTransmission(false);
  Wire.requestFrom(0x68, 6, true);
  
  int16_t GyX = Wire.read() << 8 | Wire.read();
  int16_t GyY = Wire.read() << 8 | Wire.read();
  int16_t GyZ = Wire.read() << 8 | Wire.read();

  float shakeMagnitude = sqrt((long)GyX*GyX + (long)GyY*GyY + (long)GyZ*GyZ);
  int finalSpeed = NORMAL_SPEED;

  if (shakeMagnitude > SHAKE_LIMIT) {
    finalSpeed = REDUCED_SPEED; 
  }

  // 최종 모터 출력
  analogWrite(IN1, finalSpeed);
  digitalWrite(IN2, LOW);
  analogWrite(IN3, finalSpeed);
  digitalWrite(IN4, LOW);

  delay(10); 
}
