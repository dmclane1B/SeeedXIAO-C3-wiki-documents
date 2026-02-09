---
description: Debugging and troubleshooting for Seeed Studio XIAO ESP32C3
title: Debugging & Troubleshooting
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

# XIAO ESP32C3 Debugging & Troubleshooting

This guide covers debugging techniques, built-in JTAG usage, serial monitoring, and solutions for common issues when developing with the XIAO ESP32C3.

## Serial Debugging

### Basic Serial Monitor

The most common debugging method. The XIAO ESP32C3 uses a USB-to-UART bridge, so serial output appears as a COM/tty port.

```cpp
void setup() {
  Serial.begin(115200);
  while (!Serial) delay(10);  // Wait for serial port (optional)

  Serial.println("=== XIAO ESP32C3 Debug ===");
  Serial.printf("Chip Model   : %s\n", ESP.getChipModel());
  Serial.printf("Chip Revision: %d\n", ESP.getChipRevision());
  Serial.printf("CPU Frequency: %d MHz\n", getCpuFrequencyMhz());
  Serial.printf("Flash Size   : %d KB\n", ESP.getFlashChipSize() / 1024);
  Serial.printf("Free Heap    : %d bytes\n", ESP.getFreeHeap());
  Serial.printf("SDK Version  : %s\n", ESP.getSdkVersion());
}

void loop() {
  // Periodic heap monitoring
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint > 5000) {
    Serial.printf("[%lu] Free heap: %d bytes\n", millis(), ESP.getFreeHeap());
    lastPrint = millis();
  }
}
```

### Debug Levels

Enable verbose logging from the ESP32 framework to see internal Wi-Fi, BLE, and system messages.

**Arduino IDE:** Go to **Tools > Core Debug Level** and select:
- **None** - No debug output (production)
- **Error** - Errors only
- **Warn** - Warnings and errors
- **Info** - Informational messages
- **Debug** - Detailed debug output
- **Verbose** - Everything (very noisy)

**Programmatic logging:**

```cpp
// Use ESP-IDF logging macros
#include <esp_log.h>

static const char* TAG = "MyApp";

void setup() {
  Serial.begin(115200);
  esp_log_level_set("*", ESP_LOG_VERBOSE);  // Set global log level

  ESP_LOGI(TAG, "Application started");
  ESP_LOGD(TAG, "Debug message with value: %d", 42);
  ESP_LOGW(TAG, "Warning: low memory");
  ESP_LOGE(TAG, "Error: sensor not found");
}

void loop() {}
```

## Built-in USB JTAG Debugging

The ESP32-C3 has a **built-in JTAG debugger** accessible via USB, allowing step-through debugging without any external hardware.

### Hardware Setup

The JTAG pins are shared with GPIO18 and GPIO19 which are used for the USB D-/D+ on the ESP32-C3 chip. On the XIAO ESP32C3, the USB connection goes through a USB-serial bridge, so direct JTAG access requires soldering to the GPIO18/GPIO19 pads.

:::note
For most XIAO ESP32C3 development, serial debugging (print statements and log levels) is more practical than JTAG. JTAG is most useful for hard faults, deadlocks, and bare-metal debugging.
:::

### Using JTAG with PlatformIO

If you have access to the JTAG pins, configure PlatformIO:

```ini
; platformio.ini
[env:xiao_esp32c3]
platform = espressif32
board = seeed_xiao_esp32c3
framework = arduino
debug_tool = esp-builtin
debug_init_break = tbreak setup
```

Then use **Run > Start Debugging (F5)** in VS Code.

## Memory Debugging

### Heap Monitoring

Track memory usage to catch leaks:

```cpp
void printMemoryInfo() {
  Serial.println("--- Memory Info ---");
  Serial.printf("Total heap   : %d bytes\n", ESP.getHeapSize());
  Serial.printf("Free heap    : %d bytes\n", ESP.getFreeHeap());
  Serial.printf("Min free heap: %d bytes\n", ESP.getMinFreeHeap());
  Serial.printf("Max alloc    : %d bytes\n", ESP.getMaxAllocHeap());
  Serial.println("-------------------");
}
```

### Detecting Stack Overflow

FreeRTOS can detect stack overflows. Enable in `sdkconfig` or use the Arduino menu:

```cpp
// Check remaining stack space for the current task
UBaseType_t stackHighWaterMark = uxTaskGetStackHighWaterMark(NULL);
Serial.printf("Stack remaining: %d bytes\n", stackHighWaterMark * 4);
```

### Detecting Memory Leaks

```cpp
// Take a snapshot, then compare
int heapBefore = ESP.getFreeHeap();

// ... run suspected leaky code ...

int heapAfter = ESP.getFreeHeap();
int leaked = heapBefore - heapAfter;
if (leaked > 0) {
  Serial.printf("Possible leak: %d bytes\n", leaked);
}
```

## Crash Decoding

When the ESP32-C3 crashes, it prints a backtrace to Serial. Decode it to find the source of the crash.

### Reading the Crash Dump

A typical crash dump looks like:

```
Guru Meditation Error: Core  0 panic'ed (Store access fault). Exception was unhandled.
Core  0 register dump:
MEPC    : 0x42008a2c  RA      : 0x42008a28  SP      : 0x3fc93bf0
...
Stack memory:
3fc93bf0: 0x42008a28 0x3fc93c10 ...

Backtrace: 0x42008a2c:0x3fc93bf0 0x42008a28:0x3fc93c10
```

### Decoding with addr2line

```bash
# Use the RISC-V toolchain addr2line
~/.arduino15/packages/esp32/tools/riscv32-esp-elf-gdb/*/bin/riscv32-esp-elf-addr2line \
  -e /path/to/your/sketch.ino.elf \
  -f -C \
  0x42008a2c 0x42008a28
```

This outputs the function name and source line for each address.

### ESP Exception Decoder (Arduino IDE)

Install the **ESP Exception Decoder** plugin for Arduino IDE to automatically decode crash dumps from the Serial Monitor.

## Wi-Fi Debugging

### Connection Issues

```cpp
#include <WiFi.h>

void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);
  WiFi.begin("SSID", "password");

  Serial.println("Connecting to Wi-Fi...");
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    attempts++;
    Serial.printf("Attempt %d, status: %d\n", attempts, WiFi.status());

    if (attempts > 20) {
      Serial.println("Connection failed. Status codes:");
      Serial.println("0=IDLE, 1=NO_SSID_AVAIL, 2=SCAN_COMPLETED");
      Serial.println("3=CONNECTED, 4=CONNECT_FAILED, 5=CONNECTION_LOST");
      Serial.println("6=DISCONNECTED");
      break;
    }
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("Connected!");
    Serial.printf("IP     : %s\n", WiFi.localIP().toString().c_str());
    Serial.printf("RSSI   : %d dBm\n", WiFi.RSSI());
    Serial.printf("Channel: %d\n", WiFi.channel());
  }
}

void loop() {}
```

### Signal Strength Monitor

```cpp
void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    int rssi = WiFi.RSSI();
    String quality;
    if (rssi > -50) quality = "Excellent";
    else if (rssi > -60) quality = "Good";
    else if (rssi > -70) quality = "Fair";
    else quality = "Weak";

    Serial.printf("RSSI: %d dBm (%s)\n", rssi, quality.c_str());
  }
  delay(2000);
}
```

## Common Issues & Solutions

### Board Not Detected by Computer

| Cause | Solution |
|---|---|
| Bad USB cable | Use a cable that supports data transfer (not charge-only) |
| Driver missing | Install CH340/CH9102 USB-serial drivers |
| Board in deep sleep | Press RESET button to wake |
| Firmware crash loop | Hold BOOT + press RESET to enter download mode |

### Enter Download Mode (Recovery)

If your firmware is crashing and you cannot upload new code:

1. **Hold** the BOOT button (small button near USB connector)
2. While holding BOOT, **press and release** the RESET button
3. **Release** the BOOT button
4. The board is now in download mode -- upload your sketch

### Upload Fails with Timeout

```
A fatal error occurred: Failed to connect to ESP32-C3: Timed out waiting for packet header
```

**Solutions:**
- Enter download mode manually (BOOT + RESET sequence above)
- Try a different USB port (prefer direct USB, not via hub)
- Lower upload speed in Arduino IDE: **Tools > Upload Speed > 115200**
- Close any Serial Monitor before uploading

### Brownout / Unexpected Resets

If you see `Brownout detector was triggered` in the serial output:

- **Cause**: Insufficient power supply, especially during Wi-Fi TX spikes (~300 mA)
- **Solutions**:
  - Use a quality USB cable and port (not a weak hub)
  - Add a 100 uF capacitor between 3.3V and GND
  - For battery operation, use a battery that can supply >500 mA peak

### Wi-Fi Won't Connect

- Ensure the **external antenna** is connected to the IPEX connector
- The ESP32-C3 only supports **2.4 GHz** networks (not 5 GHz)
- Check SSID and password (case-sensitive)
- Some enterprise networks (802.1X) require special configuration

### GPIO Conflicts

Some GPIOs have special functions during boot:

| GPIO | Boot Function | Caution |
|---|---|---|
| GPIO2 | Must be floating or LOW during boot | Avoid strong pull-up |
| GPIO8 | Must be HIGH during boot (has internal pull-up) | Avoid pulling LOW at boot |
| GPIO9 | Boot mode select (BOOT button) | HIGH = normal, LOW = download |

### Sketch Too Large

```
Sketch too big; see https://...
```

**Solutions:**
- Use **Tools > Partition Scheme > No OTA (2MB APP / 2MB SPIFFS)** for larger sketches
- Optimize code: remove unused libraries, use `F()` macro for string literals
- Move large data to SPIFFS/LittleFS instead of embedding in code

## Useful Debug Utilities

### System Info Dump

```cpp
void printSystemInfo() {
  Serial.println("====== System Info ======");
  Serial.printf("Chip      : %s Rev %d\n", ESP.getChipModel(), ESP.getChipRevision());
  Serial.printf("CPU Cores : %d @ %d MHz\n", ESP.getChipCores(), getCpuFrequencyMhz());
  Serial.printf("Flash     : %d KB (%d MHz)\n", ESP.getFlashChipSize() / 1024, ESP.getFlashChipSpeed() / 1000000);
  Serial.printf("Heap      : %d / %d bytes free\n", ESP.getFreeHeap(), ESP.getHeapSize());
  Serial.printf("PSRAM     : %s\n", ESP.getPsramSize() > 0 ? "Yes" : "No");
  Serial.printf("SDK       : %s\n", ESP.getSdkVersion());
  Serial.printf("MAC       : %s\n", WiFi.macAddress().c_str());
  Serial.println("========================");
}
```

### Watchdog Timer Reset Detection

```cpp
void setup() {
  Serial.begin(115200);

  esp_reset_reason_t reason = esp_reset_reason();
  switch (reason) {
    case ESP_RST_POWERON:  Serial.println("Power-on reset"); break;
    case ESP_RST_SW:       Serial.println("Software reset"); break;
    case ESP_RST_PANIC:    Serial.println("Exception/panic reset"); break;
    case ESP_RST_INT_WDT:  Serial.println("Interrupt watchdog reset"); break;
    case ESP_RST_TASK_WDT: Serial.println("Task watchdog reset"); break;
    case ESP_RST_WDT:      Serial.println("Other watchdog reset"); break;
    case ESP_RST_DEEPSLEEP: Serial.println("Deep-sleep wake"); break;
    case ESP_RST_BROWNOUT: Serial.println("Brownout reset"); break;
    default:               Serial.println("Unknown reset reason"); break;
  }
}

void loop() {}
```

## Further Reading

- [Getting Started](/XIAO_ESP32C3_Getting_Started) - Initial setup
- [Architecture Overview](/XIAO_ESP32C3_Architecture) - Boot process, memory map
- [Pin Multiplexing](/XIAO_ESP32C3_Pin_Multiplexing) - GPIO usage and pin functions
