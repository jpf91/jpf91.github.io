---
title: Quickly Estimating the Size of Digital Designs
author:
  - Johannes Pfau
description: Estimate how many gates or LUTs your design will use
ShowToc: true
series: ["OSS ASIC Design"]
categories: Hardware Development
tags:
  - OSS
  - OSS EDA
  - ASIC
  - ORFS
  - FPGA
  - Distrobox
draft: false
---
Experienced developers have some intuition how a hardware description maps to circuits, enabling them to optimize code for circuit area or speed instinctively.
But like most of the time, checking is better than (educated) guessing.
<!--more-->

When contributing to the D [standard library](https://dlang.org/phobos/), I learned that the [D community](https://dlang.org) has quite high standards on code quality.
This includes complete test coverage, extensive code and documentation reviews and runtime performance.
However, when it comes to performance, the guiding principle was to write idiomatic code first and avoid premature optimization.
If you write unidiomatic code as optimization, don't just assume that it will perform better, but prove it using benchmarks.
The idea behind that was the compilers are often much better at optimizing than we might first think.

Similar considerations apply for hardware development:
Larger gains are usually made through architectural choices anyway.
But whether it is architectural changes or micro-optimizations, we better have a way to measure the results.
Completely running the design through a RTL2GDS flow or implementing a bitstream is a possible solution, but it is often too slow for rapid application development approaches. 
For a simple, quick estimate, synthesis results with technology mapping are sufficient.
Those can be obtained in only a few seconds using `yosys`.

## Basic Setup

As always, I like to use a simple Makefile to combine all used commands.
Here's the basic template, which just sets up folders, source files etc.:
```Makefile
export TOP_MODULE = neosd
APP_SVERILOG = neosd_cmd_reg.sv \
	neosd_top.sv

######################################################################
# Generated variables
######################################################################
export OBJDIR=build
HDLDIR=src/hdl
export SYN_SVERILOG_PATHS=$(addprefix $(HDLDIR)/,$(APP_SVERILOG))
QUIET_FLAG=
ifeq ($(strip $(VERBOSE)),)
	QUIET_FLAG=-q
endif

######################################################################
# Rules
######################################################################
clean:
	rm -rf $(OBJDIR)

$(OBJDIR):
	mkdir -p $(OBJDIR)
```

This sets up the `TOP_MODULE` variable to contain the name of the module we want to synthesize.
Furthermore, `APP_SVERILOG` contains a list of sources, whereas the full paths to those will be stored in `SYN_SVERILOG_PATHS`.
Note that some variables are `export`ed.
Those are available to external tools, as we will need to access them in our synthesis scripts.

As for make rules, there are two basic ones:
The `clean` rule cleans up all generated files.
The `$(OBJDIR)` rule is used to create the build directory, as the `clean` rule deletes it completely.
This way we also don't have to check in empty build directories into git.

## Estimating FPGA Size

Synthesizing for FPGAs is quite simple and was mostly covered in [this previous post]({{< relref "./2025-02-26-tang20k-oss-development.md" >}}).
Here's the complete Makefile rule to synthesize for the Gowin FPGA targets:
```Makefile
$(OBJDIR)/gowin.syn.json: $(SYN_SVERILOG_PATHS) | $(OBJDIR)
	yosys -p "read_verilog -sv $(SYN_SVERILOG_PATHS); synth_gowin -top $(TOP_MODULE) -json $@; tee -o $(OBJDIR)/gowin.syn.stat stat" $(QUIET_FLAG) -l $(OBJDIR)/gowin.syn.log
```
One thing to note is that we use `-noflatten` here, to obtain hierarchical results.

While we're at it, let's also add a `synth` rule that synthesizes all targets and a `summary` rule to directly print the hardware resource usage:
```Makefile
synth: $(OBJDIR)/gowin.syn.json

summary:
	@echo
	@echo ========================== FPGA Summary ==========================
	@echo
	@cat $(OBJDIR)/gowin.syn.stat
```

Now we can just run `make synth` in any [distrobox with yosys]({{< relref "./2025-02-26-tang20k-oss-development.md" >}}) installed and get these results:
```
========================== FPGA Summary ==========================


4. Printing statistics.

=== neosd ===

   Number of wires:                739
   Number of wire bits:           1471
   Number of public wires:         739
   Number of public wire bits:    1471
   Number of ports:                 19
   Number of port bits:            122
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:               1163
     ALU                             8
     DFFC                            2
     DFFCE                          41
     DFFE                          209
     DFFR                           33
     GND                             1
     IBUF                           83
     LUT1                          209
     LUT2                           58
     LUT3                           46
     LUT4                          195
     MUX2_LUT5                     153
     MUX2_LUT6                      50
     MUX2_LUT7                      24
     MUX2_LUT8                      10
     OBUF                           39
     VCC                             1
     sreg                            1

=== sreg ===

   Number of wires:                 22
   Number of wire bits:             66
   Number of public wires:          22
   Number of public wire bits:      66
   Number of ports:                  8
   Number of port bits:             22
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:                 39
     DFFCE                           8
     IBUF                           13
     LUT1                            1
     LUT3                            8
     OBUF                            9

=== design hierarchy ===

   neosd                             1
     sreg                            1

   Number of wires:                761
   Number of wire bits:           1537
   Number of public wires:         761
   Number of public wire bits:    1537
   Number of ports:                 27
   Number of port bits:            144
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:               1201
     ALU                             8
     DFFC                            2
     DFFCE                          49
     DFFE                          209
     DFFR                           33
     GND                             1
     IBUF                           96
     LUT1                          210
     LUT2                           58
     LUT3                           54
     LUT4                          195
     MUX2_LUT5                     153
     MUX2_LUT6                      50
     MUX2_LUT7                      24
     MUX2_LUT8                      10
     OBUF                           48
     VCC                             1
```

## Estimating ASIC Size

Let's do the same thing for an ASIC target.
Unfortunately, things are a bit more complex for ASICs.
Let's first extend the Makefile `synth` and `summary` rules:

```Makefile
synth: $(OBJDIR)/gowin.syn.json $(OBJDIR)/ihp.syn.json summary

summary:
	@echo
	@echo ========================== FPGA Summary ==========================
	@echo
	@cat $(OBJDIR)/gowin.syn.stat
	@echo
	@echo ========================== IHP Summary ==========================
	@echo
	@cat $(OBJDIR)/ihp.syn.stat
```

Then let's add the new synthesis rule.
We're going to use the IHP [SG13G2](https://github.com/IHP-GmbH/IHP-Open-PDK) BiCMOS SiGe PDK here.
As we'll need to use some PDK related files, we need to export the `FLOW_HOME`  variable and make it point to a copy of the [OpenROAD Flow Scripts (ORFS)](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts):
```Makefile
export FLOW_HOME=/home/jpfau/Dokumente/orfs/flow

$(OBJDIR)/ihp.syn.json: $(SYN_SVERILOG_PATHS) | $(OBJDIR)
	yosys syn_ihp.tcl $(QUIET_FLAG) -l $(OBJDIR)/ihp.syn.log
```

As the ASIC synthesis script is much more complex, we won't provide it inline using the `-p` option and will save it to `syn_ihp.tcl` instead.
My synthesis script is a stripped down version of the one [shipped in ORFS](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts/blob/master/flow/scripts/synth.tcl).

Some steps have been removed, so the synthesis will be less optimal compared to putting your HDL through the full ORFS flow.
On the other hand, this simplification makes the script easier to maintain and understand.
The script also hard-codes some information for the IHP PDK, so it can not easily be used with other PDKs.
Here's the full synthesis script:
```tcl
# Import yosys commands
yosys -import

# PDK setup
set pdk_platform_dir $::env(FLOW_HOME)/platforms/ihp-sg13g2
set pdk_scripts_dir $::env(FLOW_HOME)/scripts
set pdk_stdcell_lib $pdk_platform_dir/lib/sg13g2_stdcell_typ_1p20V_25C.lib
set pdk_dont_use_cells {sg13g2_lgcp_1 sg13g2_sighold sg13g2_slgcp_1 sg13g2_dfrbp_2}
set pdk_latch_map $pdk_platform_dir/cells_latch.v
set pdk_tiehi_cell_port {sg13g2_tiehi L_HI}
set pdk_tielo_cell_port {sg13g2_tielo L_LO}

# Read app sources
read_verilog -defer -sv {*}$::env(SYN_SVERILOG_PATHS)
# Read stdcells
read_liberty -overwrite -setattr liberty_cell -lib $pdk_stdcell_lib

hierarchy -check -top $::env(TOP_MODULE)
# Synthesize, don't flatten
synth -run :fine
# Remove non-synthesizeable stuff
chformal -remove
delete t:\$print
# Optimize
opt -purge
# Technology mapping
techmap
techmap -map $pdk_latch_map
# Map D FFs to stdcell library
set dont_use_args ""
foreach cell $pdk_dont_use_cells {
  lappend dont_use_args -dont_use $cell
}
dfflibmap -liberty $pdk_stdcell_lib {*}$dont_use_args
opt
# ABC optimization
abc -script $pdk_scripts_dir/abc_area.script -liberty $pdk_stdcell_lib {*}$dont_use_args
# Set undefined values to 0
setundef -zero
# Remove unused stuff
opt_clean -purge
# Technology mapping for constant 1/0
hilomap -singleton \
        -hicell {*}$pdk_tiehi_cell_port \
        -locell {*}$pdk_tielo_cell_port
# Write out design
json -o $::env(OBJDIR)/ihp.syn.json
# Reports
tee -o $::env(OBJDIR)/ihp.syn.check check
tee -o $::env(OBJDIR)/ihp.syn.stat stat -liberty $pdk_stdcell_lib
# Check that we mapped everything to std cells
check -assert -mapped
```

The comments in the script should provide some hints at what it is doing.
Finally, here's our `make synth` output for the ASIC part:
```
========================== IHP Summary ==========================


17. Printing statistics.

=== neosd ===

   Number of wires:               1670
   Number of wire bits:           2013
   Number of public wires:          38
   Number of public wire bits:     381
   Number of ports:                 19
   Number of port bits:            122
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:               1634
     sg13g2_a21oi_1                 78
     sg13g2_a21oi_2                  1
     sg13g2_a221oi_1                11
     sg13g2_a22oi_1                 45
     sg13g2_and2_1                   2
     sg13g2_and3_1                   1
     sg13g2_and4_1                   2
     sg13g2_buf_1                   12
     sg13g2_buf_2                   35
     sg13g2_buf_4                   39
     sg13g2_buf_8                   19
     sg13g2_dfrbp_1                285
     sg13g2_inv_1                  107
     sg13g2_inv_2                   10
     sg13g2_inv_4                    1
     sg13g2_mux2_1                  18
     sg13g2_nand2_1                467
     sg13g2_nand2_2                  8
     sg13g2_nand2b_1                 8
     sg13g2_nand3_1                 48
     sg13g2_nand3b_1                 1
     sg13g2_nand4_1                138
     sg13g2_nor2_1                 117
     sg13g2_nor2_2                  20
     sg13g2_nor2b_1                 61
     sg13g2_nor2b_2                  1
     sg13g2_nor3_1                  12
     sg13g2_nor3_2                   2
     sg13g2_nor4_1                  19
     sg13g2_o21ai_1                 62
     sg13g2_tiehi                    1
     sg13g2_tielo                    1
     sg13g2_xnor2_1                  1
     sreg                            1

   Area for cell type \sreg is unknown!

   Chip area for module '\neosd': 25358.167800
     of which used for sequential elements: 13444.704000 (53.02%)

=== sreg ===

   Number of wires:                 65
   Number of wire bits:             79
   Number of public wires:           8
   Number of public wire bits:      22
   Number of ports:                  8
   Number of port bits:             22
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:                 57
     sg13g2_dfrbp_1                  8
     sg13g2_inv_2                    1
     sg13g2_nand2_1                 24
     sg13g2_nand2b_1                 8
     sg13g2_nor2_2                   8
     sg13g2_nor2b_2                  8

   Chip area for module '\sreg': 820.108800
     of which used for sequential elements: 377.395200 (46.02%)

=== design hierarchy ===

   neosd                             1
     sreg                            1

   Number of wires:               1735
   Number of wire bits:           2092
   Number of public wires:          46
   Number of public wire bits:     403
   Number of ports:                 27
   Number of port bits:            144
   Number of memories:               0
   Number of memory bits:            0
   Number of processes:              0
   Number of cells:               1690
     sg13g2_a21oi_1                 78
     sg13g2_a21oi_2                  1
     sg13g2_a221oi_1                11
     sg13g2_a22oi_1                 45
     sg13g2_and2_1                   2
     sg13g2_and3_1                   1
     sg13g2_and4_1                   2
     sg13g2_buf_1                   12
     sg13g2_buf_2                   35
     sg13g2_buf_4                   39
     sg13g2_buf_8                   19
     sg13g2_dfrbp_1                293
     sg13g2_inv_1                  107
     sg13g2_inv_2                   11
     sg13g2_inv_4                    1
     sg13g2_mux2_1                  18
     sg13g2_nand2_1                491
     sg13g2_nand2_2                  8
     sg13g2_nand2b_1                16
     sg13g2_nand3_1                 48
     sg13g2_nand3b_1                 1
     sg13g2_nand4_1                138
     sg13g2_nor2_1                 117
     sg13g2_nor2_2                  28
     sg13g2_nor2b_1                 61
     sg13g2_nor2b_2                  9
     sg13g2_nor3_1                  12
     sg13g2_nor3_2                   2
     sg13g2_nor4_1                  19
     sg13g2_o21ai_1                 62
     sg13g2_tiehi                    1
     sg13g2_tielo                    1
     sg13g2_xnor2_1                  1

   Chip area for top module '\neosd': 26178.276600
     of which used for sequential elements: 377.395200 (1.44%)
```

{{< box info >}}
The most reliable information that can be obtained here is the number and type of gates.
Chip area after synthesis does not consider any potential routing congestion or other placement related issues or overhead and is therefore only a rough estimate.
{{< /box >}}

## Conclusion

With the scripts offered here you can quickly evaluate the size of Verilog designs on FPGA and ASIC.
Of course the obtained results are somewhat technology specific.

For the FPGA target, the main difference to other vendors' FPGAs is the LUT size.
The absolute number of LUTs used therefore obviously can't be compared between different FPGA types.
However, the resource count can be a useful relative metric to guide optimization of your designs.
In addition, be careful if the synthesis uses special cells, which might be vendor specific (such as the ALU one).

For the ASIC target, results depend on the standard cell library and PDK you use.
Of course area depends on the technology node, so gate count is more reliable.
The amount of gates used will however also depend on what kind of gates are available in your standard cell library.
Again, the obtained metrics are mainly useful to compare different iterations of a design **in the same technology**.

{{< box warning >}}
Be careful when you start using specific hard IP: Make sure to also specify it using black boxes.
If you rely on inference for IP, chances are yosys will not map those to dedicated block but synthesize it to basic cells instead.
{{< /box >}}