
# ESP32 GPS Tracker with MQTT (Single Page)

This code connects an ESP32 to WiFi, reads GPS data via UART2, and publishes it to an MQTT broker in JSON format.

---

## 🚀 Libraries
```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <HardwareSerial.h>
#include <TinyGPS++.h>
````

---

## 📡 WiFi & MQTT Config

```cpp
const char* ssid = "RUST";
const char* password = "codersonly";

const char* mqtt_servers[] = {
  "broker.hivemq.com",
  "test.mosquitto.org",
  "mqtt.eclipseprojects.io"
};
const int mqtt_port = 1883;
const char* mqtt_topic = "animal_tracking/device1";
```

---

## 🧭 GPS Setup

```cpp
HardwareSerial SerialGPS(2); // UART2
TinyGPSPlus gps;
WiFiClient espClient;
PubSubClient client(espClient);

int current_broker = 0;
unsigned long lastPublishTime = 0;
const unsigned long publishInterval = 5000;
```

---

## 🔌 WiFi Connect

```cpp
void setup_wifi() {
  delay(10);
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  int attempts = 0;

  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }

  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("WiFi failed");
    ESP.restart();
  }

  Serial.println("WiFi connected");
  Serial.println(WiFi.localIP());
}
```

---

## 🔄 MQTT Reconnect

```cpp
void reconnect() {
  while (!client.connected()) {
    Serial.print("Connecting to ");
    Serial.println(mqtt_servers[current_broker]);

    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("Connected");
    } else {
      Serial.print("Failed, rc=");
      Serial.println(client.state());
      current_broker = (current_broker + 1) % (sizeof(mqtt_servers)/sizeof(mqtt_servers[0]));
      client.setServer(mqtt_servers[current_broker], mqtt_port);
      delay(5000);
    }
  }
}
```

---

## 📤 Publish GPS as JSON

```cpp
void publishGPSData() {
  if (gps.location.isValid()) {
    String payload = "{";
    payload += "\"lat\":" + String(gps.location.lat(), 6);
    payload += ",\"lng\":" + String(gps.location.lng(), 6);
    payload += ",\"speed\":" + String(gps.speed.kmph());
    payload += ",\"altitude\":" + String(gps.altitude.meters());
    payload += ",\"satellites\":" + String(gps.satellites.value());
    payload += ",\"hdop\":" + String(gps.hdop.value());
    payload += ",\"timestamp\":" + String(millis());
    payload += "}";

    if (client.publish(mqtt_topic, payload.c_str())) {
      Serial.println("Published: " + payload);
    } else {
      Serial.println("Publish failed");
    }
  } else {
    Serial.println("Invalid GPS data");
  }
}
```

---

## 🛠️ `setup()` and 🔁 `loop()`

```cpp
void setup() {
  Serial.begin(115200);
  SerialGPS.begin(9600, SERIAL_8N1, 16, 17); // RX=16, TX=17
  setup_wifi();
  client.setServer(mqtt_servers[current_broker], mqtt_port);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  while (SerialGPS.available() > 0) {
    if (gps.encode(SerialGPS.read())) {
      // Parsed successfully
    }
  }

  if (millis() - lastPublishTime >= publishInterval) {
    publishGPSData();
    lastPublishTime = millis();
  }
}
```

```

You can copy this and save it as `esp32_gps_mqtt.md` or paste it into a markdown viewer/editor like Typora, Obsidian, or GitHub.

Let me know if you want this in PDF, HTML, or formatted for a blog post.
```
