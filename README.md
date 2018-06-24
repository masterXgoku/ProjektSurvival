# ProjektSurvival
```markdown
//Einbindung von Bibliotheken
#include "MaxMatrix.h"        //Bibliothek zum Arbeiten mit der LED-Matrix
#include <time.h>             //Bibliothek zum Arbeiten mit dem Random-Befehl
#include <SoftwareSerial.h>   //Bibliohek zum Arbeiten mit dem Server
#include "DumbServer.h"       //Bibliothek zum Verbinden mit dem Server

//Anbindung des WLan-Shields(Pretzelboard)
SoftwareSerial esp_serial(11, 12);
EspServer esp_server;

//Anbindung des LED Displays mit Hilfe von "MaxMatrix.h"-Bibliothek
int data = 8;
int load = 9;
int uhr = 10;            //Eigentlich "clock", aber der Name war schon in time.h vergeben (beachte auch Zeile 47!)
int maxInUse = 1;

//Anbindung des Joysticks
const int SW_pin = 2;
const int X_pin = 0;
const int Y_pin = 1;

int spielfeld[8][8];     //8x8-großes Spielfeld wird in Form eines Arrays initialisiert 
int richtung[2];         //Array für die Richtung, in die sich bewegt werden soll

//Koordinaten des temporären Standpunktes
int altX;
int altY;

//Koordinaten des neuen Standpunktes
int neuX;
int neuY;

//Koordinaten des Apfels
int apfelX;              
int apfelY;

int zeit = 1000;         //Standardmäßig leuchtet die Position des Überlebenden 1 Sekunde (mit dieser Variable wird die Schnelligkeit verändert)
int ende = 0;            //Integer für Game Over
int anfang = 0;          //Integer für Spielstart 
int score;               //Variable für den Punktestand/die Anzahl der eingesammelten Äpfel
       
//Strings, die aus der GUI gelesen werden
String mode;
String beginn;

// Ein Objekt vom Typ MaxMatrix wird erzeugt
MaxMatrix m(data, load, uhr, maxInUse);


void setup() {
  //Joystick als Eingabe
  pinMode(SW_pin, INPUT);
  digitalWrite(SW_pin, HIGH);  
  
  srand(time(NULL));          //Initialisierung für den Random-Befehl
  m.init();                   //Display-Objekt initialisieren
  m.setIntensity(15);         //Helligkeit auf Maximalwert setzen    
  
  myInit();                   //Aufrufen der myInit-Funktion
  
  //Verbindung mit dem Server herstellen
  Serial.begin(19200);
  esp_serial.begin(19200);
  Serial.println("Starting server...");
  esp_server.begin(&esp_serial, "Fynn Hotspot", "unly1304", 30363);
  Serial.println("...server is running");
  char ip[16];
  esp_server.my_ip(ip, 16);
  Serial.print("My ip: ");
  Serial.println(ip);
}


void start() {
  if (esp_server.available()) {
    String mode = esp_server.readStringUntil('\n');    //String mode wird ausgelesen
    Serial.println(mode);

    //Je nach Inhalt des Strings, wird die Schnelligekit des Überlebenden geändert
    if (mode == "Beginner") {
      zeit = 1500;
      Serial.println(zeit);
    } else if (mode == "Normal") {
      zeit = 1000;
      Serial.println(zeit);
    } else if (mode == "Hard") {
      zeit = 500;
      Serial.println(zeit);
    }
  }
  if (esp_server.available()) {
    String beginn = esp_server.readStringUntil('\n');   //String beginn wird ausgelesen
    Serial.println(beginn);
    if (beginn == "Anfang") {   
      anfang = 1;                                       //Beinhaltet der String beginn "Anfang", wird das Spiel gestartet
    }
  }
}


void myInit() {
  //Alle Koordinaten und der Punktestand werden zurückgesetzt
  score = 0;
  altX = 0;
  altY = 0;
  neuX = 0;
  neuY = 0;
  apfelX = 0;
  apfelY = 0;
  
  spielfeld[altX][altY] = 1;    //Startpunkt ist der LED-Punkt in der linken oberen Ecke
  
  //Zu Beginn bewegt sich der Überlebende nach rechts
  richtung[0] = 1;
  richtung[1] = 0;
  
  m.setDot(altX, altY, HIGH);             //LED des Startpunkts wird eingeschaltet 
  setzeApfel();                           //Der erste Apfel wird gesetzt
  delay(zeit);                            //Der erste Schritt wird nach einer bestimmten Zeit gesetzt
}


void setzeApfel() {
  while (true) {
    //Koordinaten des Apfels werden zufällig bestimmt
    apfelX = rand() % 8;    
    apfelY = rand() % 8;
    
    if (spielfeld[apfelX][apfelY] == 0 && ( abs(neuX - apfelX) > 2 || abs(neuY - apfelY) > 2)) {
      m.setDot(apfelX, apfelY, HIGH);     //Wenn der zufällig gewählte Punkt nicht besetzt ist und nicht direkt neben dem Überlebenden liegt, schalte die LED ein
      break;
    }
  }
}


void holeRichtung() {
  //Je nachdem in welche Richtung der Joystick zeigt, wird das Array richtung mit entsprechenden Werten belegt
  if (analogRead(Y_pin) > 600) {
    richtung[0] = 0;
    richtung[1] = 1;
  } else if (analogRead(Y_pin) < 200) {
    richtung[0] = 0;
    richtung[1] = -1;
  } if (analogRead(X_pin) > 600) {
    richtung[0] = 1;
    richtung[1] = 0;
  } else if (analogRead(X_pin) < 200) {
    richtung[0] = -1;
    richtung[1] = 0;
  }
}


void setzeSpielfeld() {
  spielfeld[altX][altY] = 0;              //Temporäres Spielfeld wird geleert
  
  //Die neuen Koordinaten werden bestimmt
  neuX = altX + richtung[0];
  neuY = altY + richtung[1];
  
  spielfeld[neuX][neuY] = 1;              //Spielfeld wird aktualisiert
  if (neuX == apfelX && neuY == apfelY) {
    score++;                              //Wenn der Überlebende den Apfel eingesammlt hat, erhöht sich der Punktestand
    if (score == 5) {
      levelUp();                          //Wenn der Punktestand 5 beträgt, ist der User ein Level weiter
    } else {
      setzeApfel();                       //Sonst setze einen neuen Apfel
    }
  }
  if (neuX > 7 || neuX < 0 || neuY > 7 || neuY < 0) {
    anzeigenEnde();                       //Game Over, falls das Spielfeld verlassen wird
    ende = 1;
  }
}

void anzeigenEnde() {
  //Beim Game Over sieht der User eine Animation
  ausmachenLichter();
  traurigesGesicht();
  delay(3000);
}

void traurigesGesicht() {
  //Darstellung eines traurigen Gesichts
  m.setDot(1, 1, HIGH);
  m.setDot(6, 1, HIGH);
  m.setDot(4, 2, HIGH);
  m.setDot(4, 3, HIGH);
  m.setDot(4, 4, HIGH);
  m.setDot(3, 4, HIGH);
  m.setDot(1, 7, HIGH);
  m.setDot(2, 6, HIGH);
  m.setDot(3, 6, HIGH);
  m.setDot(4, 6, HIGH);
  m.setDot(5, 6, HIGH);
  m.setDot(6, 7, HIGH);
}

void lachendesGesicht() {
  //Darstellung eines lachenden Gesichts
  m.setDot(1, 1, HIGH);
  m.setDot(6, 1, HIGH);
  m.setDot(4, 2, HIGH);
  m.setDot(4, 3, HIGH);
  m.setDot(4, 4, HIGH);
  m.setDot(3, 4, HIGH);
  m.setDot(1, 5, HIGH);
  m.setDot(2, 6, HIGH);
  m.setDot(3, 6, HIGH);
  m.setDot(4, 6, HIGH);
  m.setDot(5, 6, HIGH);
  m.setDot(6, 5, HIGH);
}
void ausmachenLichter() {
  //Mit einer doppelten for-Schleife werden alle LEDs ausgeschaltet
  for (int i = 0; i < 8; i++) {
    for (int j = 0; j < 8; j++) {
      m.setDot(i, j, LOW);
      spielfeld[i][j] = 0;
    }
  }
}

void levelUp() {
  //Bei einem Level Up erfolgt eine Animation und die Schnelligkeit des Überlebenden wird verdoppelt 
  ausmachenLichter();
  lachendesGesicht();
  delay(3000);
  ausmachenLichter();
  zeit = zeit / 2;
  
  myInit();     //Nach einem Level Up muss alles auf ursprüngliche Werte zurückgesetzt werden, daher wird myInit aufgerufen
}

void loop() {
  if (anfang == 0) {    
    start();
  } else {
    //Diese Schritte werden erst ausgeführt, wenn der Startbutton gedrückt wurde
    holeRichtung();         
    setzeSpielfeld();
    if (ende == 1) {
      ausmachenLichter();
      exit(loop);                 //Wenn das Spiel verloren ist, wird das Spiel abgebrochen
    }
    m.setDot(altX, altY, LOW);    //Alter Standpunkt wird ausgeschaltet
    altX = neuX;
    altY = neuY;
    m.setDot(neuX, neuY, HIGH);   //Neuer Standpunkt wird eingeschaltet
    delay(zeit);                  //LED leuchtet eine bestimmte Zeitspanne lang
  }
}
```
