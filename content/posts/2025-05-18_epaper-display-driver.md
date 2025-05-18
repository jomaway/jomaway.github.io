+++
title = "Writing an E-Paper Display Driver "

[taxonomies]
tags = ["epaper", "esp32", "c++", "embedded"]

+++

# Situation and Goals

I ordered some epaper display modules from [waveshare](https://www.waveshare.com/4.2inch-e-Paper-Module-B.htm) to let my students tinker with them. It is a three color display with 4.2inch and is based on the [SSD1683](https://www.solomon-systech.com/product/ssd1683/) controller.

Waveshare provides some examples on their [wiki](https://www.waveshare.com/wiki/4.2inch_e-Paper_Module_(B)) to get started, but I found the instructions a bit tricky to follow. Especially if you are still getting started with embedded programming. For my students, I wanted to provide a simple example with a library that they can easily include in a platformio project.

I also had a quick look at the [GxEPD2](https://github.com/ZinggJM/GxEPD2) library which seemed to be more what I was looking for. But for educational reasons and for the purpose of learning something myself, I decided to write my own driver library.

{{ note(header="Goal", body="<ul>
<li>Provide a simple library to use in a PlatformIO project.</li>
<li>Make the library compatible with an ESP32 and the Arduino Framework.</li>
<li>Keep the driver implementation as simple as possible.</li>
</ul>") }}

We will be using C++ as the primary programming language for this project, leveraging its object-oriented features to create a modular and reusable library for the e-paper display.
While this guide aims to provide a starting point and easy to understand implementation, certain aspects have been intentionally simplified for clarity and ease of understanding.

So let's get started.

# How to structure the code?

To keep the design simple and modular, I decided to go with the following architecture:

- HAL: contains the microcontroller specific pin and spi initialization.
- Driver: handles the display initialization and sending image data to the display.
- Graphics: simple graphics functions to draw on a buffer.
- User API: a wrapper to use the driver and graphics functions in a arduino typical way.

## The Hardware Abstraction Layer

Before we can decide how our HAL should look like, we need to understand how the display will be connected to the microcontroller.
The display uses SPI to communicate with the microcontroller, but it also has some additional pins to control the display. The following table shows the pinout of the display:


| Pin Name | Description |
| -------- | ----------- |
| VCC      | Power supply (3.3V) |
| GND      | Ground |
| DIN      | SPI Data In (MOSI) |
| CLK      | SPI Clock (SCK) |
| CS       | Chip Select (active low) |
| DC       | Data (high) / Command (low) |
| RST      | Reset (active low) |
| BUSY     | Busy (active high) |

Our HAL should handle everything which needs knowledge about the underlying hardware. This includes the SPI bus and some GPIO pins.
So we create an abstract interface that can be implemented for different microcontrollers.
The following code shows how the HAL interface looks like:

```cpp
class EPD_HAL {
public:
    virtual void init() = 0;
    virtual void send_command(uint8_t cmd) = 0;
    virtual void send_data(uint8_t data) = 0;
    virtual void wait_busy() = 0;
    virtual void set_reset_pin(bool state) = 0;
};
```

**Initialization:**

Referring to the [Datasheet](https://files.waveshare.com/wiki/4.2inch%20e-Paper%20Module%20(B)/4.2inch%20e-Paper%20(B)%20V2.pdf) (Section 6.3) we need to set the following SPI settings.

- Clock Speed: 20MHz
- Send `MSBFIRST`
- `SPI_MODE_0` (CPOL = 0 and CPHA = 0).

**Data or Command:**

The display can be operated with either 3-wire SPI or 4-wire SPI depending if the DC Pin is used or not. The module provides the option to change the setting by soldering a resistor on the backside. The 4-wire configuration is the default option, so we will use it.

This means:
- if the DC pin is `LOW` we send a command.
- if the DC pin is `HIGH` we send some data.

The `send_command` and `send_data` methods just wrap this functionality by setting the DC Pin accordingly.

**Reset:**

To awake the panel from deep sleep mode we need to use the hardware reset pin. In the Datasheet is states that it needs to be pulled `LOW` for 200μs and then back `HIGH` for another 200μs. This behavior can vary depending on the specific version of the display panel being used. To accommodate these differences and keep the implementation flexible, I decided to encapsulate the reset logic within the driver itself and just provide a `set_reset_pin` function in the HAL.

**Wait Busy:**

The `busy` pin is used to signal that the panel is still processing and you should not send new commands or data to the display.

## The Driver

Ok now that we have our `EPD_HAL` as a foundation we can continue with the driver itself.

The Driver should handle all the initialization of the display and provide functions to show an image on the screen and set it back into sleep mode afterwards. For convenience we also want a method to clear the screen.

After some consideration this interface is what I came up with:

```cpp
class EPD_Driver {
    protected:
        EPD_HAL* hal; // Uses hardware abstraction layer
        const uint16_t width = 400;  // Display width in pixels
        const uint16_t height = 300; // Display height in pixels

    public:
        EPD_Driver(EPD_HAL* hal) : hal(hal) {}
        void reset();
        void init();
        void clear();
        void write_framebuffer(const uint8_t *image, bool use_red_ram);
        void update();
        void sleep();
        void display(const uint8_t *image, bool use_red_ram);
};
```

### Initialization and sleep mode

Let's first look into the `init` process. The description in the datasheet and the provided examples diverged somewhat, but both give us a good starting point. Basically we need to do the following:

1. Do a hardware reset to awake the display from sleep mode.
2. Do a software reset to clear previous settings.
3. Send a bunch of commands to the display to config the options we want. These might change depending on your needs, but this should work as a minimal example.
  - Set the panels border waveform.
  - Select temperature sensor.
  - Set the displays RAMs Data Entry Mode.
  - Specify initial window (full) and cursor for the RAM.
4. Wait until the display is ready.



The [Datasheet](https://files.waveshare.com/wiki/4.2inch%20e-Paper%20Module%20(B)/4.2inch%20e-Paper%20(B)%20V2.pdf) (Chapter 7) provides a comprehensive table with all available commands and their options. I let a LLM generate a complete list of all commands from this table, which worked surprisingly well.

```cpp
namespace DisplayCmd {
    // System Control
    ...
    constexpr uint8_t DEEP_SLEEP_MODE   = 0x10; // Deep Sleep Mode Control
    constexpr uint8_t DATA_ENTRY_MODE   = 0x11; // Data Entry Mode Setting
    constexpr uint8_t SOFTWARE_RESET    = 0x12; // SW RESET
    ...
}
```

There you also can find the command for the sleep mode `0x10`. There is a Deep Sleep Mode 1 and 2. However, I could not find any difference between the two. So i went with Mode 2 as this was used in the example code provided by waveshare even if the datasheet suggest to use Mode 1.

{{ note(header="Deep sleep mode", body="As you should not keep the display in a high voltage state for too long it is important to send it back to deep sleep mode after updating the screen.")}}

We can use our HALs `send_` methods, to send a command and it's options to the display.

```cpp
void EPD_Driver::sleep()
{
    hal->send_command(DisplayCmd::DEEP_SLEEP_MODE);
    hal->send_data(0x01); // Deep Sleep Mode 2
}
```

### Show an Image

To show something on the epaper display we need to load the image data into the RAM of the epaper module and then tell it to update the screen. But as the display supports `RED`, `BLACK` and `WHITE` there are two memory locations. One for the `BLACK` and `WHITE` image and the other for the `RED` and `WHITE` image. Depending on your initial settings `WHITE` is representate by a `1` and the others are `0`. Each byte which we will send to the panel represents 8 pixels. You can find a visual representation on the [waveshare wiki page](https://www.waveshare.com/wiki/4.2inch_e-Paper_Module_(B)_Manual#Overview).

With this knowledge we can implement the `write_framebuffer` and `update` method:

```cpp
void EPD_Driver::write_framebuffer(const uint8_t *data, bool use_red_ram);
{
    // select RAM
    hal->send_command(use_red_ram ? DisplayCmd::WRITE_RAM_RED: DisplayCmd::WRITE_RAM_BW);

    // Send all pixels
    uint32_t w = this->width / 8;
    for (uint32_t j = 0; j < this->height; j++)
    {
        for (uint32_t i = 0; i < w; i++)
        {
            hal->send_data(image[i + j * w]);
        }
    }
}
```

Depending on the selected color we tell the display which memory location to choose and send the data.
As our display has a width of 400 pixel, which is a multiple of 8, we can savely just divide by 8 to calculate the bytes in one row.

To make the changes visible to the user we need to call the `update` function.
To update the display we need to send two command and wait until the display is ready.
```cpp
void EPD_Driver::update()
{
    hal->send_command(DISP_UPDATE_CTRL_2);
    hal->send_data(0xF7); // EN ANALOG, LOAD TEMP, DISP MODE 1, DIS ANALOG, DIS OSC
    hal->send_command(MASTER_ACTIVATION);
    hal->wait_busy();
}
```
The display supports also an option for fast updates. But we will not look into this today.

If we only use black and white images we could include the required commands in the `write_framebuffer` method.
But if we have an image with `RED` and `BLACK` we only want to update the screen after sending both images.
So we keep both functions seperated.

The `display` method is basically just a wrapper around those two functions for the mentioned usecase.
Also `clear()` is the same as writing a complete white framebuffer.

We are now able to show an image on the screen by sending an array of bytes to the module. As the flash memory of the esp32 is big enough for storing the complete image data we don't worry about partial updates for now.

We can use a service like [image2cpp](https://javl.github.io/image2cpp/) to convert an existing image into an array of bytes. Make sure your image has the correct dimensions (400x300) before uploading it. Then we copy the resulting bytes into the source code.

```cpp
const unsigned char TEST_IMAGE[] PROGMEM = {
/* 0X00,0X01,0X90,0X01,0X2C,0X01, */
0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,
...
}
```

For a quick demo we can test our newly written driver with the following program:
```cpp
// includes and defines ...

auto hal = EPD_HAL_ESP32(EPD_CS_PIN, EPD_DC_PIN, EPD_RST_PIN, EPD_BUSY_PIN);
auto epd = EPD_Driver(hal);

void setup()
{
    epd.init();  // Init the display
    epd.clear();
    epd.display(TEST_IMAGE,false);
    epd.sleep();
}

void loop(){}
```

This works as expected and we should see our image on the screen.


##  The Graphics Library

We could call it a day and just always convert images and copy them into our program or write a simple server where to fetch new images from.
But sometimes it is useful to be able to create a new image programmatically.
This is where the graphics library comes in. Instead of writing our own graphics library from scratch we will use [Adafruit_GFX](https://github.com/adafruit/Adafruit-GFX-Library) for this purpose.
This makes it fairly simple. We only need to subclass the `Adafruit_GFX` base class and implement the `drawPixel` method.

We could implement this in a way to draw directly to the epapers RAM. But this would need some more work on our `EPD_Driver` class to make it a smooth experience. To keep it simple i decided to go with a different approach.
In fact i decided to decouple it from the driver itself and only provide a `Canvas` class which is basically a framebuffer where you can draw on. After that you can send the imge to the display using the driver.

```cpp
auto canvas = Canvas(400,300);
// draw on the canvas with all functions which are provided by Adafruit_GFX
auto img = canvas.getImage();
epd.display(img, BLACK_IMAGE);
```

## The User API

To simplify usage and make the library feel more like a typical Arduino library, I designed a streamlined Epaper class. It abstracts away lower-level tasks like creating the HAL and driver instances, as well as automatically handling actions like calling sleep after displaying an image. This allows users to get started quickly with minimal setup.

This is what is looks like:
```cpp
typedef enum{
    BLACK = 0,
    RED = 1,
} Color_t;

class EPaper {
private:
    EPD_HAL *hal;
    EPD_Driver *driver;
public:
    EPaper(unsigned int cs_pin, unsigned int dc_pin, unsigned int reset_pin, unsigned int busy_pin);

    void begin();
    void showImage(const byte *image, Color_t color);
};
```

# Conclusion

Before starting this little project i thought it would be much harder and more work to get some minimal example up and running.
But i made quick progress and the datasheet and the existing examples provided enough information to hack a first working solution together in a couple of hours.

As it is often not more productive to write something from scratch it is very educative. This is what i love and my main motivation. There are probably a lot of things which can be further improved, like partial and fast updates or using dma to transfer the framebuffer. But i think i managed to keep it as simple as possible for a minimal starting point and still be able to improve it later.

{{ note(header="Source code", body="If you are interested you can find the full library [here](https://github.com/jomaway/rdf-epd-lib). _But as this project evolves over time it will contain other functions to support more complex workflows as well._

If you have suggestions for improvements feel free to contribute.")}}
