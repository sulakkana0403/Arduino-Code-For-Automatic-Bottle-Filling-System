# Arduino-Code-For-Automatic-Bottle-Filling-System
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DIYables_Keypad.h>

#define FLOW_SENSOR_PIN 2

#define MOTOR_EN_PIN 11
#define MOTOR_IN1_PIN 12
#define MOTOR_IN2_PIN 13

const int KEYPAD_COLS = 4;
const int KEYPAD_ROWS = 4;

float flowRate = 0.0;
volatile int flowPulseCount = 0;
float dispensedVolume = 0.0;
int targetVolume = 0;

LiquidCrystal_I2C lcd(0x27, 16, 2);

char keymap[KEYPAD_ROWS][KEYPAD_COLS] = {
  { '1', '2', '3', 'A' },
  { '4', '5', '6', 'B' },
  { '7', '8', '9', 'C' },
  { '*', '0', '#', 'D' }
};

byte rowPins[KEYPAD_ROWS] = { 3, 4, 5, 6 };
byte colPins[KEYPAD_COLS] = { 7, 8, 9, 10 };

DIYables_Keypad keypad = DIYables_Keypad(makeKeymap(keymap), rowPins, colPins, KEYPAD_ROWS, KEYPAD_COLS);

void incrementPulse() {
  flowPulseCount++;
}

void setup() {
  pinMode(FLOW_SENSOR_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), incrementPulse, RISING);

  pinMode(MOTOR_EN_PIN, OUTPUT);
  pinMode(MOTOR_IN1_PIN, OUTPUT);
  pinMode(MOTOR_IN2_PIN, OUTPUT);

  digitalWrite(MOTOR_EN_PIN, LOW);
  digitalWrite(MOTOR_IN1_PIN, LOW);
  digitalWrite(MOTOR_IN2_PIN, LOW);

  lcd.begin();
  lcd.backlight();
  showWelcomeScreen();

  Serial.begin(9600);
}

void loop() {
  targetVolume = readVolumeInput();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Target: ");
  lcd.print(targetVolume);
  lcd.print(" mL");

  delay(2000);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Filling...");

  flowPulseCount = 0;
  dispensedVolume = 0.0;

  digitalWrite(MOTOR_EN_PIN, 255);
  digitalWrite(MOTOR_IN1_PIN, HIGH);
  digitalWrite(MOTOR_IN2_PIN, LOW);

  Serial.println("Pump ON");

  while (dispensedVolume < targetVolume) {
    delay(1000);

    noInterrupts();
    flowRate = (flowPulseCount / 60.0);
    interrupts();

    float currentMilliLiters = (flowRate / 60.0) * 1000;
    dispensedVolume += currentMilliLiters;

    lcd.setCursor(0, 1);
    lcd.print("Flow: ");
    lcd.print(dispensedVolume, 1);
    lcd.print(" mL   ");

    Serial.print("Flow: ");
    Serial.print(dispensedVolume);
    Serial.println(" mL");

    flowPulseCount = 0;
  }

  digitalWrite(MOTOR_EN_PIN, LOW);
  digitalWrite(MOTOR_IN1_PIN, LOW);
  digitalWrite(MOTOR_IN2_PIN, LOW);

  Serial.println("Pump OFF");

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Completed!");
  delay(5000);

  showWelcomeScreen();
}

void showWelcomeScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Enter volume:");
}

int readVolumeInput() {
  String inputBuffer = "";
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Enter volume:");

  while (true) {
    char pressedKey = keypad.getKey();
    if (pressedKey) {
      if (pressedKey == '#') {
        break;
      }
      if (pressedKey == '*') {
        return readVolumeInput();
      }
      if (pressedKey == 'C') {
        inputBuffer = "";
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Enter volume:");
        continue;
      }
      inputBuffer += pressedKey;
      lcd.setCursor(0, 1);
      lcd.print(inputBuffer);
      lcd.print(" mL ");
    }
  }
  return inputBuffer.toInt();
}

void showMainScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Enter Volume");
}
