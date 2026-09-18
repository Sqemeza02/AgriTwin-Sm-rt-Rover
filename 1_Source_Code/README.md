#define BLYNK_TEMPLATE_ID "TMPL2u1GnPFi5"
#define BLYNK_TEMPLATE_NAME "agriTwin Smart rover"
#define BLYNK_AUTH_TOKEN "n2bslE5w45hQkLExSEit6ji6yDq1w5k1"
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// ==== WiFi & Blynk ====
char ssid[] = "Samsung Galaxy A03";
char pass[] = "@Spondo12!";
BlynkTimer timer;

// ==== Pin Definitions ====
#define DHTPIN 2       // D4 on NodeMCU
#define DHTTYPE DHT11
#define MOISTURE_PIN A0
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define OLED_ADDR 0x3C
#define OLED_SDA 4     // D2 on NodeMCU
#define OLED_SCL 5     // D1 on NodeMCU

// ==== Objects ====
DHT dht(DHTPIN, DHTTYPE);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

float temperature = 0;
float humidity = 0;
int moistureRaw = 0;
int moisturePercent = 0;

unsigned long previousMillis = 0;
const long interval = 2000; // Update every 2 seconds

// BLYNK SEND FUNCTION - runs every 2s
void sendToBlynk() {
  Blynk.virtualWrite(V0, temperature);      // V0 = Temperature
  Blynk.virtualWrite(V1, humidity);         // V1 = Humidity  
  Blynk.virtualWrite(V2, moisturePercent);  // V2 = Soil %
  
  // Optional: Send alert if soil is dry
  if(moisturePercent < 30) {
    Blynk.logEvent("soil_dry", "Soil is too dry! Please water plants.");
  }
}

void setup() {
  Serial.begin(115200);
  
  // 1. I2C for OLED
  Wire.begin(OLED_SDA, OLED_SCL); 
  Wire.setClock(100000);
  
  // 2. Init OLED
  if(!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("SSD1306 allocation failed");
    for(;;);
  }
  display.ssd1306_command(SSD1306_SETCONTRAST);
  display.ssd1306_command(255);
  
  // 3. Init DHT
  dht.begin();
  
  // 4. Connect Blynk
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);
  timer.setInterval(2000L, sendToBlynk); // Send data every 2 seconds
  
  // Splash screen
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(20,20);
  display.println("Smart Farm");
  display.setCursor(15,35);
  display.println("Connecting Blynk..");
  display.display();
  delay(2000);
}

void loop() {
  Blynk.run();   // Must have
  timer.run();   // Must have
  
  unsigned long currentMillis = millis();
  
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;
    readSensors();
    updateDisplay();
  }
}

void readSensors() {
  // Read DHT11
  humidity = dht.readHumidity();
  temperature = dht.readTemperature(); // Celsius
  
  // Read Moisture
  moistureRaw = analogRead(MOISTURE_PIN);
  moisturePercent = map(moistureRaw, 1023, 300, 0, 100); // Calibrate 300=wet, 1023=dry
  moisturePercent = constrain(moisturePercent, 0, 100);

  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("Failed to read from DHT sensor!");
  }
}

void updateDisplay() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  
  // Header + WiFi status
  display.setCursor(0,0);
  display.print("Smart Farm ");
  if(Blynk.connected()) display.println("[ONLINE]");
  else display.println("[OFFLINE]");
  display.drawLine(0,10,128,10,SSD1306_WHITE);
  
  // Data
  display.setCursor(0,15);
  display.print("Temp: "); display.print(temperature, 1); display.println(" C");
  
  display.setCursor(0,28);
  display.print("Humidity: "); display.print(humidity, 1); display.println(" %");
  
  display.setCursor(0,41);
  display.print("Soil: "); display.print(moisturePercent); display.println(" %");
  
  // Moisture bar
  display.drawRect(0, 55, 100, 8, SSD1306_WHITE);
  display.fillRect(2, 57, moisturePercent, 4, SSD1306_WHITE);
  display.setCursor(105, 53); display.print(moisturePercent); display.print("%");
  
  display.display();
  delay(50);
  
  // Debug to Serial
  Serial.printf("Temp: %.1fC, Hum: %.1f%%, Soil: %d%%\n", temperature, humidity, moisturePercent);
}
<img width="971" height="483" alt="image" src="https://github.com/user-attachments/assets/e8246ee3-16ff-4e52-be2b-500dbd3ef58a" />







