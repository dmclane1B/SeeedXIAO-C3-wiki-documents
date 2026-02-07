---
description: Development Environment Setup for Seeed Studio XIAO ESP32C3
title: Development Environment Setup
keywords:
- xiao
- esp32c3
- esp-idf
- platformio
- arduino
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Dev_Environment
last_update:
  date: 02/07/2026
  author: Claude
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Development Environment Setup

This guide covers setting up the three main development environments for the XIAO ESP32C3: **Arduino IDE**, **ESP-IDF**, and **PlatformIO**. While the [Getting Started](/XIAO_ESP32C3_Getting_Started) guide covers basic Arduino setup, this page provides deeper configuration for all three frameworks.

## Environment Comparison

| Feature | Arduino IDE | PlatformIO | ESP-IDF |
|:--------|:-----------|:-----------|:--------|
| Ease of Use | Easiest | Moderate | Advanced |
| Library Ecosystem | Large (Arduino + ESP32) | Large (Arduino + native) | ESP-IDF components |
| Build System | Arduino CLI | SCons + CMake | CMake + Ninja |
| Debugging | Serial only | JTAG + Serial | JTAG + Serial + GDB |
| RTOS Access | Limited (runs on FreeRTOS) | Full | Full |
| Partition Control | Presets | Full custom | Full custom |
| Recommended For | Prototyping, beginners | Production, mixed projects | Low-level, production |

## Arduino IDE (Detailed Setup)

For basic setup, see [Getting Started](/XIAO_ESP32C3_Getting_Started). This section covers advanced Arduino configuration.

### Board Configuration Options

After selecting **XIAO_ESP32C3** from the board menu, configure these options under **Tools**:

| Option | Recommended Setting | Notes |
|:-------|:-------------------|:------|
| Upload Speed | 921600 | Fastest reliable upload speed |
| CPU Frequency | 160MHz | Default. Use 80MHz for lower power |
| Flash Frequency | 80MHz | Default |
| Flash Mode | QIO | Default, fastest |
| Flash Size | 4MB | Matches on-board flash |
| Partition Scheme | Default 4MB with spiffs | Use "Huge APP" for large sketches |
| Core Debug Level | None | Set to "Verbose" when debugging |
| USB CDC On Boot | Enabled | Disable if using UART0 for serial |

### Custom Partition Tables

For projects that need more app space or a different filesystem layout, create a custom partition table.

- **Step 1.** Create a file named `partitions.csv` in your sketch folder:

```csv
# Name,    Type, SubType, Offset,   Size,    Flags
nvs,       data, nvs,     0x9000,   0x5000,
otadata,   data, ota,     0xe000,   0x2000,
app0,      app,  ota_0,   0x10000,  0x1E0000,
spiffs,    data, spiffs,  0x1F0000, 0x200000,
```

- **Step 2.** In Arduino IDE, select **Tools > Partition Scheme > Custom** (if available), or manually reference the CSV in your `boards.local.txt`.

### Using Arduino CLI

For scripted builds and CI/CD pipelines:

```bash
# Install ESP32 core
arduino-cli core install esp32:esp32

# Compile a sketch
arduino-cli compile --fqbn esp32:esp32:XIAO_ESP32C3 MySketch/

# Upload
arduino-cli upload -p /dev/ttyACM0 --fqbn esp32:esp32:XIAO_ESP32C3 MySketch/
```

## ESP-IDF Setup

ESP-IDF is Espressif's official development framework. It gives full access to all ESP32-C3 features including FreeRTOS configuration, partition management, and low-level peripheral control.

### Prerequisites

- Python 3.8+
- Git
- CMake 3.16+
- Ninja build system

### Installation

<Tabs>
  <TabItem value="linux" label="Linux" default>

```bash
# Install dependencies
sudo apt-get install git wget flex bison gperf python3 python3-pip \
    python3-venv cmake ninja-build ccache libffi-dev libssl-dev \
    dfu-util libusb-1.0-0

# Clone ESP-IDF (v5.x recommended)
mkdir -p ~/esp
cd ~/esp
git clone -b v5.4 --recursive https://github.com/espressif/esp-idf.git

# Run the install script for ESP32-C3
cd esp-idf
./install.sh esp32c3

# Set up environment variables (add to .bashrc for persistence)
. ./export.sh
```

  </TabItem>
  <TabItem value="macos" label="macOS">

```bash
# Install dependencies via Homebrew
brew install cmake ninja dfu-util python3

# Clone ESP-IDF
mkdir -p ~/esp
cd ~/esp
git clone -b v5.4 --recursive https://github.com/espressif/esp-idf.git

# Install toolchain for ESP32-C3
cd esp-idf
./install.sh esp32c3

# Set up environment variables
. ./export.sh
```

  </TabItem>
  <TabItem value="windows" label="Windows">

Download and run the [ESP-IDF Tools Installer](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/get-started/windows-setup.html) for the simplest setup on Windows.

Alternatively, use the command line:

```powershell
# Clone ESP-IDF
mkdir %USERPROFILE%\esp
cd %USERPROFILE%\esp
git clone -b v5.4 --recursive https://github.com/espressif/esp-idf.git

# Install toolchain
cd esp-idf
install.bat esp32c3

# Set up environment
export.bat
```

  </TabItem>
</Tabs>

### Creating a Project

```bash
# Copy the hello_world example as a starting point
cp -r $IDF_PATH/examples/get-started/hello_world ~/esp/my_xiao_project
cd ~/esp/my_xiao_project

# Set the target to ESP32-C3
idf.py set-target esp32c3

# Open the configuration menu
idf.py menuconfig
```

### Key menuconfig Settings for XIAO ESP32C3

Navigate through the menuconfig menus to set:

- **Serial flasher config > Flash size**: 4 MB
- **Partition Table > Partition Table**: Custom (if needed)
- **Component config > ESP32C3-Specific**: CPU frequency 160 MHz
- **Component config > Wi-Fi**: Configure Wi-Fi memory and features
- **Component config > Bluetooth**: Enable BLE if needed

### Build, Flash, and Monitor

```bash
# Build the project
idf.py build

# Flash to XIAO ESP32C3 (auto-detects port)
idf.py -p /dev/ttyACM0 flash

# Open serial monitor
idf.py -p /dev/ttyACM0 monitor

# Or do all three at once
idf.py -p /dev/ttyACM0 flash monitor
```

:::tip
If the port is not detected, hold the **BOOT** button, plug in the USB cable, then release. This enters download mode. After flashing, press **RESET** to run the application.
:::

### Example: Blink with ESP-IDF

Create `main/main.c`:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

#define BLINK_GPIO GPIO_NUM_10  // D10 on XIAO ESP32C3

void app_main(void)
{
    gpio_reset_pin(BLINK_GPIO);
    gpio_set_direction(BLINK_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(BLINK_GPIO, 1);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
        gpio_set_level(BLINK_GPIO, 0);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}
```

## PlatformIO Setup

PlatformIO is a professional embedded development platform that integrates well with VS Code. It supports Arduino framework and ESP-IDF natively.

### Installation

- **Step 1.** Install [VS Code](https://code.visualstudio.com/)
- **Step 2.** Open VS Code, go to **Extensions** (Ctrl+Shift+X), search for **PlatformIO IDE**, and install it
- **Step 3.** Restart VS Code when prompted

### Creating a XIAO ESP32C3 Project

- **Step 1.** Click the PlatformIO icon in the sidebar, then **New Project**
- **Step 2.** Set the following:
  - **Board**: `Seeed Studio XIAO ESP32C3`
  - **Framework**: `Arduino` (or `ESP-IDF`)
  - **Location**: your preferred path

This generates a `platformio.ini` configuration file.

### platformio.ini Configuration

<Tabs>
  <TabItem value="arduino-fw" label="Arduino Framework" default>

```ini
[env:seeed_xiao_esp32c3]
platform = espressif32
board = seeed_xiao_esp32c3
framework = arduino
monitor_speed = 115200
upload_speed = 921600

; Optional: custom partition table
; board_build.partitions = partitions.csv

; Optional: filesystem support
; board_build.filesystem = littlefs

; Optional: library dependencies
lib_deps =
    adafruit/Adafruit NeoPixel@^1.12.0
    bblanchon/ArduinoJson@^7.0.0
```

  </TabItem>
  <TabItem value="espidf-fw" label="ESP-IDF Framework">

```ini
[env:seeed_xiao_esp32c3]
platform = espressif32
board = seeed_xiao_esp32c3
framework = espidf
monitor_speed = 115200

; ESP-IDF specific settings
board_build.cmake_extra_args =
    -DSDKCONFIG_DEFAULTS="sdkconfig.defaults"
```

  </TabItem>
</Tabs>

### Build and Upload

Use the PlatformIO toolbar at the bottom of VS Code:
- **Build**: Click the checkmark icon (or `Ctrl+Alt+B`)
- **Upload**: Click the arrow icon (or `Ctrl+Alt+U`)
- **Serial Monitor**: Click the plug icon (or `Ctrl+Alt+S`)

Or use the terminal:

```bash
# Build
pio run

# Upload
pio run --target upload

# Monitor
pio device monitor

# Build + Upload + Monitor
pio run --target upload && pio device monitor
```

### PlatformIO Debugging

PlatformIO supports JTAG debugging with the ESP32-C3. Add to `platformio.ini`:

```ini
debug_tool = esp-builtin
debug_init_break = tbreak setup
```

See the [Debugging Guide](/XIAO_ESP32C3_Debugging) for detailed JTAG setup.

## File System Upload

All three environments support uploading files to the XIAO ESP32C3's SPIFFS or LittleFS partition.

<Tabs>
  <TabItem value="platformio-fs" label="PlatformIO" default>

Place files in the `data/` folder of your project, then:

```bash
pio run --target uploadfs
```

  </TabItem>
  <TabItem value="arduino-fs" label="Arduino IDE">

Install the [Arduino ESP32 LittleFS Uploader Plugin](https://github.com/lorol/arduino-esp32littlefs-plugin), place files in a `data/` folder next to your sketch, and use **Tools > ESP32 LittleFS Data Upload**.

  </TabItem>
  <TabItem value="espidf-fs" label="ESP-IDF">

```bash
# Generate and flash a SPIFFS image
python $IDF_PATH/components/spiffs/spiffsgen.py \
    0x200000 data_dir spiffs.bin
esptool.py --port /dev/ttyACM0 write_flash 0x290000 spiffs.bin
```

  </TabItem>
</Tabs>

## Environment-Specific GPIO Naming

Pin references differ between frameworks:

| Board Pin | Arduino | ESP-IDF | PlatformIO (Arduino) |
|:----------|:--------|:--------|:---------------------|
| D0 | `D0` or `2` | `GPIO_NUM_2` | `D0` or `2` |
| D1 | `D1` or `3` | `GPIO_NUM_3` | `D1` or `3` |
| D2 | `D2` or `4` | `GPIO_NUM_4` | `D2` or `4` |
| D3 | `D3` or `5` | `GPIO_NUM_5` | `D3` or `5` |
| D4 (SDA) | `D4` or `6` | `GPIO_NUM_6` | `D4` or `6` |
| D5 (SCL) | `D5` or `7` | `GPIO_NUM_7` | `D5` or `7` |
| D6 (TX) | `D6` or `21` | `GPIO_NUM_21` | `D6` or `21` |
| D7 (RX) | `D7` or `20` | `GPIO_NUM_20` | `D7` or `20` |
| D8 (SCK) | `D8` or `8` | `GPIO_NUM_8` | `D8` or `8` |
| D9 (MISO) | `D9` or `9` | `GPIO_NUM_9` | `D9` or `9` |
| D10 (MOSI) | `D10` or `10` | `GPIO_NUM_10` | `D10` or `10` |

## Troubleshooting

### Upload Fails with "Connection Timeout"

1. Hold the **BOOT** button
2. While holding BOOT, press and release **RESET** (or unplug/replug USB)
3. Release BOOT
4. Retry the upload

### Port Not Detected

- Verify you are using a USB cable that supports data transfer (not charge-only)
- On Linux, ensure your user is in the `dialout` group: `sudo usermod -aG dialout $USER`
- On macOS, install the [CP210x driver](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers) if needed

### ESP-IDF Version Compatibility

| ESP-IDF Version | Status | Notes |
|:----------------|:-------|:------|
| v5.4+ | Recommended | Latest features and fixes |
| v5.0-v5.3 | Supported | Stable |
| v4.4 | Legacy | Still functional but missing newer features |

## Further Reading

- [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/)
- [PlatformIO ESP32C3 Documentation](https://docs.platformio.org/en/latest/boards/espressif32/seeed_xiao_esp32c3.html)
- [Arduino-ESP32 Documentation](https://docs.espressif.com/projects/arduino-esp32/en/latest/)
- [XIAO ESP32C3 Quick Reference](/XIAO_ESP32C3_Quick_Reference)

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
