
Proyek Mesin Cuci Otomatis Berbasis ESP32
Ringkasan Proyek
Proyek ini adalah implementasi sistem otomatis untuk mesin cuci dua tabung, yang dikendalikan secara manual menggunakan mikrokontroler ESP32. Sistem ini mengubah mesin cuci konvensional menjadi perangkat pintar dengan alur kerja yang sepenuhnya otomatis, termasuk pengisian air, pencucian, pembuangan air, dan pengeringan.
Daftar Fitur
 * Kontrol Manual: Pengendalian penuh melalui 4 tombol fisik (Menu, Start, Stop, dan Spin).
 * Alur Otomatis: Menjalankan siklus pencucian, pembilasan, dan pembuangan air secara otomatis.
 * Mode Pencucian:
   * Mode 1: Cepat (10 Menit)
   * Mode 2: Standar (15 Menit)
   * Mode 3: Berat (20 Menit)
   * Mode 4: Pengeringan (5 Menit)
 * Sensor Level Air: Menggunakan 3 sensor level air untuk mendeteksi ketinggian air secara otomatis.
 * Notifikasi: Notifikasi suara melalui buzzer untuk setiap pergantian tahapan dan saat proses selesai.
 * Display: Tampilan status dan menu pada LCD 16x2 I2C.
Komponen Hardware
| Komponen | Jumlah | Keterangan |
|---|---|---|
| Mikrokontroler | 1 | ESP32 Dev Board |
| Relay Solid-State (SSR) | 5 | Untuk mengendalikan perangkat AC 220V |
| Motor Washing | 1 | AC 220V, 3 kabel (CW, CCW, Fasa) |
| Motor Drain | 1 | AC 220V |
| Motor Spin | 1 | AC 220V |
| Solenoid Valve | 1 | AC 220V untuk keran air otomatis |
| Tombol Push-Button | 4 | Untuk kontrol manual |
| LCD 16x2 I2C | 1 | Untuk tampilan status |
| Buzzer | 1 | Untuk notifikasi suara |
| Sensor Level Air | 3 | Sensor resistif (kabel tembaga) |
| Adaptor Daya | 1 | 5V untuk ESP32 dan relay |
| Kabel Jumper | Secukupnya | Untuk koneksi antar komponen |
Diagram Rangkaian
(Di sini, Anda bisa menyisipkan gambar atau tautan ke diagram rangkaian yang menunjukkan koneksi ESP32 ke semua komponen, termasuk skema terpisah untuk bagian tegangan rendah dan 220V AC.)
Kode Program (WashingMachine.ino)
Berikut adalah kode final untuk proyek mesin cuci otomatis dalam mode manual, yang sudah Anda minta sebelumnya.
#include <LiquidCrystal_I2C.h>

//--- Konfigurasi Hardware ---
LiquidCrystal_I2C lcd(0x27, 16, 2); // Alamat I2C 0x27, 16 kolom, 2 baris. Sesuaikan jika perlu.

//--- Pin Definitions ---
const int sensorEmpty = 34; // Pin ADC untuk sensor air terendah
const int sensorHalf = 35;  // Pin ADC untuk sensor air setengah
const int sensorFull = 32;  // Pin ADC untuk sensor air penuh
const int relayInlet = 23;  // Relay untuk solenoid valve air masuk
const int relayWashCW = 18; // Relay untuk motor cuci putaran kanan
const int relayDrain = 19;  // Relay untuk motor pembuangan
const int relayWashCCW = 5; // Relay untuk motor cuci putaran kiri
const int relaySpin = 4;    // Relay untuk motor pengering
const int buttonMenu = 13;  // Tombol Menu
const int buttonStart = 12; // Tombol Start/OK
const int buttonStop = 14;  // Tombol Stop
const int buttonSpin = 25;  // Tombol tambahan untuk Pengeringan (Spin)
const int buzzer = 26;      // Buzzer notifikasi

//--- Global Variables ---
int selectedMode = 0, selectedLevel = 0;
int step = 1; // 1: Pilih Mode, 2: Pilih Level
bool isRunning = false, isFinished = false;
unsigned long lastUpdate = 0;

//--- Function Declarations ---
void displayMenu();
void stopProcess();
void finishProcess();
void runMode();
void runSpinMode();
void fillWater(int level);
void drainWater();
void washCycle(int duration, const char* modeText);
void mode1();
void mode2();
void mode3();
void mode4();
void checkButtons();
int readSensorLevel();

//--- Setup ---
void setup() {
  Serial.begin(115200);
  pinMode(sensorEmpty, INPUT);
  pinMode(sensorHalf, INPUT);
  pinMode(sensorFull, INPUT);
  pinMode(relayInlet, OUTPUT);
  pinMode(relayWashCW, OUTPUT);
  pinMode(relayDrain, OUTPUT);
  pinMode(relayWashCCW, OUTPUT);
  pinMode(relaySpin, OUTPUT);
  pinMode(buttonMenu, INPUT_PULLUP);
  pinMode(buttonStart, INPUT_PULLUP);
  pinMode(buttonStop, INPUT_PULLUP);
  pinMode(buttonSpin, INPUT_PULLUP);
  pinMode(buzzer, OUTPUT);

  digitalWrite(relayInlet, HIGH);
  digitalWrite(relayWashCW, HIGH);
  digitalWrite(relayDrain, HIGH);
  digitalWrite(relayWashCCW, HIGH);
  digitalWrite(relaySpin, HIGH);

  lcd.begin();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Mesin Cuci v1.0");
  lcd.setCursor(0, 1);
  lcd.print("by WoodenLabs");
  delay(2000);
  
  displayMenu();
}

//--- Main Loop ---
void loop() {
  checkButtons();
  
  if (isFinished && millis() - lastUpdate >= 1000) {
    tone(buzzer, 1000, 200);
    delay(800);
    lastUpdate = millis();
    finishProcess();
  }
}

//--- Fungsi-Fungsi Utama ---
void checkButtons() {
  if (digitalRead(buttonMenu) == LOW && !isRunning && !isFinished) {
    if (step == 1) {
      selectedMode = (selectedMode % 4) + 1;
    } else if (step == 2 && selectedMode != 4) {
      selectedLevel = (selectedLevel % 2) + 1;
    }
    displayMenu();
    tone(buzzer, 1000, 200);
    delay(500);
  }
  
  if (digitalRead(buttonStart) == LOW && !isRunning && !isFinished) {
    if (step == 1 && selectedMode > 0) {
      if (selectedMode == 4) {
        runSpinMode();
      } else {
        step = 2;
        selectedLevel = 1;
        displayMenu();
        tone(buzzer, 1000, 200);
      }
    } else if (step == 2 && selectedLevel > 0) {
      isRunning = true;
      tone(buzzer, 1000, 500);
      runMode();
      step = 1;
    }
    delay(500);
  }

  if (digitalRead(buttonStop) == LOW && (isRunning || isFinished)) {
    tone(buzzer, 1000, 1000);
    stopProcess();
    delay(500);
  }

  if (digitalRead(buttonSpin) == LOW) {
    runSpinMode();
    delay(500);
  }
}

void displayMenu() {
  lcd.clear();
  if (step == 1) {
    lcd.setCursor(0, 0);
    lcd.print("Pilih Mode:");
    lcd.setCursor(0, 1);
    switch (selectedMode) {
      case 1: lcd.print("> 1(Cpt 10M)"); break;
      case 2: lcd.print("> 2(Stdr 15M)"); break;
      case 3: lcd.print("> 3(Brt 20M)"); break;
      case 4: lcd.print("> 4(Spin)"); break;
      default: lcd.print("Tekan Menu.."); break;
    }
  } else if (step == 2 && selectedMode != 4) {
    lcd.setCursor(0, 0);
    lcd.print("Pilih Level Air:");
    lcd.setCursor(0, 1);
    switch (selectedLevel) {
      case 1: lcd.print("> Setengah"); break;
      case 2: lcd.print("> Penuh"); break;
      default: lcd.print("Tekan Menu.."); break;
    }
  }
}

void fillWater(int level) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Mengisi Air..");
  digitalWrite(relayInlet, LOW);
  while (readSensorLevel() < level) {
    if (digitalRead(buttonStop) == LOW) {
      digitalWrite(relayInlet, HIGH);
      stopProcess();
      return;
    }
  }
  digitalWrite(relayInlet, HIGH);
  tone(buzzer, 1000, 200);
  delay(500);
}

void drainWater() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Membuang Air..");
  digitalWrite(relayDrain, LOW);
  while (readSensorLevel() > 0) {
    if (digitalRead(buttonStop) == LOW) {
      digitalWrite(relayDrain, HIGH);
      stopProcess();
      return;
    }
  }
  digitalWrite(relayDrain, HIGH);
  tone(buzzer, 1000, 200);
  delay(500);
}

void washCycle(int duration, const char* modeText) {
  lcd.clear();
  unsigned long startTime = millis();
  unsigned long lastSwitch = 0;
  bool isCW = true;

  while (millis() - startTime < (unsigned long)duration * 1000) {
    int remaining = (duration - (millis() - startTime) / 1000);
    lcd.setCursor(0, 0);
    lcd.print(modeText);
    lcd.setCursor(0, 1);
    lcd.print("Sisa:");
    lcd.print(remaining / 60 < 10 ? "0" : "");
    lcd.print(remaining / 60);
    lcd.print(":");
    lcd.print(remaining % 60 < 10 ? "0" : "");
    lcd.print(remaining % 60);

    if (millis() - lastSwitch >= 6000) {
      digitalWrite(relayWashCW, HIGH);
      digitalWrite(relayWashCCW, HIGH);
      delay(2000); 

      if (isCW) {
        digitalWrite(relayWashCW, LOW);
        digitalWrite(relayWashCCW, HIGH);
      } else {
        digitalWrite(relayWashCW, HIGH);
        digitalWrite(relayWashCCW, LOW);
      }
      isCW = !isCW;
      lastSwitch = millis();
    }

    if (digitalRead(buttonStop) == LOW) {
      stopProcess();
      return;
    }
  }
  
  digitalWrite(relayWashCW, HIGH);
  digitalWrite(relayWashCCW, HIGH);
  tone(buzzer, 1000, 200); 
  delay(300);
  tone(buzzer, 1000, 200); 
  delay(300);
  tone(buzzer, 1000, 200); 
}

void mode1() {
  int levelTarget = (selectedLevel == 1) ? 2 : 3;
  fillWater(levelTarget); if (!isRunning) return;
  washCycle(600, "Mencuci Cpt.."); if (!isRunning) return;
  drainWater(); if (!isRunning) return;
  finishProcess();
}

void mode2() {
  int levelTarget = (selectedLevel == 1) ? 2 : 3;
  fillWater(levelTarget); if (!isRunning) return;
  washCycle(600, "Mencuci Stdr.."); if (!isRunning) return;
  drainWater(); if (!isRunning) return;
  fillWater(levelTarget); if (!isRunning) return;
  washCycle(300, "Bilas 1.."); if (!isRunning) return;
  drainWater(); if (!isRunning) return;
  finishProcess();
}

void mode3() {
  int levelTarget = (selectedLevel == 1) ? 2 : 3;
  fillWater(levelTarget); if (!isRunning) return;
  washCycle(600, "Mencuci Brt.."); if (!isRunning) return;
  drainWater(); if (!isRunning) return;
  fillWater(levelTarget); if (!isRunning) return;
  washCycle(300, "Bilas 1.."); if (!isRunning) return;
  drainWater(); if (!isRunning) return;
  fillWater(levelTarget); if (!isRunning) return;
  washCycle(300, "Bilas 2.."); if (!isRunning) return;
  drainWater(); if (!isRunning) return;
  finishProcess();
}

void mode4() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Mengeringkan..");
  digitalWrite(relaySpin, LOW);
  for (int i = 300; i > 0; i--) {
    lcd.setCursor(0, 1);
    lcd.print("Sisa:");
    lcd.print(i / 60 < 10 ? "0" : "");
    lcd.print(i / 60);
    lcd.print(":");
    lcd.print(i % 60 < 10 ? "0" : "");
    lcd.print(i % 60);
    if (digitalRead(buttonStop) == LOW) {
      stopProcess();
      return;
    }
    delay(1000);
  }
  digitalWrite(relaySpin, HIGH);
  tone(buzzer, 1000, 200);
  finishProcess();
}

void runMode() {
  if (selectedMode == 1) mode1();
  else if (selectedMode == 2) mode2();
  else if (selectedMode == 3) mode3();
  else if (selectedMode == 4) runSpinMode();
}

void runSpinMode() {
  if (isRunning) {
    stopProcess();
    delay(1000);
  }
  isRunning = true;
  selectedMode = 4;
  tone(buzzer, 1000, 500);
  mode4();
}

void stopProcess() {
  digitalWrite(relayInlet, HIGH);
  digitalWrite(relayWashCW, HIGH);
  digitalWrite(relayDrain, HIGH);
  digitalWrite(relayWashCCW, HIGH);
  digitalWrite(relaySpin, HIGH);
  isRunning = false;
  isFinished = false;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Proses Dibatalkan");
  delay(2000);
  displayMenu();
  step = 1;
  selectedMode = 0;
  selectedLevel = 0;
}

void finishProcess() {
  isRunning = false;
  isFinished = true;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Proses Selesai!");
  lcd.setCursor(0, 1);
  lcd.print("Selamat Mencuci!");
}

int readSensorLevel() {
  int empty = analogRead(sensorEmpty);
  int half = analogRead(sensorHalf);
  int full = analogRead(sensorFull);
  if (full > 2000) return 3;
  else if (half > 2000) return 2;
  else if (empty > 2000) return 1;
  else return 0;
}

