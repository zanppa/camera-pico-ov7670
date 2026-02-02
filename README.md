# Raspberry-Pi Pico OV7670 Camera Library

This library can be used to capture images from an OV7670 camera on the
Raspberry Pi Pico, using PIO and DMA.

In the [original repository](https://github.com/usedbytes/camera-pico-ov7670) 
a 74165 shift register was used to convert the 8-bit parallel bus to a 2-bit 
serial interface. [In a fork](https://github.com/dattran-itrvn/camera-pico-ov7670) 
the interface was changed to support the 8-bit parallel interface directly. In 
this repository the 8-bit parallel interface is still used.

The PIO portion uses 2-4 state machines from a single PIO instance, depending
on the number of planes in the pixel format. The PIO program(s) use 26 of the
32 bytes of PIO program memory.

## SM0

State machine 0 handles the "frame"-level timing.
It receives the number of rows and columns via the FIFO, then waits for VSYNC
(start of frame). Then, for each line it waits for HSYNC (start of line) and
triggers the "pixel" state-machine(s) for each byte of data.

## SM1-3

The other state machines handle each byte of data. They will wait for the pixel
to get latched into the 74165 shift register (based on PXCLK), then shift the
data into the PIO ISR, to be pushed into the output FIFO.

A different state machine is used for each "plane" of image data.

A different number of bits gets stored in the output FIFO depending on the pixel
format. For example, for the single-plane YUYV/RGB565 formats, 4 bytes at a time
are transferred (representing 2 pixels). For 3-plane YUV422, on the 'Y' state
machine, each transfer is 16 bits (two 'Y' samples), wheras for the 'U' and 'V'
state machines, a single byte is transferred at a time.

## DMA

The DMA is configured to read from each state machine's input FIFO and transfer
the data to the destination buffer.

## OV7670 Driver

The OV7670 sensor is quite poorly documented - there are datasheets out there
and an integration guide, but it leaves out lots of details and there's a huge
number of undocumented (or even reserved) registers which need to be configured
to get a picture out of the camera.

This library is using [Adafruit's OV7670 driver](https://github.com/adafruit/Adafruit_OV7670.git).

I'm aware that there's actually a Pico implementation in that repo, but I had
already written most of my code before I found it. Also, the "arch" they have
doesn't quite fit with how I want to integrate the code.

So, I've just taken the low-level bits (largely, the reference register settings),
and hacked it up/wrapped it with my camera code.

This could definitely be done better, but I'm in a hurry and this is working :-).

## Circuit

The code in this repo assumes a specific wiring.

The `base_pin` and `xclk_pin` can be specified in `camera_platform_config`,
other pins are relative to `base_pin`.

`xclk_pin` must be one of the `GPOUT` clock sources (GPIO numbers 21, 23, 24 or 25).

The i2c bus can be connected however you like - just pass in appropriate
functions in `camera_platform_config` (it's done this way so that the i2c
bus can be shared amongst different devices).

The default `base_pin` is 10 in the example below.

| OV7670 Module  | Pico            | Pico (`example`) |
| -------------- | --------------- | ---------------- |
| `3.3V`         | `3V3(OUT)`      | `3V3(OUT)`       |
| `GND`          | `GND`           | `GND`            |
| `SIOC`/`SCL`   | User decision   | 1                |
| `SIOD`/`SDA`   | User decision   | 0                |
| `VSYNC`/`VS`   | `base_pin` + 8  | 18               |
| `HREF`/`HS`    | `base_pin` + 9  | 19               |
| `PCLK`         | `base_pin` + 10 | 20               |
| `XCLK`/`MCLK`  | User decision   | 21               |
| `D7`           | `base_pin` + 7  | 17               |
| `D6`           | `base_pin` + 6  | 16               |
| `D5`           | `base_pin` + 5  | 15               |
| `D4`           | `base_pin` + 4  | 14               |
| `D3`           | `base_pin` + 3  | 13               |
| `D2`           | `base_pin` + 2  | 12               |
| `D1`           | `base_pin` + 1  | 11               |
| `D0`           | `base_pin`      | 10               |
| `RESET`/`RST`  | User decision   | 22               |
| `PWDN`/ `PWNN` | `GND`           | `GND`            |

`PWDN` could also be connected to a GPIO so that user could 
power down the module by writing the pin high.

## Shortcomings

I've written this for a very specific use case (my Pi Wars robot), so I haven't
put much effort into making it very flexible. I'm publishing it as a separate
library because I think it could be a good base to work from.

Many of the shortcomings below could be quite easily fixed/improved.

* Resolution is fixed to 80x60
  * The Adafruit code can support a few different resolutions, it would be easy
    to add support for those. Totally arbitrary resolutions would be harder.
* Only includes PIO code for the shift-register implementation
  * A new "byte" program which uses a parallel input would be easy to add. I've
    tried that setup during my development, but the code changed a lot since
    then.
* Fudged timings/delay cycles.
  * I've tweaked some delay numbers to make it work on my set-up with a quite
    long cable between the camera and the Pico
  * My logic analyser isn't fast enough, and my oscilloscope is too dated to
    look at the signals and make some better judgement here.
* No control of frame-rate/PLL frequency
  * The register settings just set the OV7670 frequency to 1x the XCLK frequency.
* Could use more efficient transfer sizes
  * Could always do 4 byte DMA transfers as long as the buffer sizes are
    properly aligned to give multiples of 4 samples.
* Pretty lax error checking/handling.
* `camera_term` not implemented.

Anyway, if you just want 80x60 RGB565, YUYV or YUV422 images, this should work
great :-)

## Example

The `example` directory has a simple example which captures a frame then
writes an ASCII representation of the Luma (Y) plane to `stdio_usb`. The
character map is designed to look best using **light text on a dark background**.

Here's an example picture of a cat ornament:

```
+++++************************************************+++++++<
+*******************************************************++++<
********************************************************++++<
*******************************************************+++++<
+************************************************+++++++++++<
++++++++****************************************************<
*******************=--+*************************************<
******************=::::-+***********************************<
*****************+:::::::+**********************+++*********<
*****************=.::::..:=*******************+-:::+*****+++<
*****************-..::....:-=+===--==++*****+-:....-+***++++<
****************+:..::....:::::::::::::-=++-:......:++++++++<
++**************+:........::.::::::--::::::::......:+**+++++<
+***************=.........:..::::.:---:::::::......:+***++++<
****************=...........:...:.:---:.:::::......:+***++++<
****************=........:........:---:.:::........:+****+++<
****************=................:----::....::.....:+*****++<
****************=................:-----:.....:::...-*******+<
****************+................:-----:......::...=*******+<
****************+.::.............:-----:........:..=********<
****************+:::.............:----:.........:.:+********<
****************+::..............:--=-:.........::-*********<
****************+:...............:-===:..........:-*********<
****************+::.......:......:-===:.:........:=*********<
****************+::.......:......:-===-.:.........-*********<
****************-::.......::....:-==++=::...:.....:+********<
****************-::........:::::-======-::::::....:+********<
****************-:.......:::::-===-----==::::.....:+********<
****************=:....:::----======----===-:.......=********<
****************=::...::-===========--=====-:......=********<
****************=::..::======++=============-:.....=********<
****************+:..::-====++++++===-=======--:...:+********<
*****************-..:--====++++++===-=======--:...-*********<
*****************+:.:-====++==========-=====---:..-*********<
******************-::-=================--------:.:+*********<
*******************:::-=================-------:.=**********<
*******************+:.:::-------=======-------:::+**********<
*******************=....::----------=-------:::.:-+*********<
*******************:...::-------------------::...::-=*******<
******************+:...:--------------------::....:::-+*****<
******************+::::-=======---===-------=-:::.::::-+****<
******************=::--=========-====----======-:::::::-+***<
******************+--============---=---=========::::.::-**+<
******************+===============--=====++===+===:.....:=+=<
******************+==+============-=====+++++++===-...::::--<
******************++++++==========-=====+++++++====:::----::<
*********++++*******++++=========-==--===+++++=====-::---==:<
*******++++++++++***++++========--=--==============:::-====-<
*****+++++++++++++**++++=======---=================---======<
*****+++#%*****+++****+++======---=+=============++++++++++=<
****+++*#*****##++****+++======--=++=-==========++++++++++==<
****+++********##+****+++++=-===-=+++====--=====+++++==+++==<
****+++**********#*****+++*+====++++++==--=====+++++====+===<
****+++****************++++++++++++++++=======+++++=========<
****+++*****************++++++++++++++++++++++++++==========<
****+++********++++*****++++++++++=:-++++++++++++===========<
*****++********++++*****++++++++-:-.:=-=++++++++======++====<
*****+++*******++++*****++++++++-.:-:-.:++++++=========+====<
******++*******++=+******+++++++-:-==-.:=+++================<
******+++******++==+*****++++++-::=-:--=++=================-<
```

## License

My code is BSD-3 Clause to match the Pico SDK.

Adafruit's OV7670 driver is MIT.
