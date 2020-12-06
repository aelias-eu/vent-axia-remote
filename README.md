# Vent-Axia Wired Remote Protocol
Vent-Axia [wired remote](https://www.vent-axia.com/product/sentinel-kinetic-wired-remote-controller) protocol reverse engineering.
![442899](/img/wired-remote.jpeg)

## My setup
The wired remote (item #442899) is connected to the [Sentinel Kinetic B](https://www.vent-axia.com/range/lo-carbon-sentinel-kinetic-b) MVHR (Mechanical ventilation with heat recovery) unit.
There is a RJ45 connector on the MVHR unit and free wires on the other side, connected to the Wired remote via screw terminals
![connection](/img/connection.png)

Terminal & wire colors are:

|Wire/terminal color|Signal|Memo|
|----|----|----|
|Yellow|Vcc|+5V|
|Green|RS232 Tx(relative to MVHR unit)|
|Red|RS232 Rx(relative to MVHR unit)|
|Black|GND|0V|

The wired remote is based on Microchip's [PIC16F627A](https://www.microchip.com/wwwproducts/en/PIC16F627A) and connection interface IC is the [202ECBZ](https://pdf1.alldatasheet.com/datasheet-pdf/view/532554/INTERSIL/202ECBZ.html). The connection interface IC on the PCB is marked as *U1*. Red and green terminals are connected to the U1 pins 14 and 13 via inductors L3,L2, and resistors R2 and R1 with filtration capacitors C2,C3. According to this, the MVHR unit should use standard RS232 voltage range (**Not TTL - so don't connect it directly to any MCU!!!**)

# Osciloscope measurements
As first, we need to know the voltage levels used in this system. After oscilloscope connection to the green line, we can see the data packets and that these voltage levels ale +/- 8.8V
![voltage levels](/img/green_line_voltage.jpeg)

Next, we can try to measure the baudrate - or to be more spectfic, the signal period length of shortest signal in the packet.
![Bit period](/img/bit_period.jpeg)

I like to measure the period of two signal bits and then divide it by two (just to get lesser reading error), but if you measure just one signal period, you will be fine in most cases. In this case, we have 0.208ms for two bits, that makes (divided by 2) 0.104ms == 104us (microseconds) for 1 data bit. You can look at the commond serial port speeds (https://en.wikipedia.org/wiki/Serial_port#Settings) and find out that 104us bitrate coresponds to the 9600 bits/s bitrate.
So - it looks like the **bitrate on this device is 9600bps**. 
But that's not all we need to know to succesfully communicate. We also need to know the data bit count, stop bits and parity bits. The most common setting is 8N1 - 8 data bits, no parity bit, 1 stop bit.

![Bit period](/img/packet_width.jpeg)
The packet length is 44.1ms, that makes (44.1/0.104) 424,0384 -> 424 bits in one packet
The baudrate seems to be 9600bps



## ToDo:
- [ ] Detect RS232 connection parameters (baudrate, databits, parity, stop bits)
- [ ] Connect dual serial port interface via listener and start data monitoring 


