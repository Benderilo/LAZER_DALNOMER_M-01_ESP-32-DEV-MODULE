# LAZER_DALNOMER_M-01_ESP-32-DEV-MODULE

// ================================================
// ЛАЗЕРНЫЙ ДАЛЬНОМЕР M01 - ПРАВИЛЬНАЯ ВЕРСИЯ
// ПАРСИНГ 13-БАЙТНЫХ ПАКЕТОВ И BCD ДЕКОДИРОВАНИЕ
// ================================================

#include <HardwareSerial.h>

#define RX2 16  // Желтый (TXD датчика) -> GPIO 16
#define TX2 17  // Зеленый (RXD датчика) -> GPIO 17
#define ENA_PIN 5

HardwareSerial SerialLaser(2);

const uint8_t CMD_SINGLE[] = {0xAA, 0x00, 0x00, 0x20, 0x00, 0x01, 0x00, 0x00, 0x21};
const uint8_t CMD_CONTINUOUS[] = {0xAA, 0x00, 0x00, 0x21, 0x00, 0x01, 0x00, 0x00, 0x22};
const uint8_t CMD_STOP[] = {0xAA, 0x00, 0x00, 0x21, 0x00, 0x00, 0x00, 0x00, 0x21};

uint8_t packetBuffer[32];
int packetIndex = 0;
bool waitingForPacket = false;
int expectedLen = 9; // По умолчанию большинство служебных ответов 9 байт

void setup() {
  Serial.begin(115200);
  pinMode(ENA_PIN, OUTPUT);
  digitalWrite(ENA_PIN, HIGH);
  
  SerialLaser.begin(9600, SERIAL_8N1, RX2, TX2);
  delay(500);
  
  Serial.println("\n✅ СИСТЕМА ГОТОВА!");
  Serial.println("Отправьте 'C' для непрерывного режима или 'Q' для одиночного.");
}

void loop() {
  while (SerialLaser.available() > 0) {
    uint8_t b = SerialLaser.read();
    
    if (!waitingForPacket) {
      if (b == 0xAA) {
        packetBuffer[0] = b;
        packetIndex = 1;
        waitingForPacket = true;
        expectedLen = 9; // Сбрасываем на дефолт
      }
    } else {
      packetBuffer[packetIndex++] = b;
      
      // На 4-м байте (индекс 3) мы уже знаем тип команды
      if (packetIndex == 4) {
        uint8_t func = packetBuffer[3];
        // 0x20 (Одиночное), 0x21 (Непрерывное), 0x22 (Быстрое) присылают 13 байт!
        if (func == 0x20 || func == 0x21 || func == 0x22) {
          expectedLen = 13;
        }
      }
      
      // Если собрали весь пакет
      if (packetIndex >= expectedLen) {
        processPacket(packetBuffer, expectedLen);
        waitingForPacket = false;
        packetIndex = 0;
      }
    }
  }
  
  // Управление с клавиатуры
  if (Serial.available()) {
    char cmd = Serial.read();
    if (cmd == 'q' || cmd == 'Q') {
      SerialLaser.write(CMD_SINGLE, sizeof(CMD_SINGLE));
      Serial.println("-> Одиночное измерение...");
    } else if (cmd == 'c' || cmd == 'C') {
      SerialLaser.write(CMD_CONTINUOUS, sizeof(CMD_CONTINUOUS));
      Serial.println("-> Непрерывный режим запущен...");
    } else if (cmd == 's' || cmd == 'S') {
      SerialLaser.write(CMD_STOP, sizeof(CMD_STOP));
      Serial.println("-> Остановка...");
    }
  }
}

void processPacket(uint8_t* buf, int len) {
  // 1. Считаем CRC (Сумма всех байт с 1 по предпоследний)
  uint8_t crc = 0;
  for (int i = 1; i < len - 1; i++) {
    crc += buf[i];
  }
  bool crcOk = (crc == buf[len - 1]);

  if (!crcOk) {
    Serial.println("❌ Ошибка CRC! Пакет поврежден.");
    return;
  }

  // 2. Если это пакет измерения (13 байт)
  if (len == 13) {
    // Декодируем BCD формат (Байты 6, 7, 8, 9)
    long distance_mm = 0;
    distance_mm += (buf[6] >> 4) * 10000000;
    distance_mm += (buf[6] & 0x0F) * 1000000;
    distance_mm += (buf[7] >> 4) * 100000;
    distance_mm += (buf[7] & 0x0F) * 10000;
    distance_mm += (buf[8] >> 4) * 1000;
    distance_mm += (buf[8] & 0x0F) * 100;
    distance_mm += (buf[9] >> 4) * 10;
    distance_mm += (buf[9] & 0x0F) * 1;

    // Сила сигнала (Байты 10 и 11)
    uint16_t signal = (buf[10] << 8) | buf[11];

    Serial.print("📏 Расстояние: ");
    Serial.print(distance_mm / 1000.0, 3);
    Serial.print(" м (");
    Serial.print(distance_mm);
    Serial.print(" мм) | Сигнал: ");
    Serial.println(signal);
  } 
  // 3. Обработка ошибок оборудования (обычно 9 байт, начинается с EE или содержит код)
  else if (buf[0] == 0xEE) {
     Serial.print("⚠️ Аппаратная ошибка датчика. Код: ");
     Serial.println(buf[6], HEX);
  }
}
