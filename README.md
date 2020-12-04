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

Next, we can try to measure the baudrate - or to be more spectfic, the signal periode of shortest signal in the packet.

The baudrate seems to be 9600bps

## ToDo:
- [ ] Detect RS232 connection parameters (baudrate, databits, parity, stop bits)
- [ ] Connect dual serial port interface via listener and start data monitoring 


