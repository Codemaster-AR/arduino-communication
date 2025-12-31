

---

# Arduino Communication Test (Groq AI Integration)

This project demonstrates how to connect an **Arduino MKR IoT Carrier (Rev2)** to the **Groq API** using WiFi. It polls the AI model for specific commands (`RED`, `BLUE`, `ALARM`, or `CLEAR`) and updates the hardware LEDs, buzzer, and display accordingly.

## 🚀 Getting Started

### 1. Prerequisites

Before uploading the code, ensure you have the following installed in your Arduino IDE:

* **Libraries:**
* `Arduino_MKRIoTCarrier`
* `WiFiNINA`


* **Hardware:** Arduino MKR WiFi 1010 + IoT Carrier (Rev2).

### 2. Setup

1. Open the web interface/dashboard and ensure it is fully loaded.
2. Connect your Arduino MKR IoT Carrier to your computer via USB.
3. Open the Arduino IDE and select the correct board and port.

---

## 💻 Source Code

Copy and paste the following code into your Arduino IDE.

> [!WARNING]
> **Note:** You must configure your Arduino IoT kit and enter your Wi-Fi parameters in the code before running it.

```cpp
// Download this library from Arduino:
#include <Arduino_MKRIoTCarrier.h>
// Download this library from Arduino:
#include <WiFiNINA.h>
// Download this library by Benoît Blanchon
#include <ArduinoJson.h>



// WiFi Credentials
char ssid[] = "Wifi name"; 
char pass[] = "Wifi password"; 

// Firebase Project Details
const char* firebaseHost = "firestore.googleapis.com";
const char* projectId = "agronauts-ea979";
const char* appId = "default-app-id"; 

MKRIoTCarrier carrier;
WiFiSSLClient client;

unsigned long lastCheck = 0;
const unsigned long interval = 1500; 

// Track the last command to prevent screen flickering
String lastCommand = "";

void setup() {
  Serial.begin(9600);
  
  // Initialize Carrier
  CARRIER_CASE = false; 
  if (!carrier.begin()) {
    Serial.println("Carrier initialization failed!");
    while (1);
  }

  // Flash LEDs once to show hardware is alive
  carrier.leds.fill(carrier.leds.Color(0, 50, 0), 0, 5);
  carrier.leds.show();
  delay(500);
  carrier.leds.fill(carrier.leds.Color(0, 0, 0), 0, 5);
  carrier.leds.show();

  // Force screen refresh
  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setTextColor(ST77XX_WHITE);
  carrier.display.setTextSize(2);
  carrier.display.setCursor(20, 50);
  carrier.display.println("SYSTEM START");
  carrier.display.setCursor(20, 80);
  carrier.display.println("Connecting...");

  // Connect to WiFi
  int status = WL_IDLE_STATUS;
  while (status != WL_CONNECTED) {
    status = WiFi.begin(ssid, pass);
    delay(3000);
  }

  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setCursor(20, 100);
  carrier.display.setTextColor(ST77XX_GREEN);
  carrier.display.println("CLOUD ACTIVE");
}

void loop() {
  if (millis() - lastCheck > interval) {
    checkCloudCommand();
    lastCheck = millis();
  }
}

String getFirestoreCommand() {
  client.setTimeout(8000); 

  if (client.connect(firebaseHost, 443)) {
    String path = "/v1/projects/" + String(projectId) + 
                  "/databases/(default)/documents/artifacts/" + String(appId) + 
                  "/public/data/controls/state";
    
    client.print("GET " + path + " HTTP/1.1\r\n");
    client.print("Host: " + String(firebaseHost) + "\r\n");
    client.print("Accept: application/json\r\n");
    client.print("Connection: close\r\n\r\n");

    // Skip headers
    unsigned long startHeader = millis();
    while (client.connected()) {
      if (millis() - startHeader > 4000) break;
      String line = client.readStringUntil('\n');
      if (line == "\r") break;
    }

    // Read response body carefully
    String response = "";
    unsigned long startRead = millis();
    
    // Wait for initial data
    while(client.connected() && !client.available() && (millis() - startRead < 3000));
    
    // Read the body
    while (client.available()) {
      response += (char)client.read();
      if (response.length() > 3000) break; // Memory guard
    }
    client.stop();

    // Clean response: find first { and last }
    int firstBrace = response.indexOf('{');
    int lastBrace = response.lastIndexOf('}');
    
    if (firstBrace == -1 || lastBrace == -1 || lastBrace < firstBrace) {
      return "EMPTY_RES";
    }
    
    response = response.substring(firstBrace, lastBrace + 1);

    // Dynamic document handles larger Google responses
    DynamicJsonDocument doc(3072); 
    DeserializationError error = deserializeJson(doc, response);

    if (!error) {
      if (doc.containsKey("fields") && doc["fields"].containsKey("currentCommand")) {
        const char* cmd = doc["fields"]["currentCommand"]["stringValue"];
        if (cmd) return String(cmd);
      }
      return "NO_FIELD";
    } else {
      Serial.print("Json Error: ");
      Serial.println(error.c_str());
      return "PARSE_ERR";
    }
  } else {
    return "CONN_ERR";
  }
  return "NONE";
}

void checkCloudCommand() {
  String command = getFirestoreCommand();
  
  if (command != lastCommand) {
    lastCommand = command;
    
    carrier.display.fillScreen(ST77XX_BLACK);
    carrier.display.setCursor(20, 40);
    carrier.display.setTextSize(2);
    carrier.display.setTextColor(ST77XX_WHITE);
    carrier.display.print("CLOUD: ");
    
    if (command == "PARSE_ERR" || command == "CONN_ERR" || command == "EMPTY_RES") {
       carrier.display.setTextColor(ST77XX_RED);
    } else {
       carrier.display.setTextColor(ST77XX_GREEN);
    }
    carrier.display.println(command);
    
    if (command == "RED") {
      updateHardware(255, 0, 0, "RED ALERT");
    } else if (command == "BLUE") {
      updateHardware(0, 0, 255, "BLUE MODE");
    } else if (command == "ALARM") {
      triggerAlarm();
    } else if (command == "CLEAR") {
      updateHardware(0, 0, 0, "READY");
    }
  }
}

void updateHardware(int r, int g, int b, String label) {
  carrier.leds.fill(carrier.leds.Color(r, g, b), 0, 5);
  carrier.leds.show();
  
  carrier.display.setCursor(20, 100);
  carrier.display.setTextColor(ST77XX_CYAN);
  carrier.display.setTextSize(3);
  carrier.display.println(label);
}

void triggerAlarm() {
  carrier.display.setCursor(20, 100);
  carrier.display.setTextColor(ST77XX_RED);
  carrier.display.setTextSize(3);
  carrier.display.println("ALARM!");

  for(int i=0; i<3; i++) {
    carrier.leds.fill(carrier.leds.Color(255, 0, 0), 0, 5);
    carrier.leds.show();
    carrier.Buzzer.beep(1200, 100);
    delay(100);
    carrier.leds.fill(carrier.leds.Color(0, 0, 0), 0, 5);
    carrier.leds.show();
    delay(100);
  }
}

```

---

## 🛠 Features

* **Groq AI Integration:** Uses Llama 3.1 8B via API to process logic.
* **Visual Feedback:** The OLED display shows connection status and current modes.
* **LED Indicators:** RGB LEDs change color based on the AI's response.
* **Audible Alarm:** Uses the onboard buzzer for the "ALARM" state.
* https://docs.google.com/document/d/1LEeH-elrM0l-2gsYv9CXMIsAE65ZaXYOnJdxyaRQBgI/edit?usp=sharing

