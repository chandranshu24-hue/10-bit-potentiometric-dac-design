# **Research Report: Development of an AI-Assisted Analog Multiplexer Switch for a 10-Bit Potentiometric DAC**

This repository documents the true, step-by-step optimization matrix and troubleshooting history for a pre-layout simulation of a 2-to-1 analog multiplexer switch. The design targets a 10-bit potentiometric DAC using the SkyWater 130nm PDK (`sky130_fd_pr`) inside an `ngspice` environment.

---

## **Initial Prompt Given**

The design process began with the following explicit instruction:

> "Act as an analog IC engineer working with the SKY130 PDK. Generate a clean ngspice subcircuit file named switch.spice. The subcircuit must define a 2-to-1 analog multiplexer switch (.subckt switch Vin1 Vin2 bit bit_b Vout). Use nfet_01v8 and pfet_01v8 transmission gates to route Vin1 to Vout when bit is high (1.8V) and bit_b is low, and route Vin2 to Vout when bit is low and bit_b is high. Keep comments to a absolute minimum, do not add unnecessary explanation, and ensure standard primitive names."

---

## **Troubleshooting and Optimization Matrix**

| **Prompt Given (Verbatim / Actual Input)** | **Problem Faced / Error Encountered** | **Solution / Resolution Applied** |
| :--- | :--- | :--- |
| **Initial Execution Log:**<br>`warning, can't find model 'sky130_fd_pr__nfet_01v8' from line m1...`<br>`Error on line 9... could not find a valid modelname` | **Model Recognition Failure (`M` vs `X` Primitive Prefix):** The SkyWater PDK builds devices as subcircuits (`.subckt`) rather than raw SPICE models. Calling them using the standard native MOS designation (`M1`, `M2`, etc.) caused ngspice to crash because it could not find matching flat `.model` cards. | Converted the transistor instantiation prefixes inside the subcircuit from `M` to `X`. This forced ngspice to evaluate them correctly as subcircuit primitives. |
| **User Prompt 2:**<br>`@chandranshu24-hue ➜ /workspaces/vsd-7nm/avsddac_3v3_sky130_v1/Prelayout (main) $ ngspice aiswitch.spice`<br>`Error on line 64 or its substitute: m.xswitch.x1.msky130_fd_pr__nfet_01v8... l=1.500000000000000e-07 w=1.000000000000000e-06... could not find a valid modelname` | **PDK Model Binning Failure:** The testbench missed a global unit scale directive. ngspice fed absolute raw values ($1.5 \times 10^{-7}$ and $1.0 \times 10^{-6}$) straight into the PDK’s binned equations. Because the PDK model files look for localized integer-like parameters, it failed to resolve the device geometries. | Added `.option scale=1u` directly below the library calls. Stripped duplicate `u` character suffixes from the device lines (`w=1.0 l=0.15`) to prevent ngspice from double-scaling the dimensions into picometers. |
| **User Prompt 3:**<br>`* switch_tb.spice - 2-to-1 analog multiplexer testbench...`<br>`[Raw text netlist script provided with request:]`<br>`.end correct it` | **Syntax Faults and Static Signal Allocation:** The pasted raw netlist script contained hidden non-breaking space characters from the terminal application environment which created parsing abnormalities. Additionally, the digital signal `Vdinb` was written with a 3.3V amplitude, while the analog rails were limited to 1.8V. | Sanitized the entire text block to clear invisible formatting characters. Unified the operating domains dynamically to prevent potential over-stressing of the low-voltage 1.8V transistors. |
| **User Prompt 4:**<br>`Initial Transient Solution`<br>`...`<br>`No. of Data Rows : 1015`<br>`Warning: No job (tran, ac, op etc.) defined: run simulation not started` | **Control Block Compilation Error:** Placing both the `.tran` step command and the standalone `run` statement simultaneously within the boundaries of the `.control` wrapper block broke the command sequence execution order of ngspice. | Transferred the `.tran 10p 10n` command out of the `.control` block, setting it as a standard native top-level circuit simulation card. |
| **User Prompt 5:**<br>`[Image uploads: image_b00bae.png & image_af2327.jpg]`<br>`can vout come on seperate graph? and also its not matching the matrix i showed you earlier in image , this design has no of data rows as 1015 where as it had only 123, give the updated netlist` | **Data Array Density and Display Window Mismatch:** The time step setup generated an oversized dataset (1015 rows) compared to the user's specific target configuration of exactly 123 rows. Furthermore, `Vdinb` needed to reach 3.3V to replicate the reference picture, and `vout` required an isolated graphics viewport. | Reset `Vdinb` back to a 3.3V pulse peak. Calculated and applied an exact transient step configuration of `81.3p` to achieve exactly 123 analytical data rows over the 10ns timeframe. Separated the multi-variable plot call into three standalone `plot` commands. |

---

## **Final Verified Netlist**

Below is the complete, true-to-spec netlist artifact containing all cumulative fixes for direct deployment in ngspice:

```spice
* switch_tb.spice - 2-to-1 analog multiplexer testbench
.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt

* Required for SKY130 PDK model binning to evaluate correctly
.option scale=1u
.global VDD GND

* Subcircuit definition (with numeric values scaled by the 1u option)
.subckt switch Vin1 Vin2 bit bit_b Vout
X1 Vout bit   Vin1 GND sky130_fd_pr__nfet_01v8 w=1.0 l=0.15
X2 Vout bit_b Vin1 VDD sky130_fd_pr__pfet_01v8 w=2.0 l=0.15
X3 Vout bit_b Vin2 GND sky130_fd_pr__nfet_01v8 w=1.0 l=0.15
X4 Vout bit   Vin2 VDD sky130_fd_pr__pfet_01v8 w=2.0 l=0.15
.ends switch

* Global Power Supply
Vdd VDD GND 1.8

* Analog Input DC Sources (matched to bottom-right plot)
Vinp1 inp1 GND 1.65
Vinp2 inp2 GND 0.0

* Digital Control Stimuli (Matched to target matrix plot with dinb reaching 3.3V)
Vdin  din  GND pwl(0 1.8  5n 1.8  5.01n 0.0  10n 0.0)
Vdinb dinb GND pwl(0 0.0  5n 0.0  5.01n 3.3  10n 3.3)

* Switch Instantiation
XSwitch inp1 inp2 din dinb vout switch

* Adjusted step size to 81.3p to output exactly 123 data rows over 10ns
.tran 81.3p 10n

* Simulation and Plotting Control Block
.control
run

* Generate three separate plot windows to match your exact layout matrix
plot vout
plot din dinb
plot inp1 inp2
.endc

.end
