# IoT Manual

## DryOnline: Keep control of your dryer with telegram.
In this document you will find my prototype to get a notification on your phone when your door is locked to turn your dryer of if it's running while you are away. For demonstration of the project I used a LEDstrip which is supposed to be the dryer.

### Supplies
* Hardware
  * ESP8266 NodeMCU
  * HC-SR04 Ultrasonic Sensor
  * LEDstrip
* Software
  * Arduino IDE
  * Telegram

### 1. Install Arduino IDE
Follow this [guide to install Arduino IDE](https://www.arduino.cc/en/guide/windows)

### 2. Install required libraries 
* Universal Telegram Bot Library
  1. [Click here to download the Universal Arduino Telegram Bot library](https://github.com/witnessmenow/Universal-Arduino-Telegram-Bot/archive/master.zip)
  2. In Arduino IDE: go to **Sketch > Include Library > Add.ZIP Livrary..** 
  3. Add the library you've just downloaded.

* ArduinoJson Library
  1. In Arduino IDE: Go to **Sketch > Include Library > Manage Libraries.**
  2. Search for 'ArduinoJson'.
  3. Install the library

![Image of library](https://i1.wp.com/randomnerdtutorials.com/wp-content/uploads/2020/06/Install-ArduinoJson-Library.png?w=786&quality=100&strip=all&ssl=1)

* Adafruit NeoPixel
  * Follow the same steps as before but search for Adafruit Neopixel

### 3. Preparing Telegram
1. Go to Google Play or the App Store, download and install Telegram.
2. Search for "botfather".
3. type **/newbot** and follow the instructions.
4. Save the **bot token**.
5. Go back to search for "IDBot".
6. Safe the user ID

### 4. Connecting the hardware
* LEDstrip 
  * +5v to 3v3
  * Din to D1
  * GND to GND
* HC-SR04 Ultrasonic Sensor
  * VCC to Vin
  * Trig to D6
  * Echo to D5
  * GND to GND

### 5. Code
Start by importing the required libraries
```C
#ifdef ESP32
  #include <WiFi.h>
#else
  #include <ESP8266WiFi.h>
#endif
  #include <Adafruit_NeoPixel.h>
#ifdef __AVR__
  #include <avr/power.h>
#endif
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>   // Universal Telegram Bot Library written by Brian Lough: https://github.com/witnessmenow/Universal-Arduino-Telegram-Bot
#include <ArduinoJson.h>
```
Replace this with your network credentials.
```C
const char* ssid = "network-name";
const char* password = "password";
```
Add the token and userID we got from Telegram.
```C
// Initialize Telegram BOT
#define BOTtoken "BOTTOKEN"
// UserID
#define CHAT_ID "USERID"
```
Add this for the wifi client and checking incomming messages
```C
#ifdef ESP8266
  X509List cert(TELEGRAM_CERTIFICATE_ROOT);
#endif
WiFiClientSecure client;
UniversalTelegramBot bot(BOTtoken, client);
// Checks for new messages every 1 second.
int botRequestDelay = 1000;
unsigned long lastTimeBotRan;
```
Add the information needed for the LEDpin
```C
#define NUMPIXELS 31
#define PIN D1
Adafruit_NeoPixel pixels(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);
#define DELAYVAL 500
const int ledPin = D1;
bool ledState = LOW;
```
Add the information needed for the sensor
```C
int sendMsg = 1;
const int trigPin = 12;
const int echoPin = 14;
#define SOUND_VELOCITY 0.034
long duration;
float distanceCm;
```
The code in the next part makes it possible to send messages to Telegram if the door is seen as locked by the sensor.
```C
void distanceMessage(int numNewMessages) {
  Serial.println("handleNewMessages");
  Serial.println(String(numNewMessages));

  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);
    if (chat_id != CHAT_ID) {
      bot.sendMessage(chat_id, "Unauthorized user", "");
      continue;
    }
    String text = bot.messages[i].text;
    Serial.println(text);
    String from_name = bot.messages[i].from_name;

    if (distanceCm <= 6) {
      if (ledState = HIGH) {
        bot.sendMessage(chat_id, "Je verlaat het huis en de droger staat aan, type '/off' om de droger uit te zetten.");
        Serial.print("locked, on");
      }
      else {
        Serial.print("locked, off");
      }
    }
  }
}
```
Now we make a function to handle what happens after receiving messages
**handleNewMessages()**
```C
void handleNewMessages(int numNewMessages) {
  Serial.println("handleNewMessages");
  Serial.println(String(numNewMessages));
```
this part will check available messages:
```C
  for (int i = 0; i < numNewMessages; i++) {
    // Chat id of the requester
    String chat_id = String(bot.messages[i].chat_id);
    if (chat_id != CHAT_ID) {
      bot.sendMessage(chat_id, "Unauthorized user", "");
      continue;
    }
    String text = bot.messages[i].text;
    Serial.println(text);
    String from_name = bot.messages[i].from_name;
```
Now we will add the commands to turn the device on and off.
```C
if (text == "/on") {
      bot.sendMessage(chat_id, "De droger staat aan", "");
      ledState = HIGH;
      for (int i = 0; i < NUMPIXELS; ++i) { // For each pixel...
        pixels.setPixelColor(i, pixels.Color(255, 255, 255));
        pixels.show();   // Send the updated pixel colors to the hardware.
        sendMsg = 1;
      }
      digitalWrite(ledPin, ledState);
    }
    if (text == "/off") {
      bot.sendMessage(chat_id, "De droger staat uit", "");
      ledState = LOW;
      for (int i = 0; i < NUMPIXELS; ++i) { // For each pixel...
        pixels.setPixelColor(i, pixels.Color(0, 0, 0));
        pixels.show();   // Send the updated pixel colors to the hardware.
      }
      digitalWrite(ledPin, ledState);
    }
    if (text == "/state") {
      if (digitalRead(ledPin)) {
        bot.sendMessage(chat_id, "De droger staat aan", "");
      }
      else {
        bot.sendMessage(chat_id, "De droger staat uit", "");
      }
    }
  }
}
```
**Setup()**
```C
void setup() {
  Serial.begin(115200);
#if defined(__AVR_ATtiny85__) && (F_CPU == 16000000)
  clock_prescale_set(clock_div_1);
#endif
  pixels.begin();

#ifdef ESP8266
  configTime(0, 0, "pool.ntp.org");     
  client.setTrustAnchors(&cert); 
#endif
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, ledState);
  pinMode(trigPin, OUTPUT); // Sets the trigPin as an Output
  pinMode(echoPin, INPUT); // Sets the echoPin as an Input
  // Connect to Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
#ifdef ESP32
  client.setCACert(TELEGRAM_CERTIFICATE_ROOT); // Add root certificate for api.telegram.org
#endif
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi..");
  }
  // Print ESP32 Local IP Address
  Serial.println(WiFi.localIP());
}
```
**Loop()**
```C
void loop() {
  if (millis() > lastTimeBotRan + botRequestDelay)  {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      Serial.println("got response");
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }
    lastTimeBotRan = millis();
  }
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  duration = pulseIn(echoPin, HIGH);
  distanceCm = duration * SOUND_VELOCITY / 2;
  Serial.print("Distance (cm): ");
  Serial.println(distanceCm);

  if (distanceCm <= 6) {
    if (ledState = HIGH) {
      distanceMessage(sendMsg);
    }
    sendMsg = 0;
  }

  delay(100);
}
```

### Errors
Here I will share some errors I made so you can prevent them.
* expected ',' or ';' before.
  * Never forget to end the rule.
* Everything worked except I got spammed in Telegram because of the part that the bot will send me a message if the distance is shorter than 6 and the led is high. I fixed that by adding a variable.
* Too few arguments to function 
  * in the function
  ```C
  distanceMessage(int numNewMessages)
  ```
  *  I forgot to add the variable to the function that had to switch between 1 and 0 so it could send a message when the distance is shorter than 6 AND the Ledpin was HIGH. 
  *  First i added the functions all in one void but it didn't work with if statements so i decided to make 2 voids out of it. the first one for the distance sensor only and the second one for handling commands.
