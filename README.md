# e19868m20s-LORA-module-with-ESP8266
EBYTE e19868m20s LORA module with ESP8266, how to connect and example software

The EBYTE e19868m20s has to soldered on a breadboard , see my pictures.
The EBYTE e19868m20s is normally used with a SMD designed PCB.

Connection with the ESP8266 as receiver:

ESP8266-e19868m20s
-----------------
D0-DI00

D3-RESET

D5-SCK

D6-MISO

D7-MOSI

D8-N

GND-GND

EBYTE e19868m20s needs 3V3, do not use the 3v3 from the ESP8266, use a separated supply 3v3 which a regulator with a 5V input and 3v3 output. In one of the picture you can see how this was done with the assembly.



