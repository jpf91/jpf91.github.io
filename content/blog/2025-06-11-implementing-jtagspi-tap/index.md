---
title: Implementing JTAGSPI for the NEORV32 JTAG TAP
author:
  - Johannes Pfau
description: How to tunnel SPI over JTAG to make OpenOCD read your SPI flash.
tags:
  - FPGA
  - NEORV32
  - JTAG
ShowToc: true
categories: Hardware Development
draft: true
---

FPGAs and microcontrollers often need to store configuration on SPI flash.
Whereas we could always use external programmers or special code running on these devices, various products have come up with a more clever solution: JTAGSPI.
JTAGSPI tunnels SPI over JTAG, so that you can program your flash using just your JTAG programmer.
Let's see how we can implement an OpenOCD compatible solution in our JTAG TAPs.

<!--more-->

## Background Story

We're currently planning to tape-out the RISC-V SoC developed in the PSoC lab course at my university.
When preparing the design for that, it was immediately obvious that memory(SRAM) is going to be the largest part of the chip.
NeoRV32, the SoC we use, by default loads applications completly into instruction ram memory.
As we want to be able to run fairly large FreeRTOS based programs, this would be quite wasteful.

A solution to this is the execute-in-place (XIP) peripheral included with the NeoRV32 distribution.
The XIP peripheral maps SPI flash as (read-only) memory, so programs can execute directly from flash.
You still want to configure an instruction cache, as otherwise the whole system will become incredibly slow.
But on the upside, you can get rid of the instruction SRAM and the instruction cache can be way smaller.

To boot from XIP, you need a bootloader.
The default NeoRV32 bootloader is quite large, as it uses a text based UART interface.
As we need to include the bootloader in chip ROM, we'd also like to reduce the bootloader and therefore ROM size.
The specialized [neorv32-xip-bootloader](https://github.com/betocool-prog/neorv32-xip-bootloader) XIP bootloader is already smaller, but still large.
The bootloader size is still mostly determined by UART interaction:
I've written a [minimal bootloader](https://github.com/kit-kch/psoc-xip-bootloader/blob/main/bootloader_tiny/bootloader_xip.c) that just boots straight from XIP and compiles to only [5 instructions](https://github.com/kit-kch/psoc-xip-bootloader/blob/main/bootloader_tiny/neorv32_bootloader_image.vhd). 
However, this bootloader can no longer program the flash storage via UART.
We could use external programmers for the SPI flash, but there's a better solution:
We already use the JTAG TAP in NeoRV32 for debugging and when using the SRAM based instruction memory, it can also be used by GDB to upload executables.
The only thing we need now is a way for gdb to program our flash using JTAG...

Luckily this is a problem many people had before, so there is a common solution: JTAGSPI.

## Tunneling SPI Over JTAG

The solution used by many flash-based microcontrollers and by many FPGAs is JTAGSPI.
The OpenOCD documentation explains the basic idea:

> To access this flash from the host, some FPGA device provides dedicated JTAG instructions, while other FPGA devices should be programmed with a special proxy bitstream that exposes the SPI flash on the device’s JTAG interface. The flash can then be accessed through JTAG.
> 
> Since signalling between JTAG and SPI is compatible, all that is required for a proxy bitstream is to connect TDI-MOSI, TDO-MISO, TCK-CLK and activate the flash chip select when the JTAG state machine is in SHIFT-DR.
>
> — [OpenOCD Documentation](https://openocd.org/doc/html/Flash-Commands.html")

### JTAG Basics

In order to understand this very brief description, it is necessary to understand some JTAG concepts first.
JTAG essentially forms a register scan chain, which can be used to shift data using `TCK` clock, the `TDI` input and `TDO` output signals.
In addition to those, JTAG however has a control signal called `TMS`, which drives a standardized state machine.

The FSM is described by the figure below, taken from [xjtag.com](https://www.xjtag.com/about-jtag/jtag-a-technical-overview/).
![JTAG TAP FSM](tap_state_machine.gif)

Whereas this explains the low-level working of JTAG, the high-level aspects are often assumed and not explained.
Here are the main points:
* `IR` is an instruction register.
  Shift in different instructions to achieve different effects.
  Usually `IR` values are treated like addresses, selecting what data the `DR` register accesses.
* JTAG defines some standard `IR` values:
  One for reading out the default scan chain and one for bypass, which connects `TDI` to `TDO` using 1 flip-flop to reduce chain length.
* Both are not really useful to us, but we can add our own instructions.
* Neither the length of the `IR` nor the `DR` registers are defined.
  Tools like OpenOCD however expect a fixed `IR` length, that needs to be specified in the tool configuration.
* The usual high-level programming works like this:
  1. Optional: Reset
  2. Shift in `IR`
  3. Access `DR`

### JTAGSPI Instructions

What the OpenOCD documentation was now telling us is that a JTAG tap can simply connect `TDI` with SPI `MOSI` and `TDO` with SPI `MISO` in the `shift DR` state.
When doing this, the data send and received by the JTAG link are directly forwarded to the SPI device.
OpenOCD has a generic flash access support using the `jtagspi` driver.
However, as the documentation notes, how you actually enter this specific *SPI Bypass* is different depending on the JTAG Tap vendor.
In general there are two options:
Implementing a special `IR` instruction, which will activate this mode, or using some other out-of-band method.
The out-of-band method is commonly used for FPGAs, but it makes little sense for the NeoRV32 microcontroller.
We're therefore going to implement a custom instruction in the JTAG TAP to activate JTAGSPI.

## Modifying the NEORV32 JTAG TAP

https://github.com/kit-kch/psoc-neorv32/commit/7d30065f3b0ebd355daca2ba11663422c4d95779
```verilog
module fpga_spi(
    output led_r,
    output led_g,
    output led_b,
    // SPI
    input spi_clk,
    input spi_mosi,
    input spi_nss
);

    // SPI Slave
    reg[7:0] spi_data;
    always @(posedge spi_clk) begin
        if (spi_nss == 1'b0) begin
            spi_data <= {spi_mosi, spi_data[7:1]};
        end
    end

    // Assign outputs. Note: In practice you probably want
    // to latch this data into your main clock domain
    assign led_r = ~spi_data[0];
    assign led_g = ~spi_data[1];
    assign led_b = ~spi_data[2];

endmodule
```

### Getting the Timing Right

![JTAGSPI With One Cycle Read Delay](jtagspi_1_cycle.png)

![JTAGSPI With Zero Cycles Read Delay](jtagspi_0_cycle.png)

## Adding Support in OpenOCD
https://github.com/kit-kch/psoc-openocd/commit/bd577aad8a9d6b2afd2c503b4e5400600c96edb7

## Testing with GDB

```
./src/openocd -c 'adapter serial 210249B1B925' -f ~/Downloads/openocd_neorv32_jtaghs2.cfg
```

```
# ----------------------------------------------
# Flash programming
# ----------------------------------------------
pld create neorv32.pld neorv32 -chain-position neorv32.cpu
flash bank spi_flash jtagspi 0x20000000 0 0 0 neorv32.cpu.0 -pld neorv32.pld
```

```
riscv-none-elf-gdb
target extended-remote localhost:3333
```

```
Open On-Chip Debugger 0.12.0+dev-02012-g4fe57a0c1-dirty (2025-06-10-09:38)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
CMD_ARGC: 4 
Info : clock speed 1000 kHz
Info : JTAG tap: neorv32.cpu tap/device found: 0x0cafe001 (mfg: 0x000 (<invalid>), part: 0xcafe, ver: 0x0)
Info : datacount=1 progbufsize=2
Info : Disabling abstract command reads from CSRs.
Info : Examined RISC-V core; found 1 harts
Info :  hart 0: XLEN=32, misa=0x40901106
Info : [neorv32.cpu.0] Examination succeed
Info : [neorv32.cpu.0] starting gdb server on 3333
Info : Listening on port 3333 for gdb connections
Target HALTED.
Ready for remote connections.
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : accepting 'gdb' connection on tcp/3333
Info : Found flash device 'win w25q64jv' (ID 0x1770ef)
```

```
(gdb) file main.elf
Reading symbols from main.elf...
(gdb) load
Loading section .text, size 0x1038 lma 0x20000000
Loading section .rodata, size 0x880 lma 0x20001038
Start address 0x20000000, load size 6328
Transfer rate: 17 KB/sec, 3164 bytes/write.
(gdb) break main
Breakpoint 1 at 0x200001f8
Note: automatically using hardware breakpoints for read-only addresses.
```