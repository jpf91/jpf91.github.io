---
title: OSS VHDL and Verilog Development on Tang Nano FPGAs
author:
  - Johannes Pfau
description: How to use the OSS toolchains for mixed VHDL and Verilog development for the Tang Nano FPGA
ShowToc: true
series: ["OSS RISC-V Development on Tang Nano"]
categories: Hardware Development
tags:
  - FPGA
  - OSS
  - OSS EDA
  - Tang Nano
  - VHDL
  - Verilog
draft: false
---

There's some [good documentation](https://learn.lushaylabs.com/getting-setup-with-the-tang-nano-9k/) on how to get started with OSS development for the [Tang Nano FPGA](https://wiki.sipeed.com/hardware/en/tang/index.html) series, but there's no complete tutorial for VHDL and manual compilation.
This post will explain how to set up tools, use Verilog or VHDL and how to mix them, how to compile everything manually and how to program the FPGA.
In addition, I'll show how to use the PLL and how to get a blinky demo running.

<!--more-->

## Basic OS Setup

At first, we'll have to install the tools.
As always, I'll start with a clean container system in [distrobox](https://distrobox.it).
Of course, these setup instructions should also work when you install the tools directly on your host OS, but using distrobox will make dependency management more convenient.
Alternatively, you could also use [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

So let's set up a new container based on Fedora 41, the latest release at the time this article was written:
```bash
distrobox create -i fedora:41 fpga
distrobox enter fpga
```

## Playing with the Preinstalled FPGA Firmware

Let's install picocom to talk to the serial port:
```bash
sudo dnf install picocom
```

In addition to using the USB port for programming, the Tang Nano boards emulate a USB serial device using the same connector.
This device can be used to talk to the user design and to set up various parameters for the board, such as the generated clocks.
For now, let's talk to the preprogrammed FPGA application:

```bash
picocom -b 115200 /dev/ttyUSB1
```

If you've not used picocom before: The command key is `CTRL` + `a`.
So to exit picocom for example, hold `CTRL` and then press `a` followed by `x`.

We can now interact with the preinstalled design:
```bash
# Tab to list commands
help
# All off
litex> leds 0xff
Settings Leds to 0xff
# All on
litex> leds 0   
Settings Leds to 0x0
# Button state
litex> buttons
Buttons value: 0x3
# Hold CTRL, then press a, then x to exit
```

### Entering the System Console

To enter the system console, we'll first open the serial port as if we were connecting to the application:

```bash
picocom -b 115200 /dev/ttyUSB1
```

To enter the system console, we have to enter some special commands:
Hold `CTRL`, then press `x` followed by `c` and press `Enter`:
```bash
Type [C-a] [C-h] to see available commands
Terminal ready
# Hold CTRL, then press x, then c and finally enter, to enter system console
: command not found.
TangNano20K />
```

To get the list of supported commands:
```bash
TangNano20K /> help
shell commands list:
pll_clk
pll
free
memtrace
help
reboot
choose
```

For example, to change clock 0 to 100 MHz:
```bash
# Change the clock
TangNano20K />pll_clk O0=100M
...
# Make configuration persist
TangNano20K />pll_clk -s
```

## Using APIO for Simple Applications

[Apio](https://apiodoc.readthedocs.io/en/stable/) is a simple way to get a quick FPGA toolchain setup for various development boards.
We'll use it here to get started, then switch to manual compilation for more control.
First, let's install pip in the container:
```bash
sudo dnf install python3-pip
```

Next, we'll install Apio.
We're going to get the development version as we need GOWIN FPGA support for our Tang Nano board.
```bash
pip install -U https://github.com/FPGAwars/apio/archive/refs/heads/develop.zip
```

We now follow the [Apio Quick Start](https://apiodoc.readthedocs.io/en/stable/source/quick_start.html) and install the dependencies:
```bash
apio packages install
```

Once the initial setup finished, we can have a look at available examples:
```bash
apio examples list
├──────────────────────────────────────┼───────┼────────────────────────────────────────────────┤
│ sipeed-tang-nano-4k/blinky           │ gowin │ Blinking led (untested)                        │
│ sipeed-tang-nano-9k/blinky           │ gowin │ Blinking led                                   │
│ sipeed-tang-nano-9k/blinky-sv        │ gowin │ Blinking led (system verilog)                  │
│ sipeed-tang-nano-9k/pll              │ gowin │ PLL clock multiplier                           │
└──────────────────────────────────────┴───────┴────────────────────────────────────────────────┘
```

So nothing for the Tang Nano 20k...
We can still create a new, custom project:
```bash
apio create -b sipeed-tang-nano-20k
```

### Verilog Example Application

Use the following in `main.v`:
```verilog
module main (
    input sys_clk,
    output[5:0] led
);

    localparam WAIT_CYCLES = 13500000;
    reg[25:0] counter;
    reg[5:0] led_buf = 6'b111110;

    always @(posedge sys_clk) begin
        counter <= counter + 1'b1;
        if (counter == WAIT_CYCLES) begin
            counter <= 0;
            led_buf <= {led_buf[4:0], led_buf[5]};
        end
    end
    assign led = led_buf;

endmodule
```

And this code for constraints in `main.cst`:
```ini
IO_LOC "sys_clk" 4;
IO_PORT "sys_clk" PULL_MODE=UP;

IO_LOC "led[0]" 15;
IO_LOC "led[1]" 16;
IO_LOC "led[2]" 17;
IO_LOC "led[3]" 18;
IO_LOC "led[4]" 19;
IO_LOC "led[5]" 20;
```

We can also set up clock constraints in `clk.py`:
```python
# The sys clock
ctx.addClock("sys_clk", 27)
```

With those files in place we can now build the example:
```bash
% apio build
Setting the environment.
Processing board sipeed-tang-nano-20k
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
yosys -p "synth_gowin -top main -json _build/hardware.json" -q main.v
nextpnr-himbaechel --device GW2AR-LV18QN88C8/I7 --json _build/hardware.json --write _build/hardware.pnr.json --report _build/hardware.pnr --vopt family=GW2A-18C --vopt cst=main.cst -q
gowin_pack -d GW2A-18C -o _build/hardware.fs _build/hardware.pnr.json
============================================================================ [SUCCESS] Took 12.05 seconds ============================================================================
```

And now we can upload the example:
```bash
apio upload
Setting the environment.
Processing board sipeed-tang-nano-20k
...
```

We can also get a resource and timing summary:
```bash
% apio report
Setting the environment.
Processing board sipeed-tang-nano-20k
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Formatting pnr report.

FPGA Resource Utilization
┌───────────────────┬────────┬───────────┬──────────┐
│  RESOURCE         │  USED  │    TOTAL  │   UTIL.  │
├───────────────────┼────────┼───────────┼──────────┤
│  ALU              │    28  │    15552  │      0%  │
│  ALU54D           │        │       24  │          │
│  BSRAM            │        │       46  │          │
│  BUFG             │        │       24  │          │
│  CLKDIV           │        │        8  │          │
│  CLKDIV2          │        │       16  │          │
│  DCS              │        │        8  │          │
│  DFF              │    32  │    15552  │      0%  │
│  DHCEN            │        │       24  │          │
│  DQCE             │        │       24  │          │
│  GND              │     1  │        1  │    100%  │
│  GSR              │     1  │        1  │    100%  │
│  IOB              │     7  │      384  │      1%  │
│  IOLOGICI         │        │      384  │          │
│  IOLOGICO         │        │      384  │          │
│  LUT4             │    28  │    20736  │      0%  │
│  MULT18X18        │        │       48  │          │
│  MULT36X36        │        │       12  │          │
│  MULT9X9          │        │       96  │          │
│  MULTADDALU18X18  │        │       24  │          │
│  MULTALU18X18     │        │       24  │          │
│  MULTALU36X18     │        │       24  │          │
│  MUX2_LUT5        │     5  │    10368  │      0%  │
│  MUX2_LUT6        │     2  │     5184  │      0%  │
│  MUX2_LUT7        │        │     2592  │          │
│  MUX2_LUT8        │        │     2592  │          │
│  OSC              │        │        1  │          │
│  PADD18           │        │       48  │          │
│  PADD9            │        │       96  │          │
│  RAM16SDP4        │        │      648  │          │
│  VCC              │     1  │        1  │    100%  │
│  rPLL             │        │        2  │          │
└───────────────────┴────────┴───────────┴──────────┘

Clock Information
┌────────────────┬───────────────────┐
│  CLOCK         │  MAX SPEED [Mhz]  │
├────────────────┼───────────────────┤
│  clk_IBUF_I_O  │           288.35  │
└────────────────┴───────────────────┘

Run 'apio report --verbose' for more details.
============================================================================ [SUCCESS] Took 0.20 seconds ============================================================================
```

### Apio Limitations

Apio is a great project, especially if you want to quickly use different boards.
You can also use it to see what commands it uses behind the scenes and replicate those manually.
Often the tools used behind the scenes are complex and this is valuable information.

There are however certain limitations with Apio, especially in more advanced projects.
These were the issues I ran into:
The documentation is often missing or lacking.
For example, I couldn't find any in-depth information about the `apio.ini` format.
For more advanced use cases, you can apparently use a `Sconstruct` file, but I don't know where to find documentation.
I also don't know if there is a way to put files in folders? What's the source tree structure?

Instead of trying to find that information in source code and by searching online, I decided it'd be more instructive to repeat the process manually and learn how the individual tools are used.

## Doing it All Manually

If we don't use Apio, we first have to install the OSS tools.
One major benefit here is that we can get the latest version, which may be quite useful.
For example, when doing my tests, I quickly ran [into an issue with PLLs](https://github.com/YosysHQ/nextpnr/issues/1408) and reported a bug.
It was fixed quickly, but of course you need to use the latest tool version to get those fixes.

### Installing the Tools

The most common way to get the OSS FPGA tools is using the [OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build).
This project provides nightly binary builds of all important OSS tools needed.
We can simply install the binary tarball:
```bash
# Adjust the version here
OSSCAD_VERSION="2025-02-26"
OSSCAD_VERSION_SHORT=$(echo "${OSSCAD_VERSION}" | sed 's/-//g')
curl --progress-bar -L "https://github.com/YosysHQ/oss-cad-suite-build/releases/download/${OSSCAD_VERSION}/oss-cad-suite-linux-x64-${OSSCAD_VERSION_SHORT}.tgz" -o osscad.tgz
tar -xf osscad.tgz
rm osscad.tgz
echo "${OSSCAD_VERSION}" > oss-cad-suite/VERSION
sudo mv oss-cad-suite /opt/
```

Let's also set up a profile snippet.
With this, whenever we execute `distrobox enter`, the tools will be loaded automatically:
```bash
sudo sh -c 'echo "source /opt/oss-cad-suite/environment" > /etc/profile.d/z_oss_cad.sh'
# Make sure to use bash in the container, other shells won't work
chsh -s chsh -s /bin/bash
```

Then exit and enter again to get your new shell prompt:
```bash
exit
distrobox enter fpga
⦗OSS CAD Suite⦘ 📦[jpfau@fpga ~]$
# Now test some commands:
yosys -V

Yosys 0.50+49 (git sha1 05c81b3f1, clang++ 18.1.8 -fPIC -O3)
```

And that's all that's required for the tool installation.

### Setting up the Folder Structure

We're going to use the same example as in the Apio section.
Just create the `main.v`, `main.pcf` and `clk.py` files as explained [there](#verilog-example-application), but this time, place them in a proper folder structure:
```bash
tree
.
├── Makefile
├── script
│   └── summary.py
└── src
    ├── constraints
    │   ├── top.cst
    │   └── top.py
    └── hdl
        └── main.v
```

Let's go through the programming step-by-step.

### Synthesizing Verilog

Synthesis is performed in [Yosys](https://github.com/YosysHQ/yosys).
We use this command:
```bash
yosys -p "synth_gowin -top main -json build/top.syn.json" -q src/hdl/main.v  -l build/rpt/top.syn.log
```

Where `-q` tells Yosys not to print to the standard output and `-l` will write all output to a file.
The part after `-p` is the synthesis script and determines what actions Yosys will perform.
In this case, we synthesize for the GOWIN platform (`synth_gowin`) with a top module named `main` and save the final netlist to `build/top.syn.json` in JSON format.

{{< box info >}}
  The synthesis script will be different if you're synthesizing for ASIC or for another FPGA vendor.
{{< /box >}}

### Using VHDL

If we want to use VHDL, the command for synthesis is slightly more complex.
Put the following in `src/hdl/main.vhd`:
```vhdl
library ieee;
use ieee.numeric_std.all;
use ieee.std_logic_1164.all;

entity main is
    port (
        sys_clk: in std_logic;
        led: out std_logic_vector(5 downto 0)
    );
end;

architecture impl of main is
    signal counter: unsigned(25 downto 0);
    signal led_buf: std_logic_vector(5 downto 0) := "111110";
    constant WAIT_CYCLES: integer := 13500000;
begin

    shift: process(sys_clk)
    begin
        if rising_edge(sys_clk) then
            counter <= counter + 1;
            if to_integer(counter) = WAIT_CYCLES then
                counter <= (others => '0');
                led_buf <= led_buf(4 downto 0) & led_buf(5);
            end if;
        end if;
    end process;
    led <= led_buf;
end;
```

The synthesis command now looks like this:
```bash
yosys -m ghdl -p "ghdl src/hdl/main.vhd -e main; synth_gowin -top main -json build/top.syn.json" -q -l build/rpt/top.syn.log
```

Here `-m ghdl` loads the [GHDL](https://github.com/ghdl/ghdl) [plugin](https://ghdl.github.io/ghdl/using/Synthesis.html) that is responsible for parsing the VHDL code.
The `-p` script now loads VHDL files using the `ghdl` command and `-e` tells GHDL the name of the top module.
The rest of the command remains the same.

### Mixing VHDL and Verilog

We can also mix Verilog and VHDL in the OSS toolchain (with all the limitations you'd expect if you've ever done this in commercial tools before).

Add this to `src/hdl/main.vhd`, as our top module:
```vhdl
library ieee;
use ieee.numeric_std.all;
use ieee.std_logic_1164.all;

entity main is
    port (
        sys_clk: in std_logic;
        led: out std_logic_vector(5 downto 0)
    );
end;

architecture impl of main is
    signal counter: unsigned(25 downto 0);
    signal led_buf: std_logic_vector(5 downto 0) := "111110";
    constant WAIT_CYCLES: integer := 13500000;

    component test is
        port (
            a: in std_logic_vector(5 downto 0);
            y: out std_logic_vector(5 downto 0)
        );
    end component;
begin

    shift: process(sys_clk)
    begin
        if rising_edge(sys_clk) then
            counter <= counter + 1;
            if to_integer(counter) = WAIT_CYCLES then
                counter <= (others => '0');
                led_buf <= led_buf(4 downto 0) & led_buf(5);
            end if;
        end if;
    end process;

    conn: test
        port map (
            a => led_buf,
            y => led
        );
end;
```

And this in `src/hdl/test.v`:
```verilog
module test ( 
    input[5:0] a,
	  output[5:0] y,
);
    assign y = a;
endmodule
```

The new synthesis command:
```bash
yosys -m ghdl -p "ghdl src/hdl/main.vhd -e main; read_verilog src/hdl/test.v; synth_gowin -top main -json build/top.syn.json" -q -l build/rpt/top.syn.log
```

### Implementation

So we now know how to synthesize the netlist.
This will actually be similar for all FPGA vendors and even for ASIC targets.
Implementation on the other hand is highly target specific, so the commands here are only valid for GOWIN FPGAs.
The good news is that the command is the same for VHDL, Verilog or whatever other language you're using for your sources.
```bash
nextpnr-himbaechel --device GW2AR-LV18QN88C8/I7 --json build/top.syn.json --write build/top.pnr.json \
  --report build/rpt/top.pnr.json --vopt family=GW2A-18C \
  --vopt cst=src/constraints/top.cst --pre-pack src/constraints/top.py -q -l build/rpt/top.pnr.log \
  --threads 4 --detailed-timing-report
```

So what does this do?

| Flag         | Description                                                                                  |
|--------------|----------------------------------------------------------------------------------------------|
| `--device`   | The target FPGA we use.                                                                      |
| `--json`     | The netlist input, in JSON format.                                                           |
| `--write`    | The output file, JSON format.                                                                |
| `--report`   | Report in JSON format (utilization etc).                                                     |
| `--vopt`     | GOWIN specific options. Target family and path to constraints file.                          |
| `--pre-pack` | Python script run before packing. Required to specify clock constraints for multiple clocks. |
| `-q`, `-l`   | Same as for Yosys.                                                                           |
| `--threads`  | Enable multi-threading.                                                                      |

### Generating the Bitstream

This last part is quite simple:
```bash
gowin_pack -d GW2A-18C -o build/top.fs build/top.pnr.json
```
Once again we specify the FPGA family, the output file `top.fs` and the input obtained from the implementation step.

### Uploading the Design

We now have two options to upload the design.
We can program directly to FPGA SRAM:
```bash
openFPGALoader -b tangnano20k build/top.fs
```

Alternatively, we can write to the SPI Flash:
```bash
openFPGALoader -b tangnano20k -f build/top.fs
```

## Using the PLL

With the basic examples running, let's see how to use some hard IP in the OSS tools, the `rPLL`.
Parameters for the `rPLL` can be obtained from [this website](https://juj.github.io/gowin_fpga_code_generators/pll_calculator.html).
For an overview of hard IP supported by the OSS Toolchain, refer to the [Nextpnr‐Himbaechel Wiki](https://github.com/YosysHQ/apicula/wiki/Nextpnr%E2%80%90Himbaechel-Gowin).

Put the following in `src/hdl/CLKGen.v`
```verilog
// Derive a 108 MHz clock from the 27 MHz system clock
module CLKGen ( 
        input sys_clk,
        input enable,
        output clk,
        output locked
    );

    // https://juj.github.io/gowin_fpga_code_generators/pll_calculator.html
    rPLL #(
        .DEVICE("GW2AR-18"),
        .FCLKIN("27"),
        .IDIV_SEL(0), // -> PFD = 27 MHz (range: 3-400 MHz)
        .FBDIV_SEL(3), // -> CLKOUT = 108 MHz (range: 3.125-600 MHz)
        .ODIV_SEL(8), // -> VCO = 864 MHz (range: 400-1200 MHz)
        .DYN_ODIV_SEL("false"),
        .DYN_FBDIV_SEL("false"),
        .DYN_IDIV_SEL("false"),
        .PSDA_SEL("0000"),
        .DYN_DA_EN("false"),
        .DUTYDA_SEL("1000"),
        .CLKOUT_FT_DIR(1'b1),
        .CLKOUTP_FT_DIR(1'b1),
        .CLKOUT_DLY_STEP(0),
        .CLKOUTP_DLY_STEP(0),
        .CLKFB_SEL("internal"),
        .CLKOUT_BYPASS("false"),
        .CLKOUTP_BYPASS("false"),
        .CLKOUTD_BYPASS("false"),
        .DYN_SDIV_SEL(2),
        .CLKOUTD_SRC("CLKOUT"),
        .CLKOUTD3_SRC("CLKOUT")
    ) pll (
        .CLKOUT(clk),
        .LOCK(locked),
        .CLKOUTP(),
        .CLKOUTD(),
        .CLKOUTD3(),
        .RESET(1'b0),
        .RESET_P(1'b0),
        .CLKIN(sys_clk),
        .CLKFB(1'b0),
        .FBDSEL(6'b00000),
        .IDSEL(6'b00000),
        .ODSEL(6'b0),
        .PSDA(4'b0),
        .DUTYDA(4'b0),
        .FDLY(4'b0)
    );

endmodule
```

Change the top-level Verilog in `main.v` to actually use the PLL:
```verilog
module main (
    input sys_clk,
    output[5:0] led
);
    wire clk;
    wire clk_locked;

    CLKGen cgen (
        .sys_clk(sys_clk),
        .enable(1'b1),
        .locked(clk_locked),
        .clk(clk)
    );

    localparam WAIT_CYCLES = 13500000;
    reg[25:0] counter;
    reg[4:0] led_buf = 5'b11110;

    always @(posedge clk) begin
        counter <= counter + 1'b1;
        if (counter == WAIT_CYCLES) begin
            counter <= 0;
            led_buf <= {led_buf[3:0], led_buf[4]};
        end
    end
    assign led[5:1] = led_buf;

    assign led[0] = !clk_locked;

endmodule
```

Finally, add constraints for the generated clock to `clk.py`:
```python
# The sys clock, as available on pin
ctx.addClock("sys_clk", 27)
# Derived main clock
ctx.addClock("cgen.clk", 108)
```

And that's it :-)