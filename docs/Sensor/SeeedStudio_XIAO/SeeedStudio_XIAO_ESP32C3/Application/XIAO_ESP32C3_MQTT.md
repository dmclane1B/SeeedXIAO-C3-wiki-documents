---
description: MQTT communication with Seeed Studio XIAO ESP32C3
title: MQTT Communication
keywords:
- xiao
- esp32c3
- mqtt
- iot
image: https://files.seeedstudio.com/wiki/wiki-platform/S-tempor.png
slug: /xiao_esp32c3_mqtt
last_update:
  date: 02/07/2026
  author: Claude
---

# MQTT Communication with XIAO ESP32C3

MQTT (Message Queuing Telemetry Transport) is a lightweight messaging protocol ideal for IoT devices. This guide covers connecting the XIAO ESP32C3 to MQTT brokers, publishing sensor data, subscribing to commands, and building reliable IoT communication.

## What is MQTT?

MQTT uses a **publish/subscribe** model:

```
                    ┌──────────┐
  Sensor Node ────► │  MQTT    │ ────► Dashboard App
  (publish)         │  Broker  │       (subscribe)
                    │          │
  Control App ────► │ (e.g.,   │ ────► Actuator Node
  (publish)         │ Mosquitto│       (subscribe)
                    └──────────┘
```

- **Broker**: Central server that routes messages (e.g., Mosquitto, HiveMQ, EMQX)
- **Topic**: Named channel for messages (e.g., `home/livingroom/temperature`)
- **QoS**: Quality of Service levels (0 = at most once, 1 = at least once, 2 = exactly once)

## Hardware Setup

- Connect the **Wi-Fi antenna** to the IPEX connector on the XIAO ESP32C3
- Connect the XIAO ESP32C3 to your computer via USB-C

## Installing the MQTT Library

In Arduino IDE, install the **PubSubClient** library:

1. Go to **Sketch > Include Library > Manage Libraries**
2. Search for **PubSubClient** by Nick O'Leary
3. Click **Install**

## Basic MQTT: Publish & Subscribe

### Connect to a Public Broker

This example connects to a public test broker and publishes/subscribes to topics:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

// Wi-Fi credentials
const char* WIFI_SSID = "YourSSID";
const char* WIFI_PASS = "YourPassword";

// MQTT broker (public test broker)
const char* MQTT_SERVER = "broker.hivemq.com";
const int   MQTT_PORT   = 1883;
const char* CLIENT_ID   = "xiao-esp32c3-001";

// Topics
const char* TOPIC_PUB = "xiao/esp32c3/sensor";
const char* TOPIC_SUB = "xiao/esp32c3/command";

WiFiClient espClient;
PubSubClient mqtt(espClient);

// Callback for incoming messages
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  Serial.printf("Message on [%s]: ", topic);

  String message;
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.println(message);

  // React to commands
  if (message == "led_on") {
    digitalWrite(LED_BUILTIN, LOW);   // LED on (active low)
  } else if (message == "led_off") {
    digitalWrite(LED_BUILTIN, HIGH);  // LED off
  }
}

void connectWiFi() {
  Serial.printf("Connecting to %s", WIFI_SSID);
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.printf("\nConnected. IP: %s\n", WiFi.localIP().toString().c_str());
}

void connectMQTT() {
  while (!mqtt.connected()) {
    Serial.print("Connecting to MQTT...");
    if (mqtt.connect(CLIENT_ID)) {
      Serial.println("connected!");
      mqtt.subscribe(TOPIC_SUB);
      Serial.printf("Subscribed to: %s\n", TOPIC_SUB);
    } else {
      Serial.printf("failed, rc=%d. Retrying in 5s\n", mqtt.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);
  digitalWrite(LED_BUILTIN, HIGH);

  connectWiFi();
  mqtt.setServer(MQTT_SERVER, MQTT_PORT);
  mqtt.setCallback(mqttCallback);
  connectMQTT();
}

void loop() {
  if (!mqtt.connected()) {
    connectMQTT();
  }
  mqtt.loop();

  // Publish sensor data every 10 seconds
  static unsigned long lastPublish = 0;
  if (millis() - lastPublish > 10000) {
    float temperature = temperatureRead();  // Internal temp sensor
    String payload = "{\"temp\":" + String(temperature, 1) + "}";

    mqtt.publish(TOPIC_PUB, payload.c_str());
    Serial.printf("Published: %s\n", payload.c_str());
    lastPublish = millis();
  }
}
```

## MQTT with Sensor Data (DHT22 Example)

A practical example publishing temperature and humidity readings:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHT.h>

#define DHTPIN D2       // DHT22 data pin connected to D2
#define DHTTYPE DHT22

const char* WIFI_SSID  = "YourSSID";
const char* WIFI_PASS  = "YourPassword";
const char* MQTT_SERVER = "192.168.1.100";  // Your local broker IP
const int   MQTT_PORT   = 1883;
const char* MQTT_USER   = "user";           // If authentication required
const char* MQTT_PASS   = "password";

WiFiClient espClient;
PubSubClient mqtt(espClient);
DHT dht(DHTPIN, DHTTYPE);

void connectMQTT() {
  while (!mqtt.connected()) {
    Serial.print("Connecting MQTT...");
    if (mqtt.connect("xiao-dht22", MQTT_USER, MQTT_PASS)) {
      Serial.println("connected!");
    } else {
      Serial.printf("failed (%d), retry in 5s\n", mqtt.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  dht.begin();

  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) delay(500);

  mqtt.setServer(MQTT_SERVER, MQTT_PORT);
  connectMQTT();
}

void loop() {
  if (!mqtt.connected()) connectMQTT();
  mqtt.loop();

  static unsigned long lastRead = 0;
  if (millis() - lastRead > 30000) {  // Every 30 seconds
    float temp = dht.readTemperature();
    float hum  = dht.readHumidity();

    if (!isnan(temp) && !isnan(hum)) {
      // Publish to separate topics for easy dashboard integration
      mqtt.publish("home/bedroom/temperature", String(temp, 1).c_str(), true);
      mqtt.publish("home/bedroom/humidity", String(hum, 1).c_str(), true);

      // Also publish as JSON
      String json = "{\"temperature\":" + String(temp, 1) +
                    ",\"humidity\":" + String(hum, 1) + "}";
      mqtt.publish("home/bedroom/climate", json.c_str());

      Serial.printf("Temp: %.1f C, Humidity: %.1f%%\n", temp, hum);
    }
    lastRead = millis();
  }
}
```

## Secure MQTT (TLS)

For production, use TLS-encrypted MQTT connections:

```cpp
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>

const char* MQTT_SERVER = "your-broker.com";
const int   MQTT_PORT   = 8883;  // TLS port

// Root CA certificate of your broker
const char* root_ca = R"EOF(
-----BEGIN CERTIFICATE-----
... your broker's CA certificate ...
-----END CERTIFICATE-----
)EOF";

WiFiClientSecure espClient;
PubSubClient mqtt(espClient);

void setup() {
  Serial.begin(115200);
  WiFi.begin("SSID", "password");
  while (WiFi.status() != WL_CONNECTED) delay(500);

  espClient.setCACert(root_ca);
  mqtt.setServer(MQTT_SERVER, MQTT_PORT);

  while (!mqtt.connected()) {
    if (mqtt.connect("xiao-secure", "user", "pass")) {
      Serial.println("Secure MQTT connected!");
    } else {
      delay(5000);
    }
  }
}

void loop() {
  mqtt.loop();
}
```

## MQTT with Deep Sleep

Combine MQTT with deep sleep for battery-powered sensor nodes:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <esp_sleep.h>

RTC_DATA_ATTR int messageCount = 0;

const uint64_t SLEEP_DURATION = 300 * 1000000ULL;  // 5 minutes

WiFiClient espClient;
PubSubClient mqtt(espClient);

void setup() {
  Serial.begin(115200);
  messageCount++;

  // 1. Connect Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin("SSID", "password");
  unsigned long wifiStart = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - wifiStart < 10000) {
    delay(100);
  }

  if (WiFi.status() != WL_CONNECTED) {
    goToSleep();  // Skip this cycle if Wi-Fi fails
    return;
  }

  // 2. Connect MQTT
  mqtt.setServer("broker.hivemq.com", 1883);
  if (mqtt.connect("xiao-sleepy")) {
    // 3. Read and publish sensor data
    float temp = temperatureRead();
    String payload = "{\"temp\":" + String(temp, 1) +
                     ",\"msg\":" + String(messageCount) + "}";
    mqtt.publish("xiao/sensor", payload.c_str());
    mqtt.disconnect();
    Serial.println("Published: " + payload);
  }

  // 4. Go to sleep
  goToSleep();
}

void goToSleep() {
  WiFi.disconnect(true);
  WiFi.mode(WIFI_OFF);
  esp_sleep_enable_timer_wakeup(SLEEP_DURATION);
  Serial.println("Sleeping...");
  Serial.flush();
  esp_deep_sleep_start();
}

void loop() {}
```

## Integration with Home Assistant

Home Assistant auto-discovers MQTT devices. Publish discovery messages for automatic integration:

```cpp
void publishDiscovery() {
  // Temperature sensor auto-discovery
  String discoveryTopic = "homeassistant/sensor/xiao_temp/config";
  String discoveryPayload = "{"
    "\"name\":\"XIAO Temperature\","
    "\"state_topic\":\"xiao/sensor/temperature\","
    "\"unit_of_measurement\":\"°C\","
    "\"device_class\":\"temperature\","
    "\"unique_id\":\"xiao_esp32c3_temp\","
    "\"device\":{"
      "\"identifiers\":[\"xiao_esp32c3_001\"],"
      "\"name\":\"XIAO ESP32C3\","
      "\"manufacturer\":\"Seeed Studio\","
      "\"model\":\"XIAO ESP32C3\""
    "}"
  "}";

  mqtt.publish(discoveryTopic.c_str(), discoveryPayload.c_str(), true);
}
```

## Troubleshooting

| Issue | Solution |
|---|---|
| Connection refused | Check broker IP/port. Ensure broker allows anonymous connections or provide credentials. |
| Frequent disconnects | Increase `mqtt.setKeepAlive(60)`. Ensure stable Wi-Fi. Call `mqtt.loop()` frequently. |
| Messages not received | Verify topic names match exactly (case-sensitive). Check QoS settings. |
| Payload too large | PubSubClient default max is 256 bytes. Increase: `mqtt.setBufferSize(1024)`. |
| TLS handshake fails | Verify certificate is correct. Check system time is accurate (`configTime()`). |

## Further Reading

- [WiFi Usage](/XIAO_ESP32C3_WiFi_Usage) - Wi-Fi connectivity setup
- [Power Management](/XIAO_ESP32C3_Power_Management) - Battery and deep sleep optimization
- [OTA Updates](/XIAO_ESP32C3_OTA_Updates) - Remote firmware updates
- [ESPHome for XIAO](/xiao-esp32c3-esphome) - No-code Home Assistant integration
