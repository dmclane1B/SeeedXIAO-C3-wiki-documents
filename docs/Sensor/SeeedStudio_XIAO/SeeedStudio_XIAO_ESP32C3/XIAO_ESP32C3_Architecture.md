---
description: ESP32-C3 System Architecture for Seeed Studio XIAO ESP32C3
title: System Architecture
keywords:
- xiao
- esp32c3
- architecture
- risc-v
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_Architecture
last_update:
  date: 02/07/2026
  author: Claude
---

# XIAO ESP32C3 System Architecture

This document provides a technical overview of the ESP32-C3 SoC architecture as it applies to the Seeed Studio XIAO ESP32C3. Understanding the internal architecture helps developers write more efficient firmware, debug hardware issues, and make informed design decisions.

## SoC Block Diagram

The ESP32-C3 is a single-core, 32-bit RISC-V microcontroller with integrated Wi-Fi and Bluetooth 5 (LE) connectivity. The major subsystems are:

```
┌─────────────────────────────────────────────────────────┐
│                      ESP32-C3 SoC                       │
│                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│  │  RISC-V CPU  │   │  ROM 384KB   │   │ SRAM 400KB │  │
│  │  up to 160MHz│   │  (Bootloader)│   │            │  │
│  └──────┬───────┘   └──────────────┘   └────────────┘  │
│         │                                               │
│  ┌──────┴──────────────────────────────────────────┐    │
│  │              System Bus (AHB/APB)               │    │
│  └──┬────┬────┬────┬─────┬─────┬─────┬─────┬──────┘    │
│     │    │    │    │     │     │     │     │            │
│  ┌──┴┐┌──┴┐┌──┴┐┌──┴─┐┌──┴──┐┌──┴──┐┌──┴──┐┌──┴───┐   │
│  │SPI││I2C││UAR││ ADC ││Timer││ RTC ││ DMA ││Crypto│   │
│  └───┘└───┘└───┘└────┘└─────┘└─────┘└─────┘└──────┘   │
│                                                         │
│  ┌─────────────────────┐  ┌─────────────────────────┐   │
│  │  Wi-Fi 802.11b/g/n  │  │  Bluetooth 5.0 (LE)     │   │
│  │  2.4 GHz            │  │  Bluetooth Mesh          │   │
│  └─────────────────────┘  └─────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## CPU Core

The ESP32-C3 uses a **single-core 32-bit RISC-V processor** (RV32IMC instruction set) with a four-stage pipeline:

| Feature | Details |
|:--------|:--------|
| ISA | RV32IMC (Integer, Multiply/Divide, Compressed) |
| Max Clock | 160 MHz |
| Pipeline | 4-stage (Fetch, Decode, Execute, Writeback) |
| Interrupt Controller | CLIC with 31 external interrupts |
| Debug | JTAG interface via GPIO4/5/6/7 |

The RISC-V core provides deterministic performance well-suited for real-time IoT workloads. The compressed instruction extension (C) reduces code size, which is important given the limited flash available.

## Memory Architecture

### On-Chip Memory

| Memory | Size | Description |
|:-------|:-----|:------------|
| ROM | 384 KB | First-stage bootloader, core libraries, and crypto routines |
| SRAM | 400 KB | Instruction and data memory, shared with wireless stack |
| RTC SRAM | 8 KB | Retains data during deep sleep |
| eFuse | 4096 bits | One-time programmable storage for keys, MAC, calibration |

### External Memory (On-Board)

| Memory | Size | Interface |
|:-------|:-----|:----------|
| Flash | 4 MB | SPI (connected internally) |

### Memory Map Overview

```
0x0000_0000 ┌──────────────────────┐
             │  Reserved            │
0x3C00_0000 ├──────────────────────┤
             │  Flash Data (MMU)    │  ← Code/data mapped from external flash
0x3FC8_0000 ├──────────────────────┤
             │  Internal SRAM 1     │  ← Data RAM (400KB)
0x3FCC_0000 ├──────────────────────┤
             │  Internal SRAM 0     │
0x4000_0000 ├──────────────────────┤
             │  Internal ROM        │  ← Bootloader ROM (384KB)
0x4200_0000 ├──────────────────────┤
             │  Flash Instruction   │  ← Instruction cache from flash
0x5000_0000 ├──────────────────────┤
             │  RTC FAST Memory     │  ← 8KB, retained in deep sleep
0x6000_0000 ├──────────────────────┤
             │  Peripheral Registers│  ← I/O register space
             └──────────────────────┘
```

:::tip
The 400KB SRAM is shared between your application and the Wi-Fi/BLE stack. When Wi-Fi is active, approximately 160KB of SRAM is consumed by the networking stack, leaving ~240KB for your application. Plan your memory budget accordingly.
:::

## Flash Partitions

The default 4MB flash layout used by Arduino/ESP-IDF:

| Partition | Offset | Size | Purpose |
|:----------|:-------|:-----|:--------|
| bootloader | 0x0000 | 32 KB | Second-stage bootloader |
| partition-table | 0x8000 | 4 KB | Partition table |
| nvs | 0x9000 | 20 KB | Non-Volatile Storage (Wi-Fi credentials, user data) |
| phy_init | 0xF000 | 4 KB | PHY calibration data |
| app0 | 0x10000 | 1.25 MB | Application firmware (slot A) |
| app1 | 0x150000 | 1.25 MB | OTA firmware (slot B) |
| spiffs | 0x290000 | 1.5 MB | SPIFFS/LittleFS filesystem |

:::note
The partition layout can be customized via `partitions.csv` when using ESP-IDF or PlatformIO. Arduino IDE uses a default layout but allows selecting alternatives via the **Tools > Partition Scheme** menu.
:::

## Boot Process

The ESP32-C3 follows a multi-stage boot sequence:

```
Power On / Reset
       │
       ▼
┌──────────────────┐
│  1st Stage Boot  │  ROM bootloader (in chip ROM)
│  - Reset vector  │  - Checks strapping pins (GPIO2, GPIO8, GPIO9)
│  - Clock init    │  - Selects boot mode: SPI flash / UART download
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2nd Stage Boot  │  From flash at 0x0000
│  - Hardware init │  - Initializes flash, caches, memory
│  - Partition scan│  - Loads application from active OTA slot
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Application     │  Your firmware (Arduino setup/loop, or ESP-IDF app_main)
│  - FreeRTOS init │  - FreeRTOS scheduler starts
│  - User code     │  - setup() then loop() in Arduino
└──────────────────┘
```

### Strapping Pins

These pins are sampled at boot to determine the boot mode:

| Pin | Default | Boot Mode Effect |
|:----|:--------|:-----------------|
| GPIO2 | Floating | Must be HIGH for SPI boot (default via pull-up) |
| GPIO8 | Floating | Must be HIGH for SPI boot |
| GPIO9 | Pull-up (BOOT btn) | LOW = UART download mode, HIGH = normal SPI boot |

:::caution
The combination of GPIO8 = LOW and GPIO9 = LOW is invalid and will trigger unexpected behavior. If you use GPIO8 as an output, add an external pull-up resistor to ensure it is HIGH during boot.
:::

## Peripheral Subsystems

### GPIO Matrix

The ESP32-C3 has 22 GPIOs (GPIO0-GPIO21), of which 11 are exposed on the XIAO ESP32C3 board. The GPIO matrix allows flexible routing of peripheral signals to any GPIO pin.

| Peripheral | Default XIAO Pins | Notes |
|:-----------|:-------------------|:------|
| ADC1 | GPIO2 (A0), GPIO3 (A1), GPIO4 (A2) | 12-bit SAR ADC, 0-2500mV |
| ADC2 | GPIO5 (A3) | Unreliable when Wi-Fi active |
| I2C | GPIO6 (SDA), GPIO7 (SCL) | Up to 400 kHz |
| SPI | GPIO8 (SCK), GPIO9 (MISO), GPIO10 (MOSI) | Up to 80 MHz |
| UART0 | GPIO21 (TX), GPIO20 (RX) | Default serial |
| UART1 | Configurable | Any available GPIO |
| PWM | Any GPIO | 6 channels, up to 40 MHz base clock |
| I2S | Configurable | Digital audio interface |
| JTAG | GPIO4, GPIO5, GPIO6, GPIO7 | Shared with I2C/ADC pins |

### ADC (Analog-to-Digital Converter)

The ESP32-C3 has two ADC units:

- **ADC1** (6 channels): GPIO0-GPIO4 — reliable, recommended
- **ADC2** (1 channel): GPIO5 — conflicts with Wi-Fi, avoid when wireless is active

Key characteristics:
- 12-bit resolution (0-4095)
- Attenuation configurable: 0dB, 2.5dB, 6dB, 11dB
- Default full-scale range: ~2500mV (with calibration correction in eFuse)
- Use `analogReadMilliVolts()` for calibrated readings

### Timers

| Timer | Count | Resolution | Notes |
|:------|:------|:-----------|:------|
| General Purpose Timer | 2 groups x 1 timer | 54-bit | Alarm, watchdog support |
| Systimer | 1 | 52-bit | Used by FreeRTOS tick |
| Watchdog Timer | 3 (2 MWDT + 1 RWDT) | - | RTC WDT survives deep sleep |

### DMA

The General DMA (GDMA) controller has 3 channels with linked-list descriptor support. DMA is used internally by SPI, I2S, UART, ADC, and the crypto accelerator.

## Wireless Architecture

### Wi-Fi Subsystem

| Feature | Details |
|:--------|:--------|
| Standards | IEEE 802.11 b/g/n |
| Band | 2.4 GHz |
| Bandwidth | 20 MHz / 40 MHz |
| Max TX Power | 21 dBm |
| Modes | Station, SoftAP, Station+SoftAP, Promiscuous |
| Security | WPA, WPA2, WPA3, WAPI |
| Antenna | External IPEX connector on XIAO |

### Bluetooth LE Subsystem

| Feature | Details |
|:--------|:--------|
| Version | Bluetooth 5.0 |
| Features | LE 1M/2M PHY, Coded PHY (long range), advertising extensions |
| Mesh | Bluetooth Mesh supported |
| Max TX Power | 21 dBm |
| Coexistence | Time-division with Wi-Fi |

:::tip
Wi-Fi and BLE share the same 2.4 GHz radio and use a time-division coexistence mechanism. When both are active simultaneously, expect reduced throughput on each. For latency-sensitive BLE applications, consider disabling Wi-Fi during critical communication windows.
:::

## Security Hardware

The ESP32-C3 includes hardware cryptographic accelerators:

| Accelerator | Capabilities |
|:------------|:-------------|
| AES | 128/256-bit encryption/decryption |
| SHA | SHA-1, SHA-224, SHA-256 |
| RSA | Up to 3072-bit key operations |
| HMAC | Hardware-based message authentication |
| Digital Signature | Hardware-assisted private key operations (key never exposed to software) |
| RNG | True random number generator |
| Secure Boot | Verified boot chain from ROM to application |
| Flash Encryption | Transparent AES-XTS-128 encryption of flash contents |

### Secure Boot Flow

```
ROM Bootloader ──verify──▶ 2nd Stage Bootloader ──verify──▶ Application
     (RSA)                        (RSA)
```

When secure boot is enabled, each stage verifies the digital signature of the next stage before executing it. The signing key hash is burned into eFuse (one-time programmable).

## Power Domains

The ESP32-C3 has separate power domains that can be independently controlled:

| Domain | Components | Deep Sleep State |
|:-------|:-----------|:-----------------|
| Digital | CPU, SRAM, peripherals | OFF |
| RTC | RTC controller, RTC SRAM (8KB), GPIO wakeup logic | ON |
| Wi-Fi/BT | Radio, baseband | OFF |

Deep sleep power consumption: **~44 uA** (RTC domain active, GPIO wakeup enabled)

For detailed power management strategies, see [Power Management](/XIAO_ESP32C3_Power_Management).

## Clock Tree

| Clock Source | Frequency | Purpose |
|:-------------|:----------|:--------|
| PLL | 480 MHz (derived from 40 MHz XTAL) | CPU clock source (divided to 160/80/40 MHz) |
| XTAL | 40 MHz | Reference oscillator |
| RC_FAST | 17.5 MHz (approx) | Low-power RTC clock |
| RC_SLOW | 136 kHz (approx) | RTC timer in deep sleep |
| XTAL32K | 32.768 kHz (if external crystal present) | Precision RTC timing |

The CPU runs at 160 MHz by default in Arduino. Lower clock speeds (80 MHz, 40 MHz) can be selected to reduce power consumption:

```cpp
// Set CPU frequency to 80 MHz to save power
setCpuFrequencyMhz(80);
```

## XIAO ESP32C3 Board-Level Design

Beyond the ESP32-C3 SoC itself, the XIAO board includes:

| Component | Details |
|:----------|:--------|
| Flash | 4 MB SPI flash (on-chip package) |
| Antenna | External 2.4 GHz antenna via IPEX connector |
| USB | USB Type-C with CDC-ACM (native USB serial) |
| Power Management | LDO regulator (3.3V / 700mA output), battery charge IC |
| Battery Charging | Supports 3.7V Li-Po, 380mA fast charge / 40mA trickle |
| Reset Circuit | Dedicated reset button connected to CHIP_EN |
| Boot Circuit | Boot button on GPIO9 with pull-up resistor |
| Dimensions | 21 x 17.8 mm |

### Schematic Highlights

- **USB-C**: Connected to the ESP32-C3's native USB peripheral (GPIO18/GPIO19, internal)
- **Charge LED**: Connected to VCC_3V3 line, indicates charging status
- **Strapping pin R6**: Pull-up resistor on GPIO9 ensures normal SPI boot by default
- **IPEX antenna connector**: Provides better range than a PCB antenna

## Further Reading

- [ESP32-C3 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf)
- [ESP32-C3 Datasheet](https://files.seeedstudio.com/wiki/XIAO_WiFi/Resources/esp32-c3_datasheet.pdf)
- [XIAO ESP32C3 Getting Started](/XIAO_ESP32C3_Getting_Started)
- [XIAO ESP32C3 Pin Multiplexing](/XIAO_ESP32C3_Pin_Multiplexing)

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
