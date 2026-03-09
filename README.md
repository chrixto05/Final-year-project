//ichrixto@gmail.com
//Projectiot@123


// ---------------- WiFi & Blynk ----------------
#include <LiquidCrystal.h>
#define BLYNK_TEMPLATE_ID "TMPL3-e1G9KP_"
#define BLYNK_TEMPLATE_NAME "PetFeeder"
#define BLYNK_AUTH_TOKEN "ot_V-V6OXjZkm-RZuveS95qJdTcvv8C5"
#include <Wire.h>
#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <ESP32Servo.h>
#include <time.h>

// ---------------- LCD ----------------
LiquidCrystal lcd(2, 15, 19, 18, 5, 4);

// ---------------- WiFi Credentials ----------------
const char *ssid = "projectiot";
const char *password = "123456789abcd";

// ---------------- Timezone (IST) ----------------
const long gmtOffset_sec = 19800;
const int daylightOffset_sec = 0;

// ---------------- Servo ----------------
Servo servo1;
#define window1Pin 14

// ---------------- Ultrasonic ----------------
#define trigPin 12
#define echoPin 13

long getDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH, 20000); 
  return duration * 0.034 / 2; // cm
}

// ---------------- Sound Sensor ----------------
#define soundPin 27   // digital sound sensor pin

// ---------------- Reminder Variables ----------------
int startH1=-1, startM1=-1, endH1=-1, endM1=-1; 
bool active1=false;
#define VPIN_1    V2  // Food Taken/Not Taken
#define VPIN_2    V3  // Dog Barking/Silent

BLYNK_CONNECTED()
{ 

  Blynk.syncVirtual(V2);  
  Blynk.syncVirtual(V3);  
}

void setup() {
  Serial.begin(115200);

  // WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");

  // Blynk
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, password);

  // Time (NTP)
  configTime(gmtOffset_sec, daylightOffset_sec, "pool.ntp.org");
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    Serial.println("Waiting for NTP time...");
    delay(500);
  }

  // Servo
  servo1.attach(window1Pin);
  servo1.write(0);

  // LCD
  lcd.begin(16, 2);
  lcd.setCursor(0, 0);
  lcd.print("Smart Pet Feeder");
  lcd.setCursor(0, 1);
  lcd.print(" Reminder System ");
  delay(3000);
  lcd.clear();

  // Ultrasonic
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Sound sensor
  pinMode(soundPin, INPUT);

  Serial.println("Type 1/2 in Serial Monitor to control window.");
}

// ---------------- BLYNK TIME INPUTS ----------------
BLYNK_WRITE(V0) { 
  TimeInputParam t(param); 
  if (t.hasStartTime()) { startH1=t.getStartHour(); startM1=t.getStartMinute(); } 
}
BLYNK_WRITE(V1) { 
  TimeInputParam t(param); 
  if (t.hasStartTime()) { endH1=t.getStartHour(); endM1=t.getStartMinute(); } 
}

// ---------------- Helper Function ----------------
bool inTimeWindow(int rawHour, int minute, int sh, int sm, int eh, int em) {
  if (sh == -1 || eh == -1) return false;
  int now = rawHour*60 + minute;
  int start = sh*60 + sm;
  int end   = eh*60 + em;
  return (now >= start && now < end);
}

// ---------------- MAIN LOOP ----------------
void loop() {
  Blynk.run();

  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    Serial.println("Failed to get time");
    return;
  }

  int rawHour = timeinfo.tm_hour;
  int currentMinute = timeinfo.tm_min;
  int currentSecond = timeinfo.tm_sec;
  int currentDay = timeinfo.tm_mday;
  int currentMonth = timeinfo.tm_mon + 1;
  int currentYear = (timeinfo.tm_year + 1900) % 100;

  // ---------- Convert to 12-hour format ----------
  int currentHour = rawHour;
  String ampm = "AM";
  if (currentHour == 0) currentHour = 12;
  else if (currentHour == 12) ampm = "PM";
  else if (currentHour > 12) { currentHour -= 12; ampm = "PM"; }
  // ----------------------------------------------

  // ---------- Window 1 Logic ----------
  if (inTimeWindow(rawHour, currentMinute, startH1, startM1, endH1, endM1)) {
    if (!active1) { active1 = true; servo1.write(90); Serial.println("W1 Opened!"); }
  } else if (active1) { active1 = false; servo1.write(0); Serial.println("W1 Closed!"); }
  // ----------------------------------

  // ---------- Serial Control ----------
  if (Serial.available()) {
    char cmd = Serial.read();
    switch (cmd) {
      case '1': servo1.write(90); Serial.println("Servo1 Opened (CMD 1)"); break;
      case '2': servo1.write(0);  Serial.println("Servo1 Closed (CMD 2)"); break;
    }
  }
  // -----------------------------------

  // ---------- Ultrasonic Check ----------
  long dist = getDistance();
  bool foodPresent = (dist < 10);  // <10cm = food present
  // -------------------------------------

  // ---------- Sound Sensor Check ----------
  int soundState = digitalRead(soundPin);
  bool dogBarking = (soundState == HIGH);  // HIGH means sound detected
  // ---------------------------------------

  // ---------- LCD Display ----------
  lcd.setCursor(0, 0);
  lcd.printf("%02d/%02d/%02d", currentDay, currentMonth, currentYear);

  lcd.setCursor(9, 0);
  lcd.printf("F:%s", active1 ? "Open " : "Close");

  lcd.setCursor(0, 1);
  lcd.printf("%02d:%02d:%02d%s", currentHour, currentMinute, currentSecond, ampm.c_str());

  lcd.setCursor(11, 1);
  if (foodPresent) {
    lcd.print("FT");   // Food Not Taken
   // Blynk.virtualWrite(VPIN_1, HIGH); 
  } else {
    lcd.print("FN");   // Food Taken
    //Blynk.virtualWrite(VPIN_1, LOW); 
  }

  lcd.setCursor(14, 1);
  if (dogBarking) {
    lcd.print("DB");   // Dog Barking
    //Blynk.virtualWrite(VPIN_2, HIGH); 
  } else {
    lcd.print("DS");   // Dog Silent
    //Blynk.virtualWrite(VPIN_2, LOW);
  }
  // ---------------------------------

  delay(200);
}
