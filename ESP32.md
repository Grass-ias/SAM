# ESP32 Smart AC Gateway — Technical Specification & Firmware Architecture

Dokumen ini menjelaskan arsitektur perangkat keras, penataan pinout GPIO yang aman, arsitektur firmware C++ (Arduino/PlatformIO), serta analisis teknis mendalam untuk **ESP32 Smart AC Gateway**. Node ini bertindak murni sebagai *dumb node* (sensor & actuator) yang dikontrol oleh server pusat Golang (**SAM**).

---

## 1. Analisis Arsitektur Teknis (Menjawab Pertanyaan Kunci)

### 1.1 Manajemen IR: Skrip Terpisah (Sniffer vs Produksi)
> **Rekomendasi Mutlak: PISAHKAN menjadi skrip/firmware tersendiri.**
* **Alasan Teknis**: 
  1. Library seperti `IRremoteESP8266` mengaktifkan interrupt timer berbasis hardware dengan resolusi mikrodetik (50µs) untuk mendengarkan receiver pin secara kontinyu. Jika receiver dibiarkan aktif saat menerima request HTTP melalui Wi-Fi, terjadi perebutan resource interrupt timer antara WiFi stack (Core 0/1) dan IR receiver, yang sering kali menyebabkan buffer overflow, packet drop, atau *Watchdog Timer (WDT) Reset*.
  2. Sniffing remote AC hanya dilakukan **sekali di awal** untuk menangkap protokol & state data remote fisik AC.
  3. Menyimpan sniffer di kode produksi menghabiskan memori RAM/Flash dan membuka celah crash tanpa ada nilai guna operasional sehari-hari.
* **Solusi**: Sediakan dua target di PlatformIO:
  * `sniffer`: Skrip khusus untuk kalibrasi/perekaman remote (DumpV2).
  * `production`: Skrip runtime ringan (hanya Web Server + DHT22 non-blocking + IR Sender).

### 1.2 Delay / Blocking & Web Server Asynchronous
> **Masalah**: DHT22 membutuhkan waktu konversi hingga ~250ms dan interval pembacaan minimal 2 detik (bit-banging). Jika pembacaan dilakukan sinkron di dalam HTTP handler, webserver standar (`WebServer.h`) bisa freeze atau timeout saat ada concurrent request dari Golang.
* **Solusi**:
  1. Gunakan library **`ESPAsyncWebServer`** berbasis `AsyncTCP` (non-blocking & event-driven).
  2. Terapkan pola **Background Sensor Polling**: Sensor DHT22 dibaca setiap 3-5 detik secara berkala di loop utama / timer millis, lalu hasilnya disimpan ke variabel global (`cachedTemperature`, `cachedHumidity`).
  3. Ketika endpoint `GET /temp` dipanggil, web server langsung mengembalikan nilai cache dalam hitungan mikrodetik tanpa harus menunggu DHT22 melakukan pembacaan fisik.
  4. Pengiriman sinyal AC via IR Transmitter dilakukan secara terisolasi tanpa `delay()` besar, atau menggunakan helper library `IRac` yang langsung memancarkan raw state buffer AC secara instan.

### 1.3 Stabilitas Wi-Fi & Auto-Reconnect
* **Masalah**: Koneksi Wi-Fi ESP32 sering kali mengalami *disconnect* diam-diam (silent drop) setelah berhari-hari menyala, menyebabkan IP tidak dapat dijangkau oleh SAM.
* **Solusi**:
  1. Mendaftarkan **Wi-Fi Event Listener** native bawaan ESP-IDF (`WiFi.onEvent(...)`) untuk merespons event `SYSTEM_EVENT_STA_DISCONNECTED` secara instan.
  2. Menyertakan mekanisme heartbeat non-blocking di `loop()` yang memeriksa `WiFi.status() == WL_CONNECTED` setiap 10 detik. Jika putus, panggil `WiFi.reconnect()`.
  3. Konfigurasi Static IP (atau DHCP reservation di router) agar IP tidak berganti-ganti setelah restart.

---

## 2. Daftar Alat, Komponen & Rekomendasi Pinout GPIO

### 2.1 Bill of Materials (BOM)
1. **ESP32 DevKit V1** (30-pin atau 38-pin, berbasis ESP-WROOM-32).
2. **Sensor DHT22 (AM2302)** + Resistor Pull-up 10kΩ (jika modul tidak memiliki resistor built-in).
3. **IR Transmitter Module 38kHz** (dilengkapi transistor driver NPN 2N2222 / SS8050 agar jangkauan pancaran IR kuat menembus ruangan).
4. **IR Receiver Module (misal VS1838B / HX1838)** untuk sniffing awal.
5. **Kabel Jumper & Breadboard / PCB prototipe**.
6. **Kabel Micro-USB & Adaptor 5V 1A/2A stabil**.

### 2.2 Panduan Pinout GPIO Aman (Safe GPIO Allocation)

ESP32 memiliki beberapa pin yang tidak boleh sembarangan digunakan karena merupakan *Strapping Pins* (mempengaruhi mode Boot/Flash) atau *Input Only* (tanpa pull-up internal).

| Komponen | Pin ESP32 | Tipe GPIO | Alasan & Keamanan Pin |
| :--- | :--- | :--- | :--- |
| **DHT22 Data** | **GPIO 4** | Input/Output | Aman digunakan. Bebas dari strapping pin dan tidak bentrok dengan ADC2 atau Wi-Fi. Dilengkapi pull-up internal/eksternal. |
| **IR Transmitter (LED)** | **GPIO 19** | Output | Pin VSPI MISO standar. Aman untuk output PWM berfrekuensi 38kHz tanpa mengganggu boot state. |
| **IR Receiver (Sniffer)** | **GPIO 18** | Input | Pin VSPI SCK standar. Mendukung interrupt hardware dengan sangat baik. Hanya dipasang saat kalibrasi. |
| **Status LED (Built-in)**| **GPIO 2** | Output | Indikator koneksi Wi-Fi / aktivitas (High saat terkoneksi). |

> [!CAUTION]
> **PIN YANG HARUS DIHINDARI:**
> - **GPIO 0, 2, 5, 12, 15**: Strapping pins (dapat menyebabkan ESP32 gagal boot jika kondisi tegangan ditarik high/low saat startup).
> - **GPIO 6-11**: Terhubung langsung ke SPI Flash internal (dilarang keras digunakan!).
> - **GPIO 34, 35, 36, 39**: Hanya dapat berfungsi sebagai INPUT (*no internal pull-up/pull-down*), tidak bisa digunakan untuk IR Transmitter.

---

## 3. Skema Kontrak API & Endpoint ESP32

Semua endpoint beroperasi pada port **80**.

### 3.1 `GET /temp`
* **Deskripsi**: Mengambil data suhu dan kelembapan real-time (dari cache sensor DHT22).
* **Response Header**: `Content-Type: application/json`
* **Response Status**: `200 OK`
* **Response Body**:
  ```json
  {
    "temperature": 27.8,
    "humidity": 62.4,
    "status": "healthy",
    "uptime_seconds": 12450
  }
  ```

### 3.2 `GET /ac-on`
* **Deskripsi**: Menembakkan sinyal IR untuk menyalakan AC (dengan profil default: Mode Cool, 24°C, Fan Auto).
* **Response Header**: `Content-Type: application/json`
* **Response Status**: `200 OK`
* **Response Body**:
  ```json
  {
    "success": true,
    "command": "AC_POWER_ON",
    "message": "Sinyal IR berhasil dipancarkan"
  }
  ```

### 3.3 `GET /health`
* **Deskripsi**: Diagnostic endpoint untuk memeriksa status Wi-Fi RSSI dan uptime.
* **Response Status**: `200 OK`
* **Response Body**:
  ```json
  {
    "status": "online",
    "rssi": -58,
    "free_heap": 214532
  }
  ```

---

## 4. Implementasi Firmware C++ (PlatformIO / Arduino)

### 4.1 Konfigurasi `platformio.ini`

```ini
[platformio]
default_envs = production

[env]
platform = espressif32
board = esp32doit-devkit-v1
framework = arduino
monitor_speed = 115200
lib_deps = 
    crankyoldgit/IRremoteESP8266 @ ^2.8.6
    adafruit/DHT sensor library @ ^1.4.6
    adafruit/Adafruit Unified Sensor @ ^1.1.14
    https://github.com/me-no-dev/ESPAsyncWebServer.git
    https://github.com/me-no-dev/AsyncTCP.git

[env:production]
build_src_filter = +<production_main.cpp>

[env:sniffer]
build_src_filter = +<sniffer_main.cpp>
```

---

### 4.2 Skrip 1: Skrip Sniffer Kalibrasi (`src/sniffer_main.cpp`)
*Gunakan skrip ini saat pertama kali ingin membaca dan mencatat sinyal remote fisik AC Anda.*

```cpp
#include <Arduino.h>
#include <IRrecv.h>
#include <IRremoteESP8266.h>
#include <IRutils.h>

const uint16_t RECV_PIN = 18; // GPIO 18 untuk IR Receiver
const uint16_t CAPTURE_BUFFER_SIZE = 1024;
const uint8_t TIMEOUT = 50;   // Timeout dalam milidetik

IRrecv irrecv(RECV_PIN, CAPTURE_BUFFER_SIZE, TIMEOUT, true);
decode_results results;

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("==================================================");
  Serial.println(">>> SAM ESP32 IR Sniffer Siap Merekam Remote <<<");
  Serial.println("Arahkan remote AC ke IR Receiver dan tekan tombol.");
  Serial.println("==================================================");
  
  irrecv.enableIRIn(); // Mulai menerima sinyal
}

void loop() {
  if (irrecv.decode(&results)) {
    Serial.println("\n----------------- SINYAL TERDETEKSI -----------------");
    // Mencetak decoding protokol (Panasonic, Daikin, Gree, LG, dsb.)
    Serial.print("Protokol / Brand : ");
    Serial.println(typeToString(results.decode_type));
    
    // Dump dalam format C++ kode array yang bisa langsung di-copy
    Serial.println("\nKode Raw (C++ array format):");
    Serial.println(resultToSourceCode(&results));
    
    Serial.println("-----------------------------------------------------");
    irrecv.resume(); // Siap menerima sinyal berikutnya
  }
  delay(100);
}
```

---

### 4.3 Skrip 2: Firmware Produksi Node (`src/production_main.cpp`)
*Skrip ini yang akan di-flash dan dijalankan 24/7 di kamar.*

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>
#include <DHT.h>
#include <IRremoteESP8266.h>
#include <IRsend.h>
#include <ir_Panasonic.h> // Contoh: ganti atau sesuaikan dengan library AC Anda (atau IRac generic)

// ==========================================
// 1. KONFIGURASI PIN & SENSOR
// ==========================================
#define DHTPIN 4
#define DHTTYPE DHT22
#define IR_SEND_PIN 19
#define LED_STATUS_PIN 2

// ==========================================
// 2. KREDENSIAL WI-FI
// ==========================================
const char* WIFI_SSID = "NAMA_WIFI_RUMAH";
const char* WIFI_PASS = "PASSWORD_WIFI";

// Static IP opsional (disarankan agar IP tidak berubah)
IPAddress local_IP(192, 168, 1, 50);
IPAddress gateway(192, 168, 1, 1);
IPAddress subnet(255, 255, 255, 0);

// ==========================================
// 3. OBJECT INSTANCES & GLOBAL CACHE
// ==========================================
DHT dht(DHTPIN, DHTTYPE);
IRsend irsend(IR_SEND_PIN);
AsyncWebServer server(80);

// Global Cache Sensor (Non-blocking access)
float cachedTemperature = 0.0;
float cachedHumidity = 0.0;
unsigned long lastDHTReadTime = 0;
const unsigned long DHT_INTERVAL = 3000; // Baca sensor tiap 3 detik

unsigned long lastWiFiCheck = 0;
const unsigned long WIFI_CHECK_INTERVAL = 10000; // Cek koneksi tiap 10 detik

// ==========================================
// 4. LOGIKA PANCARAN IR AC
// ==========================================
void sendACOnCommand() {
  Serial.println("[IR] Mengirimkan sinyal AC ON...");
  
  // Contoh menggunakan raw signal atau helper class
  // Jika menggunakan kode raw yang didapat dari sniffer:
  // uint16_t rawData[] = { ... };
  // irsend.sendRaw(rawData, sizeof(rawData)/sizeof(rawData[0]), 38);
  
  // Jika menggunakan brand helper (misal Panasonic/Daikin/Gree):
  // IRPanasonicAc ac(IR_SEND_PIN);
  // ac.on(); ac.setMode(kPanasonicAcCool); ac.setTemp(24); ac.send();

  // Trigger test blink LED
  digitalWrite(LED_STATUS_PIN, LOW);
  delay(100);
  digitalWrite(LED_STATUS_PIN, HIGH);
}

// ==========================================
// 5. EVENT LISTENER & WI-FI RECONNECT
// ==========================================
void WiFiEvent(WiFiEvent_t event) {
  switch (event) {
    case SYSTEM_EVENT_STA_GOT_IP:
      Serial.print("[WiFi] Terhubung! IP: ");
      Serial.println(WiFi.localIP());
      digitalWrite(LED_STATUS_PIN, HIGH);
      break;
    case SYSTEM_EVENT_STA_DISCONNECTED:
      Serial.println("[WiFi] Koneksi terputus! Mencoba menghubungkan ulang...");
      digitalWrite(LED_STATUS_PIN, LOW);
      WiFi.reconnect();
      break;
    default:
      break;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_STATUS_PIN, OUTPUT);
  digitalWrite(LED_STATUS_PIN, LOW);

  // Inisialisasi Hardware
  dht.begin();
  irsend.begin();

  // Setup WiFi
  WiFi.mode(WIFI_STA);
  WiFi.config(local_IP, gateway, subnet);
  WiFi.onEvent(WiFiEvent);
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  Serial.println("[BOOT] Menghubungkan ke Wi-Fi...");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  // ==========================================
  // 6. SETUP ENDPOINTS ASYNC WEB SERVER
  // ==========================================

  // Endpoint 1: GET /temp
  server.on("/temp", HTTP_GET, [](AsyncWebServerRequest *request) {
    String json = "{";
    json += "\"temperature\":" + String(cachedTemperature, 1) + ",";
    json += "\"humidity\":" + String(cachedHumidity, 1) + ",";
    json += "\"status\":\"healthy\",";
    json += "\"uptime_seconds\":" + String(millis() / 1000);
    json += "}";
    request->send(200, "application/json", json);
  });

  // Endpoint 2: GET /ac-on
  server.on("/ac-on", HTTP_GET, [](AsyncWebServerRequest *request) {
    sendACOnCommand();
    String json = "{\"success\":true,\"command\":\"AC_POWER_ON\",\"message\":\"Sinyal IR terkirim\"}";
    request->send(200, "application/json", json);
  });

  // Endpoint 3: GET /health
  server.on("/health", HTTP_GET, [](AsyncWebServerRequest *request) {
    String json = "{";
    json += "\"status\":\"online\",";
    json += "\"rssi\":" + String(WiFi.RSSI()) + ",";
    json += "\"free_heap\":" + String(ESP.getFreeHeap());
    json += "}";
    request->send(200, "application/json", json);
  });

  server.begin();
  Serial.println("[SERVER] HTTP Web Server berjalan di port 80");
}

void loop() {
  unsigned long currentMillis = millis();

  // Background Non-blocking Polling DHT22
  if (currentMillis - lastDHTReadTime >= DHT_INTERVAL) {
    lastDHTReadTime = currentMillis;

    float t = dht.readTemperature();
    float h = dht.readHumidity();

    if (!isnan(t) && !isnan(h)) {
      cachedTemperature = t;
      cachedHumidity = h;
    } else {
      Serial.println("[WARN] Gagal membaca data dari DHT22");
    }
  }

  // Backup Heartbeat Wi-Fi Reconnect
  if (currentMillis - lastWiFiCheck >= WIFI_CHECK_INTERVAL) {
    lastWiFiCheck = currentMillis;
    if (WiFi.status() != WL_CONNECTED) {
      Serial.println("[WiFi] Heartbeat mendeteksi offline. Reconnecting...");
      WiFi.reconnect();
    }
  }
}
```

---

## 5. Ringkasan & Langkah Integrasi dengan SAM

1. **Gunakan skrip `sniffer`** untuk merekam tombol power/on remote AC fisik Anda, catat raw timing array atau protokolnya.
2. **Masukkan kode IR ke fungsi `sendACOnCommand()`** di `production_main.cpp`.
3. **Flash environment `production`** ke ESP32.
4. Di sisi server **SAM (Golang)**:
   - SAM dapat menembak `GET http://192.168.1.50/temp` untuk mengetahui suhu ruangan real-time sebelum menyalakan AC.
   - SAM menembak `GET http://192.168.1.50/ac-on` untuk menyalakan AC secara otomatis saat protokol kedatangan tiba.
