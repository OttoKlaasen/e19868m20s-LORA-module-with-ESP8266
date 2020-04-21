# e19868m20s-LORA-module-with-ESP8266
EBYTE e19868m20s LORA module with ESP8266, how to connect and example software

The EBYTE e19868m20s has to be soldered on a breadboard , see my pictures.
The EBYTE e19868m20s is normally used with a SMD designed PCB.

Connection with the ESP8266 as receiver:

ESP8266-e19868m20s
-----------------
D0-DI00

D3-RESET

D5-SCK

D6-MISO

D7-MOSI

D8-N

GND-GND

EBYTE e19868m20s needs 3V3, do not use the 3v3 from the ESP8266, use a separated supply 3v3 which a regulator with a 5V input and 3v3 output. In one of the picture you can see how this was done with the assembly.

The following code is for the receiver, it receives temp and hum data from another LORA transmitter, the transmitter code is available on request:

/*********
  Lora receiver for Temp and Humidity
  This version shows also MQTT function, it will send the Temp and Hum recieved from the Lora Tranmitter to the MQTT server (Raspberry PI)
  Otto Klaasen 2019 (C)
*********/

#include <SPI.h>
#include <LoRa.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// Added for MQTT
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

// Replace with your network credentials
const char* ssid     = "your ssid";
const char* password = "your password";

// Change the variable to your Raspberry Pi IP address, so it connects to your MQTT broker
const char* mqtt_server = "192.168.1.25";

// Initializes the espClient. You should change the espClient name if you have multiple ESPs running in your home automation system
WiFiClient espClient;
PubSubClient client(espClient);

#define SCREEN_WIDTH 128 // OLED display width, in pixels
#define SCREEN_HEIGHT 64 // OLED display height, in pixels

// Declaration for an SSD1306 display connected to I2C (SDA, SCL pins)
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

//define the pins used by the transceiver module

#define ss D8
#define rst D3
#define dio0 D0

#define MyLed D4

int PackedCount = 0;            // Recieved packages
String LoRaData;                // Lora Packet recieved data
// Parameters to get Temperature
String SearchTemp = "#Temp: ";  // String to search for in Lora packet
String RecievedTemp;            // Extracted temperature
int TempIndex;                  // Index for temp extraction
// Parameters to get Humidity
String SearchHum = "#Humidity: ";  // String to search for in Lora packet
String RecievedHum;            // Extracted temperature
int HumIndex;                  // Index for Humidity extraction

// Lora time out parameters
unsigned long PacketMillis = millis();  // Time out parameter for the connection
const unsigned long TimeOut = 2400000;    // 2400 (40 min) sec to check if connection is out, transmitter send every 30 min.

// MQTT time out parameters
unsigned long MQTTMillis = millis();  // Time out parameter for the connection
const unsigned long MQTTTimeOut = 180000;    // 180 sec to check if connection is out

String SearchLora = "#Counter: ";  // String to search for in Lora packet

// MQTT required code:
// Timers auxiliar variables
long now = millis();
long lastMeasure = 0;
int MQTT_TimeOut = 0;

// ###############################################################################################
// Don't change the function below. This functions connects your ESP8266 to your router
void setup_wifi() {
  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.print("WiFi connected - ESP IP address: ");
  Serial.println(WiFi.localIP());
}
// ###############################################################################################
// This functions is executed when some device publishes a message to a topic that your ESP8266 is subscribed to
// Change the function below to add logic to your program, so when a device publishes a message to a topic that
// your ESP8266 is subscribed you can actually do something
void callback(String topic, byte* message, unsigned int length) {
  Serial.print("Message arrived on topic: ");
  Serial.print(topic);
  Serial.print(". Message: ");
  String messageTemp;

  for (int i = 0; i < length; i++) {
    Serial.print((char)message[i]);
    messageTemp += (char)message[i];
  }
  Serial.println();

  // Feel free to add more if statements to control more GPIOs with MQTT

  // If a message is received on the topic room/lamp, you check if the message is either on or off. Turns the lamp GPIO according to the message
  if (topic == "room/lamp") {
    Serial.print("Changing Room lamp to ");
    if (messageTemp == "true") {
      //digitalWrite(lamp, HIGH);
      Serial.print("On");
    }
    else if (messageTemp == "false") {
      //digitalWrite(lamp, LOW);
      Serial.print("Off");
    }
  }
  Serial.println();
}
// ###############################################################################################
// This functions reconnects your ESP8266 to your MQTT broker
// Change the function below if you want to subscribe to more topics with your ESP8266
void reconnect() {
  // Loop until we're reconnected, or when the MQTT_TimeOut parameter expires.

  while (!client.connected()) {

    Serial.print("Attempting MQTT connection...");
    // Attempt to connect
    /*
      YOU MIGHT NEED TO CHANGE THIS LINE, IF YOU'RE HAVING PROBLEMS WITH MQTT MULTIPLE CONNECTIONS
      To change the ESP device ID, you will have to give a new name to the ESP8266.
      Here's how it looks:
       if (client.connect("ESP8266Client")) {
      You can do it like this:
       if (client.connect("ESP1_Office")) {
      Then, for the other ESP:
       if (client.connect("ESP2_Garage")) {
      That should solve your MQTT multiple connections problem
    */
    if (client.connect("2_ESP8266Client")) {
      Serial.println("connected");
      // Subscribe or resubscribe to a topic
      // You can subscribe to more topics (to control more LEDs in this example)
      client.subscribe("room/lamp");
      MQTT_TimeOut = 0;   // Connected , reset the time out.
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
      MQTT_TimeOut++;
      if (MQTT_TimeOut > 1) {
        Serial.print("MQTT server time out reached: "); Serial.println(mqtt_server);
        break;
      }
    }
  }
}
// ###############################################################################################


void setup() {
  //initialize Serial Monitor
  Serial.begin(115200);

  // For MQTT
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);

  // On Board LEDS off to save power
  pinMode(MyLed, OUTPUT);
  digitalWrite(MyLed, HIGH);     // Set on board LED off to save power
  pinMode(dio0, OUTPUT);
  digitalWrite(dio0, HIGH);     // Set on board LED off to save power

  while (!Serial);
  Serial.println("LoRa Receiver");
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { // Address 0x3D for 128x64
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }
  delay(1500);

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(WHITE);
  display.setCursor(0, 10);
  // Display static text
  display.println("LoRa Receiver");
  display.display();
  delay(1000);

  //setup LoRa transceiver module
  LoRa.setPins(ss, rst, dio0);

  //replace the LoRa.begin(---E-) argument with your location's frequency
  //433E6 for Asia
  //866E6 for Europe
  //915E6 for North America
  while (!LoRa.begin(866E6)) {
    Serial.println(".");
    delay(500);
  }
  // Change sync word (0xF3) to match the receiver
  // The sync word assures you don't get LoRa messages from other LoRa transceivers
  // ranges from 0-0xFF
  LoRa.setSyncWord(0xF3);
  Serial.println("LoRa Initializing OK!");
  display.clearDisplay();
  display.setCursor(0, 10);
  // Display static text
  display.println("LoRa Initializing OK!");
  display.display();
  delay(1000);
}

void loop() {

  // Time out detection
  unsigned long currentMillis = millis();

  //MQTT connection code

  if (MQTT_TimeOut < 2) {
    if (!client.connected()) {
      reconnect();
    }
    if (!client.loop())
      client.connect("2_ESP8266Client");
  }
  else {
    if (currentMillis - MQTTMillis > MQTTTimeOut) {
      Serial.println("Retry connection with MQTT server");
      MQTTMillis = millis();
      MQTT_TimeOut = 0;
    }
  }

  // try to parse packet
  int packetSize = LoRa.parsePacket();
  // Time out handling for Lora signal
  if (currentMillis - PacketMillis > TimeOut) {
    // print signal Lost
    Serial.println("Lora signal lost.");
    // Update the display
    display.clearDisplay();
    display.setTextSize(1);
    display.setCursor(0, 10);
    display.print("Lora signal lost.");
    display.display();
  }

  if (packetSize) {
    // received a packet
    Serial.print("Received packet ");
    PackedCount++;
    PacketMillis = millis();
    // read packet
    while (LoRa.available()) {
      LoRaData = LoRa.readString();
      Serial.println(LoRaData);
      // Check if the Data is valid and not corrupted !
      TempIndex = LoRaData.indexOf(SearchLora);
      Serial.print("Lora string check parameter: "); Serial.println(TempIndex);
      if (TempIndex == 0) { // If zero than we have most likely a valid string from the Lora transmitter.
        Serial.println("Lora string packed approved.");
        // Get Temp
        TempIndex = LoRaData.indexOf(SearchTemp) + SearchTemp.length() ;      // Detect Last Index of string
        RecievedTemp = LoRaData.substring(TempIndex, TempIndex + 5);          // Get value from packet string
        Serial.print("Temperature from Received Packet: "); Serial.println(RecievedTemp);
        // Get Hum
        HumIndex = LoRaData.indexOf(SearchHum) + SearchHum.length() ;      // Detect Last Index of string
        RecievedHum = LoRaData.substring(HumIndex, HumIndex + 5);          // Get value from packet string
        Serial.print("Humidity from Received Packet: "); Serial.println(RecievedHum);

        // print RSSI of packet
        Serial.print("Signal RSSI ");
        Serial.println(LoRa.packetRssi());
        display.clearDisplay();
        display.setTextSize(1);
        display.setCursor(0, 10);

        // Display static text
        display.print("RSSI value: ");
        display.println(LoRa.packetRssi());
        display.print("Packet Counter: ");
        display.println(PackedCount);
        display.setCursor(30, 30);
        display.setTextSize(2);
        display.print(RecievedTemp);
        display.println(" C");
        display.setCursor(30, 47);
        display.print(RecievedHum);
        display.println(" %");
        display.display();

        // MQTT:
        // Length (with one extra character for the null terminator)
        int str_len = RecievedTemp.length() + 1;
        // Prepare the character array (the buffer)
        char temperatureTemp[str_len];
        // Copy it over
        RecievedTemp.toCharArray(temperatureTemp, str_len);
        Serial.print("Temp Data for MQTT: "); Serial.println(temperatureTemp);

        // Length (with one extra character for the null terminator)
        str_len = RecievedHum.length() + 1;
        // Prepare the character array (the buffer)
        char humidityTemp[str_len];
        // Copy it over
        RecievedHum.toCharArray(humidityTemp, str_len);
        Serial.print("humidity Data for MQTT: "); Serial.println(humidityTemp);

        // Publishes Temperature and Humidity values
        client.publish("room/temperature", temperatureTemp);
        client.publish("room/humidity", humidityTemp);
      }
    }
  }
}



