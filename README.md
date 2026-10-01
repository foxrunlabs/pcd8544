# PCD8544 LCD Library for STM32

A lightweight C++20 driver for PCD8544-based monochrome LCDs, such as the Nokia 5110 display, designed for STM32 microcontrollers using the STM32 Low-Layer (LL) peripheral APIs.

The library provides a simple object-oriented interface for configuring and driving an 84 × 48 pixel PCD8544 LCD over hardware SPI. It supports text output, direct display RAM access, contrast adjustment, and full-screen bitmap graphics.

The project was developed and tested using an **STM32F411RE** microcontroller on a **Nucleo-F411RE** development board with a SparkFun Nokia 5110 LCD.

## Features

- C++20 implementation
- Hardware SPI communication
- STM32 Low-Layer (LL) peripheral interface
- 84 × 48 pixel display support
- Built-in 6 × 8 pixel font
- 14-column × 6-row text display
- Character and string output
- Automatic text wrapping
- Newline, carriage return, and form-feed handling
- Adjustable LCD contrast
- Direct PCD8544 RAM addressing
- Raw 8-pixel column writes
- Full-screen bitmap rendering
- No dynamic memory allocation
- Packaged as an STM32CubeIDE static library

## Hardware

The PCD8544 is a low-power monochrome LCD controller originally used in Nokia mobile phones. Common Nokia 5110 LCD modules expose the controller through a serial interface and provide an **84 × 48 pixel** display.

This library uses hardware SPI for communication with the display.

The following signals are required:

| PCD8544 Signal | Function |
| --- | --- |
| SCE / CE | Chip enable |
| RST | Display reset |
| D/C | Data/command selection |
| DIN | SPI MOSI |
| CLK | SPI clock |
| VCC | Display power |
| GND | Ground |

`DIN` and `CLK` are driven by the STM32 SPI peripheral. `SCE`, `RST`, and `D/C` are controlled through GPIO.

> **Note:** Pin order and electrical characteristics can vary between Nokia 5110 display modules. Verify the pinout and voltage requirements of your specific module before connecting it to an STM32 board.

## Software Architecture

The driver is implemented as a `PCD8544` C++ class. An instance represents a single display and contains the SPI peripheral and GPIO resources required to communicate with it.

```cpp
PCD8544(
    SPI_TypeDef* spi_port,
    GPIO_TypeDef* sce_port,
    unsigned int sce_pin,
    GPIO_TypeDef* rst_port,
    unsigned int rst_pin,
    GPIO_TypeDef* dc_port,
    unsigned int dc_pin
);
```

The constructor initializes the SPI peripheral and display controller, configures the PCD8544 for normal operation, applies the default bias and contrast settings, and clears the display.

Copy and move operations are disabled because a `PCD8544` object owns a specific set of MCU peripheral resources.

## Requirements

The library was developed with:

- STM32CubeIDE
- STM32F411RE
- STM32F4 CMSIS device support
- STM32F4 Low-Layer GPIO driver
- STM32F4 Low-Layer SPI driver
- C++20

The repository was originally tested with **STM32CubeIDE 1.8.0** and a **Nucleo-F411RE** development board.

Because the current implementation includes `stm32f411xe.h` and the STM32F4 LL GPIO/SPI headers directly, it is targeted at the STM32F411/STM32F4 environment rather than being a hardware-independent PCD8544 driver.

## Adding the Library to a Project

Add the following files to your STM32 project:

```text
Inc/
└── pcd8544.hpp

Src/
└── pcd8544.cpp
```

Ensure that the STM32F4 CMSIS and LL GPIO/SPI headers are available through the project's include paths.

Include the library with:

```cpp
#include "pcd8544.hpp"
```

The application is responsible for configuring the SPI peripheral and associated GPIO pins before constructing the display object.

## Basic Usage

After configuring SPI and GPIO with STM32CubeIDE/STM32CubeMX, construct a display using the appropriate peripheral and GPIO definitions.

For example:

```cpp
#include "pcd8544.hpp"

PCD8544 lcd{
    SPI1,
    GPIOA, LL_GPIO_PIN_4,   // SCE
    GPIOA, LL_GPIO_PIN_1,   // RST
    GPIOA, LL_GPIO_PIN_0    // D/C
};

int main()
{
    lcd.clear();

    lcd.set_cursor(0, 0);
    lcd.print("Hello, STM32!");

    while (true)
    {
    }
}
```

The exact SPI instance and GPIO pins depend on the hardware configuration of the application.

## Text Output

The display is divided into character cells using the library's built-in **6 × 8 pixel font**.

With an 84 × 48 pixel display, this provides:

```text
14 columns × 6 rows
```

### Print a string

```cpp
lcd.set_cursor(0, 0);
lcd.print("PCD8544");
```

### Print individual characters

```cpp
lcd.print('A');
lcd.print('B');
lcd.print('C');
```

### Control characters

`print()` recognizes several standard control characters:

| Character | Behavior |
| --- | --- |
| `\n` | Move to the beginning of the next row |
| `\r` | Return to the beginning of the current row |
| `\f` | Clear the display |

For example:

```cpp
lcd.print("Line 1\nLine 2");
```

Characters automatically advance the internal display position. When output reaches the right edge of the display, the cursor advances to the next character row.

## Cursor Positioning

Use `set_cursor()` for character-oriented positioning:

```cpp
lcd.set_cursor(column, row);
```

Valid character positions are:

```text
column: 0–13
row:    0–5
```

Example:

```cpp
lcd.set_cursor(4, 2);
lcd.print("STM32");
```

Coordinates outside the display dimensions wrap to the corresponding valid location.

## Direct Display RAM Access

The PCD8544 organizes its 84 × 48 display RAM into six horizontal banks. Each byte represents a vertical column of eight pixels.

The library exposes this organization through:

```cpp
lcd.set_ram_addr(x, bank);
lcd.set_pixels(value);
```

where:

```text
x:    0–83
bank: 0–5
```

For example:

```cpp
lcd.set_ram_addr(10, 2);
lcd.set_pixels(0b11111111);
```

writes a vertical group of eight illuminated pixels at the selected RAM address.

This interface allows custom graphics to be generated without requiring a full framebuffer in MCU RAM.

## Bitmap Graphics

A complete display image consists of:

```text
84 columns × 6 banks = 504 bytes
```

The library can transfer a complete bitmap directly to display RAM:

```cpp
std::array<std::uint8_t,
           PCD8544::screen_width * PCD8544::banks> bitmap{};

// Populate bitmap...

lcd.draw_bitmap(bitmap);
```

Each byte represents eight vertically arranged pixels using the native memory organization of the PCD8544.

Using the controller's native format avoids the need for an intermediate framebuffer or pixel conversion during transfer.

## Contrast

Display contrast can be changed at runtime:

```cpp
lcd.set_contrast(69);
```

The library limits the maximum Vop setting to protect against exceeding the PCD8544's recommended operating voltage at low temperatures.

The default Vop value is:

```text
69
```

which corresponds to approximately **7.2 V** using the controller's Vop relationship.

Actual visible contrast varies with the LCD module, supply voltage, and temperature.

## Public API

### `PCD8544(...)`

Constructs and initializes a display using the specified SPI peripheral and GPIO connections.

### `set_contrast(int level)`

Sets the PCD8544 operating voltage used to control LCD contrast.

### `clear()`

Clears display RAM.

### `set_cursor(int column, int row)`

Positions the cursor using 6 × 8 character coordinates.

### `print(char c)`

Writes a character and interprets supported control characters.

### `print(std::string_view s)`

Writes a string to the display.

### `write(unsigned char c)`

Writes a character glyph directly without interpreting control characters.

### `set_ram_addr(int x, int y)`

Sets the native PCD8544 display RAM address.

### `set_pixels(std::uint8_t pixels)`

Writes one byte of raw pixel data at the current display RAM address.

### `draw_bitmap(...)`

Writes a complete 504-byte bitmap to the display.

## Design Notes

This project intentionally uses the STM32 **Low-Layer (LL)** API rather than implementing the driver on top of the higher-level HAL SPI interface.

Display transfers are performed directly through the SPI peripheral. The driver explicitly controls the PCD8544's D/C and SCE signals around each command or data byte and waits for the SPI peripheral to complete transmission before releasing the display.

The implementation also avoids dynamic allocation. Text is accepted through `std::string_view`, bitmaps use fixed-size `std::array` objects, and display operations are performed directly against the controller's RAM organization.

These choices keep the interface small while demonstrating the interaction between modern C++ abstractions and memory- and resource-constrained embedded hardware.

## Project Status

Version **1.0.0** was released in May 2022 following an update of the library to idiomatic C++20.

The project is primarily intended as a compact STM32 driver and embedded C++ reference implementation. The current code specifically targets the STM32F411/STM32F4 LL environment.

See `CHANGELOG.md` for the project's revision history.

## License

pcd8544 is available under the [Apache License, Version 2.0](LICENSE.txt).

Copyright © 2022 Ryan Clarke

