#include <WiFi.h>
#include <PubSubClient.h>

// WiFi & MQTT Configuration
const char* ssid = "Tik";
const char* password = "paul8899";
const char* mqtt_server = "test.mosquitto.org";
const int mqtt_port = 1883;

WiFiClient espClient;
PubSubClient client(espClient);

String clientId = "ESP32-" + String(random(1000, 9999));

// AD8232 Pin Definitions
#define LEADS_OFF_PIN1 35
#define LEADS_OFF_PIN2 34
#define EMG_PIN 36

double meanValue = 0;
double inputsCount = 0;
double diffVal;
double last50[50];
int i;
int emgLevel = 0;

bool lastLeadsOff = false;

unsigned long lastHeartbeat = 0;
unsigned long lastMQTTAttempt = 0;

// ตัวแปรสำหรับ hold ส่ง EMG level ครั้งถัดไปเป็นเวลา 10 วินาที
bool levelHoldActive = false;
unsigned long levelHoldStart = 0;

void setup_wifi() {
  Serial.begin(115200);
  delay(10);
  Serial.print("Connecting to ");
  Serial.println(ssid);
  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nWiFi connected");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void reconnect() {
  // พยายาม reconnect ทุก ๆ 5 วินาที
  if (millis() - lastMQTTAttempt < 5000) return;
  lastMQTTAttempt = millis();

  Serial.print("Attempting MQTT connection...");
  if (client.connect(clientId.c_str())) {
    Serial.println(" connected");
  } else {
    Serial.print(" failed, rc=");
    Serial.print(client.state());
    Serial.println(" try again in 5 sec");
  }
}

void setup() {
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);

  pinMode(LEADS_OFF_PIN1, INPUT);
  pinMode(LEADS_OFF_PIN2, INPUT);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // ส่ง heartbeat ทุก 3 วินาที
  if (millis() - lastHeartbeat >= 3000) {
    client.publish("emg/heartbeat", "5"); 
    lastHeartbeat = millis();
  }

  // อ่านสถานะ Leads Off
  bool leadsOffNow = (digitalRead(LEADS_OFF_PIN1) == HIGH || digitalRead(LEADS_OFF_PIN2) == HIGH);

  // ส่งสถานะ Leads on/off เมื่อมีการเปลี่ยนแปลง
  if (leadsOffNow != lastLeadsOff) {
    if (leadsOffNow) {
      Serial.println("Leads off detected!");
      client.publish("emg/status", "Leads off");
    } else {
      Serial.println("Leads on");
      client.publish("emg/status", "Leads on");
    }
    lastLeadsOff = leadsOffNow;
  }

  // ถ้า leads on (ไม่ off) ให้คำนวณ EMG แล้วส่ง (ถ้าไม่อยู่ในช่วง hold)
  if (!leadsOffNow) {
    inputsCount++;
    int rawValue = analogRead(EMG_PIN);
    meanValue = (meanValue * (inputsCount - 1) + rawValue) / inputsCount;
    diffVal = abs(rawValue - meanValue);

    if (inputsCount <= 50) {
      last50[int(inputsCount) - 1] = diffVal;
    } else {
      for (i = 0; i < 49; i++) {
        last50[i] = last50[i + 1];
      }
      last50[49] = diffVal;
    }

    double sum = 0;
    for (i = 0; i < 50; i++) {
      sum += last50[i];
    }

    double scaled = sum * 4.0 / 12000.0;

    if (scaled < 1.0)
      emgLevel = 0;
    else if (scaled < 1.75)
      emgLevel = 1;
    else if (scaled < 2.25)
      emgLevel = 2;
    else if (scaled < 3.0)
      emgLevel = 3;
    else
      emgLevel = 4;

    // ส่ง EMG Level เฉพาะเมื่อไม่ได้อยู่ในช่วง hold 10 วินาที
    if (!levelHoldActive) {
      Serial.print("EMG Level: ");
      Serial.println(emgLevel);
      char msg[10];
      sprintf(msg, "%d", emgLevel);
      client.publish("emg/level", msg);
      levelHoldActive = true;
      levelHoldStart = millis();
    } else {
      if (millis() - levelHoldStart >= 10000) {
        levelHoldActive = false;
      }
    }
  }

  delay(100);
}
