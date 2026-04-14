# Gesture-Controlled-Robot
---
## MAC Address
```cpp
#include <WiFi.h>

void setup() {
  Serial.begin(115200);
  WiFi.mode(WIFI_STA);
  Serial.println(WiFi.macAddress());
}

void loop() {}
```
---
## Transmitter Code (ESP32 + MPU6050)
```cpp
#include <Wire.h>
#include <WiFi.h>
#include <esp_now.h>
#include <MPU6050.h>

MPU6050 mpu;

// Replace with YOUR Receiver ESP32 MAC Address
uint8_t receiverMac[] = {0xF0, 0xF5, 0xBD, 0x0E, 0x8A, 0x6C};

typedef struct struct_message {
  float x;
  float y;
  float z;
} struct_message;

struct_message dataToSend;

void setup() {
  Serial.begin(115200);

  Wire.begin(21, 22);   // SDA, SCL
  mpu.initialize();

  if (!mpu.testConnection()) {
    Serial.println("MPU6050 connection failed!");
    while (1);
  }

  WiFi.mode(WIFI_STA);

  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW Init Failed");
    return;
  }

  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, receiverMac, 6);
  peerInfo.channel = 0;
  peerInfo.encrypt = false;

  if (!esp_now_is_peer_exist(receiverMac)) {
    esp_now_add_peer(&peerInfo);
  }

  Serial.println("Transmitter Ready");
}

void loop() {
  int16_t ax, ay, az;

  mpu.getAcceleration(&ax, &ay, &az);

  dataToSend.x = ax / 16384.0;
  dataToSend.y = ay / 16384.0;
  dataToSend.z = az / 16384.0;

  esp_now_send(receiverMac, (uint8_t*)&dataToSend, sizeof(dataToSend));

  Serial.print("X: ");
  Serial.print(dataToSend.x);
  Serial.print(" | Y: ");
  Serial.print(dataToSend.y);
  Serial.print(" | Z: ");
  Serial.println(dataToSend.z);

  delay(100);
}
```
---
## Receiver Code (ESP32 + L298N)
```cpp
#include <WiFi.h>
#include <esp_now.h>

// Motor Driver Pins
#define IN1 4
#define IN2 5
#define IN3 6
#define IN4 7

typedef struct struct_message {
  float x;
  float y;
  float z;
} struct_message;

struct_message incomingData;

void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void turnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void OnDataRecv(const uint8_t * mac, const uint8_t *incoming, int len) {
  memcpy(&incomingData, incoming, sizeof(incomingData));

  Serial.print("X: ");
  Serial.print(incomingData.x);
  Serial.print(" Y: ");
  Serial.print(incomingData.y);
  Serial.print(" Z: ");
  Serial.println(incomingData.z);

  // Gesture Logic
  if (incomingData.x > 0.5) {
    moveForward();
  }
  else if (incomingData.x < -0.5) {
    moveBackward();
  }
  else if (incomingData.y > 0.5) {
    turnRight();
  }
  else if (incomingData.y < -0.5) {
    turnLeft();
  }
  else {
    stopMotors();
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  stopMotors();

  WiFi.mode(WIFI_STA);

  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW Init Failed");
    return;
  }

  esp_now_register_recv_cb(OnDataRecv);

  Serial.println("Receiver Ready");
}

void loop() {
}
```
