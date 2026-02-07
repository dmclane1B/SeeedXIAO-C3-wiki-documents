---
description: Power management and deep sleep for Seeed Studio XIAO ESP32C3
title: Power Management & Deep Sleep
keywords:
- xiao
- esp32c3
- deep sleep
- power management
- battery
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Power_Management
last_update:
  date: 02/07/2026
  author: Claude
---

# XIAO ESP32C3 Power Management & Deep Sleep

This guide covers the power modes available on the XIAO ESP32C3, how to implement deep sleep for battery-powered projects, and practical techniques for minimizing power consumption.

## Power Consumption Overview

| Mode | Typical Current | Description |
|---|---|---|
| Active (Wi-Fi TX) | ~160 mA | Wi-Fi transmitting at full power |
| Active (Wi-Fi RX) | ~95 mA | Wi-Fi receiving |
| Active (CPU only) | ~35 mA | CPU running at 160 MHz, no radio |
| Modem-sleep | ~15 mA | CPU active, Wi-Fi/BLE radio off |
| Light-sleep | ~130 uA | CPU paused, RAM retained, fast wake |
| Deep-sleep | ~44 uA | Only RTC domain powered, wake via timer/GPIO |
| Hibernation | ~5 uA | Minimal power, RTC timer wake only |

## Power Modes Explained

### Active Mode

All systems are powered. Current consumption varies depending on which peripherals and radios are active.

```cpp
// Reduce active power by lowering CPU frequency
setCpuFrequencyMhz(80);  // Options: 160, 80, 40 MHz
```

### Modem-sleep

The Wi-Fi/BLE radio is powered down while the CPU continues running. This mode is entered automatically when Wi-Fi is connected but idle (with DTIM-based wake intervals).

```cpp
#include <WiFi.h>

void setup() {
  WiFi.begin("SSID", "password");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  // Enable automatic modem sleep (default when connected)
  WiFi.setSleep(true);
}

void loop() {
  // CPU continues to run; Wi-Fi wakes periodically for beacons
  delay(1000);
}
```

### Light-sleep

The CPU is paused and most clocks are gated. RAM contents are preserved. The chip wakes quickly (< 1 ms) from GPIO, timer, or UART events.

```cpp
#include <esp_sleep.h>

void setup() {
  Serial.begin(115200);

  // Configure wake-up source: timer (5 seconds)
  esp_sleep_enable_timer_wakeup(5 * 1000000);  // microseconds

  Serial.println("Entering light sleep...");
  Serial.flush();

  // Enter light sleep
  esp_light_sleep_start();

  // Execution resumes here after wake-up
  Serial.println("Woke up from light sleep!");
}

void loop() {
  // Light sleep can be called repeatedly
  esp_light_sleep_start();
  Serial.println("Woke up again!");
  delay(100);
}
```

### Deep-sleep

Only the RTC domain remains powered. All CPU state, RAM, and peripherals are lost. On wake, the chip performs a full reboot. The 8 KB RTC SRAM can store small amounts of data across deep-sleep cycles.

```cpp
#include <esp_sleep.h>

// Store data in RTC memory (survives deep sleep)
RTC_DATA_ATTR int bootCount = 0;

void setup() {
  Serial.begin(115200);
  bootCount++;
  Serial.println("Boot count: " + String(bootCount));

  // Print wake-up reason
  print_wakeup_reason();

  // Configure wake-up: timer (30 seconds)
  esp_sleep_enable_timer_wakeup(30 * 1000000ULL);

  Serial.println("Going to deep sleep for 30 seconds...");
  Serial.flush();
  esp_deep_sleep_start();
  // Code below this line never executes
}

void loop() {
  // Never reached in deep sleep usage
}

void print_wakeup_reason() {
  esp_sleep_wakeup_cause_t reason = esp_sleep_get_wakeup_cause();
  switch (reason) {
    case ESP_SLEEP_WAKEUP_TIMER:
      Serial.println("Wake-up: Timer");
      break;
    case ESP_SLEEP_WAKEUP_GPIO:
      Serial.println("Wake-up: GPIO");
      break;
    default:
      Serial.println("Wake-up: Power on / Reset");
      break;
  }
}
```

## Wake-up Sources

### Timer Wake-up

The most common deep-sleep wake source. Uses the RTC timer for periodic wake-up.

```cpp
// Wake up every 60 seconds
esp_sleep_enable_timer_wakeup(60 * 1000000ULL);  // microseconds
esp_deep_sleep_start();
```

### GPIO Wake-up

Wake the chip when an external signal (button press, sensor interrupt) triggers a GPIO.

```cpp
#include <esp_sleep.h>

#define WAKEUP_PIN GPIO_NUM_4  // D2 on XIAO ESP32C3

void setup() {
  Serial.begin(115200);

  // Configure the GPIO as input with pull-up
  pinMode(WAKEUP_PIN, INPUT_PULLUP);

  // Wake on LOW signal (button press to GND)
  esp_deep_sleep_enable_gpio_wakeup(
    BIT(WAKEUP_PIN),
    ESP_GPIO_WAKEUP_GPIO_LOW
  );

  Serial.println("Going to deep sleep. Press button on D2 to wake...");
  Serial.flush();
  esp_deep_sleep_start();
}

void loop() {}
```

### Multiple Wake-up Sources

You can combine timer and GPIO wake-up:

```cpp
// Wake on timer OR GPIO, whichever comes first
esp_sleep_enable_timer_wakeup(300 * 1000000ULL);  // 5 minutes
esp_deep_sleep_enable_gpio_wakeup(
  BIT(GPIO_NUM_4),
  ESP_GPIO_WAKEUP_GPIO_LOW
);
esp_deep_sleep_start();
```

## Battery-Powered Project Pattern

A common pattern for battery-powered sensor nodes: wake up, read sensor, transmit data, go back to sleep.

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <esp_sleep.h>

// Persist boot count across deep sleep
RTC_DATA_ATTR int bootCount = 0;

// Configuration
const char* WIFI_SSID     = "YourSSID";
const char* WIFI_PASSWORD = "YourPassword";
const char* SERVER_URL    = "http://yourserver.com/api/sensor";
const uint64_t SLEEP_US   = 600 * 1000000ULL;  // 10 minutes

void setup() {
  Serial.begin(115200);
  bootCount++;

  // 1. Read sensor data
  float temperature = readTemperature();
  float voltage     = readBatteryVoltage();

  // 2. Connect to Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  unsigned long start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 10000) {
    delay(100);
  }

  // 3. Send data (if connected)
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(SERVER_URL);
    http.addHeader("Content-Type", "application/json");

    String payload = "{\"temp\":" + String(temperature) +
                     ",\"voltage\":" + String(voltage) +
                     ",\"boot\":" + String(bootCount) + "}";
    http.POST(payload);
    http.end();
  }

  // 4. Disconnect and sleep
  WiFi.disconnect(true);
  WiFi.mode(WIFI_OFF);

  esp_sleep_enable_timer_wakeup(SLEEP_US);
  esp_deep_sleep_start();
}

void loop() {}

float readTemperature() {
  // Replace with your sensor reading code
  // Example: read from a connected DHT22 or internal temp sensor
  return 0.0;
}

float readBatteryVoltage() {
  // XIAO ESP32C3: read battery voltage via ADC on A0
  // Voltage divider: actual_voltage = adc_reading * 2 * 3.3 / 4095
  int raw = analogRead(A0);
  return (raw * 2.0 * 3.3) / 4095.0;
}
```

## XIAO ESP32C3 Battery Connection

The XIAO ESP32C3 has battery support with built-in charge management:

- **Battery pads**: Solder a 3.7V LiPo battery to the **BAT+** and **BAT-** pads on the bottom of the board
- **Charging**: Automatic charging when USB-C is connected (380 mA fast charge / 40 mA trickle)
- **Monitoring**: Read battery voltage using ADC

:::caution
The battery pads are on the **bottom** of the XIAO ESP32C3 board. Ensure correct polarity when soldering. Reverse polarity can damage the board.
:::

## Power Optimization Tips

### 1. Reduce CPU Frequency

```cpp
setCpuFrequencyMhz(80);  // Half the default, significant power savings
```

### 2. Disable Unused Peripherals

```cpp
// Turn off Bluetooth if only using Wi-Fi
btStop();

// Turn off Wi-Fi radio completely
WiFi.mode(WIFI_OFF);
esp_wifi_stop();
```

### 3. Use Static IP to Speed Wi-Fi Connection

Reduces Wi-Fi connection time from ~2-5 seconds to ~0.5-1 second, saving power in wake-transmit-sleep cycles.

```cpp
IPAddress local_IP(192, 168, 1, 200);
IPAddress gateway(192, 168, 1, 1);
IPAddress subnet(255, 255, 255, 0);
IPAddress dns(8, 8, 8, 8);

WiFi.config(local_IP, gateway, subnet, dns);
WiFi.begin(SSID, PASSWORD);
```

### 4. Minimize Wake Time

The biggest power saving comes from spending less time awake:
- Pre-format data before connecting to Wi-Fi
- Use UDP instead of TCP/HTTP for faster transmit
- Skip Wi-Fi connection if data can be batched and sent less frequently

### 5. Disable Serial During Production

```cpp
// Serial output consumes power; disable in production
// Serial.begin(115200);  // Comment out for production
```

## Estimating Battery Life

Use this formula to estimate how long a battery will last:

```
Battery Life (hours) = Battery Capacity (mAh) / Average Current (mA)

Average Current = (Active_Time * Active_Current + Sleep_Time * Sleep_Current)
                  / (Active_Time + Sleep_Time)
```

**Example**: 500 mAh battery, wake 2 seconds every 10 minutes:
- Active: 2s at 100 mA, Sleep: 598s at 0.044 mA
- Average current = (2 * 100 + 598 * 0.044) / 600 = **0.377 mA**
- Battery life = 500 / 0.377 = **1,326 hours (~55 days)**

## Further Reading

- [Architecture Overview](/XIAO_ESP32C3_Architecture) - SoC internals and power domains
- [Getting Started](/XIAO_ESP32C3_Getting_Started) - Initial setup and hardware overview
- [WiFi Usage](/XIAO_ESP32C3_WiFi_Usage) - Wi-Fi connectivity details
