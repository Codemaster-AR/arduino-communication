Here is a professionally formatted version of your `README.md`. I have organized it into logical sections, cleaned up the instructions, and highlighted the necessary hardware/software requirements.

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
> **Security Note:** This code contains a hardcoded API key and WiFi credentials. For public repositories, consider using a `config.h` file or Secret tabs in the Arduino IDE to protect your data.

```cpp
#include <Arduino_MKRIoTCarrier.h>
#include <WiFiNINA.h>

// Groq API Configuration
const char* apiKey = "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"; 
const char* groqHost = "api.groq.com";
const char* groqModel = "llama-3.1-8b-instant";

// WiFi Credentials
char ssid[] = "WIFI name caps sensitive"; 
char pass[] = "password"; 

MKRIoTCarrier carrier;
WiFiSSLClient client;

unsigned long lastCheck = 0;
const unsigned long interval = 3000; 

void setup() {
  Serial.begin(9600);
  
  CARRIER_CASE = false;
  if (!carrier.begin()) {
    Serial.println("Carrier error!");
    while (1);
  }

  carrier.Buttons.updateConfig(130);

  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setCursor(20, 100);
  carrier.display.setTextColor(ST77XX_WHITE);
  carrier.display.setTextSize(2);
  carrier.display.println("Connecting WiFi...");

  while (WiFi.begin(ssid, pass) != WL_CONNECTED) {
    delay(2000);
    Serial.print(".");
  }

  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setCursor(20, 100);
  carrier.display.println("Groq Link Active!");
  delay(1000);
}

void loop() {
  if (millis() - lastCheck > interval) {
    checkCloudCommand();
    lastCheck = millis();
  }
}

String askGroqForCommand() {
  if (client.connect(groqHost, 443)) {
    String url = "/v1/chat/completions";
    
    // Updated prompt for absolute clarity
    String systemPrompt = "Respond ONLY with the word: RED, BLUE, ALARM, or CLEAR. No punctuation.";
    String payload = "{\"model\": \"" + String(groqModel) + "\", \"messages\": [";
    payload += "{\"role\": \"system\", \"content\": \"" + systemPrompt + "\"},";
    payload += "{\"role\": \"user\", \"content\": \"Check database.\"}";
    payload += "], \"temperature\": 0.1}";

    client.print("POST " + url + " HTTP/1.1\r\n");
    client.print("Host: " + String(groqHost) + "\r\n");
    client.print("Authorization: Bearer " + String(apiKey) + "\r\n");
    client.print("Content-Type: application/json\r\n");
    client.print("Content-Length: " + String(payload.length()) + "\r\n");
    client.print("Connection: close\r\n\r\n");
    client.print(payload);

    while (client.connected()) {
      String line = client.readStringUntil('\n');
      if (line == "\r") break;
    }

    String response = "";
    while (client.available()) {
      response += (char)client.read();
    }
    client.stop();

    response.toUpperCase();
    if (response.indexOf("RED") >= 0) return "RED";
    if (response.indexOf("BLUE") >= 0) return "BLUE";
    if (response.indexOf("ALARM") >= 0) return "ALARM";
    if (response.indexOf("CLEAR") >= 0) return "CLEAR";
  }
  return "NONE";
}

void checkCloudCommand() {
  Serial.println("Polling Groq...");
  String command = askGroqForCommand();
  Serial.println("Action: " + command);
  
  if (command == "RED") updateHardware(255, 0, 0, "RED MODE");
  else if (command == "BLUE") updateHardware(0, 0, 255, "BLUE MODE");
  else if (command == "ALARM") triggerAlarm();
  else if (command == "CLEAR") updateHardware(0, 0, 0, "OFF");
}

void updateHardware(int r, int g, int b, String label) {
  carrier.leds.fill(carrier.leds.Color(r, g, b), 0, 5);
  carrier.leds.show();
  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setCursor(40, 100);
  carrier.display.setTextColor(ST77XX_CYAN);
  carrier.display.println(label);
}

void triggerAlarm() {
  carrier.display.fillScreen(ST77XX_RED);
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

Would you like me to help you set up a `secrets.h` file to hide your WiFi and API credentials?
