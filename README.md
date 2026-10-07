# esp32-laser-light

# My Arduino Project

#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#include <BleCompositeHID.h>
#include <KeyboardDevice.h>
#include <MouseDevice.h>

// =================================================
// OLED
// =================================================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define OLED_ADDRESS 0x3C

Adafruit_SSD1306 display(
  SCREEN_WIDTH,
  SCREEN_HEIGHT,
  &Wire,
  OLED_RESET
);

// =================================================
// MPU6050
// =================================================
Adafruit_MPU6050 mpu;

#define SDA_PIN 21
#define SCL_PIN 22

// =================================================
// ROTARY ENCODER
// =================================================
#define ENC_CLK 25
#define ENC_DT  26
#define ENC_SW  27

// =================================================
// MOUSE BUTTONS
// =================================================
#define LEFT_BUTTON  32
#define RIGHT_BUTTON 33

// =================================================
// PRESENTATION BUTTONS
// =================================================
#define PLAY_BUTTON     16
#define PREVIOUS_BUTTON 17
#define NEXT_BUTTON     18
#define ESC_BUTTON      19

// =================================================
// MODE BUTTON
// GPIO35 needs external 10K pull-up
// =================================================
#define MODE_BUTTON 35

// =================================================
// BATTERY
// =================================================
#define BATTERY_PIN 34

#define R1 100000.0
#define R2 100000.0

// =================================================
// BLE HID
// =================================================
KeyboardDevice* keyboard;
MouseDevice* mouse;

BleCompositeHID compositeHID(
  "ESP32 Presentation Remote",
  "MGKK",
  100
);

// =================================================
// MODE
// =================================================
enum DeviceMode
{
  MOUSE_MODE,
  PRESENTATION_MODE
};

DeviceMode currentMode = MOUSE_MODE;

// =================================================
// MOUSE SETTINGS
// =================================================
float sensitivityX = 2.5;
float sensitivityY = 2.5;

bool invertX = false;
bool invertY = true;

float deadZone = 0.8;
float smoothFactor = 0.15;

float filteredX = 0;
float filteredY = 0;

// =================================================
// GYRO
// =================================================
float gyroOffsetX = 0;
float gyroOffsetY = 0;

// =================================================
// ENCODER
// =================================================
int lastCLK;

// =================================================
// BUTTON STATES
// =================================================
bool lastLeftState = HIGH;
bool lastRightState = HIGH;
bool lastEncoderState = HIGH;

bool lastPlayState = HIGH;
bool lastPreviousState = HIGH;
bool lastNextState = HIGH;
bool lastEscState = HIGH;
bool lastModeState = HIGH;

// =================================================
// TIMERS
// =================================================
unsigned long lastLeftTime = 0;
unsigned long lastRightTime = 0;
unsigned long lastEncoderTime = 0;

unsigned long lastPlayTime = 0;
unsigned long lastPreviousTime = 0;
unsigned long lastNextTime = 0;
unsigned long lastEscTime = 0;
unsigned long lastModeTime = 0;

const unsigned long debounceTime = 40;

// =================================================
// PLAY / PAUSE STATE
// =================================================
bool presentationPlaying = false;

// =================================================
// OLED
// =================================================
unsigned long lastOLEDUpdate = 0;
const unsigned long oledUpdateInterval = 500;

// =================================================
// BATTERY VOLTAGE
// =================================================
float readBatteryVoltage()
{
  long total = 0;

  const int samples = 20;

  for (int i = 0; i < samples; i++)
  {
    total += analogRead(BATTERY_PIN);
    delayMicroseconds(100);
  }

  float adcValue = total / (float)samples;

  float adcVoltage =
    (adcValue / 4095.0) * 3.3;

  float batteryVoltage =
    adcVoltage * ((R1 + R2) / R2);

  return batteryVoltage;
}

// =================================================
// BATTERY PERCENTAGE
// =================================================
int getBatteryPercentage(float voltage)
{
  int percentage;

  if (voltage >= 4.20)
    percentage = 100;

  else if (voltage >= 4.10)
    percentage = 90 + (voltage - 4.10) * 100;

  else if (voltage >= 4.00)
    percentage = 80 + (voltage - 4.00) * 100;

  else if (voltage >= 3.90)
    percentage = 65 + (voltage - 3.90) * 150;

  else if (voltage >= 3.80)
    percentage = 45 + (voltage - 3.80) * 200;

  else if (voltage >= 3.70)
    percentage = 30 + (voltage - 3.70) * 150;

  else if (voltage >= 3.60)
    percentage = 15 + (voltage - 3.60) * 150;

  else if (voltage >= 3.50)
    percentage = 5 + (voltage - 3.50) * 100;

  else if (voltage >= 3.30)
    percentage = (voltage - 3.30) * 25;

  else
    percentage = 0;

  return constrain(percentage, 0, 100);
}

// =================================================
// GYRO CALIBRATION
// =================================================
void calibrateGyro()
{
  Serial.println("Keep MPU6050 completely still...");

  display.clearDisplay();

  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);

  display.setCursor(0, 0);
  display.println("ESP32 REMOTE");

  display.setCursor(0, 20);
  display.println("GYRO CALIBRATION");

  display.setCursor(0, 35);
  display.println("Keep sensor still");

  display.display();

  delay(1000);

  float sumX = 0;
  float sumY = 0;

  const int samples = 500;

  sensors_event_t accel;
  sensors_event_t gyro;
  sensors_event_t temp;

  for (int i = 0; i < samples; i++)
  {
    mpu.getEvent(&accel, &gyro, &temp);

    sumX += gyro.gyro.x;
    sumY += gyro.gyro.y;

    delay(5);
  }

  gyroOffsetX = sumX / samples;
  gyroOffsetY = sumY / samples;

  Serial.println("Calibration complete.");
}

// =================================================
// OLED DISPLAY
// =================================================
void updateOLED()
{
  if (millis() - lastOLEDUpdate < oledUpdateInterval)
    return;

  lastOLEDUpdate = millis();

  float batteryVoltage = readBatteryVoltage();

  int batteryPercentage =
    getBatteryPercentage(batteryVoltage);

  display.clearDisplay();

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  display.setCursor(18, 0);
  display.println("ESP32 REMOTE");

  display.setCursor(0, 16);
  display.print("STATUS: ");

  if (compositeHID.isConnected())
    display.println("CONNECTED");
  else
    display.println("DISCONNECTED");

  display.setCursor(0, 32);
  display.print("MODE: ");

  if (currentMode == MOUSE_MODE)
    display.println("MOUSE");
  else
    display.println("PRESENTATION");

  display.setCursor(0, 48);
  display.print("BATTERY: ");
  display.print(batteryPercentage);
  display.println("%");

  display.display();
}

// =================================================
// AIR MOUSE MOVEMENT
// =================================================
void handleMouseMovement()
{
  if (currentMode != MOUSE_MODE)
    return;

  sensors_event_t accel;
  sensors_event_t gyro;
  sensors_event_t temp;

  mpu.getEvent(&accel, &gyro, &temp);

  float gx =
    gyro.gyro.x - gyroOffsetX;

  float gy =
    gyro.gyro.y - gyroOffsetY;

  gx *= 57.2958;
  gy *= 57.2958;

  if (abs(gx) < deadZone)
    gx = 0;

  if (abs(gy) < deadZone)
    gy = 0;

  filteredX =
    filteredX * (1.0 - smoothFactor)
    + gx * smoothFactor;

  filteredY =
    filteredY * (1.0 - smoothFactor)
    + gy * smoothFactor;

  int moveX =
    filteredY * sensitivityX;

  int moveY =
    filteredX * sensitivityY;

  if (invertX)
    moveX = -moveX;

  if (invertY)
    moveY = -moveY;

  moveX = constrain(moveX, -30, 30);
  moveY = constrain(moveY, -30, 30);

  if (moveX != 0 || moveY != 0)
  {
    mouse->mouseMove(moveX, moveY);
  }
}

// =================================================
// MOUSE BUTTONS
// =================================================
void handleMouseButtons()
{
  if (currentMode != MOUSE_MODE)
    return;

  bool leftState =
    digitalRead(LEFT_BUTTON);

  bool rightState =
    digitalRead(RIGHT_BUTTON);

  if (
    leftState == LOW &&
    lastLeftState == HIGH &&
    millis() - lastLeftTime > debounceTime
  )
  {
    mouse->mousePress(1);
    delay(10);
    mouse->mouseRelease(1);

    lastLeftTime = millis();

    Serial.println("LEFT CLICK");
  }

  if (
    rightState == LOW &&
    lastRightState == HIGH &&
    millis() - lastRightTime > debounceTime
  )
  {
    mouse->mousePress(2);
    delay(10);
    mouse->mouseRelease(2);

    lastRightTime = millis();

    Serial.println("RIGHT CLICK");
  }

  lastLeftState = leftState;
  lastRightState = rightState;
}

// =================================================
// ROTARY ENCODER
// FAST MOUSE-LIKE SCROLLING
// =================================================
void handleEncoder()
{
  if (currentMode != MOUSE_MODE)
    return;

  static int lastEncoderCLK = HIGH;
  static unsigned long lastScrollTime = 0;

  int currentCLK =
    digitalRead(ENC_CLK);

  int currentDT =
    digitalRead(ENC_DT);

  if (currentCLK != lastEncoderCLK)
  {
    if (currentCLK == LOW)
    {
      if (millis() - lastScrollTime > 2)
      {
        if (currentDT != currentCLK)
        {
          // SCROLL UP
          mouse->mouseMove(0, 0, 0, 3);

          Serial.println("SCROLL UP");
        }
        else
        {
          // SCROLL DOWN
          mouse->mouseMove(0, 0, 0, -3);

          Serial.println("SCROLL DOWN");
        }

        lastScrollTime = millis();
      }
    }

    lastEncoderCLK = currentCLK;
  }
}

// =================================================
// ENCODER PUSH = MIDDLE CLICK
// =================================================
void handleEncoderButton()
{
  if (currentMode != MOUSE_MODE)
    return;

  bool state =
    digitalRead(ENC_SW);

  if (
    state == LOW &&
    lastEncoderState == HIGH &&
    millis() - lastEncoderTime > debounceTime
  )
  {
    // 3 = middle mouse button
    mouse->mousePress(3);

    delay(10);

    mouse->mouseRelease(3);

    lastEncoderTime = millis();

    Serial.println("MIDDLE CLICK");
  }

  lastEncoderState = state;
}

// =================================================
// SEND KEY
// =================================================
void sendKey(uint8_t key)
{
  keyboard->keyPress(key);

  delay(20);

  keyboard->keyRelease(key);
}

// =================================================
// PRESENTATION BUTTONS
// =================================================
void handlePresentationButtons()
{
  if (currentMode != PRESENTATION_MODE)
    return;

  // -------------------------------------------------
  // PLAY / PAUSE
  // GPIO16
  // -------------------------------------------------

  bool playState =
    digitalRead(PLAY_BUTTON);

  if (
    playState == LOW &&
    lastPlayState == HIGH &&
    millis() - lastPlayTime > debounceTime
  )
  {
    if (!presentationPlaying)
    {
      sendKey(KEY_SPACE);

      presentationPlaying = true;

      Serial.println("PLAY");
    }
    else
    {
      sendKey('p');

      presentationPlaying = false;

      Serial.println("PAUSE");
    }

    lastPlayTime = millis();
  }

  lastPlayState = playState;

  // -------------------------------------------------
  // PREVIOUS SLIDE
  // -------------------------------------------------

  bool previousState =
    digitalRead(PREVIOUS_BUTTON);

  if (
    previousState == LOW &&
    lastPreviousState == HIGH &&
    millis() - lastPreviousTime > debounceTime
  )
  {
    sendKey(KEY_LEFT);

    lastPreviousTime = millis();

    Serial.println("PREVIOUS SLIDE");
  }

  lastPreviousState =
    previousState;

  // -------------------------------------------------
  // NEXT SLIDE
  // -------------------------------------------------

  bool nextState =
    digitalRead(NEXT_BUTTON);

  if (
    nextState == LOW &&
    lastNextState == HIGH &&
    millis() - lastNextTime > debounceTime
  )
  {
    sendKey(KEY_RIGHT);

    lastNextTime = millis();

    Serial.println("NEXT SLIDE");
  }

  lastNextState =
    nextState;

  // -------------------------------------------------
  // ESC
  // -------------------------------------------------

  bool escState =
    digitalRead(ESC_BUTTON);

  if (
    escState == LOW &&
    lastEscState == HIGH &&
    millis() - lastEscTime > debounceTime
  )
  {
    sendKey(KEY_ESC);

    lastEscTime = millis();

    Serial.println("ESC");
  }

  lastEscState =
    escState;
}

// =================================================
// MODE BUTTON
// =================================================
void handleModeButton()
{
  bool modeState =
    digitalRead(MODE_BUTTON);

  if (modeState != lastModeState)
  {
    if (
      millis() - lastModeTime >
      debounceTime
    )
    {
      if (modeState == LOW)
      {
        if (currentMode == MOUSE_MODE)
        {
          currentMode =
            PRESENTATION_MODE;

          filteredX = 0;
          filteredY = 0;

          presentationPlaying = false;

          Serial.println(
            "MODE: PRESENTATION"
          );
        }
        else
        {
          currentMode =
            MOUSE_MODE;

          filteredX = 0;
          filteredY = 0;

          presentationPlaying = false;

          Serial.println(
            "MODE: MOUSE"
          );
        }

        lastModeTime = millis();
      }

      lastModeState = modeState;
    }
  }
}

// =================================================
// SETUP
// =================================================
void setup()
{
  Serial.begin(115200);

  // I2C
  Wire.begin(
    SDA_PIN,
    SCL_PIN
  );

  // OLED
  if (
    !display.begin(
      SSD1306_SWITCHCAPVCC,
      OLED_ADDRESS
    )
  )
  {
    Serial.println("OLED ERROR!");

    while (1)
      delay(1000);
  }

  // Battery
  pinMode(
    BATTERY_PIN,
    INPUT
  );

  analogReadResolution(12);

  // MPU6050
  if (!mpu.begin())
  {
    Serial.println(
      "MPU6050 ERROR!"
    );

    while (1)
      delay(1000);
  }

  mpu.setAccelerometerRange(
    MPU6050_RANGE_2_G
  );

  mpu.setGyroRange(
    MPU6050_RANGE_250_DEG
  );

  mpu.setFilterBandwidth(
    MPU6050_BAND_21_HZ
  );

  // Encoder
  pinMode(
    ENC_CLK,
    INPUT_PULLUP
  );

  pinMode(
    ENC_DT,
    INPUT_PULLUP
  );

  pinMode(
    ENC_SW,
    INPUT_PULLUP
  );

  lastCLK =
    digitalRead(ENC_CLK);

  // Mouse buttons
  pinMode(
    LEFT_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    RIGHT_BUTTON,
    INPUT_PULLUP
  );

  // Presentation buttons
  pinMode(
    PLAY_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    PREVIOUS_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    NEXT_BUTTON,
    INPUT_PULLUP
  );

  pinMode(
    ESC_BUTTON,
    INPUT_PULLUP
  );

  // Mode button
  // External 10K pull-up required
  pinMode(
    MODE_BUTTON,
    INPUT
  );

  // =================================================
  // BLE HID
  // =================================================
  keyboard =
    new KeyboardDevice();

  mouse =
    new MouseDevice();

  compositeHID.addDevice(
    keyboard
  );

  compositeHID.addDevice(
    mouse
  );

  compositeHID.begin();

  // Gyro calibration
  calibrateGyro();

  // =================================================
  // OLED STARTUP
  // =================================================
  display.clearDisplay();

  display.setTextColor(
    SSD1306_WHITE
  );

  display.setTextSize(1);

  display.setCursor(
    18, 0
  );

  display.println(
    "ESP32 REMOTE"
  );

  display.setCursor(
    0, 20
  );

  display.println(
    "MODE: MOUSE"
  );

  display.setCursor(
    0, 38
  );

  display.println(
    "WAITING FOR BLE..."
  );

  display.display();

  Serial.println(
    "ESP32 PRESENTATION REMOTE"
  );

  Serial.println(
    "Waiting for Bluetooth..."
  );
}

// =================================================
// LOOP
// =================================================
void loop()
{
  handleModeButton();

  updateOLED();

  if (compositeHID.isConnected())
  {
    handleMouseMovement();

    handleMouseButtons();

    handleEncoder();

    handleEncoderButton();

    handlePresentationButtons();
  }

  delay(5);
}
