---
title: Dual-Purpose Pins with Gowin FPGAs and OSS Tools 
author:
  - Johannes Pfau
description: How to configure pin reuse for dual-purpose pins in the OSS toolchain
ShowToc: true
series: ["OSS RISC-V Development on Tang Nano"]
categories: Hardware Development
tags:
  - FPGA
  - OSS
  - Tang Nano
draft: false
date: 2025-03-09T00:00:00
---

The Gowin series of FPGA support *configuration pin reuse* on some special *dual-purpose pins*.
Those pins are used during programming of the FPGA, e.g. for JTAG or as `READY` and `DONE` signals.
When enabling *pin reuse*, they can be used as normal GPIO pins by the FPGA bitstream after configuration.
The OSS toolchain supports setting these flags and here's how.
<!--more-->

The pin reuse feature is documented in Gowin [UG290E](https://cdn.gowinsemi.com.cn/UG290E.pdf), section *4.1.2 Configuration Pin Reuse*.
Whereas the documentation explains how to configure this feature using the vendor toolchain, there's little documentation on how to do it with the OSS tools.
The fact that the vendor GUI places the pin reuse configuration in the *Place & Route* section provides some useful hint though:
For Gowin FPGAs, this configuration is not part of some dedicated, static configuration on the FPGA chip.
It can therefore not be programmed independently.
These flags are part of the normal bitstream instead and therefore need to be inserted in the bitstream packing phase.

It turns out that `gowin_pack` indeed does [support](https://github.com/YosysHQ/apicula/pull/58/files) encoding these configuration bits.
Simply use the following flags as needed when calling `gowin_pack` to pack your final bitstream:
```bash
--jtag_as_gpio
--sspi_as_gpio
--mspi_as_gpio
--ready_as_gpio
--done_as_gpio
--reconfign_as_gpio
```

{{< box info >}}
If you reuse the JTAG pins this way, `openFPGALoader` might be unable to program your firmware to FPGA.
You can avoid this by only ever uploading your bitstream to SRAM and not to flash.
After resetting the FPGA, it will then load the firmware from flash, which should not have the reuse flags set.
If the firmware stored in flash is also configured to reuse the JTAG, you can prevent loading this firmware at power-on:
If you keep the `S2` button pressed when powering up the board (i.e. when connecting the USB), it will not load any FPGA firmware from flash.
{{< /box >}}