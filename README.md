# ESP8266-TimeTempControl

## ESP8266 D1 mini
Wemos D1 Mini development board has a total 16 pins in which 12 pins are active, uses ESP-12 module, onboard reset button, 3.3 voltage regulator, Micro USB, USB to UART bridge and some other components.

![ESP8266_D1_mini](/images/ESP8266_D1_mini.png)


> Das Relais schließt, sobald Pin D1 auf "High" gesetzt wird.

## ESP8266 01
ESP8266 01 Module is different but commonly as used as the above development boards. This board is not breadboard friendly often separate programming module is used for programming. It has a total 8 pins in which 6 pins are active.

![ESP8266_01i](/images/ESP8266_01.png)


## SourceCode

```
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>                   // Include the WebServer library
#include <Wire.h>
#include <NTPClient.h>
#include <WiFiUdp.h>
#include <OneWire.h>
#include <DallasTemperature.h>

const char* ssid = "WLAN";                      // Enter SSID here
const char* password = "74696325262072177928";  // Enter Password here
const long utcOffsetInSeconds = 3600;
char daysOfTheWeek[7][12] = {"Sonntag", "Montag", "Dienstag", "Mittwoch", "Donnerstag", "Freitag", "Samstag"};
boolean debug = true; 

const int GPIO = 5; // GPIO for Relais
const int TEMP = 4; // GPIO for OneWire-Bus

int day_hour = 8;
int day_minute = 00;
int day_on = 15;
int day_off = 18;

int night_hour = 20;
int night_minute = 00;
int night_on = 10;
int night_off = 12;

// Define NTP Client to get time
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "europe.pool.ntp.org", utcOffsetInSeconds);

// Setup a oneWire instance to communicate with any OneWire devices
OneWire oneWire(TEMP);
// Pass our oneWire reference to Dallas Temperature sensor 
DallasTemperature sensors(&oneWire);


// Define WebServer
ESP8266WebServer server(80);

void setup() {
  Serial.begin(9600);
  
  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(GPIO, OUTPUT);
  digitalWrite(GPIO, LOW);

  //connect to your local wi-fi network
  WiFi.begin(ssid, password);

  //check wi-fi is connected to wi-fi network
  while (WiFi.status() != WL_CONNECTED) {
    delay(50);
    digitalWrite(LED_BUILTIN, LOW);
    delay(50);
    digitalWrite(LED_BUILTIN, HIGH);
    }

   if(debug){Serial.println("WiFi connected!");}

   timeClient.begin();
   if(debug){Serial.println("NTP Client started!");}

   // Start the DS18B20 sensor
   sensors.begin();
   if(debug){Serial.println("OneWire Sensor started!");}
   
   server.begin();
   if(debug){Serial.println("WebServer started!");}

   server.on("/", handle_OnConnect);
   server.onNotFound(handle_NotFound);
}

void loop() {
  server.handleClient();
  
  float temperatureC = getTemp();
  timeClient.update();
  
  if(debug){
    Serial.print(getTemp());
    Serial.print("ºC - ");
    Serial.print(getTimeString());
    Serial.println(" Uhr");
  }

  int switch_day_time = day_hour*100 + day_minute;
  int switch_night_time = night_hour*100 + night_minute;
  int current_time = getTime();
  int current_temp = getTemp();
  
  // Day-Time
  if (current_time > switch_day_time and current_time < switch_night_time) {
    if (current_temp < day_on) {digitalWrite(GPIO, HIGH);}
    if (current_temp > day_off) {digitalWrite(GPIO, LOW);}  
  } else {
    if (current_temp < night_on) {digitalWrite(GPIO, HIGH);}
    if (current_temp > night_off) {digitalWrite(GPIO, LOW);}    
  }

  if(debug){
      Serial.print("Heater Status :");
      Serial.println(digitalRead(GPIO));
  }
  
  delay(1000);
}


void handle_OnConnect() {


  if (server.hasArg("day_hour")== false){ //Check if body received
      server.send(200, "text/html", SendHTML()); 
         digitalWrite(GPIO, LOW);
      return; 
  }
         digitalWrite(GPIO, HIGH);
  day_hour = server.arg("day_hour").toInt();
  day_minute = server.arg("day_minute").toInt();
  day_on = server.arg("day_on").toInt();
  day_off = server.arg("day_off").toInt();

  night_hour = server.arg("night_hour").toInt();
  night_minute = server.arg("night_minute").toInt();
  night_on = server.arg("night_on").toInt();
  night_off = server.arg("night_off").toInt();
 
  server.send(200, "text/html", SendHTML()); 
}

void handle_NotFound(){
  server.send(404, "text/plain", "Not found");
}

float getTemp(){
  sensors.requestTemperatures(); 
  float temperatureC = sensors.getTempCByIndex(0);
  return temperatureC;
}

String getTimeString(){
  timeClient.update();

  String day = daysOfTheWeek[timeClient.getDay()];
  int hour = timeClient.getHours();
  int minute = timeClient.getMinutes();
  String minuteString = String(minute);
  
  if (minute < 10) { 
    String minuteString = "0" + minute; 
  } 

  String timeString = day + ", " + hour + ":" + minuteString;
  return timeString;
}

int getTime(){
  timeClient.update();
  int hour = timeClient.getHours();
  int minute = timeClient.getMinutes();

  return (hour * 100 + minute);
}




String SendHTML(){
  float temperatureC = getTemp();
  String timeString =  getTimeString();
  
  String heaterStatus = "off";
  if(digitalRead(GPIO) == 1 ) {
    String heaterStatus = "on";
  }
  
  String ptr = "<!DOCTYPE html> <html>\n";
  ptr +="<head><meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0, user-scalable=no\">\n";
  ptr +="<title>ESP8266 Time-Temp-Control</title>\n";
  ptr +="<style>html { font-family: Helvetica; display: inline-block; margin: 0px auto; text-align: center;}\n";
  ptr +="body{margin-top: 50px;} h1 {color: #444444;margin: 50px auto 30px;}\n";
  ptr +="p {font-size: 14px;color: #444444;margin-bottom: 10px;}\n";
  ptr +="</style>\n";
  ptr +="</head>\n";
  ptr +="<body>\n";
  ptr +="<div id=\"webpage\">\n";
  ptr +="<h2>ESP8266 Time-Temp-Control</h2>\n";
  ptr +="<p>Temperature: ";
  ptr +=temperatureC;
  ptr +="&deg;C -- Zeit: ";
  ptr +=timeString;
  ptr +=" Uhr -- Heizluefter: ";
  ptr +=heaterStatus + "</p>\n";
  ptr +="<form action='/' method='get'><table>\n";
  ptr +="<tr><th></th><th>Stunde</th><th>Minute</th><th>Ein bei &deg;C</th><th>Aus bei &deg;C</th></tr>\n";
  ptr +="<tr><th>Tag</th>";
  ptr +=" <td><input type='text' id='day_hour' name='day_hour' value='" + String(day_hour) + "'></td>\n";
  ptr +=" <td><input type='text' id='day_minute' name='day_minute' value='" + String(day_minute) + "'></td>\n";
  ptr +=" <td><input type='text' id='day_on' name='day_on' value='" + String(day_on) + "'> &deg;C</td>\n";
  ptr +=" <td><input type='text' id='day_off' name='day_off' value='" + String(day_off) + "'> &deg;C</td>\n";
  ptr +="</tr>\n"; 
  ptr +="<tr><th>Nacht</th>";
  ptr +=" <td><input type='text' id='night_hour' name='night_hour' value='" + String(night_hour) + "'></td>\n";
  ptr +=" <td><input type='text' id='night_minute' name='night_minute' value='" + String(night_minute) + "'></td>\n";
  ptr +=" <td><input type='text' id='night_on' name='night_on' value='" + String(night_on) + "'> &deg;C</td>\n";
  ptr +=" <td><input type='text' id='night_off' name='night_off' value='" + String(night_off) + "'> &deg;C</td>\n";
  ptr +="</tr>\n"; 
  ptr +="<br>\n";
  ptr +="</table>\n";
  ptr +="<input type='submit' value='Submit'>\n";
  ptr +="</form>\n";  
  ptr +="</div>\n";
  ptr +="</body>\n";
  ptr +="</html>\n";
  return ptr;
}


```
