---
title: Debugging NEORV32 Applications on the Tang Nano 
author:
  - Johannes Pfau
description: How to debug applications on a NEORV32 setup using GDB, without external JTAG probes
ShowToc: true
series: ["OSS RISC-V Development on Tang Nano"]
categories: Hardware Development
tags:
  - FPGA
  - OSS
  - Tang Nano
  - NEORV32
  - JTAG
draft: false
date: 2025-03-09T01:00:00
---
The Tang Nano 20K board programs the Gowin FPGA using the JTAG protocol.
Reusing that JTAG connection for our applications means we don't need an external JTAG probe for software debugging using NEORV32's On-Chip-Debugger.
<!--more-->

The Tang Nano boards use a dedicated `BL616` microcontroller to handle the USB connection and emulates an FTDI `FT2232C/D/H` IC.
Tools such as `openFPGALoader` can then simply use the programmer like a real FTDI device:
```bash
openFPGALoader --detect

empty
No cable or board specified: using direct ft2232 interface
Jtag frequency : requested 6.00MHz    -> real 6.00MHz   
index 0:
	idcode 0x1
	manufacturer efinix
	family Trion
	model  T4/T8
	irlength 4
```

{{< box info >}}
Although `FT2232C/D/H` chips do not support JTAG directly, they can be programmed to handle arbitrary protocols in MPSSE mode.
Tools such as `openFPGALoader` and *OpenOCD*, the debug server we're going to use, manually implement JTAG on top of this.
For more information about MPSSE, see my article [Using MPSSE Mode with libftdi]({{< relref "./2025-02-15-libftdi-spi-mpsse.md" >}}) as an example implementing SPI this way.
{{< /box >}}

## FPGA Firmware Preparation

To use the JTAG with our application, we have to ensure a few things in our FPGA firmware.
First, we need to enable the On-Chip-Debugger and connect its JTAG ports:
```vhdl
neorv: neorv32_top
    generic map (
        ...
        -- Enable JTAG / Debugger --
        OCD_EN            => true,
        ...
    port map (
        ...
        -- JTAG for Debugging --
        jtag_tck_i  => sys_tck,
        jtag_tdi_i  => sys_tdi,
        jtag_tdo_o  => sys_tdo,
        jtag_tms_i  => sys_tms,
        ...
    );
```

Then we have to check the constraints to ensure that top-level JTAG pins are properly connected to the JTAG pins of the `BL616`:
```ini
IO_LOC "sys_tms" 5;
IO_LOC "sys_tck" 6;
IO_LOC "sys_tdi" 7;
IO_LOC "sys_tdo" 8;
```

Finally, we have to make sure the FPGA firmware can use those FPGA pins:
They are used for FPGA programming and therefore usually not accessible by the FPGA firmware.
As explained in a [previous post]({{< relref "./2025-03-09-tang20k-dual-purpose-pins.md" >}}), Gowin FPGAs however support *pin reuse*.
When enabled, this feature allows the FPGA firmware to use these pins just like any other pins.

As explained in that previous article, we will need to add `--jtag_as_gpio` to the bitstream packing flags.
Let's edit the `Makefile` accordingly:
```Makefile
PACK_FLAGS = --jtag_as_gpio
...

# Bitstream generation / Packing
$(OBJDIR)/%.fs: $(OBJDIR)/%.pnr.json | $(OBJDIR)
	gowin_pack -d $(GOWIN_FAMILY) $(PACK_FLAGS) -o $@ $<
```

And that's all.
Rebuild the FPGA bitstream using `make bitstream`, upload it using `make upload`, and you're ready to go.

{{< box info >}}
The NEORV32 setup with all required adjustments integrated is available in the latest commits of my [neorv32-tang20k](https://github.com/jpf91/neorv32-tang20k/tree/065e092f700efafb05ee7f7f0ba1d090213474dd) git repository.
{{< /box >}}

## Debugging Hello World

With this modified bitstream, debugging software applications now works as usual in the NEORV32 setup.
The default *OpenOCD* configuration in [`sw/openocd`](https://github.com/stnolting/neorv32/tree/fa0ea6904ffef7a7fa110102c1f587bf00a0955a/sw/openocd) even uses a compatible JTAG adapter type, so we don't have to make any changes.

{{< box info >}}
If you use [my distrobox setup]({{< relref "./2025-02-26-tang20k-oss-development.md" >}}) for development, you need to adjust the *OpenOCD* configuration slightly, as the `openocd` version shipped with the *OSS CAD Suite* does not support the `gdb` command. 
Edit [`sw/openocd/target.cfg`](https://github.com/stnolting/neorv32/blob/fa0ea6904ffef7a7fa110102c1f587bf00a0955a/sw/openocd/target.cfg) and comment out these lines:
```ini
  # GDB server configuration
#  gdb report_data_abort enable
#  gdb report_register_access_error enable
```
{{< /box >}}

### Starting OpenOCD

Simply change the directory to `sw/openocd` and run the `openocd` server using the `openocd_neorv32.cfg` config:
```bash
cd lib/neorv32/sw/openocd
openocd -f openocd_neorv32.cfg

Open On-Chip Debugger 0.12.0+dev-00519-gb6ee13720-dirty (2024-11-17-07:21)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
*****************************************
NEORV32 single-core openOCD configuration
*****************************************
Info : clock speed 4000 kHz
Info : JTAG tap: neorv32.cpu tap/device found: 0x00000001 (mfg: 0x000 (<invalid>), part: 0x0000, ver: 0x0)
Info : datacount=1 progbufsize=2
Info : Disabling abstract command reads from CSRs.
Info : Examined RISC-V core; found 1 harts
Info :  hart 0: XLEN=32, misa=0x40801100
Info : [neorv32.cpu] Examination succeed
Info : starting gdb server for neorv32.cpu on 3333
Info : Listening on port 3333 for gdb connections
Authentication passed.
Target HALTED. Ready for remote connections.
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
```

### Debugging with GDB

We can now debug using GDB.
As an example, we're going to step through the [`sw/example/hello_world`](https://github.com/stnolting/neorv32/tree/fa0ea6904ffef7a7fa110102c1f587bf00a0955a/sw/example/hello_world) application.

If we use the debugger, we don't need the `neorv32_exe.bin` that we used for the bootloader.
So instead of `make exe`, use `make elf` to compile the `main.elf` file:
```bash
make clean_all
make elf
```

For debugging, you also want to ensure that your application is compiled with debug data.
If you check the application [`makefile`](https://github.com/stnolting/neorv32/blob/fa0ea6904ffef7a7fa110102c1f587bf00a0955a/sw/example/hello_world/makefile), you'll see that it already has been configured for debugging:
```Makefile
# Add extended debug symbols
USER_FLAGS += -ggdb -gdwarf-3
```

Then open GDB like this:
```bash
riscv-none-elf-gdb
```

In the GDB console, we first need to connect to *OpenOCD*:
```bash
(gdb) target extended-remote localhost:3333
Remote debugging using localhost:3333
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0xffe00414 in ?? ()
```

We can then load our `main.elf` application into GDB.
```bash
(gdb) file main.elf
A program is being debugged already.
Are you sure you want to change the file? (y or n) y
Reading symbols from main.elf...
```

As this did not program the RISC-V chip yet, let's do that now:
```bash
(gdb) load
Loading section .text, size 0xd60 lma 0x0
Loading section .rodata, size 0x8e4 lma 0xd60
Start address 0x00000000, load size 5700
Transfer rate: 60 KB/sec, 2850 bytes/write.
```

Before we start executing the application, let's set a breakpoint in the `main` function:
```bash
(gdb) break main
Breakpoint 1 at 0x1f0: file main.c, line 40.
```

Finally, we can run the application:
```bash
(gdb) continue
Continuing.

Breakpoint 1, main () at main.c:40
40	  neorv32_rte_setup();

```

Then just use GDB as usual. For example, to step through statements:
```bash
(gdb) next
[neorv32.cpu] Found 1 triggers
43	  neorv32_uart0_setup(BAUD_RATE, 0);
```

### Shortcut Command Version

Instead of compiling using `make elf`, starting GDB, connecting and finally loading the file, you can use the shortcut `make gdb`.
Then simply follow the previous steps starting at the `load` command:

```bash
make gdb

(gdb) load
Loading section .text, size 0xd60 lma 0x0
Loading section .rodata, size 0x8e4 lma 0xd60
Start address 0x00000000, load size 5700
Transfer rate: 76 KB/sec, 2850 bytes/write.
(gdb) break main
Breakpoint 1 at 0x1f0: file main.c, line 40.
(gdb) continue
Continuing.

Breakpoint 1, main () at main.c:40
40	  neorv32_rte_setup();
```