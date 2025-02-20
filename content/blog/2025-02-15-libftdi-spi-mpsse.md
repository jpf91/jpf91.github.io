---
title: Using MPSSE Mode with libftdi
author:
  - Johannes Pfau
description: Here's a quick tutorial on how to use MPSSE mode available on some FTDI chips to implement synchronous communication.
tags:
  - Software
  - OSS
  - FTDI
ShowToc: true
categories: Hardware Development
draft: false
series: ["Transferring Data from USB to FPGA using SPI"]
---
FTDI chips are commonly used for FPGA boards and microcontroller programmer boards.
Apart from basic UART, advanced FTDI chips also support synchronous IO using the [MPSSE mode](https://ftdichip.com/software-examples/mpsse-projects/).

Unfortunately, examples for MPSSE are outdated, Windows-only or otherwise limited.
This tutorial therefore explains the MPSSE basics and presents a simple [SPI](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface) communication example for an [FT232H](https://ftdichip.com/products/ft232hl/) controller using [libftdi](https://www.intra2net.com/en/developer/libftdi/).

<!--more-->

## Installing Libraries

We first need to install `libftdi-devel`. Use the following commands for Fedora 41:
```bash
# If you're running in a new container, you won't have these installed
sudo dnf install gcc g++
# libftdi itself
sudo dnf install libftdi-devel
```

## Helper Code

Now we can start programming our application.
First, let's include required headers:
```c++
#include <ftdi.h>

// For usleep
#include <unistd.h>
```

To make our life a little easier, let's also define an `enforce` function that checks error codes and throws exceptions on unexpected results:
```c++
#include <stdexcept>
#include <string.h>
using std::string;

void enforce(bool condition, string msg = "Enforcement failed")
{
    if (!condition)
        throw std::runtime_error(msg);
}
```
Better error handling is needed for real applications, but this code at least ensures that we will not miss any errors.

I will put all code in a `SPIDriver` class, so let's define it here.
Let's also declare the `_ftdi` member, a handle for the `libftdi` FTDI device:
```c++
class SPIDriver
{
private:
    struct ftdi_context _ftdi;

public:
    SPIDriver() {}
}
```

To finally compile the examples in this article, use this command:
```bash
g++ main.cpp -o main -lftdi1 `pkg-config libftdi1 --cflags`
```

## Opening the FTDI Device

Before we can send commands to the FTDI device, we have to open a handle to it.
Let's define an `open` function in the `SPIDriver` class, following the usual `libftdi` tutorials:
```c++
#define VENDOR 0x0403
#define PRODUCT 0x6014

void open()
{
    auto status = ftdi_init(&_ftdi);
    enforce(status == 0, "Failed to initialize libftdi");
    status = ftdi_usb_open(&_ftdi, VENDOR, PRODUCT);
    enforce(status == 0, "Failed to open device: " + string(ftdi_get_error_string(&_ftdi)));
```

Next, we reset the FTDI device and choose the MPSSE mode.
```c++
    enforce(ftdi_usb_reset(&_ftdi) == 0, "ftdi_usb_reset failed");
    enforce(ftdi_set_interface(&_ftdi, INTERFACE_ANY) == 0, "ftdi_set_interface failed");
    enforce(ftdi_set_bitmode(&_ftdi, 0, 0) == 0, "ftdi_set_bitmode failed");
    enforce(ftdi_set_bitmode(&_ftdi, 0, BITMODE_MPSSE) == 0, "ftdi_set_bitmode failed");
```

Finally, clear the internal buffers. Just in case the device was not properly reset.
```c++
    enforce(ftdi_tcioflush(&_ftdi) == 0, "ftdi_tcioflush failed");
    usleep(50000);
}
```

## Closing the FTDI Device

Let's also add a `close` function to clean up the FTDI handle on exit:
```c++
void close()
{
    ftdi_usb_reset(&_ftdi);
    enforce(ftdi_usb_close(&_ftdi) == 0, "ftdi_usb_close failed");
}
```

## Configuring the MPSSE

{{< box info >}}
  There's unfortunately little documentation for the MPSSE engine in general and there's no documentation for the MPSSE mode in `libftdi` at all.
  If you want to check the sources for the information here, refer to App Notes [AN2232C-01 Command Processor](https://www.ftdichip.com/Documents/AppNotes/AN2232C-01_MPSSE_Cmnd.pdf) and [FTDI MPSSE Basics](https://www.ftdichip.com/Documents/AppNotes/AN_135_MPSSE_Basics.pdf).
{{< /box >}}

Before we can transfer data using the MPSSE, we have to do some initial configuration.
But before we can do that, we need to define which pins we want to use:
```c++
// ADBUS0, 1, 2
#define PIN_SCK 0 
#define PIN_MOSI 1
#define PIN_NSS 2

#define OUTPUT_PINMASK ((1 << PIN_SCK) | (1 << PIN_MOSI) | (1 << PIN_NSS))
```
As described in [FTDI MPSSE Basics](https://www.ftdichip.com/Documents/AppNotes/AN_135_MPSSE_Basics.pdf), the pins for the `SCK`, `MOSI` and `MISO` signals are fixed.
The `NSS` signal however is handled manually and can be assigned to most pins.
Here, I actually assign it to the `MISO` pin, a very specific use-case that will be explained in a follow-up post.
For more common use-cases, you should use any other pin here.

As for all MPSSE commands, the command data is then written using `ftdi_write_data`:
```c++
void setupMPSSE()
{
    // Configure Clock: 1 MHz
    uint8_t buf[] = {TCK_DIVISOR, 0x05, 0x00,
        // Disable adaptive and 3 phase clocking
        DIS_ADAPTIVE,
        DIS_3_PHASE,
        // Set ADBUS: All SPI pins as output, NSS initial high
        SET_BITS_LOW, (1 << PIN_NSS), OUTPUT_PINMASK
    };

    // Write the setup to the chip.
    enforce(ftdi_write_data(&_ftdi, buf, sizeof(buf)) == sizeof(buf), "FTDI setup failed");
}
```

The code has been formatted so that each command is on a single line.

* `TCK_DIVISOR` configures the clock rate for the data transfers.
  The first argument byte is basically the low byte of a clock divider value, the second byte the high byte.
  The exact formula for the frequency is given in [AN2232C-01 Command Processor](https://www.ftdichip.com/Documents/AppNotes/AN2232C-01_MPSSE_Cmnd.pdf) as

  `TCK/SK period = 12MHz / (( 1 +[(0xValueH * 256) OR 0xValueL] ) * 2`.
* The `DIS_ADAPTIVE` and `DIS_3_PHASE` commands take no arguments and disable adaptive and 3 phase clocking features.
  Refer to the linked sources for details.
* The `SET_BITS_LOW` command is used to set the initial value of the `ADBUS` pins.
  The first argument specifies as a bitmask all pins that should be set high, in this case only `NSS`.
  The second bitmask specifies which pins are configured as outputs.
  Here, the `SCK`, `MOSI` and `NSS` pins will be configured as outputs.
  Ultimately, `NSS` will therefore be set high and `SCK` and `MOSI` will be set low.

{{< box info >}}
  The initial value of the `SCK` pin will define the initial clock state.
  For each data bit transferred, the MPSSE will then simply toggle the `SCK` line twice.
{{< /box >}}

## Single Bit SPI Transfers

The MPSSE can write data as blocks of bytes or as bits.
If we want to write an amount of bits not divisible by 8, we need to use the bit write command:

```c++
void writeTrigger(bool on)
{
    uint8_t tbit = on ? 0b1 : 0b0;
    uint8_t buf[] = {
        // Set NSS low
        SET_BITS_LOW, 0x00, OUTPUT_PINMASK,
        // Write one Bit
        MPSSE_DO_WRITE | MPSSE_WRITE_NEG | MPSSE_LSB | MPSSE_BITMODE, 0x00, tbit,
        // Add some artitifical delay before toggling NSS
        SET_BITS_LOW, 0, OUTPUT_PINMASK,
        // Set NSS high
        SET_BITS_LOW, (1 << PIN_NSS), OUTPUT_PINMASK
    };
    enforce(ftdi_write_data(&_ftdi, buf, sizeof(buf)) == sizeof(buf), "FTDI write failed");
}
```

We again send MPSSE commands using `ftdi_write_data`.
Here's how we implement an SPI transfer that writes a single bit of data:
* `SET_BITS_LOW`: First set the `NSS` pin (and other output pins) low, as explained before.
* `MPSSE_DO_WRITE` performs the actual data transfer and can be combined with flags to configure the transfer behavior.
  `MPSSE_WRITE_NEG` writes data on the falling clock edge,
  `MPSSE_LSB` writes data starting from the LSB to the MSB and `MPSSE_BITMODE` specifies that the length argument specifies the number of bits, not bytes.

  In `MPSSE_BITMODE`, the `MPSSE_DO_WRITE` command uses one argument as the number of bits (minus one) and the data is in the second byte argument.
  Specifying length `0` therefore transfers 1 bit of data, the LSB of the `tbit` variable.
* The next `SET_BITS_LOW` command simply sets all bits low.
  It is used to introduce an artificial delay before the `NSS` signal goes back to high state.
  Please note that unlike `MPSSE_DO_WRITE`, `SET_BITS_LOW` does not toggle `SCK`.
  Accordingly, there won't be any additional data transfers caused by this command.
* Finally, `SET_BITS_LOW` sets the `NSS` signal back to the high level, the idle state.

## Byte based SPI Transfers

Writing whole bytes works similarly and can be mixed with writing bits.
Here's an example:
```c++
void writeConfig(bool trigger, uint8_t adsr_ai, uint16_t osc_count, uint8_t filter_a)
{
    uint8_t tbit = trigger ? 0b1 : 0b0;

    uint8_t buf[] = {
        SET_BITS_LOW, 0x00, OUTPUT_PINMASK,
        MPSSE_DO_WRITE | MPSSE_WRITE_NEG | MPSSE_LSB | MPSSE_BITMODE, 0x00, tbit,
        MPSSE_DO_WRITE | MPSSE_WRITE_NEG | MPSSE_LSB, 0x01, 0x00, adsr_ai, (uint8_t)(osc_count & 0xff),
        MPSSE_DO_WRITE | MPSSE_WRITE_NEG | MPSSE_LSB | MPSSE_BITMODE, 0x03, (uint8_t)((osc_count >> 8) & 0x0f),
        MPSSE_DO_WRITE | MPSSE_WRITE_NEG | MPSSE_LSB, 0x00, 0x00, filter_a
    };
    enforce(ftdi_write_data(&_ftdi, buf, sizeof(buf)) == sizeof(buf), "FTDI write failed");

    usleep(3125);

    uint8_t buf2[] = {
        SET_BITS_LOW, (1 << PIN_NSS), OUTPUT_PINMASK
    };
    enforce(ftdi_write_data(&_ftdi, buf2, sizeof(buf2)) == sizeof(buf2), "FTDI write failed");
}
```

The first set of commands does the following:
* `SET_BITS_LOW` sets NSS low to start the transaction.
* `MPSSE_DO_WRITE` with `MPSSE_BITMODE` writes a single bit.
* `MPSSE_DO_WRITE` writes 2 bytes of data.
   When in byte mode, the length is encoded as two bytes, the first being the low byte.
   Again, the length value written to the command stream is reduced by one.
* `MPSSE_DO_WRITE` with `MPSSE_BITMODE` writes four bits.
  Combined with the previous command, this shows how you can conveniently write a 12 bit data field.
* The last command `MPSSE_DO_WRITE` writes another byte of data.

Note that at the end of the command, we do not release `NSS` yet.
It's perfectly fine to do this in another command, as shown here.
This will of course affect the timing, but sometimes this is intentional.

In this special case, I want the `NSS` signal to be low for at least 3.125 ms.
I therefore added a `usleep` call after writing the initial command stream.
The final `SET_BITS_LOW` to release `NSS` is then issued in another `ftdi_write_data` call, after the `usleep`.

## Bringing it all Together

With all functions prepared, we can not write the main function:

```c++
int main(int argc, char* argv[])
{   
    auto dev = new SPIDriver();
    dev->open();
    dev->writeTrigger(true);
    dev->writeConfig(true, 0, 0, 0);
    dev->close();
    return 0;
}
```

And that's it, this code should be all you need to send SPI data using a FT232H. If you need to read data as well, refer to the linked documents. The process is however essentially the same.