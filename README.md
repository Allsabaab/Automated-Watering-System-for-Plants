'''c

#include <IRremote.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

#define Address 0x27
#define Columns 16
#define Rows 2

const int En12=6, In1=5, In2=3;
const int En34=9, In3=10, In4=11;
const int irPin=13, relayPin=12, potPin=A0, soilPin=A1;

enum IR_Commands{
    IR_STOP = 5,
    IR_FORWARD = 1,
    IR_BACKWARD = 9,
    IR_RIGHT = 6,
    IR_LEFT = 4};

LiquidCrystal_I2C lcd(Address,Columns,Rows);
int lastCommand = 0;
bool isMoving = false;

void setup(){
    Serial.begin(9600);
    IrReceiver.begin(irPin);

    lcd.init();
    lcd.backlight();
    lcd.setCursor(0, 0);
    lcd.print("    Speed &     ");
    lcd.setCursor(0, 1);
    lcd.print("    Moisture    ");
    delay(1000);
    lcd.clear();

    pinMode(En12, OUTPUT);
    pinMode(In1, OUTPUT);
    pinMode(In2, OUTPUT);
    pinMode(En34, OUTPUT);
    pinMode(In3, OUTPUT);
    pinMode(In4, OUTPUT);
    pinMode(potPin, INPUT);
    pinMode(soilPin, INPUT);
    pinMode(relayPin, OUTPUT);}

void move(int in1, int in2, int in3, int in4, int speed1, int speed2){
    digitalWrite(In1, in1);
    digitalWrite(In2, in2);
    digitalWrite(In3, in3);
    digitalWrite(In4, in4);
    analogWrite(En12, speed1);
    analogWrite(En34, speed2);
    isMoving = (speed1>0||speed2>0);}

void pause() {move(LOW, LOW, LOW, LOW, 0, 0);}
void forward(int speed) {move(HIGH, LOW, HIGH, LOW, speed, speed);}
void backward(int speed) {move(LOW, HIGH, LOW, HIGH, speed, speed);}
void f_right(int speed) {move(HIGH, LOW, HIGH, LOW, speed/2, speed);}
void f_left(int speed) {move(HIGH, LOW, HIGH, LOW, speed, speed/2);}

void updateLCD(int rpm, int moisture){
    static int lastRpm = -1, lastMoisture = -1;

    if (rpm != lastRpm){
        lcd.setCursor(0,0);
        lcd.print("Speed(RPM):");
        lcd.setCursor(11, 0);
        lcd.print("     ");
        lcd.setCursor(11, 0);
        lcd.print(rpm);
        lastRpm = rpm;}

    if (moisture != lastMoisture){
        lcd.setCursor(0, 1);
        lcd.print("Moisture:");
        if (moisture < 85){
            lcd.setCursor(9, 1);
            lcd.print("       ");
            lcd.setCursor(9, 1);
            lcd.print("PUMP ON");
            lcd.setCursor(9, 1);
            lcd.print("       ");
            lcd.setCursor(9, 1);
            lcd.print(moisture);
            lcd.print("%");}
        else{
            lcd.setCursor(9, 1);
            lcd.print("      ");
            lcd.setCursor(9, 1);
            lcd.print(moisture);
            lcd.print("%");}
        lastMoisture = moisture;}}
        
void loop(){
    int rpm = map(analogRead(potPin), 0, 1023, 0, 255);
    int moisture = map(analogRead(soilPin), 0, 876, 100, 0);

    if (IrReceiver.decode()){
        lastCommand = IrReceiver.decodedIRData.command;
        IrReceiver.resume();

    switch (lastCommand){ 
       case IR_STOP: pause(); break;
       case IR_FORWARD: forward(rpm); break;
       case IR_BACKWARD: backward(rpm); break;
       case IR_RIGHT: f_right(rpm); break;
       case IR_LEFT: f_left(rpm); break;}}

    if (isMoving){
    switch (lastCommand){
    case IR_FORWARD:
    case IR_BACKWARD: move(HIGH, LOW, HIGH, LOW, rpm, rpm);break;
    case IR_RIGHT: move(HIGH, LOW, HIGH, LOW, rpm/2, rpm);break;
    case IR_LEFT: move(HIGH, LOW, HIGH, LOW, rpm, rpm/2);break;}}

    digitalWrite(relayPin, moisture<85 ? HIGH : LOW);
    updateLCD(rpm, moisture);

    Serial.print("Speed: "); Serial.print(rpm);
    Serial.print("  Moisture: "); Serial.println(moisture);

    delay(100);}
