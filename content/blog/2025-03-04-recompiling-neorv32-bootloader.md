---
title: Recompilling the NEORV32 Bootloader
author:
  - Johannes Pfau
description: If you want to change the bootloader configuration or code, here's how to do that.
ShowToc: true
series: ["OSS RISC-V Development on Tang Nano"]
categories: Hardware Development
tags:
  - FPGA
  - OSS
  - Tang Nano
draft: false
---


The NEORV32 default bootloader has various configuration options.
Here's how to modify and recompile it.
<!--more-->

If you want to modify the bootloader, edit the [bootloader.c](https://github.com/stnolting/neorv32/blob/ff24baf41d3bf6dcfed65579408fc5e8767c6d04/sw/bootloader/bootloader.c) file first. 
Then recompile the bootloader into a `.vhd` file representing the ROM:
```bash
cd lib/neorv32/sw/bootloader
make clean_all
make make bl_image

Memory utilization:
   text    data     bss     dec     hex filename
   4092       0       8    4100    1004 main.elf
Compiling image generator...
Generating neorv32_bootloader_image.vhd
```

Finally, install this file somewhere along our modified NEORV32 sources:
```bash
mv neorv32_bootloader_image.vhd ../../../../src/hdl/neorv32/neorv32_bootloader_image.vhd
```

In our top-level [Makefile](https://github.com/jpf91/neorv32-tang20k/blob/master/Makefile), make sure you use this file instead of the original `neorv32_bootloader_image.vhd`.
Then simply re-synthesize and re-flash the FPGA firmware.

If you're making modifications and just want to test the bootloader, you can use `make exe` instead of `make bl_image` and run the bootloader like a normal application.