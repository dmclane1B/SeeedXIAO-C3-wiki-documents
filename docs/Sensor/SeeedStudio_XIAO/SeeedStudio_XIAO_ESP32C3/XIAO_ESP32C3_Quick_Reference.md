---
description: Quick Reference Card for Seeed Studio XIAO ESP32C3
title: Quick Reference
keywords:
- xiao
- esp32c3
- pinout
- cheat sheet
- quick reference
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Quick_Reference
last_update:
  date: 02/07/2026
  author: Claude
---

# XIAO ESP32C3 Quick Reference

A compact reference for pin mappings, common code patterns, and API usage. Keep this page bookmarked for fast lookups during development.

## Pin Quick Reference

### Pin Map Table

| Board Pin | GPIO | Analog | Digital | PWM | UART | I2C | SPI | JTAG | Notes |
|:----------|:-----|:-------|:--------|:----|:-----|:----|:----|:-----|:------|
| D0 | 2 | A0 (ADC1) | Yes | Yes | - | - | - | - | Strapping pin |
| D1 | 3 | A1 (ADC1) | Yes | Yes | - | - | - | - | Deep sleep wake |
| D2 | 4 | A2 (ADC1) | Yes | Yes | - | - | - | TMS | Deep sleep wake, strapping |
| D3 | 5 | A3 (ADC2) | Yes | Yes | - | - | - | TDI | Deep sleep wake, avoid w/ WiFi |
| D4 | 6 | - | Yes | Yes | - | SDA | - | TCK | Deep sleep wake |
| D5 | 7 | - | Yes | Yes | - | SCL | - | TDO | - |
| D6 | 21 | - | Yes | Yes | TX | - | - | - | HIGH at boot (UART TX) |
| D7 | 20 | - | Yes | Yes | RX | - | - | - | - |
| D8 | 8 | - | Yes | Yes | - | - | SCK | - | Strapping pin, needs pull-up |
| D9 | 9 | - | Yes | Yes | - | - | MISO | - | BOOT button (pull-up), strapping |
| D10 | 10 | - | Yes | Yes | - | - | MOSI | - | - |

### Power Pins

| Pin | Voltage | Max Current | Notes |
|:----|:--------|:------------|:------|
| 5V | 5V | USB power in/out | Use diode for external supply |
| 3V3 | 3.3V | 700 mA | Regulated output |
| GND | 0V | - | Ground |
| BAT | 3.7V | - | Solder pads on bottom (Li-Po) |

### Pin Warnings

- **D0 (GPIO2):** Strapping pin — avoid pulling LOW at boot
- **D3 (GPIO5/A3):** Uses ADC2 — unreliable when Wi-Fi is active
- **D6 (GPIO21):** UART TX output at boot — use as output only
- **D8 (GPIO8):** Must be HIGH at boot when using download mode
- **D9 (GPIO9):** Connected to BOOT button — best used as input

## Common Code Patterns

### GPIO

```cpp
// Digital output
pinMode(D10, OUTPUT);
digitalWrite(D10, HIGH);

// Digital input
pinMode(D9, INPUT);        // BOOT button
int state = digitalRead(D9);

// Digital input with internal pull-up
pinMode(D1, INPUT_PULLUP);

// PWM output (0-255)
analogWrite(D10, 128);     // 50% duty cycle

// Analog read
int raw = analogRead(A0);             // 0-4095
int mV  = analogReadMilliVolts(A0);   // Calibrated mV
```

### Serial (UART)

```cpp
// USB Serial (always available)
Serial.begin(115200);
Serial.println("Hello");

// Hardware UART on D6/D7
Serial1.begin(9600, SERIAL_8N1, D7, D6);  // RX=D7, TX=D6
Serial1.println("On UART0");

// Hardware UART on custom pins
HardwareSerial MySerial(1);
MySerial.begin(115200, SERIAL_8N1, D9, D10);  // RX=D9, TX=D10
```

### I2C

```cpp
#include <Wire.h>

// Default I2C: SDA=D4(GPIO6), SCL=D5(GPIO7)
Wire.begin();

// Custom I2C pins
Wire.begin(D2, D3);  // SDA=D2, SCL=D3

// I2C scan
for (byte addr = 1; addr < 127; addr++) {
    Wire.beginTransmission(addr);
    if (Wire.endTransmission() == 0) {
        Serial.printf("Found device at 0x%02X\n", addr);
    }
}
```

### SPI

```cpp
#include <SPI.h>

// Default SPI: SCK=D8, MISO=D9, MOSI=D10
SPI.begin();

// Use SPI with a device
digitalWrite(SS, LOW);
SPI.transfer(0x42);
digitalWrite(SS, HIGH);
```

### Wi-Fi Station

```cpp
#include <WiFi.h>

WiFi.begin("ssid", "password");
while (WiFi.status() != WL_CONNECTED) delay(500);
Serial.println(WiFi.localIP());

// Disconnect and power off radio
WiFi.disconnect(true);
WiFi.mode(WIFI_OFF);
```

### Wi-Fi Access Point

```cpp
#include <WiFi.h>

WiFi.softAP("XIAO-AP", "password123");
Serial.println(WiFi.softAPIP());  // Default: 192.168.4.1
```

### HTTP Client

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

// GET request
HTTPClient http;
http.begin("http://example.com/api");
int code = http.GET();
if (code == HTTP_CODE_OK) {
    String body = http.getString();
}
http.end();

// POST request
http.begin("http://example.com/api");
http.addHeader("Content-Type", "application/json");
http.POST("{\"key\":\"value\"}");
http.end();
```

### Web Server

```cpp
#include <WiFi.h>
#include <WebServer.h>

WebServer server(80);

void setup() {
    WiFi.begin("ssid", "password");
    while (WiFi.status() != WL_CONNECTED) delay(500);

    server.on("/", []() {
        server.send(200, "text/plain", "Hello from XIAO!");
    });
    server.begin();
}

void loop() {
    server.handleClient();
}
```

### BLE (Bluetooth Low Energy)

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"

void setup() {
    BLEDevice::init("XIAO-C3");
    BLEServer *server = BLEDevice::createServer();
    BLEService *service = server->createService(SERVICE_UUID);
    BLECharacteristic *ch = service->createCharacteristic(
        CHARACTERISTIC_UUID,
        BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE
    );
    ch->setValue("Hello BLE");
    service->start();
    BLEAdvertising *adv = BLEDevice::getAdvertising();
    adv->start();
}

void loop() { delay(1000); }
```

### Deep Sleep

```cpp
#include "esp_sleep.h"

// Timer wakeup (30 seconds)
esp_sleep_enable_timer_wakeup(30 * 1000000ULL);
esp_deep_sleep_start();

// GPIO wakeup (D1 LOW)
esp_deep_sleep_enable_gpio_wakeup(BIT(D1), ESP_GPIO_WAKEUP_GPIO_LOW);
esp_deep_sleep_start();

// Persist data across deep sleep
RTC_DATA_ATTR int counter = 0;

// Check wakeup reason
esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
```

### NVS (Non-Volatile Storage)

```cpp
#include <Preferences.h>

Preferences prefs;

// Write
prefs.begin("myapp", false);      // false = read/write
prefs.putInt("boots", bootCount);
prefs.putString("ssid", "MyNetwork");
prefs.end();

// Read
prefs.begin("myapp", true);       // true = read-only
int boots = prefs.getInt("boots", 0);       // default = 0
String ssid = prefs.getString("ssid", "");  // default = ""
prefs.end();
```

### LittleFS / SPIFFS

```cpp
#include <LittleFS.h>

void setup() {
    if (!LittleFS.begin(true)) {  // true = format on fail
        Serial.println("LittleFS mount failed");
        return;
    }

    // Write
    File f = LittleFS.open("/data.txt", "w");
    f.println("Hello from XIAO!");
    f.close();

    // Read
    f = LittleFS.open("/data.txt", "r");
    while (f.available()) {
        Serial.write(f.read());
    }
    f.close();
}
```

### Timers and Interrupts

```cpp
// Hardware timer
hw_timer_t *timer = NULL;
volatile bool timerFired = false;

void IRAM_ATTR onTimer() {
    timerFired = true;
}

void setup() {
    timer = timerBegin(1000000);           // 1 MHz
    timerAttachInterrupt(timer, &onTimer);
    timerAlarm(timer, 1000000, true, 0);   // 1 second, auto-reload
}

void loop() {
    if (timerFired) {
        timerFired = false;
        Serial.println("Timer fired!");
    }
}

// GPIO interrupt
void IRAM_ATTR buttonISR() {
    // Keep ISR short — set a flag
}

void setup() {
    pinMode(D9, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(D9), buttonISR, FALLING);
}
```

## Specs at a Glance

| Parameter | Value |
|:----------|:------|
| MCU | ESP32-C3 (RISC-V 32-bit, 160 MHz) |
| SRAM | 400 KB |
| Flash | 4 MB |
| Wi-Fi | 802.11 b/g/n, 2.4 GHz |
| Bluetooth | BLE 5.0, Mesh |
| GPIO | 11 (all PWM-capable) |
| ADC | 4 channels (12-bit, 0-2500mV) |
| UART | 2 (+ USB CDC) |
| I2C | 1 (default: D4=SDA, D5=SCL) |
| SPI | 1 (default: D8=SCK, D9=MISO, D10=MOSI) |
| USB | Type-C (data + power) |
| Battery | 3.7V Li-Po via solder pads |
| Deep Sleep | ~44 uA |
| Size | 21 x 17.8 mm |
| Voltage | 3.3V logic, 5V USB input |

## Arduino Board Settings

| Setting | Value |
|:--------|:------|
| Board | XIAO_ESP32C3 |
| Upload Speed | 921600 |
| CPU Frequency | 160MHz |
| Flash Frequency | 80MHz |
| Flash Mode | QIO |
| Flash Size | 4MB (32Mb) |
| Partition Scheme | Default 4MB with spiffs |
| USB CDC On Boot | Enabled |

## PlatformIO Quick Config

```ini
[env:seeed_xiao_esp32c3]
platform = espressif32
board = seeed_xiao_esp32c3
framework = arduino
monitor_speed = 115200
upload_speed = 921600
lib_deps =
    ; Add your libraries here
```

## Key Links

| Resource | Link |
|:---------|:-----|
| Getting Started | [XIAO_ESP32C3_Getting_Started](/XIAO_ESP32C3_Getting_Started) |
| Architecture | [XIAO_ESP32C3_Architecture](/XIAO_ESP32C3_Architecture) |
| Pin Multiplexing | [XIAO_ESP32C3_Pin_Multiplexing](/XIAO_ESP32C3_Pin_Multiplexing) |
| Power Management | [XIAO_ESP32C3_Power_Management](/XIAO_ESP32C3_Power_Management) |
| Dev Environment | [XIAO_ESP32C3_Dev_Environment](/XIAO_ESP32C3_Dev_Environment) |
| Debugging | [XIAO_ESP32C3_Debugging](/XIAO_ESP32C3_Debugging) |
| ESP32-C3 Datasheet | [PDF](https://files.seeedstudio.com/wiki/XIAO_WiFi/Resources/esp32-c3_datasheet.pdf) |
| Schematic | [PDF](https://files.seeedstudio.com/wiki/XIAO_WiFi/Resources/XIAO_ESP32C3_v1.3_SCH_260116.pdf) |
| Seeed Store | [Buy XIAO ESP32C3](https://www.seeedstudio.com/seeed-xiao-esp32c3-p-5431.html) |
| Arduino-ESP32 Docs | [Espressif Arduino](https://docs.espressif.com/projects/arduino-esp32/en/latest/) |
| ESP-IDF Docs | [ESP-IDF C3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/) |
| PlatformIO Board | [PlatformIO](https://docs.platformio.org/en/latest/boards/espressif32/seeed_xiao_esp32c3.html) |

## Tech Support & Product Discussion

Thank you for choosing our products! We are here to provide you with different support to ensure that your experience with our products is as smooth as possible. We offer several communication channels to cater to different preferences and needs.

<div class="button_tech_support_container">
<a href="https://forum.seeedstudio.com/" class="button_forum"></a>
<a href="https://www.seeedstudio.com/contacts" class="button_email"></a>
</div>

<div class="button_tech_support_container">
<a href="https://discord.gg/eWkprNDMU7" class="button_discord"></a>
<a href="https://github.com/Seeed-Studio/wiki-documents/discussions/69" class="button_discussion"></a>
</div>
