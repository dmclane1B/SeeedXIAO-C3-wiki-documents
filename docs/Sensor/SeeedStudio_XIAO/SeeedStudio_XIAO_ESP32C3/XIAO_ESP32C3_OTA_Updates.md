---
description: Over-the-Air (OTA) firmware updates for Seeed Studio XIAO ESP32C3
title: OTA Firmware Updates
keywords:
- xiao
- esp32c3
- ota
- firmware update
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /XIAO_ESP32C3_OTA_Updates
last_update:
  date: 02/07/2026
  author: Claude
---

# XIAO ESP32C3 OTA Firmware Updates

Over-the-Air (OTA) updates allow you to upload new firmware to your XIAO ESP32C3 wirelessly over Wi-Fi, without needing a USB cable. This is essential for deployed devices in the field.

## How OTA Works on ESP32-C3

The ESP32-C3 flash is divided into two application partitions (`app0` and `app1`). OTA writes the new firmware to the inactive partition, then switches the boot target on the next reboot.

```
Flash Layout (default 4MB):
┌──────────────┐ 0x000000
│  Bootloader  │
├──────────────┤ 0x008000
│ Part. Table  │
├──────────────┤ 0x009000
│     NVS      │ (20 KB)
├──────────────┤ 0x00E000
│   OTA Data   │ (8 KB)
├──────────────┤ 0x010000
│    app0      │ (1.25 MB) ← Currently running
├──────────────┤ 0x150000
│    app1      │ (1.25 MB) ← OTA writes here
├──────────────┤ 0x290000
│   SPIFFS     │ (1.5 MB)
└──────────────┘ 0x400000
```

After a successful OTA update, the bootloader switches to the newly written partition. If the new firmware fails to boot, rollback to the previous partition is possible.

## Method 1: Arduino OTA (mDNS-based)

The simplest approach for development. Upload firmware over the local network using Arduino IDE or PlatformIO.

### Setup

```cpp
#include <WiFi.h>
#include <ArduinoOTA.h>

const char* WIFI_SSID = "YourSSID";
const char* WIFI_PASS = "YourPassword";

void setup() {
  Serial.begin(115200);

  // Connect to Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected. IP: " + WiFi.localIP().toString());

  // Configure OTA
  ArduinoOTA.setHostname("xiao-esp32c3");
  ArduinoOTA.setPassword("update-password");  // Optional auth

  ArduinoOTA.onStart([]() {
    String type = (ArduinoOTA.getCommand() == U_FLASH)
                    ? "firmware" : "filesystem";
    Serial.println("OTA Start: " + type);
  });

  ArduinoOTA.onEnd([]() {
    Serial.println("\nOTA Complete!");
  });

  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });

  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) Serial.println("Auth Failed");
    else if (error == OTA_BEGIN_ERROR) Serial.println("Begin Failed");
    else if (error == OTA_CONNECT_ERROR) Serial.println("Connect Failed");
    else if (error == OTA_RECEIVE_ERROR) Serial.println("Receive Failed");
    else if (error == OTA_END_ERROR) Serial.println("End Failed");
  });

  ArduinoOTA.begin();
  Serial.println("OTA Ready");
}

void loop() {
  ArduinoOTA.handle();  // Must be called regularly
  // Your application code here
}
```

### Uploading via Arduino IDE

1. Flash the OTA-enabled sketch above via USB first
2. In Arduino IDE, go to **Tools > Port** and select the network port (e.g., `xiao-esp32c3 at 192.168.1.x`)
3. Upload your new sketch normally -- it will be sent over Wi-Fi

## Method 2: HTTP OTA (Web Server Update)

Host a firmware binary on a web server or use the ESP32 itself as a web server to accept firmware uploads.

### Self-hosted Web Update

The device runs a web server with an upload page:

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <Update.h>

const char* WIFI_SSID = "YourSSID";
const char* WIFI_PASS = "YourPassword";

WebServer server(80);

const char* uploadPage = R"(
<!DOCTYPE html>
<html>
<head><title>XIAO ESP32C3 OTA</title></head>
<body>
  <h1>Firmware Update</h1>
  <form method="POST" action="/update" enctype="multipart/form-data">
    <input type="file" name="firmware" accept=".bin">
    <br><br>
    <input type="submit" value="Upload & Update">
  </form>
</body>
</html>
)";

void setup() {
  Serial.begin(115200);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println("IP: " + WiFi.localIP().toString());

  // Serve the upload page
  server.on("/", HTTP_GET, []() {
    server.send(200, "text/html", uploadPage);
  });

  // Handle firmware upload
  server.on("/update", HTTP_POST,
    // Response after upload completes
    []() {
      server.send(200, "text/plain",
        Update.hasError() ? "Update FAILED" : "Update OK. Rebooting...");
      delay(1000);
      ESP.restart();
    },
    // Handle file upload in chunks
    []() {
      HTTPUpload& upload = server.upload();
      if (upload.status == UPLOAD_FILE_START) {
        Serial.printf("Update: %s\n", upload.filename.c_str());
        if (!Update.begin(UPDATE_SIZE_UNKNOWN)) {
          Update.printError(Serial);
        }
      } else if (upload.status == UPLOAD_FILE_WRITE) {
        if (Update.write(upload.buf, upload.currentSize) != upload.currentSize) {
          Update.printError(Serial);
        }
      } else if (upload.status == UPLOAD_FILE_END) {
        if (Update.end(true)) {
          Serial.printf("Update Success: %u bytes\n", upload.totalSize);
        } else {
          Update.printError(Serial);
        }
      }
    }
  );

  server.begin();
  Serial.println("HTTP OTA server ready at http://" + WiFi.localIP().toString());
}

void loop() {
  server.handleClient();
}
```

**Usage:**
1. Flash this sketch via USB
2. Open `http://<device-ip>` in your browser
3. Select a compiled `.bin` file and click upload
4. The device reboots with the new firmware

### Export the Firmware Binary

In Arduino IDE: **Sketch > Export Compiled Binary** to generate the `.bin` file for upload.

## Method 3: Remote HTTP OTA (Pull-based)

The device periodically checks a remote server for new firmware and auto-updates. Ideal for production deployments.

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <Update.h>

const char* WIFI_SSID     = "YourSSID";
const char* WIFI_PASS     = "YourPassword";
const char* FIRMWARE_URL  = "http://yourserver.com/firmware/latest.bin";
const char* CURRENT_VER   = "1.0.0";
const char* VERSION_URL   = "http://yourserver.com/firmware/version.txt";

void checkForUpdate() {
  HTTPClient http;

  // Check latest version
  http.begin(VERSION_URL);
  int httpCode = http.GET();
  if (httpCode != 200) {
    http.end();
    return;
  }

  String latestVersion = http.getString();
  latestVersion.trim();
  http.end();

  if (latestVersion == CURRENT_VER) {
    Serial.println("Firmware is up to date.");
    return;
  }

  Serial.println("New firmware available: " + latestVersion);

  // Download and apply firmware
  http.begin(FIRMWARE_URL);
  httpCode = http.GET();
  if (httpCode != 200) {
    http.end();
    return;
  }

  int contentLength = http.getSize();
  WiFiClient* stream = http.getStreamPtr();

  if (!Update.begin(contentLength)) {
    Serial.println("Not enough space for OTA");
    http.end();
    return;
  }

  size_t written = Update.writeStream(*stream);
  if (written == contentLength) {
    Serial.println("OTA written successfully");
  }

  if (Update.end()) {
    Serial.println("OTA update complete. Rebooting...");
    ESP.restart();
  } else {
    Serial.println("OTA error: " + String(Update.errorString()));
  }

  http.end();
}

void setup() {
  Serial.begin(115200);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);

  checkForUpdate();
}

void loop() {
  // Check for updates periodically (e.g., every hour)
  delay(3600000);
  checkForUpdate();
}
```

## OTA Best Practices

### 1. Always Include OTA Code in Your Firmware

If you deploy a firmware without OTA support, you lose the ability to update remotely. Always include an OTA handler.

### 2. Use Rollback Protection

ESP-IDF supports automatic rollback if the new firmware crashes on first boot:

```cpp
#include <esp_ota_ops.h>

void setup() {
  // Mark the current firmware as valid after successful boot
  // If this is not called, the bootloader will roll back on next reboot
  esp_ota_mark_app_valid_cancel_rollback();

  // Your setup code...
}
```

### 3. Validate Before Reboot

Check firmware integrity before committing:

```cpp
if (Update.end(true)) {  // true = set MD5 check
  if (Update.isFinished()) {
    Serial.println("Update verified. Rebooting...");
    ESP.restart();
  }
}
```

### 4. Partition Size Limits

The default partition scheme limits firmware to ~1.25 MB per slot. If your firmware exceeds this:
- Use a custom partition table with larger app partitions
- Reduce SPIFFS size if filesystem space is not needed

### 5. Secure Your OTA

For production devices:
- Use HTTPS for firmware downloads
- Authenticate OTA requests with a password or token
- Consider signed firmware images with ESP-IDF's secure OTA

## Troubleshooting

| Issue | Solution |
|---|---|
| "Not enough space" error | Firmware exceeds partition size. Use a custom partition table or optimize code size. |
| OTA upload hangs | Ensure stable Wi-Fi. Move closer to access point. Check firewall rules. |
| Device won't boot after OTA | Hold BOOT + press RESET to enter download mode. Flash a known-good firmware via USB. |
| ArduinoOTA port not visible | Ensure device and computer are on the same network/subnet. Check mDNS/Bonjour is installed. |

## Further Reading

- [Getting Started](/XIAO_ESP32C3_Getting_Started) - Initial setup and flashing via USB
- [WiFi Usage](/XIAO_ESP32C3_WiFi_Usage) - Wi-Fi connectivity fundamentals
- [Architecture Overview](/XIAO_ESP32C3_Architecture) - Flash partitions and boot process
