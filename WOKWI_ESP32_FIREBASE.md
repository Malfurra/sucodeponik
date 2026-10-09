# Panduan Integrasi Wokwi ESP32 dengan Firebase Realtime Database (Sucodeponik)

Panduan ini berisi kode program lengkap (C++ Arduino) dan skema rangkaian Wokwi untuk menghubungkan mikrokontroler **ESP32** ke **Firebase Realtime Database** pada proyek `sucodeponik`.

---

## 1. Topologi Data Firebase (`agriculture_iot`)

Struktur JSON pada database:
```json
{
  "agriculture_iot": {
    "ph": 6.2,
    "tds": 850,
    "temperature": 27.5,
    "humidity": 65.0,
    "water_level": 85,
    "relays": {
      "relay1": false,
      "relay2": false,
      "relay3": false,
      "relay4": false
    }
  }
}
```

---

## 2. Kode Lengkap ESP32 Wokwi (`sketch.ino`)

Di Wokwi, install library berikut pada tab **Library Manager**:
1. `Firebase ESP32 Client` (by Mobizt)
2. `DHT sensor library` (by Adafruit)
3. `Adafruit Unified Sensor`

```cpp
#include <WiFi.h>
#include <FirebaseESP32.h>
#include "DHT.h"

// --- KONFIGURASI WIFI WOKWI ---
#define WIFI_SSID "Wokwi-GUEST"
#define WIFI_PASSWORD ""

// --- KONFIGURASI FIREBASE SUCODEPONIK ---
#define FIREBASE_HOST "https://sucodeponik-default-rtdb.asia-southeast1.firebasedatabase.app/"
#define FIREBASE_AUTH "AIzaSyADWJLpXaQLIB7rcV31C8wm2fG54CE18JE" // Web API Key / Database Secret

// --- PIN DEFINITIONS ---
#define PIN_PH_POT     34  // Potensiometer simulator sensor pH (ADC1)
#define PIN_TDS_POT    35  // Potensiometer simulator sensor TDS (ADC1)
#define PIN_DHT        15  // Sensor Suhu & Kelembapan DHT22
#define PIN_TRIG        5  // Ultrasonic HC-SR04 Trigger (Water Level)
#define PIN_ECHO       18  // Ultrasonic HC-SR04 Echo

// Pin Aktuator Relay
#define RELAY1_PIN     25  // Pompa pH Up
#define RELAY2_PIN     26  // Pompa pH Down
#define RELAY3_PIN     27  // Pompa Nutrisi A
#define RELAY4_PIN     14  // Pompa Nutrisi B

#define DHTTYPE DHT22
DHT dht(PIN_DHT, DHTTYPE);

FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

unsigned long lastSendTime = 0;
const unsigned long sendInterval = 3000; // Kirim telemetri setiap 3 detik

// Fungsi membaca ketinggian air dari HC-SR04 (0 - 100%)
float readWaterLevel() {
  digitalWrite(PIN_TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(PIN_TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(PIN_TRIG, LOW);

  long duration = pulseIn(PIN_ECHO, HIGH, 30000);
  if (duration == 0) return 75.0; // Nilai default bila timeout
  float distance = duration * 0.034 / 2.0; // cm

  // Asumsi tinggi wadah tandon = 20 cm
  float percentage = (20.0 - distance) / 20.0 * 100.0;
  return constrain(percentage, 0.0, 100.0);
}

void setup() {
  Serial.begin(115200);

  // Setup Pin Mode
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);
  pinMode(RELAY3_PIN, OUTPUT);
  pinMode(RELAY4_PIN, OUTPUT);
  digitalWrite(RELAY1_PIN, LOW);
  digitalWrite(RELAY2_PIN, LOW);
  digitalWrite(RELAY3_PIN, LOW);
  digitalWrite(RELAY4_PIN, LOW);

  pinMode(PIN_TRIG, OUTPUT);
  pinMode(PIN_ECHO, INPUT);

  dht.begin();

  // Koneksi WiFi Wokwi
  Serial.print("Menghubungkan ke Wokwi WiFi...");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Terhubung! IP: " + WiFi.localIP().toString());

  // Setup Firebase
  config.host = FIREBASE_HOST;
  config.signer.tokens.legacy_token = FIREBASE_AUTH;

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
  Serial.println("Firebase Client Siap!");
}

void loop() {
  // 1. DENGARKAN STATUS SAKELAR DARI WEB DASHBOARD
  if (Firebase.getBool(fbdo, "/agriculture_iot/relays/relay1")) {
    digitalWrite(RELAY1_PIN, fbdo.boolData() ? HIGH : LOW);
  }
  if (Firebase.getBool(fbdo, "/agriculture_iot/relays/relay2")) {
    digitalWrite(RELAY2_PIN, fbdo.boolData() ? HIGH : LOW);
  }
  if (Firebase.getBool(fbdo, "/agriculture_iot/relays/relay3")) {
    digitalWrite(RELAY3_PIN, fbdo.boolData() ? HIGH : LOW);
  }
  if (Firebase.getBool(fbdo, "/agriculture_iot/relays/relay4")) {
    digitalWrite(RELAY4_PIN, fbdo.boolData() ? HIGH : LOW);
  }

  // 2. KIRIM TELEMETRI SENSOR SETIAP 3 DETIK KE FIREBASE
  if (millis() - lastSendTime > sendInterval) {
    lastSendTime = millis();

    // Membaca Potensiometer pH (0 - 4095 dipetakan ke 0.0 - 14.0)
    int rawPh = analogRead(PIN_PH_POT);
    float phVal = (rawPh / 4095.0) * 14.0;

    // Membaca Potensiometer TDS (0 - 4095 dipetakan ke 0 - 2000 PPM)
    int rawTds = analogRead(PIN_TDS_POT);
    float tdsVal = (rawTds / 4095.0) * 2000.0;

    // Membaca DHT22
    float tempVal = dht.readTemperature();
    float humVal = dht.readHumidity();
    if (isnan(tempVal)) tempVal = 27.5;
    if (isnan(humVal)) humVal = 65.0;

    // Membaca Ultrasonic Ketinggian Air
    float waterVal = readWaterLevel();

    // Kirim data secara terstruktur ke Firebase
    FirebaseJson json;
    json.set("ph", phVal);
    json.set("tds", tdsVal);
    json.set("temperature", tempVal);
    json.set("humidity", humVal);
    json.set("water_level", waterVal);

    if (Firebase.updateNode(fbdo, "/agriculture_iot", json)) {
      Serial.printf("[TERKIRIM] pH: %.2f | TDS: %.0f PPM | Suhu: %.1f C | Air: %.0f%%\n",
                    phVal, tdsVal, tempVal, waterVal);
    } else {
      Serial.println("[ERROR FIREBASE]: " + fbdo.errorReason());
    }
  }

  delay(200);
}
```

---

## 3. Skema Komponen di Wokwi (`diagram.json`)

Di Wokwi, gunakan komponen berikut:
1. **ESP32 DevKit v1**
2. **Potensiometer 1 (Sensor pH)**:
   - Pin VCC -> ESP32 3V3
   - Pin GND -> ESP32 GND
   - Pin SIG -> ESP32 GPIO 34
3. **Potensiometer 2 (Sensor TDS)**:
   - Pin VCC -> ESP32 3V3
   - Pin GND -> ESP32 GND
   - Pin SIG -> ESP32 GPIO 35
4. **Sensor DHT22 (Suhu & Kelembapan)**:
   - Pin VCC -> ESP32 3V3
   - Pin GND -> ESP32 GND
   - Pin SDA / DATA -> ESP32 GPIO 15
5. **Sensor Ultrasonic HC-SR04 (Water Level)**:
   - Pin TRIG -> ESP32 GPIO 5
   - Pin ECHO -> ESP32 GPIO 18
6. **4 Buah LED / Relay Module (Indikator Pompa)**:
   - Relay 1 -> GPIO 25 (pH Up)
   - Relay 2 -> GPIO 26 (pH Down)
   - Relay 3 -> GPIO 27 (Nutrisi A)
   - Relay 4 -> GPIO 14 (Nutrisi B)

---

## 4. File `diagram.json` (Langsung Salin ke Tab `diagram.json` di Wokwi)

```json
{
  "version": 1,
  "author": "SMKN 2 Yogyakarta - Sucodeponik IoT",
  "editor": "wokwi",
  "parts": [
    { "type": "board-esp32-devkit-c-v4", "id": "esp", "top": 100, "left": 100, "attrs": {} },
    { "type": "wokwi-potentiometer", "id": "pot_ph", "top": -90, "left": -120, "attrs": { "label": "Sensor pH" } },
    { "type": "wokwi-potentiometer", "id": "pot_tds", "top": 20, "left": -120, "attrs": { "label": "Sensor TDS" } },
    { "type": "wokwi-dht22", "id": "dht", "top": -120, "left": 260, "attrs": { "temperature": "27.5", "humidity": "65" } },
    { "type": "wokwi-hc-sr04", "id": "ultrasonic", "top": -120, "left": 420, "attrs": { "distance": "5" } },
    { "type": "wokwi-led", "id": "led_ph_up", "top": 360, "left": -60, "attrs": { "color": "green", "label": "Pompa pH Up" } },
    { "type": "wokwi-led", "id": "led_ph_down", "top": 360, "left": 40, "attrs": { "color": "red", "label": "Pompa pH Down" } },
    { "type": "wokwi-led", "id": "led_nutrisi_a", "top": 360, "left": 140, "attrs": { "color": "yellow", "label": "Pompa Nutrisi A" } },
    { "type": "wokwi-led", "id": "led_nutrisi_b", "top": 360, "left": 240, "attrs": { "color": "blue", "label": "Pompa Nutrisi B" } },
    { "type": "wokwi-resistor", "id": "r1", "top": 420, "left": -60, "attrs": { "value": "220" } },
    { "type": "wokwi-resistor", "id": "r2", "top": 420, "left": 40, "attrs": { "value": "220" } },
    { "type": "wokwi-resistor", "id": "r3", "top": 420, "left": 140, "attrs": { "value": "220" } },
    { "type": "wokwi-resistor", "id": "r4", "top": 420, "left": 240, "attrs": { "value": "220" } }
  ],
  "connections": [
    [ "esp:TX", "$if:serial:RX", "", [] ],
    [ "esp:RX", "$if:serial:TX", "", [] ],

    [ "esp:3V3", "pot_ph:VCC", "red", [ "v0" ] ],
    [ "esp:GND.1", "pot_ph:GND", "black", [ "v0" ] ],
    [ "esp:34", "pot_ph:SIG", "green", [ "v0" ] ],

    [ "esp:3V3", "pot_tds:VCC", "red", [ "v0" ] ],
    [ "esp:GND.1", "pot_tds:GND", "black", [ "v0" ] ],
    [ "esp:35", "pot_tds:SIG", "orange", [ "v0" ] ],

    [ "esp:3V3", "dht:VCC", "red", [ "v0" ] ],
    [ "esp:GND.2", "dht:GND", "black", [ "v0" ] ],
    [ "esp:15", "dht:SDA", "yellow", [ "v0" ] ],

    [ "esp:5V", "ultrasonic:VCC", "red", [ "v0" ] ],
    [ "esp:GND.2", "ultrasonic:GND", "black", [ "v0" ] ],
    [ "esp:5", "ultrasonic:TRIG", "purple", [ "v0" ] ],
    [ "esp:18", "ultrasonic:ECHO", "blue", [ "v0" ] ],

    [ "esp:25", "led_ph_up:A", "green", [ "v0" ] ],
    [ "led_ph_up:C", "r1:1", "black", [ "v0" ] ],
    [ "r1:2", "esp:GND.1", "black", [ "v0" ] ],

    [ "esp:26", "led_ph_down:A", "red", [ "v0" ] ],
    [ "led_ph_down:C", "r2:1", "black", [ "v0" ] ],
    [ "r2:2", "esp:GND.1", "black", [ "v0" ] ],

    [ "esp:27", "led_nutrisi_a:A", "yellow", [ "v0" ] ],
    [ "led_nutrisi_a:C", "r3:1", "black", [ "v0" ] ],
    [ "r3:2", "esp:GND.1", "black", [ "v0" ] ],

    [ "esp:14", "led_nutrisi_b:A", "blue", [ "v0" ] ],
    [ "led_nutrisi_b:C", "r4:1", "black", [ "v0" ] ],
    [ "r4:2", "esp:GND.1", "black", [ "v0" ] ]
  ],
  "dependencies": {}
}
```

