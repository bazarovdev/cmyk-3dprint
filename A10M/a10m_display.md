---
layout: default
title: "Fixing Display in Klipper"
parent: "Geeetech A10M"
---

# A10M Display with 12-Pin Header

The printer was supplied with Marlin firmware, and the display worked correctly.  
However, after switching to Klipper, the display stopped working. I even found that I'm not the first:  
[How to diagnose why the Geeetech A10X display shows nothing](https://klipper.discourse.group/t/how-to-diagnose-why-the-geeetech-a10x-display-shows-nothing/679)

I initially thought it was just a matter of pin definitions, so I decided to map the pins from Marlin firmware to Klipper firmware.  
Using the board configuration for [Geeetech A10 V4.1](https://github.com/MarlinFirmware/Marlin/tree/bugfix-2.1.x/Marlin/src/pins/mega/pins_GT2560_V3.h) and the mapping from [Arduino pin numbers (e.g., 1) to AVR ports (e.g., E1)](https://github.com/MarlinFirmware/Marlin/tree/bugfix-2.1.x/Marlin/src/HAL/AVR/fastio/fastio_1280.h)  

We get the following for `YHCB2004`:

| #define in Marlin   | Arduino # | AVR port | Schematic NET | H2 pin | Reality   |
|---------------------|-----------|----------|---------------|--------|-----------|
| YHCB2004_CLK        | 5         | PE3      | LCM_D6        | 4      | CS        |
| YHCB2004_MOSI       | 21        | PD0      | LCM_D5        | 5      | CLK       |
| YHCB2004_MISO       | 36        | PC1      | LCM_D7        | 3      | MOSI      |
| BTN_EN1             | 16        | PH1      | LCM_D4        | 7      | BTN_EN1   |
| BTN_EN2             | 17        | PH0      | LCM_EN        | 8      | BTN_EN2   |
| BTN_ENC             | 19        | PD2      | EC_PRESS      | 9      | BTN_ENC   |
|                     |           |          | RESET         | 11     |           |
|                     |           |          | BEEP          | 12     |           |
|                     |           |          | V5            | 1      |           |
|                     |           |          | GND           | 2,6,10 |           |

According to the source code, the display is based on the  
[AiP31068](https://support.newhavendisplay.com/hc/en-us/article_attachments/4414498095511/AiP31068.pdf)  
and uses the following library for communication:  
[LiquidCrystal_AIP31068](https://github.com/red-scorp/LiquidCrystal_AIP31068)

In summary, the pin definitions are incorrect in Marlin firmware (they were reordered to make it work under the wrong names). See the [issue in MarlinFirmware](https://github.com/MarlinFirmware/Marlin/issues/25196).

To make it work in Klipper firmware, it was necessary to send commands in a 9-bit format as required by the `AIP31068`, but the existing driver only supported 8 bits.  
I solved the issue by creating a flexible [SPI driver](https://github.com/Klipper3d/klipper/pull/6633) that supports any number of bits. However, it wasn't accepted due to the risks of changing a core driver, especially since this display is rare. Kevin (the author of Klipper) proposed grouping eight 9-bit commands into nine bytes of data and sending them at once.  
This solution worked, and a [new driver](https://github.com/Klipper3d/klipper/pull/6639) was created and merged into Klipper.

![A10M display working](resources/display_works_klipper.png)
