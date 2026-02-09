---
description: Architecture deep dive for Seeed Studio XIAO ESP32C3
title: Architecture Overview
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

# XIAO ESP32C3 Architecture Overview

This guide provides a deep dive into the internal architecture of the ESP32-C3 SoC that powers the Seeed Studio XIAO ESP32C3, helping you understand the hardware foundations for better firmware development.

## ESP32-C3 SoC Block Diagram

The ESP32-C3 is a single-core Wi-Fi and Bluetooth 5 (LE) microcontroller SoC built on a 32-bit RISC-V architecture. Below is a high-level overview of the major subsystems.

```
┌─────────────────────────────────────────────────────┐
│                    ESP32-C3 SoC                     │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  RISC-V CPU  │  │   Wi-Fi      │  │ Bluetooth │ │
│  │  (160 MHz)   │  │  802.11 b/g/n│  │  5.0 (LE) │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                 │                │        │
│  ┌──────┴─────────────────┴────────────────┴─────┐  │
│  │              System Bus / Interconnect         │  │
│  └──┬────────┬──────────┬──────────┬──────────┬──┘  │
│     │        │          │          │          │      │
│  ┌──┴──┐ ┌──┴──┐  ┌────┴───┐ ┌───┴───┐ ┌───┴───┐  │
│  │SRAM │ │ ROM │  │ Flash  │ │Periph.│ │  DMA  │  │
│  │400KB│ │     │  │ 4MB    │ │ Matrix│ │       │  │
│  └─────┘ └─────┘  └────────┘ └───────┘ └───────┘  │
└─────────────────────────────────────────────────────┘
```

## RISC-V CPU Core

The ESP32-C3 features a single-core **32-bit RISC-V** processor (RV32IMC instruction set) with a four-stage pipeline.

- **Max Clock Frequency**: 160 MHz
- **Instruction Set**: RV32IMC (Integer, Multiply/Divide, Compressed)
- **Pipeline**: 4-stage (Fetch, Decode, Execute, Write-back)
- **Interrupt Controller**: CLIC (Core-Local Interrupt Controller) with 31 external interrupts
- **Debug**: JTAG-based debugging support

### Comparing to Xtensa (ESP32/ESP32-S3)

| Feature | ESP32-C3 (RISC-V) | ESP32 (Xtensa LX6) | ESP32-S3 (Xtensa LX7) |
|---|---|---|---|
| Architecture | RISC-V RV32IMC | Xtensa LX6 | Xtensa LX7 |
| Cores | 1 | 2 | 2 |
| Max Clock | 160 MHz | 240 MHz | 240 MHz |
| FPU | No | Yes | Yes |
| Pipeline | 4-stage | 5-stage | 5-stage |

The RISC-V core is simpler and more power-efficient, making the ESP32-C3 ideal for cost-sensitive and low-power IoT applications where dual-core performance or floating-point hardware is not required.

## Memory Architecture

### On-chip Memory

| Memory Type | Size | Description |
|---|---|---|
| SRAM | 400 KB | General-purpose RAM (16 KB for cache) |
| ROM | 384 KB | Boot and core function ROM |
| RTC SRAM | 8 KB | Retained during deep sleep |
| eFuse | 4096 bits | One-time programmable storage for keys and config |

### External Flash

The XIAO ESP32C3 ships with **4 MB** of external SPI flash, partitioned as follows (default Arduino partition):

| Partition | Offset | Size | Purpose |
|---|---|---|---|
| nvs | 0x9000 | 20 KB | Non-volatile storage (Wi-Fi config, user prefs) |
| otadata | 0xe000 | 8 KB | OTA state tracking |
| app0 | 0x10000 | 1.25 MB | Application firmware (slot 0) |
| app1 | 0x150000 | 1.25 MB | Application firmware (slot 1, OTA) |
| spiffs | 0x290000 | 1.5 MB | SPIFFS filesystem for user data |

You can inspect and customize the partition table using Arduino IDE or `esptool.py`:

```bash
# Read current partition table
esptool.py --port /dev/ttyACM0 read_flash 0x8000 0x1000 partition-table.bin
gen_esp32part.py partition-table.bin
```

## Peripheral Matrix

The ESP32-C3 includes a flexible peripheral matrix that allows most digital peripherals to be mapped to any GPIO pin.

### Available Peripherals

| Peripheral | Count | Notes |
|---|---|---|
| GPIO | 22 (11 exposed on XIAO) | Input/output, interrupt capable |
| ADC | 2 channels (4 exposed) | 12-bit SAR ADC, up to 100 ksps |
| SPI | 3 (1 usable: SPI2) | SPI0/SPI1 reserved for flash |
| I2C | 1 | Standard (100 kHz) and Fast (400 kHz) modes |
| UART | 2 | UART0 for USB-serial, UART1 for general use |
| I2S | 1 | Supports PCM/PDM audio |
| LED PWM | 6 channels | 14-bit resolution, configurable frequency |
| Timer | 2 groups x 1 timer | 54-bit general-purpose timers |
| Watchdog | 3 | 2x MWDT (Main) + 1x RWDT (RTC) |
| Temperature Sensor | 1 | Internal, -40 to 125 C |
| RNG | 1 | True random number generator (TRNG) |

### XIAO ESP32C3 Pin Map

```
         ┌─────────────┐
    D0  ─┤ GPIO2  (A0) ├─ D10  GPIO10
    D1  ─┤ GPIO3  (A1) ├─ D9   GPIO9
    D2  ─┤ GPIO4  (A2) ├─ D8   GPIO8 (SCK)
    D3  ─┤ GPIO5  (A3) ├─ D7   GPIO7 (MOSI)
    D4  ─┤ GPIO6 (SDA) ├─ D6   GPIO21
    D5  ─┤ GPIO7 (SCL) ├─
   3V3  ─┤             ├─ GND
         │   USB-C     │
         └─────────────┘
```

**Key GPIO assignments:**
- **I2C default**: SDA = GPIO6 (D4), SCL = GPIO7 (D5)
- **SPI default**: SCK = GPIO8 (D8), MOSI = GPIO10 (D10), MISO = GPIO9 (D9), CS = user-defined
- **UART0**: TX = GPIO21 (D6), RX = GPIO20 (D7) — used for USB-serial bridge
- **ADC pins**: GPIO2 (A0), GPIO3 (A1), GPIO4 (A2), GPIO5 (A3)

## Wireless Subsystems

### Wi-Fi

- **Standards**: IEEE 802.11 b/g/n (2.4 GHz only)
- **Modes**: Station (STA), SoftAP, STA+AP, Promiscuous (sniffer)
- **Security**: WPA/WPA2/WPA3-Personal, WPA2-Enterprise
- **Max TX Power**: 21 dBm
- **Antenna**: External IPEX connector on XIAO (antenna included)

### Bluetooth Low Energy (BLE) 5.0

- **Features**: BLE 5.0 with long range, 2M PHY, advertising extensions
- **Roles**: Broadcaster, Observer, Peripheral, Central
- **Mesh**: Bluetooth Mesh supported
- **Coexistence**: Wi-Fi and BLE can operate simultaneously with time-division multiplexing

## Clock Tree

The ESP32-C3 has multiple clock sources:

| Clock Source | Frequency | Used For |
|---|---|---|
| PLL | 480 MHz (divided to 160/80/40 MHz) | CPU, peripherals |
| XTAL | 40 MHz | PLL reference, RTC |
| RC_FAST | 17.5 MHz (approx.) | RTC, low-power modes |
| RC_SLOW | 136 kHz (approx.) | RTC watchdog, deep sleep timer |
| XTAL32K | 32.768 kHz (if external crystal present) | Precise RTC timekeeping |

In Arduino, you can read the CPU frequency:

```cpp
void setup() {
  Serial.begin(115200);
  Serial.print("CPU Frequency: ");
  Serial.print(getCpuFrequencyMhz());
  Serial.println(" MHz");

  // Reduce CPU frequency to save power
  setCpuFrequencyMhz(80);  // Options: 160, 80, 40
  Serial.print("New CPU Frequency: ");
  Serial.print(getCpuFrequencyMhz());
  Serial.println(" MHz");
}

void loop() {}
```

## Boot Process

The ESP32-C3 boot sequence proceeds through these stages:

```
1. First-stage bootloader (ROM)
   ├── Hardware initialization
   ├── Determine boot mode (normal / download)
   └── Load second-stage bootloader from flash @ 0x0

2. Second-stage bootloader (flash)
   ├── Read partition table
   ├── Select active app partition (OTA support)
   └── Load application image into SRAM

3. Application startup
   ├── FreeRTOS scheduler initialization
   ├── System peripheral initialization
   ├── Call app_main() / Arduino setup()
   └── Enter main loop
```

### Boot Mode Selection

| GPIO9 State (BOOT) | GPIO8 State | Mode |
|---|---|---|
| HIGH (default) | - | Normal boot from flash |
| LOW (held during reset) | - | Serial download mode (flashing) |

On the XIAO ESP32C3, hold the **BOOT button** (GPIO9) while pressing **RESET** to enter download mode for firmware flashing.

## DMA (Direct Memory Access)

The ESP32-C3 includes a GDMA (General DMA) controller with 3 channels, enabling zero-copy data transfers between peripherals and memory:

- **Supported peripherals**: SPI, I2S, UART, ADC, AES, SHA
- **Channels**: 3 TX + 3 RX (linked-list descriptor based)
- **Benefit**: Offloads data transfer from the CPU, crucial for continuous data streaming (e.g., audio, sensor sampling)

DMA is configured automatically by the ESP-IDF drivers when you use higher-level APIs.

## Security Hardware

The ESP32-C3 includes hardware-accelerated security features:

| Feature | Description |
|---|---|
| AES | 128/256-bit hardware accelerator |
| SHA | SHA-1, SHA-224, SHA-256 hardware accelerator |
| RSA | Up to 3072-bit key length accelerator |
| HMAC | Hardware HMAC generation |
| Digital Signature | Hardware-based digital signature verification |
| Flash Encryption | AES-128/256 transparent flash encryption |
| Secure Boot | RSA-PSS based secure boot chain |
| TRNG | True random number generator |
| eFuse | One-time programmable secure key storage |

For details on using these features, see [Security Features](/XIAO_ESP32C3_Security).

## Power Domains

The ESP32-C3 is divided into power domains that can be independently controlled:

| Domain | Contents | Powered In |
|---|---|---|
| Digital | CPU, digital peripherals, Wi-Fi, BLE | Active, Modem-sleep, Light-sleep |
| RTC | RTC controller, RTC SRAM, RTC peripherals | Active, Modem-sleep, Light-sleep, Deep-sleep |
| Analog | ADC, temperature sensor | Active |

Understanding these domains is critical for optimizing power consumption. See [Power Management](/XIAO_ESP32C3_Power_Management) for practical deep-sleep and battery optimization guides.

## Further Reading

- [Getting Started with XIAO ESP32C3](/XIAO_ESP32C3_Getting_Started) - Setup and first project
- [Pin Multiplexing](/XIAO_ESP32C3_Pin_Multiplexing) - GPIO, PWM, I2C, SPI, UART usage
- [Power Management](/XIAO_ESP32C3_Power_Management) - Deep sleep, battery, low-power modes
- [Security Features](/XIAO_ESP32C3_Security) - Flash encryption and secure boot
- [Debugging & Troubleshooting](/XIAO_ESP32C3_Debugging) - JTAG, serial debugging, common fixes
