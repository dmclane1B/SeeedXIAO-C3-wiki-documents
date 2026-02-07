---
description: Power Management and Sleep Modes for Seeed Studio XIAO ESP32C3
title: Power Management
keywords:
- xiao
- esp32c3
- deep sleep
- low power
- battery
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Power_Management
last_update:
  date: 02/07/2026
  author: Claude
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Power Management

The XIAO ESP32C3 is designed for low-power IoT applications. This guide covers the ESP32-C3's sleep modes, wake-up sources, and strategies for maximizing battery life.

## Power Consumption Overview

| Mode | Typical Current | Description |
|:-----|:---------------|:------------|
| Active (Wi-Fi TX) | ~130 mA | Wi-Fi transmitting at max power |
| Active (Wi-Fi RX) | ~75 mA | Wi-Fi receiving |
| Active (CPU only) | ~30 mA | CPU at 160 MHz, radio off |
| Modem-sleep (Wi-Fi) | ~25 mA | CPU active, Wi-Fi radio off between beacons |
| Modem-sleep (BLE) | ~27 mA | CPU active, BLE radio off between events |
| Light-sleep (Wi-Fi) | ~4 mA | CPU paused, wakes for Wi-Fi beacons |
| Light-sleep (BLE) | ~10 mA | CPU paused, wakes for BLE events |
| Deep sleep | ~44 uA | Only RTC domain active |
| Hibernation | ~5 uA | Only RTC timer, no GPIO wakeup |

## Sleep Modes Explained

### Active Mode

The default operating mode. CPU runs at the configured frequency (40/80/160 MHz), all peripherals are available.

**Reduce active-mode power by lowering CPU frequency:**

```cpp
#include "esp32-hal-cpu.h"

void setup() {
    // Drop to 80 MHz — halves CPU power, most code still runs fine
    setCpuFrequencyMhz(80);
    Serial.begin(115200);
    Serial.printf("CPU Freq: %d MHz\n", getCpuFrequencyMhz());
}

void loop() {
    // Your application code
}
```

### Modem-Sleep

Automatically enabled when Wi-Fi or BLE is connected. The radio powers down between DTIM beacons (Wi-Fi) or connection events (BLE), while the CPU stays active.

No code changes required — modem-sleep is enabled by default when using `WiFi.begin()` or BLE.

You can configure the DTIM interval for deeper Wi-Fi sleep:

```cpp
#include <WiFi.h>
#include "esp_wifi.h"

void setup() {
    WiFi.begin("ssid", "password");
    while (WiFi.status() != WL_CONNECTED) delay(500);

    // Set listen interval to 10 beacons (deeper modem sleep)
    esp_wifi_set_ps(WIFI_PS_MAX_MODEM);
}
```

| Power Save Mode | Function Call | Behavior |
|:----------------|:-------------|:---------|
| None | `WIFI_PS_NONE` | Radio always on, lowest latency |
| Min Modem | `WIFI_PS_MIN_MODEM` | Sleep between DTIM beacons |
| Max Modem | `WIFI_PS_MAX_MODEM` | Longer sleep, higher latency |

### Light-Sleep

The CPU is paused and clock is gated. Peripheral states, RAM contents, and Wi-Fi/BLE connections are maintained. Wake-up is fast (~1 ms).

**Automatic light-sleep** (with Wi-Fi connection maintained):

```cpp
#include <WiFi.h>
#include "esp_wifi.h"
#include "esp_pm.h"

void setup() {
    Serial.begin(115200);
    WiFi.begin("ssid", "password");
    while (WiFi.status() != WL_CONNECTED) delay(500);

    // Enable automatic light sleep
    esp_pm_config_esp32c3_t pm_config = {
        .max_freq_mhz = 160,
        .min_freq_mhz = 40,
        .light_sleep_enable = true
    };
    esp_pm_configure(&pm_config);
}

void loop() {
    // CPU will automatically enter light-sleep when idle
    // and wake for Wi-Fi beacons or scheduled tasks
    delay(1000);
}
```

**Manual light-sleep** (for precise timing control):

```cpp
#include "esp_sleep.h"

void enterLightSleep(uint64_t sleep_us) {
    esp_sleep_enable_timer_wakeup(sleep_us);
    esp_light_sleep_start();  // Blocks until wakeup

    // Execution resumes here after wakeup
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    Serial.printf("Woke from light sleep, cause: %d\n", cause);
}

void setup() {
    Serial.begin(115200);
}

void loop() {
    Serial.println("Working...");
    delay(2000);

    Serial.println("Entering light sleep for 5 seconds...");
    Serial.flush();  // Ensure serial output completes
    enterLightSleep(5 * 1000000);  // 5 seconds in microseconds
}
```

### Deep Sleep

CPU, most RAM, and all peripherals are powered off. Only the RTC domain remains active (8KB RTC SRAM, wakeup logic). Wake-up causes a full reboot — `setup()` runs again.

**Timer Wake-Up:**

```cpp
#include "esp_sleep.h"

#define uS_TO_S_FACTOR 1000000ULL
#define TIME_TO_SLEEP  30  // seconds

RTC_DATA_ATTR int bootCount = 0;  // Survives deep sleep in RTC SRAM

void setup() {
    Serial.begin(115200);
    delay(1000);

    bootCount++;
    Serial.printf("Boot #%d\n", bootCount);

    // Print wakeup reason
    esp_sleep_wakeup_cause_t reason = esp_sleep_get_wakeup_cause();
    switch (reason) {
        case ESP_SLEEP_WAKEUP_TIMER:
            Serial.println("Wakeup: timer");
            break;
        default:
            Serial.printf("Wakeup: other (%d)\n", reason);
            break;
    }

    // Do work here (read sensor, send data, etc.)

    // Configure timer wakeup
    esp_sleep_enable_timer_wakeup(TIME_TO_SLEEP * uS_TO_S_FACTOR);

    Serial.println("Going to deep sleep...");
    Serial.flush();
    esp_deep_sleep_start();
    // Execution never reaches here
}

void loop() {
    // Never reached in deep sleep pattern
}
```

**GPIO Wake-Up:**

```cpp
#include "esp_sleep.h"

void setup() {
    Serial.begin(115200);
    delay(1000);

    Serial.println("Configuring GPIO wakeup on D1 (GPIO3), LOW trigger");

    // Enable GPIO wakeup — D0(GPIO2), D1(GPIO3), D2(GPIO4), D3(GPIO5) supported
    esp_deep_sleep_enable_gpio_wakeup(BIT(D1), ESP_GPIO_WAKEUP_GPIO_LOW);

    Serial.println("Entering deep sleep...");
    Serial.flush();
    esp_deep_sleep_start();
}

void loop() {}
```

:::caution
On the XIAO ESP32C3, only **D0 (GPIO2), D1 (GPIO3), D2 (GPIO4), and D3 (GPIO5)** support deep sleep GPIO wakeup. These are the RTC-capable GPIOs.
:::

**Multiple Wake-Up Sources:**

You can configure multiple wakeup sources simultaneously. The chip wakes on whichever triggers first:

```cpp
#include "esp_sleep.h"

void setup() {
    Serial.begin(115200);
    delay(1000);

    // Wake up on timer OR GPIO — whichever comes first
    esp_sleep_enable_timer_wakeup(60 * 1000000ULL);  // 60 seconds
    esp_deep_sleep_enable_gpio_wakeup(BIT(D1), ESP_GPIO_WAKEUP_GPIO_LOW);

    // Check what woke us
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    switch (cause) {
        case ESP_SLEEP_WAKEUP_TIMER:
            Serial.println("Woke from timer — time to report data");
            break;
        case ESP_SLEEP_WAKEUP_GPIO:
            Serial.println("Woke from button press — user interaction");
            break;
        default:
            Serial.println("First boot or reset");
            break;
    }

    // ... do work ...

    Serial.println("Going back to sleep...");
    Serial.flush();
    esp_deep_sleep_start();
}

void loop() {}
```

## RTC Data Persistence

Variables stored in RTC SRAM survive deep sleep. Use the `RTC_DATA_ATTR` attribute:

```cpp
// These persist across deep sleep cycles (8KB max total)
RTC_DATA_ATTR int bootCount = 0;
RTC_DATA_ATTR float lastTemperature = 0.0;
RTC_DATA_ATTR uint32_t errorFlags = 0;

// Regular variables are reset on each deep sleep wakeup
int normalVar = 0;  // Always 0 after deep sleep
```

:::note
RTC SRAM is limited to 8KB. Use it for small counters, flags, and critical state. For larger data, write to NVS or flash before sleeping.
:::

## Battery-Powered Design Patterns

### Pattern 1: Periodic Sensor Reporting

The most common IoT pattern — wake, read sensor, send data, sleep:

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include "esp_sleep.h"

#define SLEEP_SECONDS 300  // Report every 5 minutes

RTC_DATA_ATTR int reportCount = 0;

void setup() {
    Serial.begin(115200);
    reportCount++;

    // Read sensor
    float temperature = readSensor();

    // Connect to Wi-Fi
    WiFi.begin("ssid", "password");
    int retries = 0;
    while (WiFi.status() != WL_CONNECTED && retries < 20) {
        delay(500);
        retries++;
    }

    if (WiFi.status() == WL_CONNECTED) {
        // Send data
        HTTPClient http;
        http.begin("http://your-server.com/api/data");
        http.addHeader("Content-Type", "application/json");

        String payload = "{\"temp\":" + String(temperature) +
                         ",\"count\":" + String(reportCount) + "}";
        http.POST(payload);
        http.end();
    }

    // Disconnect Wi-Fi to save power during shutdown
    WiFi.disconnect(true);
    WiFi.mode(WIFI_OFF);

    // Sleep
    esp_sleep_enable_timer_wakeup(SLEEP_SECONDS * 1000000ULL);
    esp_deep_sleep_start();
}

void loop() {}

float readSensor() {
    // Replace with your actual sensor reading code
    return analogReadMilliVolts(A0) * 0.1;
}
```

### Pattern 2: Event-Driven with Timeout

Wake on button press (event) or after a timeout (heartbeat):

```cpp
#include "esp_sleep.h"

#define HEARTBEAT_SECONDS 3600  // 1 hour heartbeat

RTC_DATA_ATTR int eventCount = 0;

void setup() {
    Serial.begin(115200);
    delay(1000);

    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();

    if (cause == ESP_SLEEP_WAKEUP_GPIO) {
        eventCount++;
        Serial.printf("Button event #%d!\n", eventCount);
        handleButtonEvent();
    } else if (cause == ESP_SLEEP_WAKEUP_TIMER) {
        Serial.println("Heartbeat timer");
        sendHeartbeat();
    } else {
        Serial.println("First boot — initializing");
    }

    // Configure both wakeup sources
    esp_sleep_enable_timer_wakeup(HEARTBEAT_SECONDS * 1000000ULL);
    esp_deep_sleep_enable_gpio_wakeup(BIT(D1), ESP_GPIO_WAKEUP_GPIO_LOW);

    Serial.println("Sleeping...");
    Serial.flush();
    esp_deep_sleep_start();
}

void loop() {}

void handleButtonEvent() {
    // Respond to the button press
}

void sendHeartbeat() {
    // Send a periodic "I'm alive" message
}
```

## Battery Voltage Monitoring

The XIAO ESP32C3 does not have a dedicated battery voltage pin. You can add external monitoring with a resistor divider:

```
BAT+ ──[200K]──┬──[200K]── GND
                │
               A0 (GPIO2)
```

```cpp
void setup() {
    Serial.begin(115200);
    pinMode(A0, INPUT);
}

void loop() {
    uint32_t sum = 0;
    for (int i = 0; i < 16; i++) {
        sum += analogReadMilliVolts(A0);
    }
    float voltage = 2.0 * sum / 16 / 1000.0;  // Divider ratio = 1:2

    Serial.printf("Battery: %.2f V\n", voltage);

    if (voltage < 3.3) {
        Serial.println("WARNING: Battery low!");
    }

    delay(1000);
}
```

:::tip
This monitoring circuit is described in detail by community member **msfujino** in the [Seeed Studio Forum](https://forum.seeedstudio.com/t/battery-voltage-monitor-and-ad-conversion-for-xiao-esp32c/267535). Good soldering skills are required.
:::

## Power Optimization Checklist

Use this checklist to minimize power consumption in your project:

- [ ] **Lower CPU frequency** to 80 MHz or 40 MHz if processing demands are low
- [ ] **Use deep sleep** between sensor readings instead of `delay()`
- [ ] **Disconnect Wi-Fi** (`WiFi.disconnect(true); WiFi.mode(WIFI_OFF);`) before sleeping
- [ ] **Use `RTC_DATA_ATTR`** to preserve state across sleep cycles instead of flash writes
- [ ] **Minimize Wi-Fi on-time** — connect, send data, disconnect immediately
- [ ] **Use static IP** to skip DHCP negotiation and reduce Wi-Fi connection time
- [ ] **Disable unused peripherals** (BLE, ADC, etc.) when not needed
- [ ] **Use light-sleep** instead of `delay()` when waiting in active mode
- [ ] **Batch sensor readings** — wake less frequently and send more data per wake cycle
- [ ] **Monitor battery voltage** and enter a low-power state when voltage drops

### Static IP for Faster Wi-Fi Connection

DHCP negotiation can take 1-3 seconds. A static IP saves significant power in deep-sleep cycles:

```cpp
IPAddress local_IP(192, 168, 1, 200);
IPAddress gateway(192, 168, 1, 1);
IPAddress subnet(255, 255, 255, 0);
IPAddress dns(8, 8, 8, 8);

void connectWiFiFast() {
    WiFi.config(local_IP, gateway, subnet, dns);
    WiFi.begin("ssid", "password");

    // Typically connects in <500ms with static IP
    int retries = 0;
    while (WiFi.status() != WL_CONNECTED && retries < 10) {
        delay(100);
        retries++;
    }
}
```

## Estimated Battery Life Calculator

Use these rough estimates to plan your battery capacity:

| Scenario | Avg Current | 500mAh Battery Life |
|:---------|:-----------|:-------------------|
| Deep sleep only | 44 uA | ~473 days |
| Wake every 5 min, 2s Wi-Fi TX | ~0.9 mA avg | ~23 days |
| Wake every 15 min, 2s Wi-Fi TX | ~0.35 mA avg | ~59 days |
| Wake every 60 min, 2s Wi-Fi TX | ~0.12 mA avg | ~173 days |
| Light-sleep with Wi-Fi keepalive | ~4 mA | ~5 days |
| Always-on Wi-Fi (modem-sleep) | ~25 mA | ~20 hours |

:::note
These are estimates. Actual battery life depends on sensor power, Wi-Fi signal strength, data payload size, and battery self-discharge rate. Always profile with a current meter for production designs.
:::

## Further Reading

- [XIAO ESP32C3 Architecture](/XIAO_ESP32C3_Architecture) — power domains and clock tree details
- [ESP32-C3 Low Power Consumption Report](https://files.seeedstudio.com/wiki/Seeed-Studio-XIAO-ESP32/Low_Power_Consumption.pdf)
- [ESP-IDF Sleep Modes Documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-reference/system/sleep_modes.html)

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
