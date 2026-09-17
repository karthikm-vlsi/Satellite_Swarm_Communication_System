#include <SPI.h>
#include <nRF24L01.h>
#include <RF24.h>

RF24 radio(9, 10);

#define NODE_ID 1

const int ledPin = 3;  // LED pin

const byte address[4][6] = {"00001","00002","00003","00004"};

struct DataPacket {
  int source;
  int destination;
  int lastNode;
  char message[32];
};

DataPacket data;

bool sendTo(int target) {
  if (target == NODE_ID) return false;

  radio.stopListening();
  radio.openWritingPipe(address[target - 1]);

  bool ok = radio.write(&data, sizeof(data));

  radio.startListening();
  return ok;
}

void setup() {
  Serial.begin(9600);
  radio.begin();

  radio.openReadingPipe(1, address[NODE_ID - 1]);
  radio.startListening();

  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW); // LED OFF
}

void loop() {

  //  RECEIVE
  if (radio.available()) {

    digitalWrite(ledPin, HIGH);  // LED ON (data received)

    radio.read(&data, sizeof(data));

    Serial.print("Node ");
    Serial.print(NODE_ID);
    Serial.println(" RECEIVED");

    delay(500);  // visible glow

    // If destination
    if (data.destination == NODE_ID) {
      Serial.println("Final Node Display");
      Serial.println(data.message);
    } 
    else {

      data.lastNode = NODE_ID;

      bool sent = false;

      for (int i = 1; i <= 4; i++) {
        if (i != NODE_ID) {

          sent = sendTo(i);

          if (sent) {
            Serial.print("Forwarded to ");
            Serial.println(i);
            break;
          }
        }
      }

      if (!sent) {
        Serial.println("Backup Node Display");
        Serial.println(data.message);
      }
    }

    digitalWrite(ledPin, LOW); // LED OFF after processing
  }

  //  Initial send (Node A only)
  if (NODE_ID == 1) {
    static bool sentOnce = false;

    if (!sentOnce) {
      delay(2000);

      data.source = 1;
      data.destination = 4;
      data.lastNode = 1;
      strcpy(data.message, "Temp=30C");

      bool sent = false;

      for (int i = 2; i <= 4; i++) {
        sent = sendTo(i);

        if (sent) {
          Serial.print("A sent to ");
          Serial.println(i);
          break;
        }
      }

      sentOnce = true;
    }
  }
}
