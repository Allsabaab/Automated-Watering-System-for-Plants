#include <IRremote.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

int En12 = 6;
int In1 = 5;
int In2 = 3;
int En34 = 9;
int In3 = 10;
int In4 = 11;
const int ir = 13;
const int relay = 12;
const int pot = A0;
const int soil = A1;

int lastCommand = 0;
bool isMoving = false;

void setup() {
  Serial.begin(9600);
  
  IrReceiver.begin(ir);
  
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
  pinMode(pot, INPUT);
  pinMode(soil, INPUT);
  pinMode(relay, OUTPUT);
}

void pause() {
  digitalWrite(In1, LOW);
  digitalWrite(In2, LOW);
  digitalWrite(In3, LOW);
  digitalWrite(In4, LOW);
  analogWrite(En12, 0);
  analogWrite(En34, 0);
  isMoving = false;
}

void forward(int speed) {
  digitalWrite(In1, HIGH);
  digitalWrite(In2, LOW);
  digitalWrite(In3, HIGH);
  digitalWrite(In4, LOW);
  analogWrite(En12, speed);
  analogWrite(En34, speed);
  isMoving = true;
}

void f_right(int speed) {
  digitalWrite(In1, HIGH);
  digitalWrite(In2, LOW);
  digitalWrite(In3, HIGH);
  digitalWrite(In4, LOW);
  analogWrite(En12, speed/2);
  analogWrite(En34, speed);
  isMoving = true;
}

void f_left(int speed) {
  digitalWrite(In1, HIGH);
  digitalWrite(In2, LOW);
  digitalWrite(In3, HIGH);
  digitalWrite(In4, LOW);
  analogWrite(En12, speed);
  analogWrite(En34, speed/2);
  isMoving = true;
}

void backward(int speed) {
  digitalWrite(In1, LOW);
  digitalWrite(In2, HIGH);
  digitalWrite(In3, LOW);
  digitalWrite(In4, HIGH);
  analogWrite(En12, speed);
  analogWrite(En34, speed);
  isMoving = true;
}

void loop() {
  int val1 = analogRead(pot);
  int rpm = map(val1, 0, 1023, 0, 255);

  int val2 = analogRead(soil);
  int moisture = map(val2, 0, 876, 100, 0);

  if (IrReceiver.decode()) {
    int button = IrReceiver.decodedIRData.command;
    IrReceiver.resume();

    lastCommand = button;
    
    if (lastCommand == 5) { pause(); }
    else if (lastCommand == 1) { forward(rpm); }
    else if (lastCommand == 9) { backward(rpm); }
    else if (lastCommand == 6) { f_right(rpm); }
    else if (lastCommand == 4) { f_left(rpm); }
  }

  if (isMoving==true) {
    if (lastCommand==1||lastCommand==9){
      analogWrite(En12, rpm);
      analogWrite(En34, rpm);
    } 
    else if (lastCommand==6){
      analogWrite(En12, rpm/2);
      analogWrite(En34, rpm);
    } 
    else if (lastCommand==4){
      analogWrite(En12, rpm);
      analogWrite(En34, rpm/2);
    }
  }

  lcd.setCursor(0, 0);
  lcd.print("Speed(RPM):");
  lcd.setCursor(11, 0);
  lcd.print("     ");
  lcd.setCursor(11, 0);
  lcd.print(rpm);

  lcd.setCursor(0, 1);
  lcd.print("Moisture:");

  if (moisture < 85) {
    digitalWrite(relay, HIGH);
    lcd.setCursor(9, 1);
    lcd.print("       ");
    lcd.setCursor(9, 1);
    lcd.print("PUMP ON");
    delay(100);
    lcd.setCursor(9, 1);
    lcd.print("       ");
    lcd.setCursor(9, 1);
    lcd.print(moisture);
    lcd.print("%  ");
  } 
  else {
    digitalWrite(relay, LOW);
    lcd.setCursor(9, 1);
    lcd.print("       ");
    lcd.setCursor(9, 1);
    lcd.print(moisture);
    lcd.print("%  ");
  }

  Serial.print("Speed: ");
  Serial.println(rpm);
  Serial.print("Moisture: ");
  Serial.println(moisture);

  delay(100);
}
