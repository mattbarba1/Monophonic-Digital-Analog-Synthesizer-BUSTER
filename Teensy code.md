```
#include <MIDI.h>
#include "USBHost_t36.h"
#include <SPI.h>

// usb host setup
USBHost myusb;
MIDIDevice buster(myusb);

const int statusLED = 6;
// MOSI pin 11, goes to SDIN
//SCK pin 13, goes to SCLK on DAC
const int DAC_CS = 10;  //goes to SYNC on DAC  

float V_semitone = 85.33333;      // 12-bit steps per semitone (resolution/total keys = 4096/48 for 4 octaves)

// note on/off
bool noteHeld = false;
byte heldNote = 0;
float bend = 0.0;               // Current pitch bend value (-8192 to +8191)
int lastDacValue = -1;          // Tracks last sent DAC value to avoid unnecessary writes

// constantly recalculates note when pitch bends
int calculate() {
    // Normalize bend to -1.0/+1.0 then scale to ±2 semitones.
    float semi_bend = (bend / 8192.0f) * (2 * V_semitone);

    // C3 (MIDI #48) = 0V, each semitone adds V_semitone steps
    int result = ((heldNote - 60) * V_semitone) + semi_bend;

    // notes <#48 is at 0 V, >#108 is at VREF 2.5V)
    return constrain(result, 0, 4095);
}

// send the 16-bit value to the AD5684 DAC over SPI
void writeDAC(uint16_t value) {
    // AD5684 is a 24-bit word:
    // Bits 23-20: command (0011 = write and update)
    // Bits 19-16: address (0001 = DAC A)
    // Bits 15-4:  12-bit data
    // Bits 3-0: dont care
    uint32_t packet = 0b00110001UL << 16 | value;    // update value + DAC A address       
    packet |= (value & 0x0FFF) << 4;             //12 bit value left alligned

    digitalWrite(DAC_CS, LOW);             // Begin SPI transaction (SYNC low)
    SPI.transfer((packet >> 16) & 0xFF);   // Send top byte
    SPI.transfer((packet >> 8) & 0xFF);    // Send middle byte
    SPI.transfer(packet & 0xFF);           // Send bottom byte
    digitalWrite(DAC_CS, HIGH);            // End SPI transaction (SYNC high)
}

void setup()
{
    Serial.begin(115200);
    while (!Serial && millis() < 3000) {
        delay(10);
    }

    pinMode(DAC_CS, OUTPUT);
    pinMode(statusLED, OUTPUT);
    digitalWrite(DAC_CS, HIGH);        //CS is active low, set high at startup
    SPI.begin();
    SPI.beginTransaction(SPISettings(30000000, MSBFIRST, SPI_MODE1)); // 

    Serial.println("USB Host MIDI starting...");
    myusb.begin();
    Serial.println("Waiting for USB MIDI device...");
}

void loop() {
    myusb.Task();
    //Green LED turns on when MIDI connected
    if(buster){
 digitalWrite(statusLED, HIGH); 
  } else {
    digitalWrite(statusLED, LOW);
 }           
    //TESTING VALUES FOR ARDUINO              
    if (buster.read())
    {
        switch (buster.getType())
        {
            case midi::ProgramChange:
                Serial.print("Program Change: ");
                Serial.println(buster.getData1());
                break;

            case midi::NoteOn:
                if (buster.getData2() > 0) {   // Ignore velocity 0
                //IMPORTANT: TURNS NOTE ON WHEN PLAYED, KEEPS CS AT LOW FOR NANOSECONDS CONTINUOSLY
                    noteHeld = true;
                    heldNote = buster.getData1();
                    int dacValue = calculate();  //calculate() calculates the note played
                    writeDAC((uint16_t)dacValue); //sends value to DAC
                    lastDacValue = dacValue;
                    Serial.print("12-bit Value Sent to DAC: ");
                    Serial.println(dacValue);
                }
                break;

            case midi::NoteOff:
                if (buster.getData1() == heldNote) {
                    noteHeld = false;          // Release the held note flag
                    lastDacValue = -1;         // Reset so next note always sends fresh value
                }
                Serial.println("Note Off");
                break;

            case midi::PitchBend:
                // Combine two 7-bit data bytes into a 14-bit value, centered at 0
                bend = (int16_t)(buster.getData1() | (buster.getData2() << 7)) - 8192;
                Serial.print("Pitch Bend: ");
                Serial.println(bend);
                if (noteHeld) {
                    int dacValue = calculate();
                    writeDAC((uint16_t)dacValue);
                    lastDacValue = dacValue;
                    Serial.print("12-bit Value Updated: ");
                    Serial.println(dacValue);
                }
                break;

            default:
                break;
        }
    }

    // Continuously update DAC while note is held, but only if value has changed
    if (noteHeld) {
        int dacValue = calculate();
        if (dacValue != lastDacValue) {    // Only write to DAC if value has changed
            writeDAC((uint16_t)dacValue);
            lastDacValue = dacValue;
            Serial.print("Note On - Note: ");
            Serial.print(heldNote);
            Serial.print("  12-bit Value: ");
            Serial.println(dacValue);
        }
    }
}
