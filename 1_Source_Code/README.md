#define BLYNK_TEMPLATE_ID "TMPL2-NeRMNUC"
#define BLYNK_TEMPLATE_NAME "AgriTwin Smart Rover"
#define BLYNK_AUTH_TOKEN "qExa8cVaUL14jy44WfIM-qvQ9SoN_HNO"

#include <WiFiS3.h>
#include <BlynkSimpleWifiS3.h>
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- Network Credentials ---
char ssid[] = "Pamela";
char pass[] = "Khayalethu0317";

// --- Hardware Pin Allocations ---
#define DHT_PIN 2         // DHT11 Data Pin
#define DHTTYPE DHT11

#define TRIG_PIN 3        // Ultrasonic Sensor Trigger Pin
#define ECHO_PIN 4        // Ultrasonic Sensor Echo Pin

// L298N Channel A & B: 4WD Motors (Left & Right Sides)
#define ENA 5             // Left Motors Speed (PWM)
#define IN1 6             // Left Motors Direction 1
#define IN2 7             // Left Motors Direction 2

#define IN3 9             // Right Motors Direction 1
#define IN4 10            // Right Motors Direction 2
#define ENB 11            // Right Motors Speed (PWM)

// 5V Relay Pin for Water Pump
#define RELAY_PIN 8       // Relay Signal Pin (IN / SIG)

#define SOIL_PIN A0       // Analog Soil Moisture Sensor Pin
#define GREEN_LED A1      // Green LED (Optimal Soil Indicator)
#define RED_LED 13        // Red LED (Dry Soil Warning Indicator)

// Configuration Thresholds
const int SOIL_DRY_THRESHOLD = 30; // Soil moisture trigger level (%)
const int OBSTACLE_LIMIT_CM = 15;   // Obstacle limit distance (cm)
int driveSpeed = 200;               // Speed controlled via Blynk Virtual Pin

// System Mode Variable (0 = Manual Control, 1 = Auto Patrol)
int autoMode = 0;

// OLED Setup (128x64 I2C)
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// Sensor & Timer Initialization
DHT dht(DHT_PIN, DHTTYPE);
BlynkTimer timer;

void setup() {
  Serial.begin(9600);

  // Pin Setup
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);

  // Relay Control Pin Setup
  pinMode(RELAY_PIN, OUTPUT);

  dht.begin();

  // OLED Setup
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("OLED Allocation Failed"));
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println(F("AgriTwin R4 WiFi..."));
  display.display();

  stopRover();
  stopPump();

  // Connect to Blynk
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  // Setup timed loop for telemetry updates every 1 second
  timer.setInterval(1000L, sendTelemetryData);
}

void loop() {
  Blynk.run();
  timer.run();

  // If set to Autonomous Patrol Mode, execute auto navigation and irrigation
  if (autoMode == 1) {
    runAutonomousLogic();
  }
}

// --- Blynk Input Handler Virtual Pins ---

BLYNK_WRITE(V0) { // Joystick/Button: Forward
  if (param.asInt() && autoMode == 0) moveForward(driveSpeed);
  else if (autoMode == 0) stopRover();
}

BLYNK_WRITE(V1) { // Joystick/Button: Backward
  if (param.asInt() && autoMode == 0) moveReverse(driveSpeed);
  else if (autoMode == 0) stopRover();
}

BLYNK_WRITE(V2) { // Button: Turn Right
  if (param.asInt() && autoMode == 0) turnRightAngle(90, driveSpeed);
  else if (autoMode == 0) stopRover();
}

BLYNK_WRITE(V3) { // Switch: Water Pump Relay Toggle
  if (param.asInt() && autoMode == 0) startPump();
  else if (autoMode == 0) stopPump();
}

BLYNK_WRITE(V4) { // Switch: Mode Toggle (0 = Manual, 1 = Auto Patrol)
  autoMode = param.asInt();
  if (autoMode == 0) {
    stopRover();
    stopPump();
  }
}

BLYNK_WRITE(V5) { // Slider: Speed Adjustment (0-255)
  driveSpeed = param.asInt();
}

// --- Telemetry Dispatch to Blynk & OLED ---

void sendTelemetryData() {
  int rawSoil = analogRead(SOIL_PIN);
  int soilMoisture = map(rawSoil, 0, 876, 0, 100);
  soilMoisture = constrain(soilMoisture, 0, 100);

  float tempC = dht.readTemperature();
  float humidity = dht.readHumidity();
  long distanceCm = readDistance();

  // Push updates to Blynk App Dashboards
  Blynk.virtualWrite(V10, soilMoisture);
  Blynk.virtualWrite(V11, tempC);
  Blynk.virtualWrite(V12, humidity);
  Blynk.virtualWrite(V13, distanceCm);

  // Update Status LEDs
  if (soilMoisture < SOIL_DRY_THRESHOLD) {
    digitalWrite(RED_LED, HIGH);
    digitalWrite(GREEN_LED, LOW);
  } else {
    digitalWrite(GREEN_LED, HIGH);
    digitalWrite(RED_LED, LOW);
  }

  // Refresh OLED Display
  display.clearDisplay();
  display.setCursor(0, 0);
  display.println(F("=== AGRITWIN R4 WIFi ==="));
  display.println();
  display.print(F("Soil: ")); display.print(soilMoisture); display.println(F("%"));
  display.print(F("Temp: ")); display.print(tempC, 1); display.println(F("C"));
  display.print(F("Dist: ")); display.print(distanceCm); display.println(F("cm"));
  display.print(F("Mode: ")); display.println(autoMode == 1 ? F("AUTO") : F("MANUAL"));
  display.display();
}

// --- Autonomous Mode Logic ---

void runAutonomousLogic() {
  int rawSoil = analogRead(SOIL_PIN);
  int soilMoisture = map(rawSoil, 0, 876, 0, 100);
  soilMoisture = constrain(soilMoisture, 0, 100);
  long distanceCm = readDistance();

  if (soilMoisture < SOIL_DRY_THRESHOLD) {
    stopRover();
    startPump();
  } else {
    stopPump();
    if (distanceCm > 0 && distanceCm < OBSTACLE_LIMIT_CM) {
      stopRover();
      delay(300);
      moveReverse(driveSpeed);
      delay(500);
      turnRightAngle(90, driveSpeed);
    } else {
      moveForward(driveSpeed);
    }
  }
}

// --- Relay Control Functions ---

void startPump() {
  // Turn Relay ON (Active HIGH modules)
  // Note: If using an Active LOW relay, change HIGH to LOW
  digitalWrite(RELAY_PIN, HIGH);
}

void stopPump() {
  // Turn Relay OFF
  // Note: If using an Active LOW relay, change LOW to HIGH
  digitalWrite(RELAY_PIN, LOW);
}

// --- Motor & Sensor Helper Functions ---

long readDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duration = pulseIn(ECHO_PIN, HIGH, 25000);
  if (duration == 0) return 999;
  return (duration * 0.034 / 2);
}

void moveForward(int speed) {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, speed);
  analogWrite(ENB, speed);
}

void moveReverse(int speed) {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  analogWrite(ENA, speed);
  analogWrite(ENB, speed);
}

void turnRightAngle(int angle, int speed) {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
  analogWrite(ENA, speed);
  analogWrite(ENB, speed);
  delay(angle == 45 ? 350 : 700);
}

void stopRover() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
}
<img width="971" height="483" alt="image" src="https://github.com/user-attachments/assets/e8246ee3-16ff-4e52-be2b-500dbd3ef58a" />







