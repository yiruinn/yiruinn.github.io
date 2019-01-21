---
title: Automatic Cat Feeder
date: 2018-08-19 12:30:04
category: Project
---

We want to design and build a cat feeder that can distribute food at times set by the user. The feeder needs to be able to maintain a temperature cold enough to contain the food an entire day.  

## Parts List
- Arduino Nano
- DS18S20 Thermal Probe
- DS3231 Real Time Clock
- 5V Relay
- 28BYJ48 Stepper Motor and ULN2003 Driver
- 16x2 LCD Screen
- H22A2 Photointerrupter & Break Beam Sensor
- Peltier Cooler + Fans
- Push Button (x4)
- 12V Power Supply
- PVC Foam Board
- DC-DC Step Down Regulator (x2)


## Building Process

### Mockup
The first thing we did is build a mockup out of cardboard to get a general feeling of how everything will look.

![](/assets/cat_feeder/mockup.jpg)

### Schematic
We laid out a schematic to see how many components can be used and to make the wiring and soldering process easier.

![](/assets/cat_feeder/schematic.png)

### Module Testing
Before putting the components together, each module is individually tested to see how they work, and if there's any problems with them.

###### Temperature Sensor
The temperature sensor is used to get the current temperature inside the box and determine if the cooler should be on and off.

The OneWire and DallasTemperature libraries are required to use this module.

```c
#include <OneWire.h>
#include <DallasTemperature.h>

#define ONE_WIRE_BUS 10

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

double getCurrentTemperature() {
  sensors.requestTemperatures();
  return sensors.getTempFByIndex(0);
}

void setup(void)
{
  Serial.begin(9600);
  sensors.begin();
}
void loop(void)
{
  Serial.print(getCurrentTemperature());
  delay(1000);
}
```
###### Real Time Clock
The time module is used to determine which tray should be rotated out at that moment in time. We decided used an external clock instead of the built in one on the Arduino so that we wouldn't need to reset the time each time the power is disconnected.

The Wire and RTClib libraries are required to use this module.

```c
#include <Wire.h>
#include "RTClib.h"

RTC_DS3231 rtc;

void printTime() {
  DateTime now = rtc.now();
  Serial.println(now.hour());
  Serial.println(now.minute());
}

void setup() {
  if (!rtc.begin()) {
    while(1);
  }
  if (rtc.lostPower()) {
    // following line sets the RTC to the date & time this sketch was compiled
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }
  Serial.begin(9600);
}

void loop() {
  printTime();
  delay(3000);
}
```

###### Beam Sensors
The two beam sensors (H22A2 and Break Beam) is used for tray calibration and obstruction sensing, respectively. On the initial start up, the tray won't know where it currently is, so it keeps rotating until it finds the calibration sensor and then uses it as a reference point. The obstruction sensor prevents the tray from rotating when there's something blocking it, so a cat won't have its face caught inside when it tries to eat from the tray.

It's important to note that the pins used (A6 and A7) are analog only pins, so digital reads won't work.
```c
void setup() {
  pinMode(A6, INPUT);
  Serial.begin(9600);
}

void loop() {
  if(analogRead(A6) < 512) Serial.println("LOW");
  else Serial.println("HIGH");
}
```

###### Stepper Motor
The stepper motor rotates the tray to a position with food (1,2,3,4) corresponding to the tray times set. Outside of the feeding time, it returns back to position 0 to prevent the food from spoiling.

It needs 2048 steps for a full rotation, so we can figure out how many steps it takes to get from one position to another by taking the proportion of the tray and multiply it by 2048.
![](/assets/cat_feeder/trayposition.jpg)

The Stepper library is required to use this module.

```c
#include <Stepper.h>

const int steps = 64;
int steps[5] = {568, 370, 370, 370, 370};
int currentPosition = 0;

// initialize the stepper library on pins 8 through 11:
// Order is wrong on purpose due to motor wire mismatch
Stepper myStepper(steps, 2, 4, 3, 5);

void moveTo(int destPosition) {
  int currentStep = getStepOf(currentPosition);
  int destStep = getStepOf(destPosition);
  // calculates steps to go ccw or cw
  int diffStep = (destStep - currentStep + 2048) % 2048;
  // chooses direction based on which way is shorter
  if(diffStep < 1024) {
    // clockwise
    Serial.println("cw");
    Serial.println(diffStep);
    myStepper.step(diffStep);
  } else {
    // counter-clockwise
    Serial.println("ccw");
    Serial.println(diffStep - 2048);
    myStepper.step(diffStep - 2048);
  }
  currentPosition = destPosition;
}

int getStepOf(int position) {
  int i;
  int ret = 0;
  for(i = 0; i < position; i++) {
    ret += steps[i];
  }
  return ret;
}

void calibrateMotor() {
  // turn motor to 0 position
  while(analogRead(A7) < 512) {
    myStepper.step(1);
  }
  myStepper.step(-110);
  currentPosition = 1;
}

void setup() {
  // set the speed at 60 rpm:
  myStepper.setSpeed(60);
  // initialize the serial port:
  Serial.begin(9600);
}

void loop() {
  // assuming tray is initially in the 0 position
  Serial.println("Move from 0 to 3");
  moveTo(3);
  delay(3000);
  Serial.println("Move from 3 to 4");
  moveTo(4);
  delay(3000);
  Serial.println("Move from 4 to 0");
  moveTo(0);
  delay(3000);
  Serial.println("Move from 0 to 2");
  moveTo(2);
  delay(3000);
  Serial.println("Move from 2 to 0");
  moveTo(0);
  delay(3000);
}
```

###### LCD Screen and Push Buttons
The LCD screen is used to display the menu, and four buttons will control the menu.

The LiquidCrystal library is required to use this module.

```c
#include <LiquidCrystal.h>

LiquidCrystal lcd(12,11,9,8,7,6);

const int LCD_LEFT_FIELD = 1;
const int LCD_RIGHT_FIELD = 10;
const int LCD_TOP_ROW = 0;
const int LCD_BOTTOM_ROW = 1;

void setup() {
  pinMode(A0, INPUT_PULLUP);
  pinMode(A1, INPUT_PULLUP);
  pinMode(A2, INPUT_PULLUP);
  pinMode(A3, INPUT_PULLUP);

  lcd.begin(16,2);
}

void loop() {
  lcd.clear();
  lcd.setCursor(LCD_LEFT_FIELD,LCD_TOP_ROW);
  lcd.print("A0: ");
  if(digitalRead(A0) == HIGH) lcd.print("H");
  else lcd.print("L");
  lcd.setCursor(LCD_RIGHT_FIELD,LCD_TOP_ROW);
  lcd.print("A1: ");
  if(digitalRead(A1) == HIGH) lcd.print("H");
  else lcd.print("L");
  lcd.setCursor(LCD_LEFT_FIELD,LCD_BOTTOM_ROW);
  lcd.print("A2: ");
  if(digitalRead(A2) == HIGH) lcd.print("H");
  else lcd.print("L");
  lcd.setCursor(LCD_RIGHT_FIELD,LCD_BOTTOM_ROW);
  lcd.print("A3: ");
  if(digitalRead(A3) == HIGH) lcd.print("H");
  else lcd.print("L");
  delay(250);
}
```

### Building the Box

The box for the cat feeder is made of PVC foam. It's easy to work with, and is insulating enough for our objective.

The stepper motor will be the only major heat emitting element inside the box. The rest of the electronics will be kept in a case mounted externally.
![](/assets/cat_feeder/build1.jpg)
![](/assets/cat_feeder/build2.jpg)

Rollers allow the tray to rotate without much friction and alleviate some strain from the motor.
![](/assets/cat_feeder/build3.jpg)

The rollers are distributed such that the tray will remain stable when force is applied on it unevenly.
![](/assets/cat_feeder/build4.jpg)

The holding disc is built with edges that fit perfectly with the tray so that the tray will not slide around when a cat is eating from it.
![](/assets/cat_feeder/build5.jpg)
![](/assets/cat_feeder/build6.jpg)
![](/assets/cat_feeder/build7.jpg)

### Soldering and Wiring
Most of the components have to be placed at specific places, which leaves only the Arduino, relay, RTC, and voltage regulator to be soldered on the board. We soldered pins for the Arduino and its connections for easy removal during debugging.

There are two voltage regulators stacked together, one tuned to 9.5V for the Arduino, and one tuned to 5.1V for the stepper motor. The cooler and fan uses 12V directly from the power supply.
![](/assets/cat_feeder/solder2.jpg)

A 5V and a ground rail is created for the components outside of the board.
![](/assets/cat_feeder/solder1.jpg)
![](/assets/cat_feeder/wire.jpg)

Idle menu displays current tray position, current time, and current temperature inside the feeder.
![](/assets/cat_feeder/idlemenu.jpg)

Users can select a tray and edit the time for it. How long the food stays out can also be editted. (not pictured)
![](/assets/cat_feeder/traymenu.jpg)

With everything put together.
![](/assets/cat_feeder/topview.jpg)
![](/assets/cat_feeder/internal.jpg)

### Final Code
The final code can be found [here](https://github.com/yiruinn/Automated-Cat-Feeder).

## Problems Encountered
During the process of building and testing, we encountered a few bugs. There were two major ones that we resolved.

During initial testing, some lines of code not related to the relay/cooler(D13) would cause the fan to turn on permanently. We did some tests to see if we could explicitly set it off every loop, but digitalWrites to D13 did nothing. We hypothesized that some library was interfering with pin D13 and reinitializing it as an input, so we took out libraries one by one to narrow down the problem. In the end, we figured out that the RTC library was the problem: more specifically, the rtc.adjust in the setup loop.

When we integrated all the components together, we noticed that the LCD screen would sometimes start displaying random characters. No amount of rewriting to it would fix it, so it was most likely caused by an electrical problem. After some further testing, we noticed that it would bug out immediately after the fan turned on, so we thought the problem was due to the cooler loading down the circuit, causing random behavior on the LCD signal pins. We tried fixing it by putting a capacitor across the voltage regulator, so that noise in the power supply would be filtered out. It still didn't solve the problem, so the only other possible cause could be ground bounce. The cooler and LCD screen grounds were initially wired together because of their close proximity, so we moved them to different grounds (one to arduino one to power supply) and that fixed the problem.
