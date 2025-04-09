+++
title = "Writing an E-Paper Display Driver "

draft = true

[taxonomies]
tags = [ "esp32", "c++", "embedded"]
+++


# Writing an E-Paper Display Driver

I ordered some epaper display modules from [waveshare](https://www.waveshare.com/4.2inch-e-Paper-Module-B.htm) to let my students tinker with them. It's a three color display with 4.2inch.

Waveshare provides some examples on their [wiki](https://www.waveshare.com/wiki/4.2inch_e-Paper_Module_(B)) to get started, but I found the instructions a bit tricky to follow if you not have the same board as they do. Especially if you are still getting started with embedded programming. For my students, I wanted to provide a simple example with a library that they can easily include in a platformio project.

I also had a quick look at the [GxEPD2](https://github.com/ZinggJM/GxEPD2) library but found the documentation and code still a bit complex for my usecase. As i only need the basic functionality for one specific display I decided to write my own driver and library.

{{ note(header="Goal", body="<ul>
<li>Make the library compatible with an ESP32 and the Arduino Framework.</li>
<li>Provide a simple library to use the display in a Platformio project.</li>
<li>Keep the driver implementation as simple as possible.</li>
</ul>") }}

## How to structure the code?

To keep the design simple and modular, I decided to go with the following architecture:

- HAL: contains the microcontroller specific pin and spi initialization.
- Driver: handles the display initialization and sending image data to the display.
- Graphics: simple graphics functions to draw on a buffer.
- User API: a wrapper to use the driver and graphics functions in a simple way.

### The Hardware Abstraction Layer

Before we can decide how our HAL should look like, we need to understand how the display will be connected to the microcontroller.
The display uses SPI to communicate with the microcontroller, but it also has some additional pins to control the display. The following table shows the pinout of the display:

| Pin Name | Description |
| -------- | ----------- |
| VCC      | Power supply (3.3V) |
| GND      | Ground |
| DIN      | SPI Data In (MOSI) |
| CLK      | SPI Clock (SCK) |
| CS       | Chip Select (active low) |
| DC       | Data (high) /Command (low) |
| RST      | Reset |
| BUSY     | Busy (active high) |

Our HAL should handle everything which needs knowledge about the underlying hardware. This includes the SPI bus and some GPIO pins.
I could think of two different approaches to this requirement:

- Either we provide just a simple wrapper around the SPI library and the GPIO functions of the microcontroller.
- Or we create a more abstract interface that can be implemented for different microcontrollers.

I went with the second option. The following code shows how the HAL interface looks like:

```cpp
class EPD_HAL {
public:
    virtual void init() = 0;
    virtual void send_command(byte cmd) = 0;
    virtual void send_data(byte data) = 0;
    virtual void reset() = 0;
    virtual void wait_busy() = 0;
};
```

You can see my concrete implementation for the ESP32 with the Arduino Framework [here](/github).

### The Driver


###  The Graphics Library


### The User API


## Conclusion
