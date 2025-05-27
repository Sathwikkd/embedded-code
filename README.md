# embedded-code
#include <WiFi.h>
#include <PubSubClient.h>
#include <HardwareSerial.h>
#include <TinyGPS++.h>

// WiFi Configuration
const char* ssid = "RUST";
const char* password = "codersonly";

// MQTT Configuration
const char* mqtt_servers[] = {
  "broker.hivemq.com",    // Primary
  "test.mosquitto.org",   // Secondary
  "mqtt.eclipseprojects.io" // Tertiary
};
const int mqtt_port = 1883;
const char* mqtt_topic = "animal_tracking/device1";

// GPS Configuration
HardwareSerial SerialGPS(2); // UART2 on ESP32
TinyGPSPlus gps;

WiFiClient espClient;
PubSubClient client(espClient);
int current_broker = 0;
unsigned long lastPublishTime = 0;
const unsigned long publishInterval = 5000; // 5 seconds

void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }

  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("\nWiFi connection failed");
    ESP.restart();
  }

  Serial.println("\nWiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection to ");
    Serial.print(mqtt_servers[current_broker]);
    Serial.println("...");
    
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      Serial.println("Connected!");
    } else {
      Serial.print("Failed, rc=");
      Serial.print(client.state());
      Serial.println(" Trying next broker in 5s");
      
      current_broker = (current_broker + 1) % (sizeof(mqtt_servers)/sizeof(mqtt_servers[0]));
      client.setServer(mqtt_servers[current_broker], mqtt_port);
      delay(5000);
    }
  }
}

void publishGPSData() {
  if (gps.location.isValid()) {
    // Create JSON payload
    String payload = "{";
    payload += "\"lat\":" + String(gps.location.lat(), 6);
    payload += ",\"lng\":" + String(gps.location.lng(), 6);
    payload += ",\"speed\":" + String(gps.speed.kmph());
    payload += ",\"altitude\":" + String(gps.altitude.meters());
    payload += ",\"satellites\":" + String(gps.satellites.value());
    payload += ",\"hdop\":" + String(gps.hdop.value());
    payload += ",\"timestamp\":" + String(millis());
    payload += "}";

    // Publish to MQTT
    if (client.publish(mqtt_topic, payload.c_str())) {
      Serial.println("Published: " + payload);
    } else {
      Serial.println("Publish failed");
    }
  } else {
    Serial.println("Invalid GPS data");
  }
}

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

  // Read GPS data
  while (SerialGPS.available() > 0) {
    if (gps.encode(SerialGPS.read())) {
      // GPS data decoded successfully
    }
  }

  // Publish at regular intervals
  if (millis() - lastPublishTime >= publishInterval) {
    publishGPSData();
    lastPublishTime = millis();
  }
}
