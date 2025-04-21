+++
title = "Writing an E-Paper Display Driver "

draft = true

[taxonomies]
tags = [ "esp32", "c++", "embedded"]
+++


# Writing an E-Paper Display Driver

I ordered some epaper display modules from [waveshare](https://www.waveshare.com/4.2inch-e-Paper-Module-B.htm) to let my students tinker with them. It's a three color display with 4.2inch.

Waveshare provides some examples on their [wiki](https://www.waveshare.com/wiki/4.2inch_e-Paper_Module_(B)) to get started, but I found the instructions a bit tricky to follow if you don't have the same board as they do. Especially if you are still getting started with embedded programming. For my students, I wanted to provide a simple example with a library that they can easily include in a platformio project.

I also had a quick look at the [GxEPD2](https://github.com/ZinggJM/GxEPD2) library but found the documentation and code still a bit complex for my usecase. As i only need the basic functionality for one specific display I decided to write my own driver and library.

{{ note(header="Goal", body="<ul>
<li>Make the library compatible with an ESP32 and the Arduino Framework.</li>
<li>Provide a simple library to use in a PlatformIO project.</li>
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
| DC       | Data (high) / Command (low) |
| RST      | Reset (active low) |
| BUSY     | Busy (active high) |

Our HAL should handle everything which needs knowledge about the underlying hardware. This includes the SPI bus and some GPIO pins.
I could think of two different approaches to this requirement:

- Either we provide just a simple wrapper around the SPI and the GPIO functions of the microcontroller.
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

**Initialization:**

Refering to the [Datasheet](https://files.waveshare.com/wiki/4.2inch%20e-Paper%20Module%20(B)/4.2inch%20e-Paper%20(B)%20V2.pdf) (Section 6.3) we need to set the following SPI settings.

- Clock Speed: 20MHz
- Send `MSBFIRST`
- `SPI_MODE_0` (CPOL = 0 and CPHA = 0).

**Data or Command:**

The display can be operated with either 3-wire SPI or 4-wire SPI depending if the DC Pin is used or not. The module provides the option to change the setting by soldering a resistor on the backside. The 4-wire configuration is the default option, so we will use it. The `send_command` and `send_data` methods just wrap this functionality by setting the DC Pin accordingly.

**Reset:**

To awake the panel from deep sleep mode we need to use the hardware reset pin. In the Datasheet is states that it needs to be pulled `LOW` for 200μs and then back `HIGH` for another 200μs. But in the example code provided by waveshare i found this code snippet which works well:

```cpp
digitalWrite(reset_pin, HIGH);
delay(200); // 200ms
digitalWrite(reset_pin, LOW);
delay(2);  // 2ms
digitalWrite(reset_pin, HIGH);
delay(200); // 200ms
```

**Wait Busy:**

The `busy` pin is used to signal that the panel is still processing and you should not send new commands or data to the display.

{{ note(header="", body="You can see my concrete implementation for the ESP32 with the Arduino Framework [here](/github).") }}


### The Driver

Ok now that we have our `EPD_HAL` as a foundation we can continue with the driver itself.

The Driver should handle all the initialization of the display and provide some functions to load an image on the screen and set it back into sleep mode afterwards. For convenience we also want a method to clear the screen.

After some consideration this interface is what i came up with:

```cpp
class EPD_Driver {
    protected:
        EPD_HAL* hal; // Uses hardware abstraction layer
        const int width = 400;  // Display width in pixels
        const int height = 300; // Display height in pixels
    public:
        EPD_Driver(EPD_HAL* hal) : hal(hal) {}
        void init();
        void clear();
        void display(const byte *image, uint8_t image_color, bool update = true);
        void update();
        void sleep();
};
```

#### Initialization and sleep mode

Let's first look into the `init` process. As with the `reset` function the Datasheet and provided examples diverged somewhat, but both give us a good starting point. Basically we need to do the following:

- Do a hardware reset to awake the display from sleep mode
- Do a software reset to clear previous settings.
- Send a bunch of commands to the display to config the options we want. These might change depending on your needs, but i went with the following:
  - Set Data Entry Mode `0x11`.
  - Set RAM x `0x44` and y `0x45` address.
  - Set the panels border waveform `0x3C`.
  - Reset RAMs address counter for x `0x4E` and y `0x4F`.

The datasheet provides a comprehensive table with all available commands and their options.
There you also can find the command for the sleep mode `0x10`. There is a Deep Sleep Mode 1 and 2. However, I could not find any difference between the two. So i went with Mode 2 as this was used in the example code provided by waveshare even if the datasheet suggest to use Mode 1.

{{ note(header="Deep sleep mode", body="As you should not keep the display in a high voltage state for too long it is important to send it back to deep sleep mode after updating the screen.")}}

#### Show an Image

To show something on the epaper display we need to load the image data into the RAM of the epaper module and then tell it to update the screen. But as the display supports `RED`, `BLACK` and `WHITE` there are two memory locations. One for the `BLACK` and `WHITE` image and the other for the `RED` and `WHITE` image. Depending on your initial settings `WHITE` is representate by a `1` and the others are `0`. Each byte which we will send to the panel represents 8 pixels. You can find a visual representation on the [waveshare wiki page](https://www.waveshare.com/wiki/4.2inch_e-Paper_Module_(B)_Manual#Overview).

With this knowledge we can implement the `display` method:

```cpp
void EPD_Driver::display(const byte *image, uint8_t image_color, bool update)
{
    uint32_t w = (this->width % 8 == 0) ? (this->width / 8) : (this->width / 8 + 1);

    // select RAM
    if (RED_IMAGE == image_color)
    {
        hal->send_command(WRITE_RAM_RED);
    }
    else if (BLACK_IMAGE == image_color)
    {
        hal->send_command(WRITE_RAM_BW);
    }
    else
    {
        return; // return early because of error.
    }

    // Send all pixels
    for (uint32_t j = 0; j < this->height; j++)
    {
        for (uint32_t i = 0; i < w; i++)
        {
            hal->send_data(image[i + j * w]);
        }
    }

    // update display
    if (update)
    {
        this->update();
    }
}
```

If used in landscape mode the width will always be 400 pixel. So the first line could be simplified to `this->width / 8`. But to make it a bit more flexible we add one byte if this is not the case.
Then depending on the selected color we tell the display which memory location to choose and send the data.

To make the changes visible to the user we need to call the `update` function. If we need to only set one color we could always call update, but if we have an image with `RED` and `BLACK` we only want to update the screen after sending both images. This is why i deciced to add the update flag as parameter.

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

We are now able to show an image on the screen by sending an array of bytes to the module. As the flash memory of the esp32 is big enough for storing the complete image data we don't worry about partial updates for now.

We can use a service like [image2cpp](https://javl.github.io/image2cpp/) to convert an existing image into an array of bytes. Make sure your image has the correct dimensions (400x300) before uploading it. Then we copy the resulting bytes into the source code.

```cpp
const unsigned char TEST_IMAGE[] PROGMEM = {
/* 0X00,0X01,0X90,0X01,0X2C,0X01, */
0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,
...
}
```

For a quick demo we can test our newly written driver with the following programm:
```cpp
// includes and defines ...

auto hal = EPD_HAL_ESP32(EPD_CS_PIN, EPD_DC_PIN, EPD_RST_PIN, EPD_BUSY_PIN);
auto epd = EPD_Driver(hal);

void setup()
{
    epd.init();  // Init the display

    epd.clear();
    epd.display(TEST_IMAGE,BLACK_IMAGE);
    epd.sleep();
}

void loop(){}
```

This works as expected and we should see our image on the screen.

###  The Graphics Library

We could call it a day and just always convert images and copy them into our programm or write a simple server where to fetch new images from.
But sometimes it is usefull to be able to create a new image programmatically.
This is where the graphics library comes in. Instead of writing our own graphics library from scratch we will use [Adafruit_GFX](https://github.com/adafruit/Adafruit-GFX-Library) for this purpose.
This makes it fairly simple.


### The User API


## Conclusion
