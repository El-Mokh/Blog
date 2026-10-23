---
title: "Driver for the Joy-it OLED display"
date: 2026-10-23

showSummary: true
---


[main.c code for the STM32F4x microcontroller](main.c)

## How to use
### Board compatibility
You'll need an `STM32 NUCLEO-F446RE` to run this code on considering the pin and port configurations can change on other board. If you still wanna use this code, you can change the ports and pins directly on the code but you'll have to check your own board's reference manual.
### Wiring 

```text
 _____________
|             | 
|             |-- GND !--------- MALE TO MALE !--------- GND
|     OLED    |-- VCC !--------- MALE TO MALE !--------- 3V3
|             |-- SCL !--------- MALE TO MALE !--------- D15
|             |-- SDA !--------- MALE TO MALE !--------- D14
|_____________|

```
## How It Works

### To Initialize I2C (Considering the clock to be already enabled RCC)

1. Reset and clear the I2C peripheral by setting the `SWRST` bit in `CR1` to `HIGH and then LOW`
2. Configuring the input clock by setting the `value in CR2`, the Value is to the `bus's frequency`, in my case 42Mhz so either 42U or 0x2A
3. Configuring the clock control register, `CCR`. You write the value in the register, which you get by `dividing the frequency` you have(42MHz) by the `clock speed` you want (100kHz) -> 420 ticks and `then divide by 2` cause 420 ticks half of it is high and half is low -> 210U or 0xD2
4. configuring max rise time `TRISE` which because we chose 100kHz, it simplifies the formula to your bus frequency + 1 -> 42 +1 = 43U or 0x2B
5. Finally we enable the peripheral in the CR1 register by setting the appropriate bit to 1

### To Send data/ commands :

1. We set the `START` bit in `CR1` and **!!WAIT!!** until `SB` flag in `SR1` becomes 1
2. Write the target address `shifted left by 1` into DR and **!!WAIT!!** until the `ADDR flag` in `SR1` becomes 1
3. We have to `clear the ADDR` flag or else the bus will *freeze*, you have to `read both register SR1 and SR2`, the code for that :
        (void) I2C1 -> SR1;
        (void) I2C1 -> SR2;
4. wait for TXE(Transmit data register) in SR1 register to become 1 and then write the command 

------------ The next part is for the Joy-it Oled display//

* 4a. Either write 0x00 to tell the display that the next byte is a command or 0x40 to write it directly into GDDRAM to light up pixels 

* 4b. wait for TXE(Transmit data register) in SR1 register to become 1

* 4c. Write your actual command

---------------------------------------------------------//

5. wait for the BTF flag in the SR1 register to become 1 and then set the STOP bit CR1


### How the Joy-It Display works

**IT DOESN'T TAKE AS INPUT (X,Y) COORDS FOR PIXELS**

We have pages and columns: 
* we have `8 horizontal pages`, 0-7, and `for each page 128 columns`: each page has 8 rows of pixels
* `1 byte of pixel data` affects the `first column` of the page
* the display `auto increments`, so if you send a byte for a specific column, the pointer of the display will point to the next column after drawing your command.