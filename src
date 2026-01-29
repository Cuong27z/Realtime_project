#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>

#include <LiquidCrystal_I2C.h>
#include <Keypad.h>
#include <Adafruit_Fingerprint.h>
#include <HardwareSerial.h>

// ===================== WIFI / MQTT CONFIG =====================
const char* WIFI_SSID     = "ThienMinh";
const char* WIFI_PASSWORD = "20032019";

const char* MQTT_HOST = "0deb728082f94ac89c3920bc4dcb78b9.s1.eu.hivemq.cloud";
const uint16_t MQTT_PORT = 8883; // TLS

const char* MQTT_USERNAME = "cuong27";
const char* MQTT_PASSWORD = "Cuong123";

const char* TOPIC_CMD    = "home/door2/cmd";       // subscribe
const char* TOPIC_STATUS = "home/door2/status";    // publish

WiFiClientSecure secureClient;
PubSubClient mqttClient(secureClient);

// ===================== HARDWARE CONFIG =====================
int lcdColumns = 16;
int lcdRows = 2;
LiquidCrystal_I2C lcd(0x27, lcdColumns, lcdRows);

#define RX_PIN 16  // ESP32 RX <- sensor TX
#define TX_PIN 17  // ESP32 TX -> sensor RX
#define relayPin 23

uint8_t id;
boolean mode = 0;  // giữ nguyên như bạn (luôn 0)

// Keypad 4x4
const byte ROWS = 4;
const byte COLS = 4;

char hexaKeys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};

byte rowPins[ROWS] = {13, 12, 14, 27};
byte colPins[COLS] = {26, 25, 33, 32};

Keypad customKeypad = Keypad(makeKeymap(hexaKeys), rowPins, colPins, ROWS, COLS);
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&Serial2);

// ===================== Non-blocking LCD helper =====================
String lcdMsg = "";
unsigned long lcdMsgUntil = 0;

// ===================== Minimal RMS-like scheduler periods =====================
unsigned long nextFpMs = 0;
unsigned long nextMqttSrvMs = 0;
unsigned long nextNetMs = 0;
unsigned long nextUiMs = 0;

const unsigned long P_FP_MS       = 20;   // high priority
const unsigned long P_MQTT_SRV_MS = 20;   // high priority
const unsigned long P_NET_MS      = 50;   // medium
const unsigned long P_UI_MS       = 200;  // low

// ===================== Command mailbox + timing =====================
volatile bool cmdOpenPending = false;
volatile unsigned long t0_cmd_us = 0;  // T0 nhận MQTT "Open"

unsigned long t0_event_us = 0;         // T0 chung để đo E2E (FP hoặc MQTT)

// Overload handling
unsigned long lastActMs = 0;
const unsigned long ACT_COOLDOWN_MS = 500; // drop lệnh Open nếu quá dày

// ===================== Helper: unlock door pulse =====================
void unlockDoorPulse(uint16_t ms = 100) {
  digitalWrite(relayPin, HIGH);
  delay(ms);
  digitalWrite(relayPin, LOW);
}

// ===================== LCD helper (non-blocking) =====================
void displayText(String text, uint16_t showMs = 800) {
  lcd.clear();
  int length = text.length();
  lcd.setCursor(0, 0);
  lcd.print(text.substring(0, min(16, length)));

  if (length > 16) {
    lcd.setCursor(0, 1);
    lcd.print(text.substring(16, min(32, length)));
  }
  lcdMsg = text;
  lcdMsgUntil = millis() + showMs;
}

// ===================== Read ID from keypad =====================
uint8_t readIDFromKeypad() {
  String input = "";
  char key;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Enter ID:#=OK");
  lcd.setCursor(0, 1);

  while (true) {
    // Keep MQTT alive while waiting for keypad input
    mqttClient.loop();

    key = customKeypad.getKey();
    if (key) {
      if (key == '#') {
        if (input.length() > 0) {
          uint8_t idValue = input.toInt();
          if (idValue >= 1 && idValue <= 127) {
            displayText("ID entered", 500);
            return idValue;
          } else {
            displayText("Invalid ID! 1-127", 900);
            input = "";
            lcd.clear();
            lcd.setCursor(0, 0);
            lcd.print("Enter ID:#=OK");
            lcd.setCursor(0, 1);
          }
        }
      } else if (isdigit((unsigned char)key)) {
        if (input.length() < 3) {
          input += key;
          lcd.print(key);
        }
      } else {
        displayText("Invalid key! #", 800);
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Enter ID:#=OK");
        lcd.setCursor(0, 1);
      }
    }
    delay(5);
  }
}

// ===================== Fingerprint functions =====================
void DeleteChooseFingerprint() {
  displayText("Enter ID delete:", 700);
  uint8_t deleteID = readIDFromKeypad();

  int p = finger.deleteModel(deleteID);
  if (p == FINGERPRINT_OK) {
    displayText("Deleted ID " + String(deleteID), 800);
    mqttClient.publish(TOPIC_STATUS, ("DeletedID:" + String(deleteID)).c_str());
  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    displayText("Comm error del", 900);
  } else if (p == FINGERPRINT_BADLOCATION) {
    displayText("Bad location", 900);
  } else if (p == FINGERPRINT_FLASHERR) {
    displayText("Flash error", 900);
  } else {
    displayText("Unknown error", 900);
  }
}

void DeleteAllFingerprints() {
  displayText("Erasing all...", 900);
  finger.emptyDatabase();
  displayText("All erased", 800);
  mqttClient.publish(TOPIC_STATUS, "AllErased");
}

uint8_t addFingerprint() {
  int p = -1;
  displayText("Enter ID (1-127):", 900);
  id = readIDFromKeypad();

  displayText("Place finger", 800);
  delay(800);

  while (p != FINGERPRINT_OK) {
    mqttClient.loop();
    p = finger.getImage();
    switch (p) {
      case FINGERPRINT_OK: displayText("Image taken", 500); break;
      case FINGERPRINT_NOFINGER: /* no LCD spam */ break;
      case FINGERPRINT_PACKETRECIEVEERR: displayText("Comm error", 700); break;
      case FINGERPRINT_IMAGEFAIL: displayText("Imaging error", 700); break;
      default: displayText("Unknown error", 700); break;
    }
    delay(20);
  }

  p = finger.image2Tz(1);
  switch (p) {
    case FINGERPRINT_OK: displayText("Image converted", 600); break;
    case FINGERPRINT_IMAGEMESS: displayText("Too messy", 700); return p;
    case FINGERPRINT_PACKETRECIEVEERR: displayText("Comm error", 700); return p;
    case FINGERPRINT_FEATUREFAIL:
    case FINGERPRINT_INVALIDIMAGE: displayText("No features", 700); return p;
    default: displayText("Unknown error", 700); return p;
  }

  displayText("Remove finger", 700);
  delay(700);
  displayText("Place again", 700);

  p = 0;
  while (p != FINGERPRINT_NOFINGER) {
    mqttClient.loop();
    p = finger.getImage();
    delay(20);
  }

  p = -1;
  while (p != FINGERPRINT_OK) {
    mqttClient.loop();
    p = finger.getImage();
    switch (p) {
      case FINGERPRINT_OK: displayText("Image taken", 500); break;
      case FINGERPRINT_NOFINGER: /* no LCD spam */ break;
      case FINGERPRINT_PACKETRECIEVEERR: displayText("Comm error", 700); break;
      case FINGERPRINT_IMAGEFAIL: displayText("Imaging error", 700); break;
      default: displayText("Unknown error", 700); break;
    }
    delay(20);
  }

  p = finger.image2Tz(2);
  switch (p) {
    case FINGERPRINT_OK: displayText("Image converted", 600); break;
    case FINGERPRINT_IMAGEMESS: displayText("Too messy", 700); return p;
    case FINGERPRINT_PACKETRECIEVEERR: displayText("Comm error", 700); return p;
    case FINGERPRINT_FEATUREFAIL:
    case FINGERPRINT_INVALIDIMAGE: displayText("No features", 700); return p;
    default: displayText("Unknown error", 700); return p;
  }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Creating model");
  lcd.setCursor(0, 1);
  lcd.print("#"); lcd.print(String(id));
  delay(700);

  p = finger.createModel();
  if (p == FINGERPRINT_OK) {
    displayText("Matched!", 600);
  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    displayText("Comm error", 700);
    return p;
  } else if (p == FINGERPRINT_ENROLLMISMATCH) {
    displayText("Not match", 700);
    return p;
  } else {
    displayText("Unknown error", 700);
    return p;
  }

  p = finger.storeModel(id);
  if (p == FINGERPRINT_OK) {
    displayText("Stored!", 600);
    mqttClient.publish(TOPIC_STATUS, ("EnrolledID:" + String(id)).c_str());
    return p;
  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    displayText("Comm error", 700);
    return p;
  } else if (p == FINGERPRINT_BADLOCATION) {
    displayText("Bad location", 700);
    return p;
  } else if (p == FINGERPRINT_FLASHERR) {
    displayText("Flash error", 700);
    return p;
  } else {
    displayText("Store fail", 700);
    return p;
  }
}

void checkFingerprint() {
  displayText("Place finger...", 800);
  delay(300);

  uint8_t p = finger.getImage();
  if (p != FINGERPRINT_OK) return;

  p = finger.image2Tz();
  if (p != FINGERPRINT_OK) {
    displayText("Convert fail", 700);
    return;
  }

  p = finger.fingerFastSearch();
  if (p == FINGERPRINT_OK) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("FP ID ");
    lcd.print(finger.fingerID);
    lcd.setCursor(0, 1);
    lcd.print("conf ");
    lcd.print(finger.confidence);
    lcdMsgUntil = millis() + 1500;
  } else if (p == FINGERPRINT_NOTFOUND) {
    displayText("Not found", 700);
  } else {
    displayText("Search error", 700);
  }
}

// ===== Critical path: Fingerprint unlock + latency logging =====
void checkFingerprintAndUnlock() {
  uint8_t p = finger.getImage();
  if (p != FINGERPRINT_OK) return;

  p = finger.image2Tz();
  if (p != FINGERPRINT_OK) return;

  p = finger.fingerFastSearch();
  if (p == FINGERPRINT_OK) {
    // T0: xác thực thành công
    t0_event_us = micros();

    unsigned long t1_us = micros();  // T1: trước relay
    unlockDoorPulse(100);
    unsigned long t2_us = micros();

    unsigned long e2e_us = t1_us - t0_event_us;
    char buf[160];
    snprintf(buf, sizeof(buf),
             "UnlockedByFP;id=%u;e2e_us=%lu;act_us=%lu",
             finger.fingerID, e2e_us, (t2_us - t1_us));
    mqttClient.publish(TOPIC_STATUS, buf);

    displayText("Door unlocked!", 500);
    lastActMs = millis();
  }
}

// ===================== MQTT callback (enqueue only) =====================
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String msg;
  msg.reserve(length);
  for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];

  if (String(topic) == TOPIC_CMD) {
    if (msg == "Open") {
      // enqueue into mailbox
      cmdOpenPending = true;
      t0_cmd_us = micros();  // T0 nhận lệnh
    }
  }
}

// ===== Periodic server: consume cmdOpenPending + cooldown drop =====
void processCmdServer() {
  if (!cmdOpenPending) return;

  unsigned long nowMs = millis();
  if (nowMs - lastActMs < ACT_COOLDOWN_MS) {
    cmdOpenPending = false; // drop
    mqttClient.publish(TOPIC_STATUS, "CmdDropped:Cooldown");
    return;
  }

  cmdOpenPending = false;
  t0_event_us = t0_cmd_us;  // T0 cho E2E

  unsigned long t1_us = micros();  // T1 trước relay
  unlockDoorPulse(100);
  unsigned long t2_us = micros();

  lastActMs = nowMs;

  unsigned long e2e_us = t1_us - t0_event_us;
  char buf[160];
  snprintf(buf, sizeof(buf),
           "UnlockedByMQTT;e2e_us=%lu;act_us=%lu",
           e2e_us, (t2_us - t1_us));
  mqttClient.publish(TOPIC_STATUS, buf);

  displayText("MQTT Open", 500);
}

// ===================== Connect WiFi =====================
void connectWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("WiFi connecting");
  int tries = 0;
  while (WiFi.status() != WL_CONNECTED) {
    mqttClient.loop();
    delay(500);
    lcd.setCursor(0, 1);
    lcd.print("Try ");
    lcd.print(++tries);
    lcd.print("    ");
    if (tries > 40) break; // ~20s
  }
  lcd.clear();

  if (WiFi.status() == WL_CONNECTED) {
    displayText("WiFi connected", 700);
  } else {
    displayText("WiFi failed", 900);
  }
}

// ===================== Connect MQTT =====================
void connectMQTT() {
  mqttClient.setServer(MQTT_HOST, MQTT_PORT);
  mqttClient.setCallback(mqttCallback);

  // Quick setup: insecure TLS (no CA check)
  secureClient.setInsecure();

  while (!mqttClient.connected()) {
    mqttClient.loop();
    displayText("MQTT connecting", 600);

    String clientId = "door2-" + String((uint32_t)ESP.getEfuseMac(), HEX);
    bool ok = mqttClient.connect(clientId.c_str(), MQTT_USERNAME, MQTT_PASSWORD);

    if (ok) {
      displayText("MQTT connected", 700);
      mqttClient.subscribe(TOPIC_CMD);
      mqttClient.publish(TOPIC_STATUS, "Online");
    } else {
      displayText("MQTT retry...", 600);
      delay(800);
    }
  }
}

// ===================== Arduino setup/loop =====================
void setup() {
  lcd.init();
  lcd.backlight();

  pinMode(relayPin, OUTPUT);
  digitalWrite(relayPin, LOW);

  Serial.begin(57600);
  delay(100);

  Serial2.begin(57600, SERIAL_8N1, RX_PIN, TX_PIN);
  finger.begin(57600);

  if (finger.verifyPassword()) {
    displayText("Found finger!", 800);
  } else {
    displayText("No finger sensor", 1200);
    while (1) { delay(1); }
  }

  connectWiFi();
  connectMQTT();

  // init scheduler ticks
  unsigned long now = millis();
  nextFpMs = now + P_FP_MS;
  nextMqttSrvMs = now + P_MQTT_SRV_MS;
  nextNetMs = now + P_NET_MS;
  nextUiMs = now + P_UI_MS;
}

void loop() {
  // (1) Network keep-alive
  if (WiFi.status() != WL_CONNECTED) connectWiFi();
  if (!mqttClient.connected()) connectMQTT();
  mqttClient.loop();

  unsigned long now = millis();

  // (2) RMS-like high priority periodic tasks first
  if ((long)(now - nextFpMs) >= 0) {
    nextFpMs = now + P_FP_MS;
    if (mode == 0) {
      checkFingerprintAndUnlock(); // critical
    }
  }

  if ((long)(now - nextMqttSrvMs) >= 0) {
    nextMqttSrvMs = now + P_MQTT_SRV_MS;
    processCmdServer(); // critical
  }

  // (3) medium periodic
  if ((long)(now - nextNetMs) >= 0) {
    nextNetMs = now + P_NET_MS;
    // để trống hoặc thêm kiểm tra nhẹ (đừng blocking)
  }

  // (4) low priority UI periodic
  if ((long)(now - nextUiMs) >= 0) {
    nextUiMs = now + P_UI_MS;

    char customKey = customKeypad.getKey();
    switch (customKey) {
      case 'A':
        displayText("Add mode", 500);
        mode = 0;
        addFingerprint();
        break;
      case 'B':
        DeleteChooseFingerprint();
        break;
      case 'C':
        DeleteAllFingerprints();
        break;
      case 'D':
        checkFingerprint();
        break;
      default:
        break;
    }
  }

  // (5) clear LCD after timeout (non-blocking)
  if (lcdMsgUntil && millis() > lcdMsgUntil) {
    lcd.clear();
    lcdMsgUntil = 0;
  }

  delay(1);
}
