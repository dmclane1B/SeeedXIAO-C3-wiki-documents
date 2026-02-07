---
description: Debugging and Troubleshooting for Seeed Studio XIAO ESP32C3
title: Debugging and Troubleshooting
keywords:
- xiao
- esp32c3
- debugging
- jtag
- troubleshooting
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Debugging
last_update:
  date: 02/07/2026
  author: Claude
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Debugging and Troubleshooting

This guide covers debugging techniques for the XIAO ESP32C3, from basic serial debugging to advanced JTAG debugging, plus solutions to common issues.

## Serial Debugging

Serial output is the most accessible debugging method and works with all development environments.

### Basic Serial Debug Output

```cpp
void setup() {
    Serial.begin(115200);
    while (!Serial) { delay(10); }  // Wait for USB serial

    Serial.println("=== XIAO ESP32C3 Debug ===");
    Serial.printf("CPU Freq: %d MHz\n", getCpuFrequencyMhz());
    Serial.printf("Free Heap: %d bytes\n", ESP.getFreeHeap());
    Serial.printf("Flash Size: %d bytes\n", ESP.getFlashChipSize());
    Serial.printf("SDK Version: %s\n", ESP.getSdkVersion());
}

void loop() {
    // Periodic health check
    static unsigned long lastReport = 0;
    if (millis() - lastReport > 10000) {
        Serial.printf("[%lu] Heap free: %d | Min free: %d\n",
            millis(),
            ESP.getFreeHeap(),
            ESP.getMinFreeHeap());
        lastReport = millis();
    }
}
```

### Arduino Core Debug Levels

Set via **Tools > Core Debug Level** in Arduino IDE, or in `platformio.ini`:

```ini
build_flags = -DCORE_DEBUG_LEVEL=5
```

| Level | Name | Output |
|:------|:-----|:-------|
| 0 | None | No debug output |
| 1 | Error | Critical errors only |
| 2 | Warn | Warnings and errors |
| 3 | Info | Informational messages |
| 4 | Debug | Detailed debug messages |
| 5 | Verbose | Everything, including Wi-Fi internals |

### ESP-IDF Logging

When using ESP-IDF (directly or through Arduino), use the structured logging API:

```cpp
#include "esp_log.h"

static const char *TAG = "my_app";

void setup() {
    Serial.begin(115200);

    ESP_LOGE(TAG, "This is an error message");
    ESP_LOGW(TAG, "This is a warning message");
    ESP_LOGI(TAG, "This is an info message");
    ESP_LOGD(TAG, "This is a debug message");
    ESP_LOGV(TAG, "This is a verbose message");

    // Set log level per component
    esp_log_level_set("wifi", ESP_LOG_WARN);     // Quiet Wi-Fi logs
    esp_log_level_set("my_app", ESP_LOG_DEBUG);   // Verbose for your code
}
```

## JTAG Debugging

The ESP32-C3 has a built-in USB JTAG interface that enables hardware debugging with breakpoints, variable inspection, and single-stepping.

### JTAG Pin Mapping

The ESP32-C3 exposes JTAG signals on these pins:

| JTAG Signal | ESP32-C3 GPIO | XIAO Pin |
|:------------|:-------------|:---------|
| MTMS (TMS) | GPIO4 | D2 |
| MTDI (TDI) | GPIO5 | D3 |
| MTCK (TCK) | GPIO6 | D4 (SDA) |
| MTDO (TDO) | GPIO7 | D5 (SCL) |

:::caution
JTAG shares pins with I2C (SDA/SCL) and ADC (A2/A3). You cannot use I2C and JTAG simultaneously. Disconnect any I2C devices before JTAG debugging.
:::

### Built-in USB JTAG (Recommended)

The ESP32-C3 has a built-in USB JTAG/serial controller on its native USB pins (GPIO18/GPIO19). The XIAO ESP32C3's USB-C connector connects to these pins, so you can use JTAG debugging over the same USB cable used for programming.

#### Setup with PlatformIO

Add to `platformio.ini`:

```ini
[env:seeed_xiao_esp32c3]
platform = espressif32
board = seeed_xiao_esp32c3
framework = arduino

; Enable JTAG debugging via built-in USB
debug_tool = esp-builtin
debug_init_break = tbreak setup
debug_speed = 40000
```

Then click the **Debug** icon in VS Code's sidebar, or press `F5` to start debugging.

#### Setup with ESP-IDF + OpenOCD

```bash
# Start OpenOCD (in one terminal)
openocd -f board/esp32c3-builtin.cfg

# Connect GDB (in another terminal)
riscv32-esp-elf-gdb -x gdbinit build/my_project.elf
```

Example `gdbinit`:

```
target extended-remote :3333
set remote hardware-breakpoint-limit 8
monitor reset halt
thb app_main
continue
```

### What You Can Do with JTAG

- **Set breakpoints** — pause execution at specific lines
- **Step through code** — line by line or instruction by instruction
- **Inspect variables** — view current values at any breakpoint
- **View call stack** — trace how execution reached the current point
- **View memory** — read/write raw memory addresses
- **View peripheral registers** — inspect hardware register states

### Hardware Breakpoint Limits

The ESP32-C3 supports up to **8 hardware breakpoints**. Software breakpoints are not available for code running from flash (only from IRAM). If you need more breakpoints, place frequently debugged functions in IRAM:

```cpp
void IRAM_ATTR myDebugFunction() {
    // This function can use software breakpoints
}
```

## Memory Debugging

### Heap Monitoring

Track memory leaks and fragmentation:

```cpp
void printHeapInfo() {
    Serial.printf("Free Heap:       %d bytes\n", ESP.getFreeHeap());
    Serial.printf("Min Free Heap:   %d bytes\n", ESP.getMinFreeHeap());
    Serial.printf("Max Alloc Heap:  %d bytes\n", ESP.getMaxAllocHeap());
    Serial.printf("PSRAM Free:      %d bytes\n", ESP.getFreePsram());

    // Detailed multi-heap info (ESP-IDF)
    multi_heap_info_t info;
    heap_caps_get_info(&info, MALLOC_CAP_8BIT);
    Serial.printf("Total free:      %d\n", info.total_free_bytes);
    Serial.printf("Total allocated: %d\n", info.total_allocated_bytes);
    Serial.printf("Largest block:   %d\n", info.largest_free_block);
}
```

### Stack Overflow Detection

FreeRTOS can detect stack overflows. Enable in `menuconfig` or add to your code:

```cpp
// Check remaining stack space for current task
UBaseType_t stackHighWaterMark = uxTaskGetStackHighWaterMark(NULL);
Serial.printf("Stack remaining: %d bytes\n", stackHighWaterMark * 4);
```

If you see guru meditation errors mentioning "Stack canary watchpoint triggered," increase the stack size for the offending task.

### Task Monitoring

View all running FreeRTOS tasks and their status:

```cpp
void printTaskList() {
    char buffer[2048];
    vTaskList(buffer);
    Serial.println("Task Name\tState\tPrio\tStack\tNum");
    Serial.println(buffer);
}
```

:::note
Enable `CONFIG_FREERTOS_USE_TRACE_FACILITY` and `CONFIG_FREERTOS_USE_STATS_FORMATTING_FUNCTIONS` in menuconfig (ESP-IDF) to use `vTaskList()`.
:::

## Wi-Fi Debugging

### Connection Diagnostics

```cpp
#include <WiFi.h>

void printWiFiDiag() {
    Serial.printf("Status:  %d\n", WiFi.status());
    Serial.printf("SSID:    %s\n", WiFi.SSID().c_str());
    Serial.printf("BSSID:   %s\n", WiFi.BSSIDstr().c_str());
    Serial.printf("RSSI:    %d dBm\n", WiFi.RSSI());
    Serial.printf("Channel: %d\n", WiFi.channel());
    Serial.printf("IP:      %s\n", WiFi.localIP().toString().c_str());
    Serial.printf("Gateway: %s\n", WiFi.gatewayIP().toString().c_str());
    Serial.printf("DNS:     %s\n", WiFi.dnsIP().toString().c_str());
}
```

### Wi-Fi Status Codes

| WiFi.status() | Meaning |
|:--------------|:--------|
| WL_IDLE_STATUS (0) | Initializing |
| WL_NO_SSID_AVAIL (1) | SSID not found |
| WL_SCAN_COMPLETED (2) | Scan complete |
| WL_CONNECTED (3) | Connected |
| WL_CONNECT_FAILED (4) | Connection failed |
| WL_CONNECTION_LOST (5) | Connection lost |
| WL_DISCONNECTED (6) | Disconnected |

### Wi-Fi Event Callbacks

Register event handlers for detailed connection monitoring:

```cpp
void WiFiEvent(WiFiEvent_t event) {
    switch (event) {
        case ARDUINO_EVENT_WIFI_STA_START:
            Serial.println("[WiFi] Station started");
            break;
        case ARDUINO_EVENT_WIFI_STA_CONNECTED:
            Serial.println("[WiFi] Connected to AP");
            break;
        case ARDUINO_EVENT_WIFI_STA_GOT_IP:
            Serial.printf("[WiFi] Got IP: %s\n",
                WiFi.localIP().toString().c_str());
            break;
        case ARDUINO_EVENT_WIFI_STA_DISCONNECTED:
            Serial.println("[WiFi] Disconnected");
            break;
        default:
            Serial.printf("[WiFi] Event: %d\n", event);
            break;
    }
}

void setup() {
    Serial.begin(115200);
    WiFi.onEvent(WiFiEvent);
    WiFi.begin("ssid", "password");
}
```

## Crash Analysis

### Decoding Guru Meditation Errors

When the ESP32-C3 crashes, it outputs a backtrace like:

```
Guru Meditation Error: Core  0 panic'ed (Store access fault). Exception was unhandled.
Core  0 register dump:
MEPC    : 0x42005678  RA      : 0x42005432 ...
Stack memory:
...
Backtrace: 0x42005678:0x3fc8a210 0x42005432:0x3fc8a230
```

#### Decoding with addr2line

```bash
# Use the RISC-V toolchain addr2line
riscv32-esp-elf-addr2line -pfiaC -e .pio/build/seeed_xiao_esp32c3/firmware.elf \
    0x42005678 0x42005432
```

This outputs the source file and line number for each address.

#### Using ESP Exception Decoder (Arduino IDE)

Install the [ESP Exception Decoder](https://github.com/me-no-dev/EspExceptionDecoder) plugin for Arduino IDE. Copy the crash output into the decoder window to get human-readable stack traces.

### Common Crash Types

| Panic Reason | Likely Cause |
|:-------------|:-------------|
| Store access fault | Writing to invalid/null pointer |
| Load access fault | Reading from invalid/null pointer |
| Illegal instruction | Corrupted code, stack overflow |
| Stack canary watchpoint | Stack overflow in a FreeRTOS task |
| Interrupt wdt timeout | ISR running too long, or system deadlock |
| Task watchdog timeout | A task hogging CPU without yielding |

## Common Issues and Solutions

### Issue: Upload Fails / Board Not Detected

**Symptoms:** "Failed to connect", "No serial port found"

**Solution:**
1. Hold the **BOOT** button
2. While holding, press and release **RESET** (or reconnect USB)
3. Release **BOOT** — board is now in download mode
4. Retry the upload

If the port still doesn't appear:
- Try a different USB cable (must support data, not charge-only)
- On Linux: `sudo usermod -aG dialout $USER` (then log out/in)
- On macOS: Check System Preferences > Security for blocked drivers

### Issue: Wi-Fi Fails to Connect

**Common causes:**
- Antenna not connected — attach the external IPEX antenna
- Wrong credentials — double-check SSID and password (case-sensitive)
- 5 GHz network — ESP32-C3 only supports 2.4 GHz
- Too many retries — add a timeout and reset Wi-Fi:

```cpp
WiFi.disconnect(true);
WiFi.mode(WIFI_OFF);
delay(1000);
WiFi.mode(WIFI_STA);
WiFi.begin(ssid, password);
```

### Issue: Analog Readings on A3 (GPIO5) Are Erratic

GPIO5 uses ADC2, which is unreliable when Wi-Fi is active. Use A0 (GPIO2), A1 (GPIO3), or A2 (GPIO4) instead — these use ADC1.

### Issue: GPIO8 Causes Boot Failure

GPIO8 is a strapping pin. If pulled LOW externally during boot (with GPIO9 also LOW), the chip enters an undefined state. Add a **10K pull-up resistor** to GPIO8 if using it as an output.

### Issue: Random Reboots / Watchdog Resets

- **Task watchdog:** A `loop()` or task runs for too long without yielding. Add `delay(1)` or `vTaskDelay(1)` in tight loops.
- **Interrupt watchdog:** An ISR takes too long. Keep ISR handlers short — set a flag and process in `loop()`.
- **Brownout:** Insufficient power supply. Use a quality USB cable and ensure the power source can deliver 500mA+. Disable brownout detection only for debugging:

```cpp
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"

void setup() {
    WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);  // Debug only!
}
```

### Issue: Serial Output Is Garbled

- Verify baud rate matches between code (`Serial.begin(115200)`) and serial monitor
- Ensure **USB CDC On Boot** is set to **Enabled** in board settings (for USB serial)
- Boot messages from the ROM bootloader are always at 115200 baud

### Issue: Flash Is Full / OTA Partition Too Small

Change the partition scheme:
- Arduino: **Tools > Partition Scheme > Huge APP (3MB No OTA)**
- PlatformIO: `board_build.partitions = huge_app.csv`

Or create a custom partition table. See [Development Environment](/XIAO_ESP32C3_Dev_Environment) for details.

### Issue: Deep Sleep Current Is Higher Than Expected

- Ensure `WiFi.disconnect(true)` and `WiFi.mode(WIFI_OFF)` are called before sleep
- Disconnect external peripherals or put them in their own sleep mode
- Check for floating GPIO pins — configure unused pins as inputs with pull-down:

```cpp
// Before deep sleep, set unused pins to avoid leakage current
gpio_set_direction(GPIO_NUM_2, GPIO_MODE_INPUT);
gpio_pulldown_en(GPIO_NUM_2);
gpio_pullup_dis(GPIO_NUM_2);
```

## Diagnostic Sketch

Upload this comprehensive diagnostic sketch to verify your XIAO ESP32C3 hardware:

```cpp
#include <WiFi.h>
#include "esp_system.h"
#include "esp_chip_info.h"

void setup() {
    Serial.begin(115200);
    while (!Serial) { delay(10); }
    delay(1000);

    Serial.println("\n=== XIAO ESP32C3 Hardware Diagnostic ===\n");

    // Chip info
    esp_chip_info_t chip;
    esp_chip_info(&chip);
    Serial.printf("Chip:        ESP32-C3 rev %d\n", chip.revision);
    Serial.printf("Cores:       %d\n", chip.cores);
    Serial.printf("Features:    WiFi%s%s\n",
        (chip.features & CHIP_FEATURE_BLE) ? " BLE" : "",
        (chip.features & CHIP_FEATURE_IEEE802154) ? " 802.15.4" : "");
    Serial.printf("Flash:       %d MB (%s)\n",
        spi_flash_get_chip_size() / (1024 * 1024),
        (chip.features & CHIP_FEATURE_EMB_FLASH) ? "embedded" : "external");

    // Memory
    Serial.printf("\nFree Heap:   %d bytes\n", ESP.getFreeHeap());
    Serial.printf("Min Heap:    %d bytes\n", ESP.getMinFreeHeap());
    Serial.printf("Max Alloc:   %d bytes\n", ESP.getMaxAllocHeap());

    // CPU
    Serial.printf("\nCPU Freq:    %d MHz\n", getCpuFrequencyMhz());
    Serial.printf("SDK:         %s\n", ESP.getSdkVersion());

    // MAC addresses
    Serial.printf("\nWiFi MAC:    %s\n", WiFi.macAddress().c_str());

    // Scan GPIO pin states
    Serial.println("\nGPIO States:");
    int gpios[] = {2, 3, 4, 5, 6, 7, 8, 9, 10, 20, 21};
    const char* names[] = {"D0","D1","D2","D3","D4","D5","D8","D9","D10","D7","D6"};
    for (int i = 0; i < 11; i++) {
        pinMode(gpios[i], INPUT);
        Serial.printf("  %s (GPIO%d): %s\n", names[i], gpios[i],
            digitalRead(gpios[i]) ? "HIGH" : "LOW");
    }

    // ADC readings
    Serial.println("\nADC Readings (mV):");
    Serial.printf("  A0 (GPIO2): %d mV\n", analogReadMilliVolts(A0));
    Serial.printf("  A1 (GPIO3): %d mV\n", analogReadMilliVolts(A1));
    Serial.printf("  A2 (GPIO4): %d mV\n", analogReadMilliVolts(A2));

    // Wi-Fi scan
    Serial.println("\nWi-Fi Scan:");
    WiFi.mode(WIFI_STA);
    WiFi.disconnect();
    delay(100);
    int n = WiFi.scanNetworks();
    Serial.printf("  Found %d networks\n", n);
    for (int i = 0; i < min(n, 5); i++) {
        Serial.printf("  %d: %-32s %d dBm\n", i+1,
            WiFi.SSID(i).c_str(), WiFi.RSSI(i));
    }

    Serial.println("\n=== Diagnostic Complete ===");
}

void loop() {
    delay(10000);
    Serial.printf("[%lu] Heap: %d bytes\n", millis(), ESP.getFreeHeap());
}
```

## Further Reading

- [XIAO ESP32C3 Architecture](/XIAO_ESP32C3_Architecture) — understanding the hardware you're debugging
- [XIAO ESP32C3 Quick Reference](/XIAO_ESP32C3_Quick_Reference) — pin maps and code patterns
- [ESP-IDF Debugging Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-guides/jtag-debugging/)
- [PlatformIO Debugging Documentation](https://docs.platformio.org/en/latest/plus/debugging.html)

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
