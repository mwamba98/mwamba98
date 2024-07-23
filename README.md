#include <SPI.h>
#include <MFRC522.h>
#include <WiFi.h>
#include <WebServer.h>
#include <time.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// WiFi credentials
const char* ssid = "Exam Server";
const char* password = "Exam_Admin";

// RFID pins
#define SS_PIN 27
#define RST_PIN 5

// OLED display settings
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1  // Reset pin # (or -1 if sharing Arduino reset pin)
#define OLED_I2C_ADDRESS 0x3C  // I2C address for the OLED display

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

MFRC522 rfid(SS_PIN, RST_PIN);  // Create MFRC522 instance

// Structure to hold user details
struct User {
  String id;
  String name;
  String serialNumber;
  String gender;
  String dateTimeIn;
  String uid;
};

// Array to hold users
User users[] = {
  {"1", "Prudence Mwelwe", "20211351001", "Female", "", "257B07AD"},
  {"2", "Hassan Chilumba", "20211351074", "Male", "", "041A0F63A85280"}
};

const int userCount = sizeof(users) / sizeof(users[0]);

// Array to store logged entries
User loggedEntries[10];
int loggedCount = 0;

WebServer server(80);

void setup() {
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  Serial.println("RFID reader initialized");

  // Initialize OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_I2C_ADDRESS)) {
    Serial.println(F("SSD1306 allocation failed"));
    for (;;);
  }
  display.display();
  delay(2000);
  display.clearDisplay();

  // Initialize WiFi
  WiFi.softAP(ssid, password);
  Serial.println("ESP32 Hotspot started");
  Serial.print("IP Address: ");
  Serial.println(WiFi.softAPIP());
  serial.println("internet connection web")

  // Initialize time
  configTime(0, 0, "pool.ntp.org");

  // Start web server
  server.on("/", handleRoot);
  server.on("/data", handleData); // Endpoint for JSON data
  server.begin();
  Serial.println("Web server started");
}

void loop() {
  server.handleClient();

  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    String uid = "";
    for (byte i = 0; i < rfid.uid.size; i++) {
      uid += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "");
      uid += String(rfid.uid.uidByte[i], HEX);
    }
    uid.toUpperCase();

    String scannedUIDMessage = "Scanned UID: " + uid;
    Serial.println(scannedUIDMessage);
    printToOLED(scannedUIDMessage);

    int userIndex = matchUID(uid);
    if (userIndex != -1) {
      addLoggedEntry(userIndex);
      String successMessage = users[userIndex].name + " has successfully signed in.";
      Serial.println(successMessage);
      printToOLED(successMessage);
    } else {
      String unknownCardMessage = "Unknown card scanned.";
      Serial.println(unknownCardMessage);
      printToOLED(unknownCardMessage);
    }
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }
}

void handleRoot() {
  String html = "<html><body><h1>EXAMINATION BIOMETRIC VERIFICATION</h1>";

  // Table for logged entries
  html += "<h2>Logged Entries</h2><table id='entriesTable' border='1'><tr><th>ID</th><th>Name</th><th>Serial Number</th><th>Gender</th><th>Date and Time In</th></tr>";
  html += "</table>";

  // Add JavaScript for AJAX
  html += "<script>";
  html += "function updateData() {";
  html += "  var xhr = new XMLHttpRequest();";
  html += "  xhr.open('GET', '/data', true);";
  html += "  xhr.onreadystatechange = function () {";
  html += "    if (xhr.readyState == 4 && xhr.status == 200) {";
  html += "      var data = JSON.parse(xhr.responseText);";
  html += "      var entriesTable = document.getElementById('entriesTable');";
  html += "      entriesTable.innerHTML = '<tr><th>ID</th><th>Name</th><th>Serial Number</th><th>Gender</th><th>Date and Time In</th></tr>';";
  html += "      for (var i = 0; i < data.loggedEntries.length; i++) {";
  html += "        var entry = data.loggedEntries[i];";
  html += "        entriesTable.innerHTML += '<tr><td>' + entry.id + '</td><td>' + entry.name + '</td><td>' + entry.serialNumber + '</td><td>' + entry.gender + '</td><td>' + entry.dateTimeIn + '</td></tr>';";
  html += "      }";
  html += "    }";
  html += "  };";
  html += "  xhr.send();";
  html += "}";
  html += "setInterval(updateData, 5000);"; // Update data every 5 seconds
  html += "updateData();"; // Initial load
  html += "</script>";
  html += "</body></html>";

  server.send(200, "text/html", html);
}

void handleData() {
  String json = "{";
  
  // Add logged entries data
  json += "\"loggedEntries\":[";
  for (int i = 0; i < loggedCount; i++) {
    if (i > 0) json += ",";
    json += "{";
    json += "\"id\":\"" + loggedEntries[i].id + "\",";
    json += "\"name\":\"" + loggedEntries[i].name + "\",";
    json += "\"serialNumber\":\"" + loggedEntries[i].serialNumber + "\",";
    json += "\"gender\":\"" + loggedEntries[i].gender + "\",";
    json += "\"dateTimeIn\":\"" + loggedEntries[i].dateTimeIn + "\"";
    json += "}";
  }
  json += "]";
  
  json += "}";
  
  server.send(200, "application/json", json);
}

String getTimeStamp() {
  time_t now = time(nullptr);
  struct tm* timeinfo = localtime(&now);
  char buffer[80];
  strftime(buffer, 80, "%Y-%m-%d %H:%M:%S", timeinfo);
  return String(buffer);
}

void addLoggedEntry(int userIndex) {
  if (loggedCount < 10) {
    loggedEntries[loggedCount] = users[userIndex];
    loggedEntries[loggedCount].dateTimeIn = getTimeStamp();
    loggedCount++;
  }
}

int matchUID(String uid) {
  uid.toUpperCase(); // Ensure UID is in uppercase
  for (int i = 0; i < userCount; i++) {
    if (users[i].uid == uid) {
      Serial.print("Match found for UID: ");
      Serial.println(uid);
      return i;
    }
  }
  Serial.print("No match found for UID: ");
  Serial.println(uid);
  return -1;
}

void printToOLED(String message) {
  display.clearDisplay();
  display.setTextSize(1);             // Normal 1:1 pixel scale
  display.setTextColor(SSD1306_WHITE); // Draw white text
  display.setCursor(0, 0);            // Start at top-left corner
  display.println(message);
  display.display();
}
