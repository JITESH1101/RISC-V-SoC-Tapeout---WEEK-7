# RISC-V-SoC-Tapeout---WEEK-7

 The main goal this week was to take a synthesized RISC-V SoC design, the `VsdBabySoC`, and run it through the entire physical design (PD) flow using the **OpenROAD-flow-scripts (ORFS)**.

This task was all about connecting the dots—seeing how the digital logic we write in Verilog (`RTL`) gets translated into a physical, routable layout (`GDSII`) ready for fabrication. It's where the theory of chip design meets the practical reality of layout, timing, and parasitic extraction.

## 🎯 The Objective: A Full Physical Design Flow

Up to this point, many steps like synthesis, STA, and layout were separate exercises. The goal of this task was to integrate them into one continuous flow, just like in a real-world ASIC design process.

My mission was to:
1.  **Set up the Environment:** Install and configure the OpenROAD-flow-scripts.
2.  **Configure the Project:** Teach the flow about our `BabySoC` design, its Verilog files, and its special macros (like the PLL and DAC).
3.  **Run the Flow:** Execute all the main physical design stages:
    * Synthesis (if not already done)
    * Floorplanning
    * Placement
    * Clock Tree Synthesis (CTS)
    * Routing
4.  **Extract Parasitics:** Generate the critical **SPEF** (Standard Parasitic Exchange Format) file, which is essential for accurate, post-layout timing signoff.

---

## 🛠️ Part 1: Setting Up the Environment (OpenROAD)

First things first, I had to get the toolchain (ORFS) installed and built. This flow automates the entire process, using a set of open-source tools like Yosys, OpenROAD, and Magic.

Here are the commands I used to get the environment ready:

```bash
# 1. Clone the repository (recursively to get all submodules)
git clone --recursive [https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)

# 2. Change into the directory
cd OpenROAD-flow-scripts

# 3. Run the dependency setup script
sudo ./setup.sh

# 4. Build the OpenROAD tool itself (locally)
./build_openroad.sh --local

# 5. VERY IMPORTANT: Source the environment file to set up all the paths
# You have to do this every time you open a new terminal
source ./env.sh

# 6. Run a quick check to make sure the tools are in the path
yosys -help
openroad -help
```

<img width="3790" height="450" alt="directory" src="https://github.com/user-attachments/assets/16f3cf1a-bdc8-4284-b6ea-c208a3f0b880" />

To verify the installation , enter the following code to run the entire automated flow to check if the design works.

```
source ./env.sh
yosys -help
openroad -help
cd flow
make
```

<img width="3986" height="2302" alt="make_openroad_flow_scripts" src="https://github.com/user-attachments/assets/f43f6aff-a4ca-469b-a8c8-5608215c79b8" />

<img width="3975" height="2499" alt="gui_final" src="https://github.com/user-attachments/assets/70e9f7fc-168a-4f2c-8219-d974b4cbed6b" />


---

## 📂 Part 2: Configuring the BabySoC Project

This was the most important setup step. The OpenROAD flow needs to know *where* our design files are and *how* to build them for our target technology (Skywater 130nm, or `sky130hd`).

The flow cleverly separates the **design-specific files** (`src`) from the **platform-specific files** (`sky130hd`).

1.  **Design Source (`/flow/designs/src/vsdbabysoc/`)**
    * This is the PDK-independent folder.
    * I placed all the core Verilog files here (`vsdbabysoc.v`, `rvmyth.v`, `clk_gate.v`).

2.  **Platform Config (`/flow/designs/sky130hd/vsdbabysoc/`)**
    * This folder tells the flow *how* to build our design for `sky130hd`.
    * I copied all the tech-specific files here:
        * `gds/`: GDS files for the hard macros (`avsddac.gds`, `avsdpll.gds`).
        * `lef/`: LEF files for the macros (`avsddac.lef`, `avsdpll.lef`).
        * `lib/`: Timing `.lib` files for the macros (`avsddac.lib`, `avsdpll.lib`).
        * `sdc/`: The timing constraints for synthesis (`vsdbabysoc_synthesis.sdc`).
        * `macro.cfg` & `pin_order.cfg`: Files to guide the floorplanner.

<img width="1815" height="753" alt="tree" src="https://github.com/user-attachments/assets/232b6095-878f-401a-825a-919f956a01b3" />


### The Most important File: `config.mk`

This file, which I created inside `/flow/designs/sky130hd/vsdbabysoc/`, is the main "control panel" for the flow. It points to all the files we just organized and sets our design parameters.

```makefile
# --- Basic Design Info ---
export DESIGN_NICKNAME = vsdbabysoc
export DESIGN_NAME = vsdbabysoc
export PLATFORM    = sky130hd

# --- Source Files ---
# Point to all the Verilog files needed for synthesis
export VERILOG_FILES = $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/module/vsdbabysoc.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/module/rvmyth.v \
                       $(DESIGN_HOME)/src/$(DESIGN_NICKNAME)/module/clk_gate.v

# Point to the SDC (timing constraints) file
export SDC_FILE      = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)/sdc/vsdbabysoc_synthesis.sdc

# --- Macro / IP Configuration ---
export vsdbabysoc_DIR = $(DESIGN_HOME)/$(PLATFORM)/$(DESIGN_NICKNAME)
export VERILOG_INCLUDE_DIRS = $(wildcard $(vsdbabysoc_DIR)/include/)
export ADDITIONAL_GDS  = $(wildcard $(vsdbabysoc_DIR)/gds/*.gds)
export ADDITIONAL_LEFS  = $(wildcard $(vsdbabysoc_DIR)/lef/*.lef)
export ADDITIONAL_LIBS = $(wildcard $(vsdbabysoc_DIR)/lib/*.lib)

# --- Clock Configuration ---
export CLOCK_PORT = CLK
export CLOCK_NET = $(CLOCK_PORT)

# --- Floorplan & Die Config ---
export DIE_AREA   = 0 0 1600 1600
export CORE_AREA  = 20 20 1590 1590
export FP_PIN_ORDER_CFG = $(wildcard $(vsdbabysoc_DIR)/pin_order.cfg)
export MACRO_PLACEMENT_CFG = $(wildcard $(vsdbabysoc_DIR)/macro.cfg)

# --- Placement Config ---
export PLACE_PINS_ARGS = -exclude left:0-600 -exclude left:1000-1600: -exclude right:* -exclude top:* -exclude bottom:*

# --- CTS Tuning ---
export CTS_BUF_DISTANCE = 600
export SKIP_GATE_CLONING = 1
```

---

## 🗺️ Part 3: Running the Full Flow (The Fun Part!)

With the setup complete, all I had to do was `cd` into the `flow/` directory and run the `make` commands for each stage. The `DESIGN_CONFIG` variable points to the `config.mk` file we just made.

### 1. Synthesis

This step runs Yosys to turn the Verilog into a gate-level netlist.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk synth
```
The following is the synthesis log

<img width="3964" height="2506" alt="synthesis_1" src="https://github.com/user-attachments/assets/427da15f-b2f5-4e9c-97dc-d8d3b09f897f" />
<img width="3964" height="2506" alt="synthesis_2" src="https://github.com/user-attachments/assets/4976dff9-1a8c-44af-a000-ed14f8f54e9c" />
<img width="3964" height="2506" alt="synthesis_3" src="https://github.com/user-attachments/assets/0554caa2-6e53-4302-8316-92dac7c2aadc" />
<img width="3964" height="2506" alt="synthesis_4" src="https://github.com/user-attachments/assets/545f21cf-2a93-4c24-83a8-df8bf775a084" />
<img width="3964" height="2506" alt="synthesis_5" src="https://github.com/user-attachments/assets/b0da1aea-46f2-48cc-8be7-41af232f4585" />
<img width="3964" height="754" alt="synthesis_6" src="https://github.com/user-attachments/assets/54e624e1-cdf3-4781-b21e-e948cde9c882" />

Now the netlist generated is as follows

<img width="3978" height="2560" alt="netlist" src="https://github.com/user-attachments/assets/f6a260db-e360-43f5-8086-bcd3a4b3cb01" />
<img width="3978" height="2560" alt="netlist_1" src="https://github.com/user-attachments/assets/b5cac365-fe9a-402d-b71d-1208cc67ba25" />

The following is from ``synth_check.txt``

<img width="3978" height="512" alt="synth_check" src="https://github.com/user-attachments/assets/0c8c4fe3-8159-4486-8f82-4bb4e0424b83" />

Now the following picture is from ``Synth_stat`` file

<img width="3978" height="2510" alt="synth_stat" src="https://github.com/user-attachments/assets/30002dd2-f007-4d29-b3c6-e0b0d87bfb9d" />
<img width="3978" height="2510" alt="synth_stat1" src="https://github.com/user-attachments/assets/615009e3-57b7-4128-895f-ae0061f6f9b3" />
<img width="3978" height="2510" alt="synth_stat2" src="https://github.com/user-attachments/assets/9a33a719-e043-4172-8b97-4401e41738e2" />

With this detailed analysis , we see that everything is set in place and is working fine and now we can directly go through the entire PNR flow unless we are stopped by any possible error.

### 2. Floorplan
Here, the flow defines the die area, places the I/O pins, and positions our hard macros (PLL and DAC) according to the `macro.cfg`.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk floorplan
```
To see the result, I used the GUI command:
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_floorplan
```


### 3. Placement
This step takes all the standard cells from the netlist and finds a legal, optimized position for them inside the core area.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk place
```
And to view the placement heatmap (to check for congestion):
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_place
```


### 4. Clock Tree Synthesis (CTS)
This is a critical step where the tool builds a balanced buffer tree to distribute the clock signal (`CLK`) to all the flip-flops with minimal skew.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk cts
```
After this step, the tool runs a Static Timing Analysis (STA). The reports showed **no setup or hold violations**, which is great!

> **Post-CTS Timing Report Summary:**
> * **WNS (Worst Negative Slack):** 5.55 ns (MET)
> * **TNS (Total Negative Slack):** 0.00 ns (MET)
> * **Setup Violations:** 0
> * **Hold Violations:** 0

### 5. Routing
This is the final layout step. The global router plans the paths for all the nets, and the detailed router draws the actual metal wires to make the connections.
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk route
```
And the final view of the fully routed chip:
```bash
make DESIGN_CONFIG=./designs/sky130hd/vsdbabysoc/config.mk gui_route
```


---

## ⚡ Part 4: What is SPEF?

The final step of the flow, and a key goal of this week, was to generate the **SPEF (Standard Parasitic Exchange Format)** file.

### Why is SPEF so important?

Before routing, our timing analysis (pre-STA) is just an *estimate*. It uses a "Wire Load Model" to guess the resistance (R) and capacitance (C) of the wires that *will* be drawn.

After routing, we have the *actual, physical wires*. The SPEF file contains the **real, extracted parasitic R and C values** for every single net in the design.

**In short: SPEF = Ground Truth.**

When we run our final Static Timing Analysis (known as **post-route STA**), we feed it this SPEF file. This allows the timer to calculate the *actual* delays through the wires. This is the "signoff" analysis that determines if our chip will *really* work at the target frequency.

Without a SPEF file, you're just guessing. With it, you *know*.

---

## ✅ Conclusion: Week 7 Complete!

The BabySoC physical design flow for Week 7 is complete. The OpenROAD-flow-scripts environment was configured, and the chip was successfully processed through synthesis, floorplanning, placement, CTS, and routing. The primary objective was met with the generation of the post-route SPEF file, capturing the real parasitic data essential for signoff-level timing analysis. This concludes the tasks for Week 7, with Week 8 to follow.
