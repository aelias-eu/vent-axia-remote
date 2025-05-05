# Vent-Axia Wired Remote Protocol
Vent-Axia [wired remote](https://www.vent-axia.com/product/sentinel-kinetic-wired-remote-controller) protocol reverse engineering.

![442899](/img/wired-remote.jpeg)

## My setup
The wired remote (item #442899) is connected to the [Sentinel Kinetic B](https://www.vent-axia.com/range/lo-carbon-sentinel-kinetic-b) MVHR (Mechanical ventilation with heat recovery) unit.

There is a ?RJ11? connector on the MVHR unit and 4 free wires on the other side, connected to the Wired remote via screw terminals
![connection](/img/connection.png)
![connection_at_main_unit](/img/vent-axia-kbd-rj11.jpg)

Terminal & wire colors are:

|Wire/terminal color|Signal|Memo|
|----|----|----|
|Yellow|Vcc|+ 5V|
|Green|RS232 Tx(relative to MVHR unit)|± 10V|
|Red|RS232 Rx(relative to MVHR unit)|± 10V|
|Black|GND|0V|

The wired remote is based on Microchip's [PIC16F627A](https://www.microchip.com/wwwproducts/en/PIC16F627A) and connection interface IC is the [202ECBZ](https://pdf1.alldatasheet.com/datasheet-pdf/view/532554/INTERSIL/202ECBZ.html). The connection interface IC on the PCB is marked as *U1*. Red and green terminals are connected to the U1 pins 14 and 13 via inductors L3,L2, and resistors R2 and R1 with filtration capacitors C2,C3. Looking at the specs of the RS232 driver (202ECBZ), we can see, that the maximum Transmitter output voltage is ± 10V.

According to this, the MVHR unit uses standard RS232 voltage range (**Not TTL - so don't connect it directly to any MCU!!!**). 

# Oscilloscope measurements & bus parameters identification
As first, we will check the voltage levels used in this system. After oscilloscope connection to the green line, we can see the data packets and that these voltage levels ale ± 8.8V. According the datasheet, the ±9V is typical Transmitter output voltage for the RS232 driver 202ECBZ.
![voltage levels](/img/green_line_voltage.jpeg)

Next, we can try to measure the baudrate - or to be more spectfic, the signal period length of shortest signal in the packet.
![bit period](/img/bit_period.jpeg)

I like to measure the period of two signal bits and then divide it by two (just to get lower reading error), but if you measure just one signal period, you will be fine in most cases. In this case, we have 0.208ms for two bits, that makes (divided by 2) 0.104ms == 104us (microseconds) for 1 data bit. You can look at the common serial [port speeds](https://en.wikipedia.org/wiki/Serial_port#Settings) and find out that 104us period coresponds to the **9600 bits/s bitrate**.

But that's not all we need to know to succesfully communicate. We also need to know the data bit count, stop bits and parity bit. The most common setting is 8N1 - 8 data bits, no parity bit, 1 stop bit. We could narrow down the search by looking at the hardware. The MCU is PIC16F627 and by looking at it's [datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/40044F.pdf), we can find out that the USART is connected via pins RB1/**RX**/DT and RB2/**TX**/CK. We can also see, that his chip (as many members of the Microchip family do) supports 8 and 9 databits (but the 9th bit could still be a parity bit). Implementing anything else than 8 databits means too much work as this is 8-bit MCU. 

![packet width](/img/packet_width.jpeg)

The packet length is 44.1ms, that makes (44.1/0.104) 424.0384 -> 424 bits in one packet. These bit are not just the data bits, but also the bits, required for succesfull [USART/UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter) communication - the start bit, optional parity bit and stop bit(s). So to transmit 8 bits of our data, we have to send at least one start bit, 8 bits of our data and the stop bit. That makes 10bits (or 11 if we are using the parity bit). 
If we assume the 10 bits for 8N1 configuration, we get the number 424/10 = 42.4 - that's 42 Bytes in one packet. And the MVHR unit is sending a packet every 300ms, so theoretically, we get 126 bytes of data every second - and that could be a lot of informations or not - we will see more after connecting to the computer and looking at the data.

# The data
After connecting a full-duplex [RS232 sniffing cable](https://www.lammertbies.nl/comm/cable/rs-232-spy-monitor) and selecting parameters **96008N1** I finally saw the data and...
...this is a bit awkward - but I can't say that I didn't expect this when I found out, that this is a plain RS232 link. And the result is:

The wired remote is sort-of a dumb serial connected display/keyboard. That means - it shows on the LCD diplay exactly what it receives. And why did I think of this before? Because of the language support - it's easier to maintain and upgrade just one device - in this case the MVHR main control board. If the manufacturer adds language suppoort there, he doesn't need to do anything with wired or wireless remotes as long as the firmware change does not change the wired remote communication protocol.  

The LCD display is 2 lines of 16 characters
For example the packet is (everything is HEX data):

`02 00 00 08 07 15 53 74 72 65 64 6E 69 20 50 72 75 74 6F 6B 20 20 16 33 35 20 25 20 20 20 20 20 20 20 20 20 20 20 20 F7 D8`

where the first 6 bytes look like a header:

`02 00 00 08 07 15` 

then there is the main data for the first display line:

`53 74 72 65 64 6E 69 20 50 72 75 74 6F 6B 20 20`

folowed by new line character 

`16` 

then there is the main data for the first display line:

`33 35 20 25 20 20 20 20 20 20 20 20 20 20 20 20`

and packet end - is the CRC

`F7 D8`

If you try to decode the data into ASCII, you will get **"Stredni Prutok  "** for the first line and **"35 %            "** for the second line. 
It may look gibberish to you, but it means "Middle flow" in Czech language, I suppose I don't have to translate the 35% :)

So it won't be hard to recreate a virtual keypad via ESP32, but I have to say, I was looking forward to analyze & break a bit more sophisticated communication protocol... But hey - at least creating a multi-drop connection from this standard RS232 could be a bit challenge. Why? Because we have to solve control command sending and I'd like to make it collision free.  

There are 4 keys on the keypad and by pressing a button, the keypad will send set of repeated packets with short delay between packets (18-20ms). If you hold the button longer, the keypad will just keep sending the apropriate packet.
The keys are sent as much smaller packets (8 byte):
- `04 05 AF EF FB 08 FD 55` - Main button (ventilator icon)
- `04 05 AF EF FB 01 FD 5C` - Down button
- `04 05 AF EF FB 02 FD 5B` - Up button
- `04 05 AF EF FB 04 FD 59` - SET button

The main change is on the 6th byte (01 / 02 / 04 / 08) and the last (CRC). Key combinations are just a simple add operation on the 6th byte and of course the CRC change.

- `04 05 AF EF FB 09 FD 54` - Main + Down
- `04 05 AF EF FB 0A FD 53` - Main + Up
- `04 05 AF EF FB 0C FD 51` - Main + SET
- `04 05 AF EF FB 03 FD 5A` - Down + Up
- `04 05 AF EF FB 05 FD 58` - Down + SET
- `04 05 AF EF FB 06 FD 57` - Up + SET

Nothing changes if the key / key combination is pressed for a long time (5s and more). 

## VentAxia CRC calculation
The CRC in this case is essentially what you get by subtracting all byte values from 0xFFFF (as seen in [vent-axia-bridge](https://github.com/ryancdotorg/vent-axia-bridge) by @ryancdotorg.

Here is PoC code in arduino compatible code. You can simulate it on wokwi.com or try it here: [vent-axia-kbd-protocol@wokwi](https://wokwi.com/projects/430088578742810625)
It just writes the HEX representation of generated packets to the serial console.
```cpp
#define VA_PACKET_MAX_SIZE 42

#define VA_KEY_DOWN   1
#define VA_KEY_UP     2
#define VA_KEY_SER    4
#define VA_KEY_BOOST  8

uint8_t outBuff[VA_PACKET_MAX_SIZE];  // output buffer
uint8_t outSize=0;                    // output data size
String  outLine1;
String  outLine2;

/* function: printBuffer
*  - Print HEX data representation of the buffer content
*/
void printBuffer(uint8_t buffer[], uint8_t buffSize) {
  if (buffSize > 0) {
    for (int i=0; i<buffSize; i++) {
      if (buffer[i]<16) { Serial.print('0'); }  // add leading "0" if needed
      Serial.print(buffer[i],HEX);
      Serial.print(" ");
    }
    Serial.println();
  }
}

/* function: VentAxiaSendLcdText
*  - Generate packet for showing message on LCD display with correct CRC 
*/
void VentAxiaSendLcdText(String Line1, String Line2, uint8_t buffer[], uint8_t* buffSize) {
  *buffSize=39;
  buffer[0]=0x02;
  buffer[1]=0x00;
  buffer[2]=0x00;
  buffer[3]=0x0c;
  buffer[4]=0x07;
  buffer[5]=0x15;
  for (int i=0; i<16; i++) { 
    if (Line1.length() >= i+1){
      buffer[6+i]=Line1.charAt(i);
    }
    else {
      buffer[6+i]=0x20; 
    }
  } 
  buffer[22]=0x16;
  for (int i=0; i<16; i++) { 
    if (Line1.length() >= i+1){
      buffer[23+i]=Line1.charAt(i);
    }
    else {
      buffer[23+i]=0x20; 
    }
  } 
  uint16_t crc=VentAxiaCrcCalc(buffer, *buffSize); // calculate & attach CRC
  buffer[39] = crc >> 8;
  buffer[40] = crc & 255;
  *buffSize=41;
}

/* function: VentAxiaSendKeyPress 
*  - Generate keyboard press packet with correct CRC 
*/
void VentAxiaSendKeyPress(uint8_t key, uint8_t buffer[], uint8_t* buffSize) {
  // KeyPress Packet: 04 05 AF EF FB [KEY_CODE] [CRC_BYTE1] [CRC_BYTE2]
  // Orther version:  04 06 FF FF FF [KEY_CODE] [CRC_BYTE1] [CRC_BYTE2]
  buffer[0]=0x04;
  buffer[1]=0x05;
  buffer[2]=0xAF;
  buffer[3]=0xEF;
  buffer[4]=0xFB;
  buffer[5]=key;
  *buffSize = 6;
  uint16_t crc=VentAxiaCrcCalc(buffer, *buffSize); // calculate & attach CRC
  buffer[6] = crc >> 8;
  buffer[7] = crc & 255;
  *buffSize = 8;
}

/* function: VentAxiaCrcCalc 
*  - Calculate CRC for VentAxia packets
*/
uint16_t VentAxiaCrcCalc(uint8_t buffer[], uint8_t buffSize) {
  if ((buffSize > 0) && (buffSize+2 <= VA_PACKET_MAX_SIZE)) {
    uint16_t crc = 0xFFFF;
    for (int i=0; i<buffSize; i++) {
      crc -= buffer[i];
    }
    return crc;
  }
  else {
    return 0x0000;
  }
}  

void SendSerial(uint8_t buffer[], uint8_t buffSize) {
   
}

void setup() {
  Serial.begin(115200);

  // Send max. 16 characters per line. Will be stripped down automatically
  outLine1="Text Message L1 "; 
  outLine2="Text Message L2";
  VentAxiaSendLcdText(outLine1, outLine2, outBuff, &outSize);
  Serial.println("LCD Message:");
  printBuffer(outBuff, outSize);
  delay(10000);
  for (uint8_t i=0; i<255; i++){
    Serial.print("key_code: ");
    Serial.println(i);
    VentAxiaSendKeyPress(i, outBuff, &outSize);
    printBuffer(outBuff, outSize);
  }
}

void loop() {
}
```

## To Do:
- [x] Detect RS232 connection parameters (baudrate, databits, parity, stop bits)
- [x] Connect dual serial port interface via listener and start data monitoring 
- [x] Sniff data for key combinations and check for changes for long-pressed buttons
- [ ] Implement virtual keypad with ESP32 with interface to MQTT

## Possible future usage
- Create virtual keypad on the phone or webinterface
- Integrate MVHR unit into any smart-home solution

# Used tools
- Serial port terminal program - in my case [hterm](http://der-hammer.info/pages/terminal.html) - set approx. 10ms timeout for packet separation
- Oscilloscope - in my case a DSO (Digital Signal Oscilloscope)
- Full duplex serial port sniffing cable - you can find the connection & explanation [here](https://www.lammertbies.nl/comm/cable/rs-232-spy-monitor)
- Arduino & ESP online simulator & more [Wokwi.com](https://www.wokwi.com)

# Links & Other projects
 - [vent-axia-remote-oled-pic](https://github.com/brianmarchant/vent-axia-remote-oled-pic) - @brianmarchant did some nice work on the custom firmware
 - [ESP8266-Sentinel-Kinetic-Wired-Remote](https://github.com/alextrical/ESP8266-Sentinel-Kinetic-Wired-Remote) - Completely custom build keyboard with ESP8266, including 3d printed case by @alextrical
 - [vent-axia-bridge](https://github.com/ryancdotorg/vent-axia-bridge) - @ryancdotorg created an ESP8266 based controller via the BMS port.
