---
description: Build a wireless camera with Seeed Studio XIAO ESP32C3 using SPI and UART camera modules
title: Wireless Camera
keywords:
- xiao
- esp32c3
- camera
- wireless camera
- arducam
- spi camera
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /xiao_esp32c3_wireless_camera
last_update:
  date: 02/09/2026
  author: Claude
---

# Wireless Camera with XIAO ESP32C3

This guide shows how to build a wireless camera using the XIAO ESP32C3. Because the ESP32-C3 **does not have a native parallel camera interface** (DVP/CSI), we use external camera modules connected via **SPI** or **UART** to capture JPEG images and serve them over Wi-Fi.

:::tip Which XIAO for camera projects?
If you need **real-time video streaming** (10+ FPS), the [XIAO ESP32S3 Sense](/xiao_esp32s3_camera_usage) is a much better choice -- it has a built-in OV2640 camera, dedicated camera interface, and 8MB PSRAM. The ESP32C3 approach described here is best for **periodic image capture** (security snapshots, time-lapse, motion-triggered photos) where low power and small size matter more than frame rate.
:::

## Hardware Comparison

| Approach | Module | Interface | Max Resolution | Typical FPS | Cost |
|---|---|---|---|---|---|
| **SPI Camera** | ArduCam Mini 2MP (OV2640) | SPI + I2C | 1600x1200 | 1-3 FPS (JPEG over Wi-Fi) | ~$15 |
| **UART Camera** | Grove Serial Camera Kit | UART | 640x480 | < 1 FPS | ~$25 |
| **Better Alternative** | XIAO ESP32S3 Sense | Native DVP | 1600x1200 | 12-15 FPS | ~$14 |

## Option 1: SPI Camera (ArduCam Mini 2MP) -- Recommended

The ArduCam Mini 2MP Plus uses an OV2640 sensor with SPI output, making it compatible with the ESP32-C3. It captures JPEG-compressed images internally, so the limited 400KB SRAM on the ESP32-C3 can handle the compressed data.

### Materials Required

| Component | Quantity | Notes |
|---|---|---|
| Seeed Studio XIAO ESP32C3 | 1 | Main controller |
| ArduCam Mini 2MP Plus (OV2640 SPI) | 1 | SPI camera module |
| Wi-Fi antenna | 1 | Included with XIAO |
| Jumper wires | 7 | Female-to-female |
| USB-C cable | 1 | For programming and power |

### Wiring

Connect the ArduCam Mini to the XIAO ESP32C3:

| ArduCam Pin | XIAO Pin | GPIO | Function |
|---|---|---|---|
| CS | D7 | GPIO21 | SPI Chip Select |
| MOSI | D10 | GPIO10 | SPI Data In |
| MISO | D9 | GPIO9 | SPI Data Out |
| SCK | D8 | GPIO8 | SPI Clock |
| SDA | D4 | GPIO6 | I2C Data (camera config) |
| SCL | D5 | GPIO7 | I2C Clock (camera config) |
| VCC | 3V3 | - | 3.3V Power |
| GND | GND | - | Ground |

```
  ArduCam Mini 2MP          XIAO ESP32C3
  ┌──────────────┐          ┌──────────┐
  │ CS  ─────────┼──────────┤ D7       │
  │ MOSI ────────┼──────────┤ D10      │
  │ MISO ────────┼──────────┤ D9       │
  │ SCK  ────────┼──────────┤ D8       │
  │ SDA  ────────┼──────────┤ D4 (SDA) │
  │ SCL  ────────┼──────────┤ D5 (SCL) │
  │ VCC  ────────┼──────────┤ 3V3      │
  │ GND  ────────┼──────────┤ GND      │
  └──────────────┘          └──────────┘
```

### Install the ArduCam Library

1. In Arduino IDE, go to **Sketch > Include Library > Manage Libraries**
2. Search for **ArduCAM** and install it
3. Also install **ArduCAM ESP32 OV2640** if available

Or install via the Library Manager URL: Add `https://www.arducam.com/downloads/esp32/package_ArduCAM_index.json` to Arduino Preferences > Additional Board Manager URLs.

### Sketch: Wi-Fi Image Capture Server

This sketch starts a web server on the XIAO ESP32C3. When you visit the device's IP address in a browser, it captures a JPEG image from the ArduCam and serves it.

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <SPI.h>
#include <ArduCAM.h>
#include "memorysaver.h"

// Wi-Fi credentials
const char* WIFI_SSID = "YourSSID";
const char* WIFI_PASS = "YourPassword";

// Pin definitions
const int CS_PIN = D7;

// Initialize camera and server
ArduCAM myCAM(OV2640, CS_PIN);
WebServer server(80);

// Buffer for reading SPI data
#define CHUNK_SIZE 512
uint8_t buffer[CHUNK_SIZE];

void setup() {
  Serial.begin(115200);

  // Initialize I2C (for camera register configuration)
  Wire.begin(D4, D5);

  // Initialize SPI
  SPI.begin(D8, D9, D10, CS_PIN);
  pinMode(CS_PIN, OUTPUT);
  digitalWrite(CS_PIN, HIGH);

  // Test SPI connection
  myCAM.write_reg(ARDUCHIP_TEST1, 0x55);
  uint8_t testVal = myCAM.read_reg(ARDUCHIP_TEST1);
  if (testVal != 0x55) {
    Serial.println("SPI interface error! Check wiring.");
    while (1);
  }
  Serial.println("SPI OK");

  // Check camera module
  uint8_t vid, pid;
  myCAM.wrSensorReg8_8(0xFF, 0x01);
  myCAM.rdSensorReg8_8(OV2640_CHIPID_HIGH, &vid);
  myCAM.rdSensorReg8_8(OV2640_CHIPID_LOW, &pid);
  if ((vid != 0x26) && ((pid != 0x41) && (pid != 0x42))) {
    Serial.println("Camera module not detected!");
    while (1);
  }
  Serial.printf("Camera detected: 0x%02X%02X\n", vid, pid);

  // Configure camera
  myCAM.set_format(JPEG);
  myCAM.InitCAM();
  myCAM.OV2640_set_JPEG_size(OV2640_640x480);  // Start with 640x480
  delay(1000);
  myCAM.clear_fifo_flag();

  // Connect to Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.printf("\nConnected! Open http://%s in your browser\n",
                WiFi.localIP().toString().c_str());

  // Web routes
  server.on("/", handleRoot);
  server.on("/capture", handleCapture);
  server.on("/stream", handleStream);
  server.on("/resolution", handleResolution);
  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();
}

// Serve the main page with controls
void handleRoot() {
  String html = R"(
<!DOCTYPE html>
<html>
<head>
  <title>XIAO ESP32C3 Camera</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: Arial, sans-serif; text-align: center;
           background: #1a1a2e; color: #eee; margin: 20px; }
    img  { max-width: 100%; border-radius: 8px;
           border: 2px solid #444; margin: 10px 0; }
    button { padding: 10px 24px; margin: 5px; font-size: 16px;
             border: none; border-radius: 6px; cursor: pointer;
             background: #0f3460; color: #eee; }
    button:hover { background: #16213e; }
    select { padding: 8px; font-size: 14px; margin: 5px;
             border-radius: 6px; }
    h1 { color: #e94560; }
    .controls { margin: 15px 0; }
  </style>
</head>
<body>
  <h1>XIAO ESP32C3 Wireless Camera</h1>
  <div class="controls">
    <select id="res" onchange="setRes()">
      <option value="0">320x240</option>
      <option value="1" selected>640x480</option>
      <option value="2">800x600</option>
      <option value="3">1024x768</option>
      <option value="4">1280x960</option>
      <option value="5">1600x1200</option>
    </select>
    <button onclick="snap()">Capture</button>
    <button onclick="startAuto()">Auto Refresh</button>
    <button onclick="stopAuto()">Stop</button>
  </div>
  <div><img id="photo" src="/capture" alt="Camera"></div>
  <script>
    let timer = null;
    function snap() {
      document.getElementById('photo').src = '/capture?' + Date.now();
    }
    function startAuto() {
      stopAuto();
      timer = setInterval(snap, 2000);
    }
    function stopAuto() {
      if (timer) { clearInterval(timer); timer = null; }
    }
    function setRes() {
      fetch('/resolution?val=' + document.getElementById('res').value)
        .then(() => setTimeout(snap, 500));
    }
  </script>
</body>
</html>
  )";
  server.send(200, "text/html", html);
}

// Capture a JPEG and send it
void handleCapture() {
  myCAM.flush_fifo();
  myCAM.clear_fifo_flag();
  myCAM.start_capture();

  // Wait for capture to complete (timeout 5 seconds)
  unsigned long start = millis();
  while (!myCAM.get_bit(ARDUCHIP_TRIG, CAP_DONE_MASK)) {
    if (millis() - start > 5000) {
      server.send(500, "text/plain", "Capture timeout");
      return;
    }
    delay(1);
  }

  uint32_t length = myCAM.read_fifo_length();
  if (length >= 0x5FFFF || length == 0) {
    server.send(500, "text/plain", "Invalid image size");
    myCAM.clear_fifo_flag();
    return;
  }

  // Stream the JPEG data
  server.setContentLength(length);
  server.send(200, "image/jpeg", "");

  myCAM.CS_LOW();
  myCAM.set_fifo_burst();

  WiFiClient client = server.client();
  while (length > 0) {
    size_t toRead = (length < CHUNK_SIZE) ? length : CHUNK_SIZE;
    for (size_t i = 0; i < toRead; i++) {
      buffer[i] = SPI.transfer(0x00);
    }
    client.write(buffer, toRead);
    length -= toRead;
  }

  myCAM.CS_HIGH();
  myCAM.clear_fifo_flag();
}

// Change camera resolution
void handleResolution() {
  if (server.hasArg("val")) {
    int res = server.arg("val").toInt();
    switch (res) {
      case 0: myCAM.OV2640_set_JPEG_size(OV2640_320x240);  break;
      case 1: myCAM.OV2640_set_JPEG_size(OV2640_640x480);  break;
      case 2: myCAM.OV2640_set_JPEG_size(OV2640_800x600);  break;
      case 3: myCAM.OV2640_set_JPEG_size(OV2640_1024x768); break;
      case 4: myCAM.OV2640_set_JPEG_size(OV2640_1280x960); break;
      case 5: myCAM.OV2640_set_JPEG_size(OV2640_1600x1200);break;
    }
    delay(100);
  }
  server.send(200, "text/plain", "OK");
}

// Simple auto-refresh stream page
void handleStream() {
  String html = R"(
<!DOCTYPE html>
<html><head><title>Stream</title></head>
<body style="margin:0;background:#000;text-align:center">
  <img id="s" style="max-width:100%">
  <script>
    function refresh() {
      let img = document.getElementById('s');
      img.onload = () => setTimeout(refresh, 100);
      img.onerror = () => setTimeout(refresh, 1000);
      img.src = '/capture?' + Date.now();
    }
    refresh();
  </script>
</body></html>
  )";
  server.send(200, "text/html", html);
}
```

### Usage

1. Flash the sketch to your XIAO ESP32C3 via USB
2. Open the Serial Monitor (115200 baud) to see the device's IP address
3. Open `http://<device-ip>` in your browser
4. Use the controls to capture images and change resolution

**Endpoints:**
- `http://<ip>/` -- Main page with UI controls
- `http://<ip>/capture` -- Raw JPEG capture (returns image)
- `http://<ip>/stream` -- Auto-refreshing stream page
- `http://<ip>/resolution?val=1` -- Change resolution (0-5)

### Performance Expectations

| Resolution | JPEG Size (typical) | Capture + Transfer Time | Effective FPS |
|---|---|---|---|
| 320x240 | 10-20 KB | ~200 ms | ~4-5 |
| 640x480 | 30-50 KB | ~400 ms | ~2-3 |
| 1024x768 | 80-120 KB | ~800 ms | ~1 |
| 1600x1200 | 150-250 KB | ~1.5 s | < 1 |

:::note
The ESP32-C3 has only 400 KB SRAM and no PSRAM. Higher resolutions work because the ArduCam module has its own 2 MB frame buffer -- the ESP32-C3 reads the JPEG data in small chunks via SPI, never holding the entire image in memory.
:::

## Option 2: UART Serial Camera (Grove Serial Camera)

A simpler option using the [Grove Serial Camera Kit](https://www.seeedstudio.com/Grove-Serial-Camera-Kit.html). Lower resolution and speed, but easier wiring.

### Wiring

| Grove Camera Pin | XIAO Pin | Function |
|---|---|---|
| TX | D7 (GPIO21) | Camera TX to XIAO RX |
| RX | D6 (GPIO20) | Camera RX to XIAO TX |
| VCC | 5V | Power |
| GND | GND | Ground |

### Sketch: UART Camera Capture

```cpp
#include <WiFi.h>
#include <WebServer.h>

const char* WIFI_SSID = "YourSSID";
const char* WIFI_PASS = "YourPassword";

// Use Serial1 for camera communication
#define CAM_SERIAL Serial1
#define CAM_RX D7   // Camera TX -> XIAO D7
#define CAM_TX D6   // Camera RX -> XIAO D6

WebServer server(80);

// JPEG buffer (limited by available RAM)
#define MAX_IMAGE_SIZE 40000
uint8_t imageBuffer[MAX_IMAGE_SIZE];
uint32_t imageSize = 0;

// Camera commands (VC0706 protocol)
const uint8_t CMD_RESET[]       = {0x56, 0x00, 0x26, 0x00};
const uint8_t CMD_TAKE_PHOTO[]  = {0x56, 0x00, 0x36, 0x01, 0x00};
const uint8_t CMD_GET_SIZE[]    = {0x56, 0x00, 0x34, 0x01, 0x00};
const uint8_t CMD_STOP_FRAME[]  = {0x56, 0x00, 0x36, 0x01, 0x03};
const uint8_t CMD_RESUME[]      = {0x56, 0x00, 0x36, 0x01, 0x02};

// Set resolution to 640x480
const uint8_t CMD_SET_640[]     = {0x56, 0x00, 0x31, 0x05, 0x04, 0x01, 0x00, 0x19, 0x00};

void sendCommand(const uint8_t* cmd, uint8_t len) {
  CAM_SERIAL.write(cmd, len);
}

bool waitForResponse(uint8_t* buf, uint8_t expectedLen, unsigned long timeout = 1000) {
  unsigned long start = millis();
  uint8_t idx = 0;
  while (idx < expectedLen && millis() - start < timeout) {
    if (CAM_SERIAL.available()) {
      buf[idx++] = CAM_SERIAL.read();
    }
  }
  return idx == expectedLen;
}

bool captureImage() {
  uint8_t response[16];

  // Stop current frame
  sendCommand(CMD_STOP_FRAME, sizeof(CMD_STOP_FRAME));
  waitForResponse(response, 5);

  // Take photo
  sendCommand(CMD_TAKE_PHOTO, sizeof(CMD_TAKE_PHOTO));
  waitForResponse(response, 5);

  // Get JPEG size
  sendCommand(CMD_GET_SIZE, sizeof(CMD_GET_SIZE));
  if (!waitForResponse(response, 9)) return false;

  imageSize = (response[7] << 8) | response[8];
  if (imageSize > MAX_IMAGE_SIZE) {
    Serial.printf("Image too large: %d bytes\n", imageSize);
    imageSize = 0;
    return false;
  }

  // Read JPEG data in chunks
  uint32_t offset = 0;
  uint32_t remaining = imageSize;

  while (remaining > 0) {
    uint16_t chunkSize = (remaining > 256) ? 256 : remaining;

    // Read data command
    uint8_t readCmd[] = {0x56, 0x00, 0x32, 0x0C, 0x00, 0x0A, 0x00, 0x00,
                         (uint8_t)(offset >> 8), (uint8_t)(offset & 0xFF),
                         0x00, 0x00,
                         (uint8_t)(chunkSize >> 8), (uint8_t)(chunkSize & 0xFF),
                         0x00, 0x0A};
    CAM_SERIAL.write(readCmd, sizeof(readCmd));

    // Skip header (5 bytes)
    waitForResponse(response, 5);

    // Read image data
    unsigned long start = millis();
    uint16_t bytesRead = 0;
    while (bytesRead < chunkSize && millis() - start < 2000) {
      if (CAM_SERIAL.available()) {
        imageBuffer[offset + bytesRead] = CAM_SERIAL.read();
        bytesRead++;
      }
    }

    // Skip footer (5 bytes)
    waitForResponse(response, 5);

    offset += chunkSize;
    remaining -= chunkSize;
  }

  // Resume frame
  sendCommand(CMD_RESUME, sizeof(CMD_RESUME));
  waitForResponse(response, 5);

  Serial.printf("Captured: %d bytes\n", imageSize);
  return true;
}

void handleCapture() {
  if (captureImage() && imageSize > 0) {
    server.send_P(200, "image/jpeg", (const char*)imageBuffer, imageSize);
  } else {
    server.send(500, "text/plain", "Capture failed");
  }
}

void handleRoot() {
  String html = R"(
<!DOCTYPE html><html><head>
<title>XIAO ESP32C3 UART Camera</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>body{font-family:sans-serif;text-align:center;background:#1a1a2e;color:#eee;margin:20px}
img{max-width:100%;border-radius:8px;margin:10px 0}
button{padding:10px 24px;margin:5px;font-size:16px;border:none;border-radius:6px;
cursor:pointer;background:#0f3460;color:#eee}</style>
</head><body>
<h1>UART Camera</h1>
<button onclick="snap()">Capture</button>
<div><img id="photo" alt="Camera"></div>
<script>function snap(){document.getElementById('photo').src='/capture?'+Date.now()}</script>
</body></html>
  )";
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  CAM_SERIAL.begin(38400, SERIAL_8N1, CAM_RX, CAM_TX);

  // Reset camera
  sendCommand(CMD_RESET, sizeof(CMD_RESET));
  delay(3000);

  // Drain any startup messages
  while (CAM_SERIAL.available()) CAM_SERIAL.read();

  // Set resolution
  sendCommand(CMD_SET_640, sizeof(CMD_SET_640));
  delay(100);

  Serial.println("Camera initialized");

  // Connect Wi-Fi
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.printf("Open http://%s\n", WiFi.localIP().toString().c_str());

  server.on("/", handleRoot);
  server.on("/capture", handleCapture);
  server.begin();
}

void loop() {
  server.handleClient();
}
```

## Motion-Triggered Capture with Deep Sleep

A power-efficient security camera pattern: the XIAO ESP32C3 sleeps until a PIR motion sensor triggers a wake-up, then captures an image and sends it via HTTP or MQTT.

### Additional Hardware

| Component | Pin | Purpose |
|---|---|---|
| PIR Motion Sensor (HC-SR501) | D2 (GPIO4) | Wake trigger |
| ArduCam Mini 2MP | SPI (see wiring above) | Image capture |

### Sketch

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <SPI.h>
#include <ArduCAM.h>
#include <esp_sleep.h>
#include "memorysaver.h"

const char* WIFI_SSID   = "YourSSID";
const char* WIFI_PASS   = "YourPassword";
const char* UPLOAD_URL  = "http://yourserver.com/upload";

const int CS_PIN  = D7;
const int PIR_PIN = D2;

ArduCAM myCAM(OV2640, CS_PIN);
RTC_DATA_ATTR int captureCount = 0;

#define CHUNK_SIZE 512
uint8_t buffer[CHUNK_SIZE];

void setup() {
  Serial.begin(115200);
  captureCount++;
  Serial.printf("Motion event #%d\n", captureCount);

  // Initialize camera
  Wire.begin(D4, D5);
  SPI.begin(D8, D9, D10, CS_PIN);
  pinMode(CS_PIN, OUTPUT);
  digitalWrite(CS_PIN, HIGH);

  myCAM.set_format(JPEG);
  myCAM.InitCAM();
  myCAM.OV2640_set_JPEG_size(OV2640_640x480);
  delay(500);

  // Capture image
  myCAM.flush_fifo();
  myCAM.clear_fifo_flag();
  myCAM.start_capture();

  unsigned long start = millis();
  while (!myCAM.get_bit(ARDUCHIP_TRIG, CAP_DONE_MASK)) {
    if (millis() - start > 5000) {
      Serial.println("Capture timeout");
      goToSleep();
      return;
    }
  }

  uint32_t imgLen = myCAM.read_fifo_length();
  Serial.printf("Image size: %d bytes\n", imgLen);

  // Connect Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  start = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - start < 10000) {
    delay(100);
  }

  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("Wi-Fi failed");
    goToSleep();
    return;
  }

  // Upload image via HTTP POST
  HTTPClient http;
  http.begin(UPLOAD_URL);
  http.addHeader("Content-Type", "image/jpeg");
  http.addHeader("X-Capture-Number", String(captureCount));

  // Read from ArduCam FIFO and send
  // For simplicity, read into a buffer (limited by RAM)
  if (imgLen < 40000) {
    uint8_t* imgBuf = (uint8_t*)malloc(imgLen);
    if (imgBuf) {
      myCAM.CS_LOW();
      myCAM.set_fifo_burst();
      for (uint32_t i = 0; i < imgLen; i++) {
        imgBuf[i] = SPI.transfer(0x00);
      }
      myCAM.CS_HIGH();

      int httpCode = http.POST(imgBuf, imgLen);
      Serial.printf("Upload response: %d\n", httpCode);
      free(imgBuf);
    }
  }

  http.end();
  myCAM.clear_fifo_flag();

  goToSleep();
}

void goToSleep() {
  WiFi.disconnect(true);
  WiFi.mode(WIFI_OFF);

  // Wake on PIR motion (HIGH signal)
  esp_deep_sleep_enable_gpio_wakeup(
    BIT(GPIO_NUM_4),  // D2 = GPIO4
    ESP_GPIO_WAKEUP_GPIO_HIGH
  );

  Serial.println("Sleeping until motion detected...");
  Serial.flush();
  esp_deep_sleep_start();
}

void loop() {}
```

## Time-Lapse Camera

Capture images at regular intervals and serve a gallery over Wi-Fi. Images are stored in the ArduCam's FIFO buffer and served one at a time (the ESP32-C3 doesn't have enough RAM to store multiple high-res images).

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <SPI.h>
#include <ArduCAM.h>
#include "memorysaver.h"

const char* WIFI_SSID = "YourSSID";
const char* WIFI_PASS = "YourPassword";

const int CS_PIN = D7;
ArduCAM myCAM(OV2640, CS_PIN);
WebServer server(80);

unsigned long lastCapture = 0;
unsigned long captureInterval = 60000;  // 1 minute
uint32_t captureCount = 0;

#define CHUNK_SIZE 512
uint8_t buffer[CHUNK_SIZE];

void captureAndStore() {
  myCAM.flush_fifo();
  myCAM.clear_fifo_flag();
  myCAM.start_capture();

  unsigned long start = millis();
  while (!myCAM.get_bit(ARDUCHIP_TRIG, CAP_DONE_MASK)) {
    if (millis() - start > 5000) return;
  }

  captureCount++;
  Serial.printf("Time-lapse capture #%d, size: %d bytes\n",
                captureCount, myCAM.read_fifo_length());
}

void handleLatest() {
  // Serve the most recent capture from ArduCam FIFO
  uint32_t length = myCAM.read_fifo_length();
  if (length == 0 || length >= 0x5FFFF) {
    server.send(404, "text/plain", "No image available");
    return;
  }

  server.setContentLength(length);
  server.send(200, "image/jpeg", "");

  myCAM.CS_LOW();
  myCAM.set_fifo_burst();
  WiFiClient client = server.client();

  while (length > 0) {
    size_t toRead = (length < CHUNK_SIZE) ? length : CHUNK_SIZE;
    for (size_t i = 0; i < toRead; i++) {
      buffer[i] = SPI.transfer(0x00);
    }
    client.write(buffer, toRead);
    length -= toRead;
  }
  myCAM.CS_HIGH();
}

void handleRoot() {
  String html = R"(
<!DOCTYPE html><html><head>
<title>XIAO Time-Lapse</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta http-equiv="refresh" content="60">
<style>body{font-family:sans-serif;text-align:center;background:#1a1a2e;color:#eee;margin:20px}
img{max-width:100%;border-radius:8px}</style>
</head><body>
<h1>Time-Lapse Camera</h1>
<p>Capture #)" + String(captureCount) + R"( | Interval: )" +
String(captureInterval / 1000) + R"(s</p>
<img src="/latest" alt="Latest capture">
<p>Page auto-refreshes every 60 seconds</p>
</body></html>
  )";
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  Wire.begin(D4, D5);
  SPI.begin(D8, D9, D10, CS_PIN);
  pinMode(CS_PIN, OUTPUT);
  digitalWrite(CS_PIN, HIGH);

  myCAM.set_format(JPEG);
  myCAM.InitCAM();
  myCAM.OV2640_set_JPEG_size(OV2640_640x480);
  delay(1000);

  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.printf("Open http://%s\n", WiFi.localIP().toString().c_str());

  server.on("/", handleRoot);
  server.on("/latest", handleLatest);
  server.begin();

  // Take initial capture
  captureAndStore();
}

void loop() {
  server.handleClient();

  if (millis() - lastCapture > captureInterval) {
    captureAndStore();
    lastCapture = millis();
  }
}
```

## Sending Images to MQTT / Telegram

### MQTT Image Publishing

For sending captured images to a Home Assistant dashboard or MQTT broker:

```cpp
#include <PubSubClient.h>

// After capturing image into a buffer...
void publishImageToMQTT(uint8_t* imgData, uint32_t imgSize) {
  // PubSubClient default max payload is 256 bytes -- increase it
  mqtt.setBufferSize(imgSize + 128);

  // Publish raw JPEG to an MQTT topic
  mqtt.publish("camera/xiao/image", imgData, imgSize, false);

  // Publish metadata
  String meta = "{\"size\":" + String(imgSize) +
                ",\"timestamp\":" + String(millis()) + "}";
  mqtt.publish("camera/xiao/meta", meta.c_str());
}
```

### Telegram Bot Notification

Send captured images to a Telegram chat:

```cpp
#include <WiFiClientSecure.h>

void sendToTelegram(uint8_t* imgData, uint32_t imgSize) {
  const char* botToken = "YOUR_BOT_TOKEN";
  const char* chatId   = "YOUR_CHAT_ID";

  WiFiClientSecure client;
  client.setInsecure();  // Skip cert verification for simplicity

  if (!client.connect("api.telegram.org", 443)) {
    Serial.println("Telegram connection failed");
    return;
  }

  String boundary = "----XiaoBoundary";
  String bodyStart = "--" + boundary + "\r\n"
    "Content-Disposition: form-data; name=\"chat_id\"\r\n\r\n" +
    String(chatId) + "\r\n" +
    "--" + boundary + "\r\n"
    "Content-Disposition: form-data; name=\"photo\"; filename=\"capture.jpg\"\r\n"
    "Content-Type: image/jpeg\r\n\r\n";
  String bodyEnd = "\r\n--" + boundary + "--\r\n";

  uint32_t contentLength = bodyStart.length() + imgSize + bodyEnd.length();

  client.println("POST /bot" + String(botToken) + "/sendPhoto HTTP/1.1");
  client.println("Host: api.telegram.org");
  client.println("Content-Type: multipart/form-data; boundary=" + boundary);
  client.println("Content-Length: " + String(contentLength));
  client.println();
  client.print(bodyStart);
  client.write(imgData, imgSize);
  client.print(bodyEnd);

  Serial.println("Image sent to Telegram");
  client.stop();
}
```

## Troubleshooting

| Issue | Solution |
|---|---|
| "SPI interface error" | Check wiring. Ensure CS, MOSI, MISO, SCK are connected correctly. Verify 3.3V power. |
| "Camera module not detected" | Check I2C wiring (SDA/SCL). Ensure camera module is an OV2640 variant. |
| Image is garbled/corrupted | Lower the SPI clock speed. Check for loose connections. Try a lower resolution. |
| Captures are very slow | Use lower resolution. The SPI bus speed and Wi-Fi transfer are the bottlenecks. |
| Out of memory | Use chunked reading (as shown in examples). Never allocate the full image in RAM for resolutions above 640x480. |
| Wi-Fi drops during transfer | Ensure strong Wi-Fi signal. The antenna must be connected. Consider lowering resolution. |

## Further Reading

- [XIAO ESP32S3 Sense Camera Usage](/xiao_esp32s3_camera_usage) -- Native camera with 15 FPS streaming
- [WiFi Usage](/XIAO_ESP32C3_WiFi_Usage) -- Wi-Fi connectivity details
- [Power Management](/XIAO_ESP32C3_Power_Management) -- Deep sleep for battery-powered cameras
- [MQTT Communication](/xiao_esp32c3_mqtt) -- Sending images to MQTT brokers
- [Architecture Overview](/XIAO_ESP32C3_Architecture) -- SPI peripheral and memory details
