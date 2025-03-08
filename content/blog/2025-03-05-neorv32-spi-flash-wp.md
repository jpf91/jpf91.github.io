---
title: SPI Flash Write Protect Signals and NEORV32
author:
  - Johannes Pfau
description: How to drive the SPI flash write protect signal with NEORV32
ShowToc: true
series: ["OSS RISC-V Development on Tang Nano"]
categories: Hardware Development
tags:
  - FPGA
  - OSS
  - Tang Nano
  - NEORV32
draft: false
---


NEORV32 does not have any dedicated output pin for the SPI flash write protect signal.
On boards like the Tang Nano 20K, not driving this pin however means we can't write to SPI flash.
This breaks the upload firmware function in the bootloader, so here's how to fix this.
<!--more-->

In my [initial NEORV32 port]({{< relref "./2025-02-27-tang20k-neorv32.md" >}}) article, I solved this by compiling a custom bootloader.
An alternative seemed to be permanently driving this pin high (which means disable write protect permanently, as it's a low active signal).
However, for some reason this breaks programming using the USB programmer and corrupts the flashed firmware.

As discussed in NEORV32 [issue 1195](https://github.com/stnolting/neorv32/issues/1195), there is now a better solution with very recent NEORV32 revisions:
The `rstn_wdt_o` output of the `neorv32_top` module is high when the NEORV32 is executing software, but it is `Z`, as long as the core is in reset.

If we connect this signal to the SPI flash write protect pin like this:
```vhdl
-- Write protect (low active) for flash chip
mspi_wp <= '1' when rstn_wdt_o = '1' else '0';
```

The `mspi_wp` pin will be high when the core is executing and low when it is in reset.
This means when the core is running, it will drive the low-active `mspi_wp` pin high and writing to SPI flash is possible.

This is a cleaner solution as it does not require a patched bootloader, and it does not need to have a GPIO pin connected for this signal.
The solution is implemented in [the latest revision](https://github.com/jpf91/neorv32-tang20k/commit/a40a49d96e91049487f85428be74f1cbc22bc8c2) in my NEORV32 repo.


When updating the NEORV32 you might notice that the RTL files it uses changed. file_list_soc.f