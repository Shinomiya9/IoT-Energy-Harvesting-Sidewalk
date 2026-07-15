#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH110X.h>
#include <FluxGarage_RoboEyes.h>

// ============ DISPLAY CONFIGURATION ============
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1

#define LCD_ADDRESS 0x27
#define LCD_COLS 16
#define LCD_ROWS 2

// ============ ESP32 PIN CONFIGURATION ============
#define I2C_SDA 21
#define I2C_SCL 22
#define BUTTON_PIN 4

// ============ ADC SHUNT PINS ============
#define PIN_HIGH 35   // High side of shunt (piezo/rectifier side)
#define PIN_LOW  32   // Low side of shunt (load/GND side)

// ============ SHUNT RESISTOR ============
const float SHUNT_RESISTOR = 10.0;  // CHANGE to your actual resistor value (ohms)

// ============ CURRENT THRESHOLD ============
const float CURRENT_THRESHOLD_mA = 3.0 ;  // below this = "Waiting for input"

// ============ CREATE OBJECTS ============
LiquidCrystal_I2C lcd(LCD_ADDRESS, LCD_COLS, LCD_ROWS);
Adafruit_SH1106G display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
RoboEyes<Adafruit_SH1106G> *roboEyes = nullptr;

// ============ PIEZO SENSOR VARIABLES ============
float piezoCurrent_mA = 0;
float offsetVoltage = 0;     // auto-zero baseline
unsigned long lastActivityTime = 0;
unsigned long lastRezeroTime = 0;
const unsigned long rezeroIdleTime = 15000;   // if idle 15s, allow re-zero
const unsigned long rezeroInterval = 60000;   // re-zero at most once a minute

// ============ LCD MANAGEMENT ============
unsigned long lastLcdUpdate = 0;
const unsigned long lcdUpdateInterval = 500;

// ============ OLED BILLBOARD (button-controlled only) ============
unsigned long lastButtonPress = 0;
int debounceTime = 300;
unsigned long firstPressTime = 0;
int pressCount = 0;
const unsigned long multiPressWindow = 2000;

unsigned long moodChangedTime = 0;
bool moodChanged = false;
String currentMood = "DEFAULT";

unsigned long textDisplayTime = 0;
bool showingText = false;
String currentText = "";

// ============ OLED TEXT OVERLAY ============

void showTextWithEyes(String text) {
  currentText = text;
  showingText = true;
  textDisplayTime = millis();
}

void drawTextOverlay() {
  if (!showingText || roboEyes == nullptr) return;

  display.setTextSize(1);
  display.setTextColor(SH110X_WHITE);

  int16_t x1, y1;
  uint16_t w, h;
  display.getTextBounds(currentText, 0, 0, &x1, &y1, &w, &h);
  int x = (SCREEN_WIDTH - w) / 2;
  int y = SCREEN_HEIGHT - h - 2;

  display.fillRect(x - 2, y - 2, w + 4, h + 4, SH110X_BLACK);
  display.drawRect(x - 2, y - 2, w + 4, h + 4, SH110X_WHITE);
  display.setCursor(x, y);
  display.print(currentText);

  if (millis() - textDisplayTime > 2000) {
    showingText = false;
  }
}

void setMoodAndAnimate(String mood, String textMessage) {
  if (roboEyes == nullptr) return;

  if (mood == "ANGRY") {
    roboEyes->setMood(ANGRY);
    roboEyes->blink();
    roboEyes->open();
    currentMood = "ANGRY";
  }
  else if (mood == "HAPPY") {
    roboEyes->setMood(HAPPY);
    roboEyes->anim_laugh();
    currentMood = "HAPPY";
  }
  else if (mood == "TIRED") {
    roboEyes->setMood(TIRED);
    roboEyes->anim_confused();
    currentMood = "TIRED";
  }
  else if (mood == "DEFAULT") {
    roboEyes->setMood(DEFAULT);
    currentMood = "DEFAULT";
  }

  if (textMessage != "") {
    showTextWithEyes(textMessage);
  }
}

// ============ PIEZO MEASUREMENT (auto-zero ADC shunt method) ============

void calibrateOffset() {
  long sumHigh = 0, sumLow = 0;
  for (int i = 0; i < 100; i++) {
    sumHigh += analogRead(PIN_HIGH);
    sumLow += analogRead(PIN_LOW);
    delay(2);
  }
  float avgHigh = (sumHigh / 100.0) * (3.3 / 4095.0);
  float avgLow  = (sumLow / 100.0) * (3.3 / 4095.0);
  offsetVoltage = avgHigh - avgLow;
  lastRezeroTime = millis();

  Serial.print("Auto-Zero Offset Voltage: ");
  Serial.print(offsetVoltage, 6);
  Serial.println(" V");
}

void readPiezoADC() {
  long sumHigh = 0, sumLow = 0;
  for (int i = 0; i < 10; i++) {
    sumHigh += analogRead(PIN_HIGH);
    sumLow += analogRead(PIN_LOW);
    delay(2);
  }
  float rawHigh = sumHigh / 10.0;
  float rawLow  = sumLow / 10.0;

  float voltageHigh = rawHigh * (3.3 / 4095.0);
  float voltageLow  = rawLow  * (3.3 / 4095.0);

  float shuntVoltage = (voltageHigh - voltageLow) - offsetVoltage;
  float current_A = shuntVoltage / SHUNT_RESISTOR;

  piezoCurrent_mA = fabs(current_A * 1000.0);

  if (piezoCurrent_mA >= CURRENT_THRESHOLD_mA) {
    lastActivityTime = millis();
  } else {
    piezoCurrent_mA = 0;
  }

  // Periodic re-zero while idle, to correct for slow drift
  if (piezoCurrent_mA == 0 &&
      millis() - lastActivityTime > rezeroIdleTime &&
      millis() - lastRezeroTime > rezeroInterval) {
    calibrateOffset();
  }
}

// ============ UPDATE LCD (current value only) ============

void updateLCD() {
  lcd.clear();

  if (piezoCurrent_mA < CURRENT_THRESHOLD_mA) {
    lcd.setCursor(0, 0);
    lcd.print("Waiting for");
    lcd.setCursor(0, 1);
    lcd.print("input...");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Current:");
    lcd.setCursor(0, 1);
    if (piezoCurrent_mA >= 1000) {
      lcd.print(piezoCurrent_mA / 1000, 3);
      lcd.print(" A");
    } else {
      lcd.print(piezoCurrent_mA, 2);
      lcd.print(" mA");
    }
  }
}

// ============ I2C SCANNER ============

void scanI2C() {
  Serial.println("Scanning I2C devices...");
  for (uint8_t addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
      Serial.print("Found device at 0x");
      Serial.println(addr, HEX);
    }
  }
}

// ============ SETUP ============

void setup() {
  delay(2000);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  Serial.begin(115200);
  Serial.println("\n\n=== PIEZO CURRENT MONITOR ===");

  Wire.begin(I2C_SDA, I2C_SCL);
  Wire.setClock(100000);
  scanI2C();

  // LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Calibrating...");

  // Auto-zero calibration — keep piezo untouched during this
  calibrateOffset();

  lcd.clear();

  // OLED + RoboEyes (billboard, button-controlled only)
  if (!display.begin(0x3C)) {
    Serial.println("OLED not found!");
  } else {
    Serial.println("OLED found!");
    roboEyes = new RoboEyes<Adafruit_SH1106G>(display);
    roboEyes->begin(SCREEN_WIDTH, SCREEN_HEIGHT, 100);
    roboEyes->setAutoblinker(ON, 8, 2);
    roboEyes->setIdleMode(ON, 2, 1);
    roboEyes->setMood(DEFAULT);
    roboEyes->open();
  }

  Serial.println("=== SETUP COMPLETE ===");
}

// ============ LOOP ============

void loop() {
  unsigned long currentMillis = millis();

  // ===== READ PIEZO CURRENT =====
  readPiezoADC();

  // ===== UPDATE LCD =====
  if (currentMillis - lastLcdUpdate > lcdUpdateInterval) {
    updateLCD();
    lastLcdUpdate = currentMillis;

    Serial.print("Current: ");
    Serial.print(piezoCurrent_mA, 2);
    Serial.println(" mA");
  }

  // ===== BUTTON PRESS -> OLED BILLBOARD (only trigger for OLED) =====
  bool buttonPressed = (digitalRead(BUTTON_PIN) == LOW);
  if (buttonPressed && (currentMillis - lastButtonPress > debounceTime)) {
    lastButtonPress = currentMillis;

    if (firstPressTime == 0 || (currentMillis - firstPressTime > multiPressWindow)) {
      firstPressTime = currentMillis;
      pressCount = 1;
    } else {
      pressCount++;
    }

    if (pressCount == 1) {
      String messages[] = {"Do that again?", "Not cool!", "Eco angry!"};
      setMoodAndAnimate("ANGRY", messages[random(0, 3)]);
      moodChangedTime = currentMillis;
      moodChanged = true;
    }
    else if (pressCount == 2) {
      String messages[] = {"Yay! Double love!", "Best day ever!", "Eco loves you!"};
      setMoodAndAnimate("HAPPY", messages[random(0, 3)]);
      moodChangedTime = currentMillis;
      moodChanged = true;
    }
    else if (pressCount >= 3) {
      String messages[] = {"Huaaaaa...", "So tired...", "Need a nap!"};
      setMoodAndAnimate("TIRED", messages[random(0, 3)]);
      pressCount = 0;
      firstPressTime = 0;
      moodChangedTime = currentMillis;
      moodChanged = true;
    }
  }

  // ===== RESET MOOD BACK TO DEFAULT (a few seconds after a button press) =====
  if (moodChanged && currentMillis - moodChangedTime > 3000) {
    setMoodAndAnimate("DEFAULT", "");
    moodChanged = false;
  }

  // ===== DRAW OLED =====
  if (roboEyes != nullptr) {
    roboEyes->update();
    drawTextOverlay();
    display.display();
  }

  // Reset multi-press counter
  if (firstPressTime > 0 && currentMillis - firstPressTime > multiPressWindow) {
    pressCount = 0;
    firstPressTime = 0;
  }

  delay(30);
}

/*
=============== NOTES ===============

1. LCD now shows ONLY current. Below CURRENT_THRESHOLD_mA (0.4mA) it shows
   "Waiting for input" instead of a number.

2. OLED is now purely a button-controlled billboard:
   - No automatic reaction to piezo activity (no wattage bubble, no
     energy-triggered HAPPY mood).
   - Only the button on pin 4 changes mood/text: 1 press = ANGRY,
     2 presses (within 2s) = HAPPY, 3+ presses = TIRED.
   - Mood/text resets to DEFAULT (idle blinking eyes, no text) 3 seconds
     after the last press.

3. CURRENT_THRESHOLD_mA is set to 0.4 per your request. If you still see
   noise crossing that line with the piezo disconnected, raise it slightly
   — check Serial output with the piezo unplugged to confirm your real
   noise floor before finalizing this number for the report.

4. Auto-zero re-calibrates automatically after 15s of no activity, but no
   more than once a minute, to correct for slow drift without needing a
   reboot.
=======================================
*/
