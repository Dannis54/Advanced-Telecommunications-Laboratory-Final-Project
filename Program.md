#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Adafruit_SSD1306.h>
#include <RTClib.h>
#include <Servo.h>

#define BUZZER_PIN 4        
#define ONE_WIRE_BUS 2      
#define WATER_LEVEL_PIN A0  
#define SERVO_PIN 9 
#define OLED_RESET -1 

Servo myServo;
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature tempSensor(&oneWire);
Adafruit_SSD1306 display(128, 64, &Wire, OLED_RESET);
RTC_DS1307 rtc;

// Timers & States
unsigned long lastServoTime = 0;
unsigned long servoActionStart = 0;
const unsigned long servoInterval = 10000; 
enum ServoState { IDLE, MOVING_FORWARD, MOVING_BACKWARD };
ServoState currentServoState = IDLE;

// Thresholds
const float TEMP_HIGH = 35.0;
const float TEMP_LOW = 15.0;
const int WATER_THRESHOLD = 15;

void setup() {
  Wire.begin();
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  pinMode(BUZZER_PIN, OUTPUT);
  tempSensor.begin();
  rtc.begin();
  
  // Initial Servo Reset
  myServo.attach(SERVO_PIN);
  myServo.write(90); 
  delay(200);
  myServo.detach();
  
  display.setTextColor(WHITE);
}

void loop() {
  unsigned long currentTime = millis();

  // 1. SENSOR READINGS
  tempSensor.requestTemperatures();
  float currentTemp = tempSensor.getTempCByIndex(0);
  int waterPercent = constrain(map(analogRead(WATER_LEVEL_PIN), 0, 1023, 0, 100), 0, 100);

  // 2. SERVO STATE MACHINE (Non-Blocking)
  if (currentServoState == IDLE && (currentTime - lastServoTime >= servoInterval)) {
    tone(BUZZER_PIN, 1500, 80); delay(100);
    tone(BUZZER_PIN, 1800, 80);+
    myServo.attach(SERVO_PIN);
    myServo.write(180);
    servoActionStart = currentTime;
    currentServoState = MOVING_FORWARD;
  }
  if (currentServoState == MOVING_FORWARD && (currentTime - servoActionStart >= 1000)) {
    myServo.write(0);
    servoActionStart = currentTime;
    currentServoState = MOVING_BACKWARD;
  }
  if (currentServoState == MOVING_BACKWARD && (currentTime - servoActionStart >= 1000)) {
    myServo.detach();
    lastServoTime = currentTime;
    currentServoState = IDLE;
  }

  // 3. SOUND ALARMS
  if (waterPercent <= WATER_THRESHOLD) tone(BUZZER_PIN, 2500, 100);
  else if (currentTemp >= TEMP_HIGH) tone(BUZZER_PIN, 3000, 50);
  else if (currentTemp <= TEMP_LOW) tone(BUZZER_PIN, 800, 150);
  else noTone(BUZZER_PIN);

  // 4. DRAW OLED
  updateDisplay(currentTemp, waterPercent);
}

void updateDisplay(float t, int w) {
  DateTime now = rtc.now();
  display.clearDisplay();
  
  // --- HEADER: Small Date & Time ---
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print(now.day()); display.print("/"); display.print(now.month());
  display.print(" ");
  if(now.hour() < 10) display.print('0'); display.print(now.hour()); 
  display.print(':'); 
  if(now.minute() < 10) display.print('0'); display.print(now.minute());
  display.print(':');
  if(now.second() < 10) display.print('0'); display.print(now.second());

  display.drawFastHLine(0, 11, 128, WHITE);

  // --- CENTER: Large Data ---
  display.setCursor(0, 18);
  display.print("TEMP: "); display.setTextSize(2); display.print(t, 1); display.print("C");
  
  display.setTextSize(1);
  display.setCursor(0, 38);
  display.print("WATER: "); display.setTextSize(2); display.print(w); display.print("%");

  // --- FOOTER: Dynamic Alert Bar ---
  display.drawFastHLine(0, 54, 128, WHITE);
  display.setTextSize(1);
  display.setCursor(0, 57);
  
  if (w <= WATER_THRESHOLD) {
    display.print("!! LOW WATER ALERT !!");
  } else if (t >= TEMP_HIGH) {
    display.print("ALRT: HIGH TEMO!");
  } else if (t <= TEMP_LOW) {
    display.print("ALRT: LOW TEMP");
  } else if (currentServoState != IDLE) {
    display.print("Feeding time :)");
  } else {
    display.print("SYSTEM STATUS: OK");
  }

  display.display();
}
