# Electric Unicycle Interface Library

Arduino library to interface electric unicycles.  

## Intro

Comment from Jakub :
This is a fork of T-vK project. 
I made adjustments to library and arduino sketch as well due to changes between bluetooth 2.0 and 4.0 BLE that has been made over the years.
I am using 128x64px display. 

This library allows you to receive and automatically decode the data that EUCs (=Electric Unicycles) send via their Bluetooth interface.  
It has been tested on several GotWay EUCs.  
I strongly advise to use this library with HM-10 bluetooth module, because it is capable of auto-connect and it is reliable.

## Features

### Receiving / calculating data

 - Read Voltage
 - Read Current
 - Read Speed
 - Read Total Mileage
 - Read Temperature

#### Using these values you can calculate: 

 - Battery status
 - Acceleration
 - If the unicycle is breaking
 - How much power (Watt) it is currently using
 - and much more...
 
### Sending commands

 - Make the unicylce beep
 - Set the unicycle to madden mode
 - Set the unicycle to comfort mode
 - Set the unicycle to soft mode
 - Set horizontal alignment
 - Turn off level 1 alarm
 - Turn off level 2 alarm
 - Turn on speed alarms
 - Enable tilt-back at 6km/h
 - Disable tilt-back at 6km/h
 
### Video: Electric Unicycle Interface Example: Led Rainbow and Breaklight
[![Video](https://img.youtube.com/vi/0SC_-P2epRE/0.jpg)](https://youtu.be/0SC_-P2epRE)  

### Usage 

#### via Bluetooth

HOW TO CONNECT : 

1. Set up bluetooth
- set up Arduino IDE, connect with Bluetooth HM-10
At this point RED light will be blinking slowly. Couple of times per second.

- send a sketch to Arduino (with serial write)

```C++
#include <SoftwareSerial.h>
SoftwareSerial BTSerial(4, 5); // TX on bluetooth to 4 | RX on bluetooth to 5

void setup()
{
  //pinMode(9, OUTPUT);  // this pin will pull the HC-05 pin 34 (key pin) HIGH to switch module to AT mode
  //digitalWrite(9, HIGH);
  Serial.begin(19200);
  Serial.println("Enter AT commands:");
  BTSerial.begin(9600);  // HC-05 default speed in AT command more
}

void loop()
{
  if (BTSerial.available())
    {
    // PRINTING HEX
    //Serial.print(BTSerial.read(), HEX);
    //Serial.print('\n');
    // OR PRINTING DECIMAL RECEIVED
    Serial.write(BTSerial.read());
    }

//Keep reading when serial is available
if (Serial.available())
    BTSerial.write(Serial.read());
}
```

- propper baud rate, in my case 9600
- not the same baud rate for bluetooth and serial monitor
- AT (should be OK)
 
- commands to be send :

- Set up master mode : AT+ROLE1, Set mode to Transmittion : AT+MODE0, Discovery : AT+DISC?, Connect to discovered device : AT+CONN[number]

- When the device will be connected, RED light will stop blinking. Will be solid red.
Now send another sketch to capture data :

```C++
#include <SoftwareSerial.h>

SoftwareSerial BTSerial(4, 5); // TX on bluetooth to 4 | RX on bluetooth to 5

void setup()
{
  Serial.begin(19200);
  Serial.println("Enter AT commands:");
  BTSerial.begin(9600);  // HC-10 is default speed in AT command more
}

void loop()
{
// Keep reading from HC-05 and send to Arduino Serial Monitor

  if (BTSerial.available())
    {
    // PRINTING HEX
    Serial.print(BTSerial.read(), HEX);
    Serial.print('\n');
    // OR PRINTING DECIMAL RECEIVED
    // Serial.write(BTSerial.read());
    }


  // Keep reading from Arduino Serial Monitor and send to HC-05
  if (Serial.available())
    BTSerial.write(Serial.read());
}

```


MAIN CODE

``` C++
 /* Receive the data the electric unicycle sends 
 * and print it to an 128x32 pixel OLED display.
 */

#include <math.h>
#include <SoftwareSerial.h>
#include <EucInterface.h>
#include <SPI.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH1106.h>
#define OLED_RESET A4
Adafruit_SH1106 display(OLED_RESET);

#include <Wire.h>

//Bluetooth serial with rx on pin 4 and tx on pin 5
SoftwareSerial BluetoothSerial(4, 5);

Euc Euc(BluetoothSerial, BluetoothSerial); // Receive and transmit data via bluetooth

float StartMileage = 0.00;
float DailyMileage = 0.00;
float CurrentMileage = 0.00;

// 'display', 128x64px
const unsigned char Background [] PROGMEM = {
  0x1c, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x7f, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x80, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0f, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x88, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 
  0x88, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00, 0x00, 0x05, 0x00, 
  0x98, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 
  0x98, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0xb8, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x8e, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0b, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x8c, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0a, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x8c, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0e, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x88, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x11, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x80, 0x80, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x15, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x7f, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x11, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x1c, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x0e, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x0f, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0x80, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x0f, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xc0, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x60, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x50, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x60, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x40, 0xa0, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x40, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x40, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x40, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x6a, 0xc0, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00
};


void setup() {
  Serial.begin(9600); // We'll use the normal hardware serial to print out all the received data
  
  BluetoothSerial.begin(9600); // Most unicycles communicate @9600 baud over bluetooth
  Euc.setCallback(eucLoop); // Set a callback function in which we can receive all the data the unicycle sends

  display.begin(SH1106_SWITCHCAPVCC, 0x3C);  // initialize with the I2C addr 0x3D (for the 128x64)
  display.display();
  // Clear the buffer.
  display.clearDisplay();
}

void loop() {
  Euc.tick(); // This simply needs to be called regularely
}

  void eucLoop(float voltage, float speed, float current, float tempMileage, float temperature, float mileage, float battery, bool dataIsNew) 
   {
    
   if (dataIsNew) 
      {
        if (StartMileage = 0.00)
        {
        static int StartMileage = (mileage);
        }
        else
        {
        float CurrentMileage = (mileage); 
        }
        DailyMileage = CurrentMileage - StartMileage;
        
      display.clearDisplay(); // first number horizontal, then vertical
      display.drawBitmap(0,0,Background,128,64,WHITE);
      display.setTextSize(1);
      display.setTextColor(WHITE);
      
      display.setCursor(32,30);
      display.setTextSize(2);
      display.print(abs((int)speed));   display.println(" km/h");
      display.setTextSize(1);
  
      display.setCursor(85,5);
      display.print(temperature);    display.println(" C");
      
      display.setCursor(0,20);
      display.print(battery,0);    display.println("%");
      
      display.setCursor(15,5);
      display.print(voltage,1);    display.println(" V");
      
      display.setCursor(20,55);
      display.print(mileage,2); display.println(" / ");
      display.setCursor(78,55);
      display.print(DailyMileage,2); display.println(" km");
      display.display();
     }
     
   }
    
```

### API docs
``` c++
Euc(ReceiverSerial, TransmitterSerial); //create new instance of this class

Euc.tick(); // simply has to be called regularly
Euc.setCallback(callbackFunction); // you have to specify a callback function to which the class can send the data it receives from the unicycle
//Example callback function:
void callbackFunction(float voltage, float speed, float tempMileage, float current, float temperature, float mileage, bool dataIsNew) {
  // Do something with the received data
}
Euc.beep(); // make the unicycle beep
Euc.maddenMode(); // set madden mode
Euc.comfortMode(); // set confort mode
Euc.softMode(); // set soft mode
Euc.calibrateAlignment(); // calibrate alignment
Euc.disableLevel1Alarm(); // disable level 1 alarm
Euc.disableLevel2Alarm(); // disable level 2 alarm
Euc.enableAlarms(); // enable alarms
Euc.enable6kmhTiltback(); // enable  6km/h tiltback
Euc.disable6kmhTiltback(); // disable 6km/h tiltback
```

### More info

The electric unicycles I tested send their data 5 times persecond with a consistent 200 millisecond delay between them.

### Disclaimer

I will not be responsible or liable, directly or indirectly, in any way for any loss or damage of any kind incurred as a result of using this library or parts thereof.  
