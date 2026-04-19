# Tech-Cultivation-
#define BLYNK_TEMPLATE_ID "TMPLxxxxxx"
#define BLYNK_TEMPLATE_NAME "Greenhouse"
#define BLYNK_AUTH_TOKEN "FgmJ98L_QKh4T5h5igrX0Z4NOfna1_qP"

#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

// WiFi
char ssid[] = "teh";
char pass[] = "1921519215";

// LCD
LiquidCrystal_I2C lcd(0x27, 16, 4);

// Blynk
BlynkTimer timer;

// Sensors
#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

#define SOIL_PIN 34
#define LDR_PIN 35

// Relays
#define FAN_RELAY 26
#define PUMP_RELAY 27
#define LIGHT_RELAY 25

// Button
#define BTN_SELECT 13

// Plants
int plantType = 0;
String plantName[] = {"Kale", "Rosemary", "Ginseng"};

float tempMin[] = {14, 18, 15};
float tempMax[] = {25, 29, 22};

float soilMin[] = {10, 10, 20};
float soilMax[] = {30, 20, 40};

int lightMin[] = {20, 30, 10};
int lightMax[] = {80, 90, 60};

// Variables
float temperature = 0, humidity = 0;
int soilMoisture = 0, lightLevel = 0;

String statusText = "Normal";

int page = 0;
unsigned long lastPageChange = 0;
unsigned long lastWiFiTry = 0;

// ========== SETUP ==========
void setup() {
  Serial.begin(115200);

  Wire.begin(21, 22);
  lcd.init();
  lcd.backlight();

  lcd.setCursor(0,1);
  lcd.print("GREEN HOUSE      LOADING,,,");
  delay(10000);
  lcd.clear();

  pinMode(BTN_SELECT, INPUT_PULLUP);

  pinMode(FAN_RELAY, OUTPUT);
  pinMode(PUMP_RELAY, OUTPUT);
  pinMode(LIGHT_RELAY, OUTPUT);

  digitalWrite(FAN_RELAY, LOW);
  digitalWrite(PUMP_RELAY, LOW);
  digitalWrite(LIGHT_RELAY, LOW);

  dht.begin();

  WiFi.begin(ssid, pass);
  Blynk.config(BLYNK_AUTH_TOKEN);

  timer.setInterval(2000L, sendToBlynk);
}

// ========== LOOP ==========
void loop() {

  // WiFi reconnect بدون تهنيج
  if (WiFi.status() != WL_CONNECTED) {
    if (millis() - lastWiFiTry > 5000) {
      WiFi.begin(ssid, pass);
      lastWiFiTry = millis();
    }
  } else {
    Blynk.run();
  }

  timer.run();

  readSensors();
  controlSystem();
  readButtons();
  updateLCD();
}

// ========== SENSORS ==========
void readSensors() {
  temperature = dht.readTemperature();
  humidity = dht.readHumidity();

  soilMoisture = map(analogRead(SOIL_PIN), 1300, 4095, 100, 0);
  lightLevel = map(analogRead(LDR_PIN), 0, 4095, 100, 0);

  if (isnan(temperature) || isnan(humidity)) {
    temperature = 0;
    humidity = 0;
  }
}

// ========== CONTROL ==========
void controlSystem() {

  if (temperature < tempMin[plantType]) {
    digitalWrite(FAN_RELAY, HIGH);
    statusText = "Heating";
  } 
  else if (temperature > tempMax[plantType]) {
    digitalWrite(FAN_RELAY, LOW);
    statusText = "Cooling";
  } 
  else {
    digitalWrite(FAN_RELAY, HIGH);
    statusText = "Normal";
  }

  if (soilMoisture < soilMin[plantType]) {
    digitalWrite(PUMP_RELAY, HIGH);
    statusText = "Watering";
  } 
  else {
    digitalWrite(PUMP_RELAY, LOW);
  }

  if (lightLevel < lightMin[plantType]) {
    digitalWrite(LIGHT_RELAY, HIGH);
  } 
  else if (lightLevel > lightMax[plantType]) {
    digitalWrite(LIGHT_RELAY, LOW);
  }
}

// ========== LCD ==========
void updateLCD() {

  
  if (millis() - lastPageChange > 5000) {
    page++;
    if (page > 2) page = 0;
    lastPageChange = millis();
    lcd.clear(); 
  }


  lcd.setCursor(0, 0);
  if (WiFi.status() == WL_CONNECTED)
    lcd.print("WiFi:OK ");
  else
    lcd.print("WiFi:OFF");

  lcd.setCursor(10, 0);
  lcd.print(plantName[plantType] + "   ");

  if (page == 0) {
    lcd.setCursor(0, 1);
    lcd.print("Temp:");
    lcd.print(temperature,1);
    lcd.print("C   ");

    lcd.setCursor(0, 2);
    lcd.print("Soil:");
    lcd.print(soilMoisture);
    lcd.print("%   ");

    lcd.setCursor(0, 3);
    lcd.print("Light:");
    lcd.print(lightLevel);
    lcd.print("%   ");
  }

  else if (page == 1) {
    lcd.setCursor(0, 1);
    lcd.print("T:");
    lcd.print(tempMin[plantType]);
    lcd.print("-");
    lcd.print(tempMax[plantType]);
    lcd.print("   ");

    lcd.setCursor(0, 2);
    lcd.print("S:");
    lcd.print(soilMin[plantType]);
    lcd.print("-");
    lcd.print(soilMax[plantType]);
    lcd.print("   ");

    lcd.setCursor(0, 3);
    lcd.print("L:");
    lcd.print(lightMin[plantType]);
    lcd.print("-");
    lcd.print(lightMax[plantType]);
    lcd.print("   ");
  }

  else {
    lcd.setCursor(0, 1);
    lcd.print("Status:");
    lcd.print(statusText);
    lcd.print("   ");

    lcd.setCursor(0, 2);
    lcd.print("System Running");
  }
}

// ========== BUTTON ==========
void readButtons() {
  static bool lastState = HIGH;
  bool current = digitalRead(BTN_SELECT);

  if (current == LOW && lastState == HIGH) {
    plantType++;
    if (plantType > 2) plantType = 0;
  }

  lastState = current;
}

// ========== BLYNK ==========
void sendToBlynk() {
  if (WiFi.status() == WL_CONNECTED) {
    Blynk.virtualWrite(V0, temperature);
    Blynk.virtualWrite(V1, humidity);
    Blynk.virtualWrite(V2, soilMoisture);
    Blynk.virtualWrite(V3, lightLevel);
  }
}
