# **Research Report: Development of an AI-Assisted Analog Multiplexer Switch for a 10-Bit Potentiometric DAC**

This repository documents the comprehensive troubleshooting history, prompt optimization paths, and comparative benchmarks for a pre-layout simulation of a 2-to-1 analog multiplexer pass-gate switch. This block is designed for integration into a 10-bit potentiometric DAC architecture using the SkyWater 130nm PDK (`sky130_fd_pr`) simulated within an `ngspice` environment.

---

## **Initial Prompt Given**

The project initiated with the following explicit engineering design instruction:

> "Act as an analog IC engineer working with the SKY130 PDK. Generate a clean ngspice subcircuit file named switch.spice. The subcircuit must define a 2-to-1 analog multiplexer switch (.subckt switch Vin1 Vin2 bit bit_b Vout). Use nfet_01v8 and pfet_01v8 transmission gates to route Vin1 to Vout when bit is high (1.8V) and bit_b is low, and route Vin2 to Vout when bit is low and bit_b is high. Keep comments to a absolute minimum, do not add unnecessary explanation, and ensure standard primitive names."

---

## **Troubleshooting and Optimization Matrix**

| **Prompt Given (Verbatim / Actual Input)** | **Problem Faced / Error Encountered** | **Solution / Resolution Applied** |
| :--- | :--- | :--- |
| **Initial Simulation Attempt (Log Output):**<br>`warning, can't find model 'sky130_fd_pr__nfet_01v8' from line m1...`<br>`Error on line 9... could not find a valid modelname` | **Model Recognition Failure (`M` vs `X` Primitive Prefix):** The SkyWater PDK models physical transistors as complex subcircuits (`.subckt`) rather than standard flat models. Instantiating them with the default native SPICE MOS designation (`M1`, `M2`, etc.) caused ngspice to look for native `.model` cards and crash. | Converted the internal transistor instance prefixes from `M` to `X`. This forced ngspice to recognize and map them correctly as subcircuit primitives. |
| **User Prompt 2:**<br>`@chandranshu24-hue ➜ /workspaces/vsd-7nm/avsddac_3v3_sky130_v1/Prelayout (main) $ ngspice aiswitch.spice`<br>`Error on line 64 or its substitute: m.xswitch.x1.msky130_fd_pr__nfet_01v8... l=1.500000000000000e-07 w=1.000000000000000e-06... could not find a valid modelname` | **PDK Model Binning Failure:** The netlist missed a global unit scale directive. ngspice passed raw metric parameters ($1.5 \times 10^{-7}$ and $1.0 \times 10^{-6}$) straight into the PDK’s binned equations. Because the model files look for localized micron-scaled expressions, it could not resolve the channel geometries. | Appended `.option scale=1u` directly beneath the library inclusion cards. Stripped the absolute `u` suffixes from the length and width parameters (`w=1.0 l=0.15`) to prevent double-scaling the dimensions down into picometers. |
| **User Prompt 3:**<br>`* switch_tb.spice - 2-to-1 analog multiplexer testbench...`<br>`[Raw text netlist script provided with request:]`<br>`.end correct it` | **Syntax Faults and Static Signal Allocation:** The pasted raw netlist text block contained hidden non-breaking space characters from the terminal environment wrapper, causing parsing layout bugs. Additionally, the digital signal `Vdinb` was configured to step to 3.3V while running on a low-voltage 1.8V analog rail power domain. | Sanitized the text file to wipe out all invisible escape formatting characters. Unified the active operating domains temporarily to ensure simulation stability without clipping. |
| **User Prompt 4:**<br>`Initial Transient Solution`<br>`...`<br>`No. of Data Rows : 1015`<br>`Warning: No job (tran, ac, op etc.) defined: run simulation not started` | **Control Block Compilation Error:** Placing both the transient analysis statement (`.tran`) and the execution command (`run`) inside the `.control` boundary wrapper syntax broke the internal command processing pipeline of the ngspice compiler. | Extracted the `.tran` step command out of the `.control` block wrapper, placing it cleanly as a top-level native circuit analysis card. |
| **User Prompt 5:**<br>`[Image uploads: image_b00bae.png & image_af2327.jpg]`<br>`can vout come on seperate graph? and also its not matching the matrix i showed you earlier in image , this design has no of data rows as 1015 where as it had only 123, give the updated netlist` | **Data Array Density and Display Mismatch:** The transient step was set too fine (`10p`), yielding an oversized dataset of 1015 rows. The target specification required matching the 123-row layout profile, updating `Vdinb` to scale up to 3.3V to fit the layout matrix properly, and decoupling the output trace into an isolated viewport. | Configured `Vdinb` to a 3.3V pulse peak. Adjusted the transient time step precisely to `81.3p` to generate exactly 123 analytical data rows over the 10ns window. Separated the multi-variable plot call into three standalone `plot` commands to yield detached display windows. |

---

## **Comparative Analysis: Handcoded vs. AI-Assisted Designs**

| **Parameter / Metric** | **Handcoded Approach (Without AI)** | **AI-Assisted Approach** |
| :--- | :--- | :--- |
| **Hierarchy Style** | **Flat Netlist:** Transistors are declared linearly in the global space without subcircuit blocks. | **Hierarchical Macro:** Switch core is cleanly encapsulated inside a reusable `.subckt switch` definition block. |
| **Local Gate Drive Logic** | **Integrated Control:** Features 2 local CMOS inverter stages (`XM1`/`XM2` and `XM7`/`XM8`) to dynamically derive localized complementary select logic lines inside the deck. | **Externalized Control:** Relies entirely on external differential inputs (`din` and `dinb`) driven directly by the system testbench configuration. |
| **Transistor Geometries** | **Narrower Channels:** <br>• NFET: $W = 0.6\,\mu\text{m},\, L = 0.15\,\mu\text{m}$<br>• PFET: $W = 1.2\,\mu\text{m},\, L = 0.15\,\mu\text{m}$ | **Wider Channels (Lower On-Resistance):** <br>• NFET: $W = 1.0\,\mu\text{m},\, L = 0.15\,\mu\text{m}$<br>• PFET: $W = 2.0\,\mu\text{m},\, L = 0.15\,\mu\text{m}$ |
| **PDK Scaling Mechanics** | **Literal Character Definitions:** Uses text-appended dimension strings (e.g., `W=0.6`, `L=0.15`) without a parent scaling statement. | **Global Matrix Scaling:** Incorporates a top-level `.option scale=1u` option card to seamlessly streamline model geometry processing. |
| **Simulation Data Rows** | **123 Data Rows:** Generated during output execution analysis. | **143 Data Rows:** Obtained during intermediate troubleshooting phase analysis. |
| **Output Node Protection** | **Explicit Capacitive Load:** Includes an explicit $1\,\text{fF}$ load shunt capacitor (`C1`) to keep the output node from floating and causing execution matrix errors. | **Native High-Impedance Boundary:** Simulates steadily without requiring external dummy shunt capacitance additions on the output node. |
| **Plot Call Syntax Style** | **Functional Wrapping:** Requires explicit formatting wrapping expressions (e.g., `plot v(vout)`) to correctly trigger display routines. | **Raw Vector Variable Name:** Directly targets active node variable handles (e.g., `plot vout`) across detached viewport frames. |

---

## **Simulation Netlist Artifacts**

### **1. Handcoded Baseline Netlist (Without AI)**
```spice
* Fully Corrected Transmission Gate Testbench

* 1. Fixed: Restored working absolute library path
.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt

* Inverter 1 (din -> dinb)
XM1 dinb din 0 0 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM2 dinb din vdd vdd sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

* Inverter 2 (dinb -> dd)
XM7 dd dinb 0 0 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM8 dd dinb vdd vdd sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

* Transmission Gate (Fixed: Removed trailing non-breaking hidden spaces)
XM3 vout dinb inp2 inp2 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM4 inp1 dd vout vout sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM5 vout dinb inp1 inp1 sky130_fd_pr__pfet_01v8 L=0.15 W=1.2
XM6 inp2 dd vout vout sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

* Voltage Sources (Fixed: Dropped 'V' from '3.3V' to prevent syntax confusion)
V1 din 0 PULSE 0 1.8 0ns 100p 100p 5n 10n
V2 vdd 0 dc 3.3
V4 inp2 0 dc 0
V3 inp1 0 dc 1.65

* Added a tiny 1fF capacitor to prevent vout from floating and crashing the transient matrix
C1 vout 0 1f

.tran 0.1n 10n
.control
run 
* 2. Fixed: Wrapped node names inside v() for ngspice plotting
plot v(din) v(dinb) 
plot v(inp1) v(inp2)
plot v(vout)
.endc
.end
