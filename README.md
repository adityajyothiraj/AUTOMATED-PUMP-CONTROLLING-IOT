# AUTOMATED-PUMP-CONTROLLING-IOT
The Automatic Pump Controlling System is an IoT-based solution designed to manage pump operations efficiently. The system consists of three main controllers that interact with each other and a central Firebase database to monitor and control the pump.

![neer_main (1)](https://github.com/user-attachments/assets/9bc845ec-f9f7-4b31-91c6-f38c46c28318)

## CODE MAIN
```
#include <Arduino.h>
#include <Wire.h>
#include <FirebaseClient.h>
#include "ExampleFunctions.h" // Provides the functions used in the examples.
#include <otadrive_esp.h>
#include <LiquidCrystal_I2C.h>

#define led_Off 5
#define led_On 16
#define led_Auto 17
#define button_Off 34
#define button_On 35
#define button_Auto 21
#define relay 23
#define I2C_SDA 32
#define I2C_SCL 33

#define WIFI_SSID "PUMP"
#define WIFI_PASSWORD "Pump*123"

#define APIKEY "c6eab4fa-591a-4672-ac9d-d040bc4935cd"
#define FW_VER "1.0.1"

#define API_KEY "AIzaSyBMaU5oFpnm64xA1wJoGN3u2c3LyQtX4oM"
#define USER_EMAIL "adityajyothirajyt@gmail.com"
#define USER_PASSWORD "Firebase123"
#define DATABASE_URL "https://neer-d1234-default-rtdb.firebaseio.com/"
/*
IPAddress local_ip(192, 168, 1, 100);  // Static IP address
IPAddress gateway(192, 168, 1, 1);     // Router's gateway IP
IPAddress subnet(255, 255, 255, 0);    // Subnet mask
*/
SSL_CLIENT ssl_client;
// This uses built-in core WiFi/Ethernet for network connection.
// See examples/App/NetworkInterfaces for more network examples.
using AsyncClient = AsyncClientClass;
AsyncClient aClient(ssl_client);

UserAuth user_auth(API_KEY, USER_EMAIL, USER_PASSWORD, 3000 /* expire period in seconds (<3600) */);
FirebaseApp app;
RealtimeDatabase Database;

void Connect_WiFi();
LiquidCrystal_I2C lcd(0x27, 16, 2);
String filter(String);

unsigned long lastFirebaseQueryTime = 0;
unsigned long firebaseQueryInterval = 500;  // 5 seconds interval

String dataBase = "  ";
int On = 0, Off = 0, Auto = 0;

const int debounceDelay = 50;  // Debounce delay in milliseconds
unsigned long lastDebounceTime_Off = 0;
unsigned long lastDebounceTime_On = 0;
unsigned long lastDebounceTime_Auto = 0;

// Flags for interrupt handling
volatile bool buttonOffPressed = false;
volatile bool buttonOnPressed = false;
volatile bool buttonAutoPressed = false;

// Interrupt Service Routines (ISRs) for each button press
void IRAM_ATTR handleButtonOff() {
  buttonOffPressed = true;
}

void IRAM_ATTR handleButtonOn() {
  buttonOnPressed = true;
}

void IRAM_ATTR handleButtonAuto() {
  buttonAutoPressed = true;
}

void setup() {
  Connect_WiFi();
  
  pinMode(led_Off, OUTPUT);
  pinMode(led_On, OUTPUT);
  pinMode(led_Auto, OUTPUT);
  pinMode(button_Off, INPUT_PULLUP);  // Use pull-up resistor for button
  pinMode(button_On, INPUT_PULLUP);
  pinMode(button_Auto, INPUT_PULLUP);
  pinMode(relay, OUTPUT);

  lcd.init(I2C_SDA, I2C_SCL);                      
  lcd.backlight();                                  
  lcd.clear();    
  lcd.setCursor(4, 0);
  lcd.print("NEER H20");
  lcd.setCursor(3, 1);
  lcd.print("SAVE WATER");

  OTADRIVE.setInfo(APIKEY, FW_VER);

  // Attach interrupts for each button
  attachInterrupt(digitalPinToInterrupt(button_Off), handleButtonOff, FALLING);
  attachInterrupt(digitalPinToInterrupt(button_On), handleButtonOn, FALLING);
  attachInterrupt(digitalPinToInterrupt(button_Auto), handleButtonAuto, FALLING);
}

void loop() {
  app.loop();
  //yield();

  unsigned long currentMillis = millis();

  // Firebase operations at intervals
  if (currentMillis - lastFirebaseQueryTime >= firebaseQueryInterval) {
    lastFirebaseQueryTime = currentMillis;
    dataBase = filter(Database.get<String>(aClient, "DATABASE"));
  }

  // Handle button presses using interrupt flags (debouncing already handled in ISRs)
  if (buttonOffPressed && (millis() - lastDebounceTime_Off) > debounceDelay) {
    if(dataBase[1] == '0'){
      dataBase[0] = '1';
    }
    else if(dataBase[1] == '1'){
      dataBase[1] = '0';
      dataBase[0] = '1';
    }
    Database.set<String>(aClient, "DATABASE", dataBase);
    lastDebounceTime_Off = millis();
    buttonOffPressed = false;  // Reset the flag
  }

  if (buttonOnPressed && (millis() - lastDebounceTime_On) > debounceDelay) {
    if(dataBase[1] == '0'){
      dataBase[0] = '2';
    }
    else if(dataBase[1] == '1'){
      dataBase[1] = '0';
      dataBase[0] = '2';
    }
    Database.set<String>(aClient, "DATABASE", dataBase);
    lastDebounceTime_On = millis();
    buttonOnPressed = false;  // Reset the flag
  }

  if (buttonAutoPressed && (millis() - lastDebounceTime_Auto) > debounceDelay) {
    if (dataBase[1] == '0') {
      dataBase[1] = '1';
    } else if (dataBase[1] == '1') {
      dataBase[1] = '0';
    }
    Database.set<String>(aClient, "DATABASE", dataBase);
    lastDebounceTime_Auto = millis();
    buttonAutoPressed = false;  // Reset the flag
  }

  // Logic for button and auto state
  if (dataBase[0] == '1') {
    lcd.setCursor(0, 1);
    lcd.print("     PUMP OFF    ");
    digitalWrite(led_On, LOW);
    digitalWrite(led_Off, HIGH);
    digitalWrite(relay, LOW);
  } else if (dataBase[0] == '2') {
    lcd.setCursor(0, 1);
    lcd.print("     PUMP ON   ");
    digitalWrite(led_Off, LOW);
    digitalWrite(led_On, HIGH);
    digitalWrite(relay, HIGH);
  }

  if (dataBase[1] == '0' && dataBase[0] == '1') {
    digitalWrite(led_Auto, LOW);
    lcd.setCursor(0, 1);
    lcd.print("     PUMP OFF    ");
  } else if (dataBase[1] == '0' && dataBase[0] == '2') {
    digitalWrite(led_Auto, LOW);
    lcd.setCursor(0, 1);
    lcd.print("     PUMP ON    ");
  } else if (dataBase[1] == '1' && dataBase[0] == '1') {
    digitalWrite(led_Auto, HIGH);
    lcd.setCursor(0, 1);
    lcd.print("  PUMP OFF  AUTO");
    delay(500);
  } else if (dataBase[1] == '1' && dataBase[0] == '2') {
    digitalWrite(led_Auto, HIGH);
    lcd.setCursor(0, 1);
    lcd.print("  PUMP ON   AUTO");
    delay(500);
  }
  //yield();
}

String filter(String msg) {
  msg.replace("[", "");
  msg.replace("]", "");
  msg.replace("\\", "");
  msg.replace("\"", "");
  return msg;
}

void Connect_WiFi() {
  Serial.begin(9600);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  //WiFi.config(local_ip, gateway, subnet);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED)
  {
    if (OTADRIVE.timeTick(30)) {
      OTADRIVE.updateFirmware();
    }
    Serial.print(".");
    delay(300);
  }
  Serial.println();
  Serial.print("Connected with IP: ");
  Serial.println(WiFi.localIP());
  Serial.println();

  Firebase.printf("Firebase Client v%s\n", FIREBASE_CLIENT_VERSION);
  set_ssl_client_insecure_and_buffer(ssl_client);

  Serial.println("Initializing app...");
  initializeApp(aClient, app, getAuth(user_auth), 120 * 1000, auth_debug_print);
  app.getApp<RealtimeDatabase>(Database);
  Database.url(DATABASE_URL);
}

```
## CODE TANK
```
#include <Arduino.h>
#include <stdlib.h>
#include "time.h"
#include <otadrive_esp.h>
#include <WiFi.h>
#include <FirebaseClient.h>
#include "ExampleFunctions.h" // Provides the functions used in the examples.

#define float_bottom 32
#define float_top 33

#define WIFI_SSID "PUMP"
#define WIFI_PASSWORD "Pump*123"

#define APIKEY "81844a75-57ba-4ca7-9f21-17e0e73a5582"
#define FW_VER "1.0.1"

#define API_KEY "AIzaSyBMaU5oFpnm64xA1wJoGN3u2c3LyQtX4oM"
#define USER_EMAIL "adityajyothirajyt@gmail.com"
#define USER_PASSWORD "Firebase123"
#define DATABASE_URL "https://neer-d1234-default-rtdb.firebaseio.com/"

const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 19800;
const int   daylightOffset_sec = 3600;

SSL_CLIENT ssl_client;
// This uses built-in core WiFi/Ethernet for network connection.
// See examples/App/NetworkInterfaces for more network examples.
using AsyncClient = AsyncClientClass;
AsyncClient aClient(ssl_client);

UserAuth user_auth(API_KEY, USER_EMAIL, USER_PASSWORD, 3000 /* expire period in seconds (<3600) */);
FirebaseApp app;
RealtimeDatabase Database;

void Connect_WiFi();
String filter(String);

unsigned long lastFirebaseQueryTime = 0;
unsigned long firebaseQueryInterval = 50;  // 5 seconds interval
String buttonState; String autoState;
int bottom; int top; int hour=0;
char timeHour[3];

void setup() {
  Connect_WiFi();
  pinMode(float_bottom, INPUT_PULLUP);
  pinMode(float_top, INPUT_PULLUP);
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  OTADRIVE.setInfo(APIKEY, FW_VER);
}

void loop() {
  app.loop();
  yield(); 

  unsigned long currentMillis = millis();
  if (currentMillis - lastFirebaseQueryTime >= firebaseQueryInterval) {
    lastFirebaseQueryTime = currentMillis;
    buttonState = filter(Database.get<String>(aClient, "BUTTONSTATE"));
    autoState = filter(Database.get<String>(aClient, "AUTOSTATE"));
    //Serial.print("BUTTONSTATE = ");
    //Serial.println(buttonState);
    //Serial.print("AUTOSTATE = ");
    //Serial.println(autoState);
  }

  bottom = digitalRead(float_bottom);
  top = digitalRead(float_top);
  Serial.print("bottom = ");
  Serial.println(bottom);
  Serial.print("top = ");
  Serial.println(top);

  struct tm timeinfo;
  if(!getLocalTime(&timeinfo)){
    Serial.println("Failed to obtain time");
    //WiFi.disconnect();
    //WiFi.begin();
    return;
  }
  
  strftime(timeHour,3, "%H", &timeinfo);
  hour = atoi(timeHour);
  Serial.print("Time = ");
  Serial.println(hour);
  
  if(autoState == "1")
  {
    if(bottom == 0 && top == 0 && hour >=5 && hour <= 23)
    {
      Database.set<String>(aClient, "BUTTONSTATE", "1");
    }

    if(bottom == 1 && top == 1)
    {
      Database.set<String>(aClient, "BUTTONSTATE", "0");
    }
  }

   if(autoState == "0")
  {
    if(bottom == 1 && top == 1)
    {
      Database.set<String>(aClient, "BUTTONSTATE", "0");
    }
  }
  //yield();
}

String filter(String msg)
{
  msg.replace("[", "");
  msg.replace("]", "");
  msg.replace("\\", "");
  msg.replace("\"", "");
  return msg;
}


void Connect_WiFi() {
  Serial.begin(9600);
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

    Serial.print("Connecting to Wi-Fi");
    while (WiFi.status() != WL_CONNECTED)
    {
      if (OTADRIVE.timeTick(30)) {
        //OTADRIVE.updateFirmware();
      }
        Serial.print(".");
        delay(300);
    }
    Serial.println();
    Serial.print("Connected with IP: ");
    Serial.println(WiFi.localIP());
    Serial.println();

    Firebase.printf("Firebase Client v%s\n", FIREBASE_CLIENT_VERSION);
    set_ssl_client_insecure_and_buffer(ssl_client);

    Serial.println("Initializing app...");
    initializeApp(aClient, app, getAuth(user_auth), 120 * 1000, auth_debug_print);
    app.getApp<RealtimeDatabase>(Database);
    Database.url(DATABASE_URL);
  }
```

## CODE PANEL
```
```
