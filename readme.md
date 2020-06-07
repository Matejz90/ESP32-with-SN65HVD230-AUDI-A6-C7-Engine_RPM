What you need:
1x ESP-WROOM32
1x SN65HVD230

ESP32CAN library from miwagner (THANKS!!!!!) https://github.com/miwagner/ESP32-Arduino-CAN

Wiring:
ESP32------SN65HVD230------CAR
3v3--------3v3
GND--------GND
CTX--------GPIO_5
CRX--------GPIO_4
           CANH-----------HUB-CANH
           CANL-----------HUB-CANL


How to start:
Fist start esp32can_basic so you can read some data with what can you begin. I recommend that you delete lines from // Send CAN Message, because in my case intterupt data reading.
Then build project and send to ESP32. Fire up serial monitor and wait some seconds. Then copy data to excel or notepad++ and find messages with most entrys.
When you have sorted this, then try to figure whitch entry is for RPM. You can start searching with string "0x0C". For VAG group you need bit 2 and bit 3 (if you count from 0).
You can calculate this hex with google and formule 0.25 * (256 * B + A) //if B = bit 3 and A = bit 2

Mine was for idling car:
11:18:38.612 -> New standard frame from 0x00000105, DLC 8, Data 0xC2 0x0A 0xEC 0x0C 0x83 0x53 0x00 0xFC 
This adress throw 0.25 * (256 * 0x0C + 0xEC) = 827 RPM

Now code for calculating

```arduino
#include <Arduino.h>
#include <ESP32CAN.h>
#include <CAN_config.h>

CAN_device_t CAN_cfg;               // CAN Config
unsigned long previousMillis = 0;   // will store last time a CAN Message was send
const int interval = 1000;          // interval at which send CAN Messages (milliseconds)
const int rx_queue_size = 10;       // Receive Queue size
char temp[100];
char teml[100]; 

void setup() {
  Serial.begin(115200);
  Serial.println("Basic Demo - ESP32-Arduino-CAN");
  CAN_cfg.speed = CAN_SPEED_500KBPS;
  CAN_cfg.tx_pin_id = GPIO_NUM_5;
  CAN_cfg.rx_pin_id = GPIO_NUM_4;
  CAN_cfg.rx_queue = xQueueCreate(rx_queue_size, sizeof(CAN_frame_t));
  // Init CAN Module
  ESP32Can.CANInit();
}
   
void loop() {

  CAN_frame_t rx_frame;

  // Receive next CAN frame from queue
  if (xQueueReceive(CAN_cfg.rx_queue, &rx_frame, 3 * portTICK_PERIOD_MS) == pdTRUE) {
      for (int i = 0; i < rx_frame.FIR.B.DLC; i++) {
        //get data from bit 3 and 4 and save to temp array
        if (i == 2 && rx_frame.MsgID == 0x00000105){
          sprintf(teml, "0x%02X ", rx_frame.data.u8[2]);
          }
        if (i == 3 && rx_frame.MsgID == 0x00000105){
          sprintf(temp, "0x%02X ", rx_frame.data.u8[3]);
          }
      }
      //convert hex string from array to long int
      long int valueHex = strtol(temp, NULL, 16); //convert hex string to decimal
      long int valueHex1 = strtol(teml, NULL, 16); //convert hex string to decimal
      //calculate values for engine_rpm
      long int finalRpm = 0.25 * (256 * valueHex + valueHex1);
      Serial.println(finalRpm);
  }
}```
