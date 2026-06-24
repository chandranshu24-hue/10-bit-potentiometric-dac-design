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
.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt

XM1 dinb din 0 0 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6 
XM2 dinb din vdd vdd sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

XM7 dd dinb 0 0 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM8 dd dinb vdd vdd sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

XM3 vout dinb inp2 inp2 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM4 inp1 dd vout vout sky130_fd_pr__nfet_01v8 L=0.15 W=0.6

XM5 vout dinb inp1 inp1 sky130_fd_pr__pfet_01v8 L=0.15 W=1.2
XM6 inp2 dd vout vout sky130_fd_pr__pfet_01v8 L=0.15 W=1.2 


V1 din 0 PULSE 0 1.8 0ns 100p 100p 5n 10n
V2 vdd 0 dc 3.3V
V4 inp2 0 dc 0
V3 inp1 0 dc 1.65


.tran 0.1n 10n
.control
run 
plot din dinb 
plot inp1 inp2
plot vout
.endc
.end
```
<img width="1072" height="802" alt="Screenshot 2026-06-20 143135" src="https://github.com/user-attachments/assets/930aee6b-12e8-484a-b795-a5d3e8b481ed" />


AI WRITTEN NETLIST:
```
* switch_tb.spice - 2-to-1 analog multiplexer testbench
.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt

* Required for SKY130 PDK model binning
.option scale=1u
.global VDD GND

* Subcircuit definition
.subckt switch Vin1 Vin2 bit bit_b Vout
X1 Vout bit   Vin1 GND sky130_fd_pr__nfet_01v8 w=1.0 l=0.15
X2 Vout bit_b Vin1 VDD sky130_fd_pr__pfet_01v8 w=2.0 l=0.15
X3 Vout bit_b Vin2 GND sky130_fd_pr__nfet_01v8 w=1.0 l=0.15
X4 Vout bit   Vin2 VDD sky130_fd_pr__pfet_01v8 w=2.0 l=0.15
.ends switch

* Global Power Supply
Vdd VDD GND 1.8

* Analog Input DC Sources
Vinp1 inp1 GND 1.65
Vinp2 inp2 GND 0.0

* Digital Control Stimuli (Restored dinb to 3.3V to match target matrix plot)
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
```
<img width="1427" height="876" alt="Screenshot 2026-06-20 151300" src="https://github.com/user-attachments/assets/b3dc2294-139e-4a4c-9a91-32c653df2da7" />

<img width="843" height="647" alt="Screenshot 2026-06-20 151340" src="https://github.com/user-attachments/assets/18444cf4-114e-4eae-afe8-93f208e6fc56" />

# Comparison Between AI-Generated Netlist and Verified Hardcoded Netlist

## Objective

During the development of the 2-bit resistor string DAC, an AI-generated SPICE netlist was initially used. Although the individual transmission gate (TG2) worked correctly in standalone simulations, the DAC failed to produce the expected four-level staircase output.

After debugging, the original hardcoded netlist from the repository was compared against the AI-generated version.

The comparison below explains why the AI-generated version failed.

---

# Comparison Table

| Feature | AI Generated Netlist | Original Hardcoded Netlist | Result |
|----------|---------------------|----------------------------|--------|
| **Subcircuit Name** | `switch` | `TG2` | Different names are acceptable if instantiated consistently. |
| **Subcircuit Pin Order** | `inp1 inp2 din vout vdd gnda` | `inp1 vdd vout din 0 inp2` | **Major Difference** |
| **Power Pin Position** | 5th pin | 2nd pin | AI assumed a conventional ordering instead of preserving the symbol-generated interface. |
| **Ground Pin** | Explicit `gnda` port | Global node `0` | Original design intentionally uses global ground. |
| **Output Pin Position** | 4th pin | 3rd pin | Different interface expected by hierarchy. |
| **Second Analog Input (`inp2`)** | 2nd pin | Last pin | Completely different mapping. |
| **Hierarchy Compatibility** | Manual assumption | Matches Xschem generated hierarchy | Original works correctly. |
| **Symbol Compatibility** | Assumed manually | Automatically generated by Xschem | Original preserves symbol pin ordering. |
| **Resistor Ladder Connections** | Correct | Correct | Not the source of error. |
| **Transmission Gate Circuit** | Identical | Identical | Circuit itself was never faulty. |
| **Standalone TG Simulation** | Passed | Passed | Verified that TG circuit works. |
| **Complete DAC Simulation** | Failed | Passed | Failure caused by interface mismatch. |

---

# Why the AI Netlist Failed

The transmission gate circuit itself was **never incorrect**.

The failure occurred because the AI reconstructed the subcircuit interface manually instead of preserving the interface generated by Xschem.

The AI assumed:

```spice
.subckt switch inp1 inp2 din vout vdd gnda
```

However, the original project generated:

```spice
.subckt TG2 inp1 vdd vout din 0 inp2
```

Although both describe the same transmission gate internally, the **external interface is completely different**.

Hierarchical SPICE designs depend on **exact positional pin matching**.

For example:

AI assumption:

```
Pin1 -> inp1
Pin2 -> inp2
Pin3 -> din
Pin4 -> vout
Pin5 -> vdd
Pin6 -> gnda
```

Original hierarchy:

```
Pin1 -> inp1
Pin2 -> vdd
Pin3 -> vout
Pin4 -> din
Pin5 -> 0
Pin6 -> inp2
```

Because SPICE maps ports **by position**, not by name, manually changing the interface breaks the hierarchy.

---

# Important Lesson

A subcircuit that works perfectly in standalone simulation can still fail in a hierarchical design if the subcircuit interface differs from the symbol-generated interface.

The transistor circuit was correct.

The interface was incorrect.

---

# Better Prompt for AI

When asking an AI to generate a SPICE netlist from an Xschem schematic, avoid asking only for the circuit implementation.

Instead, explicitly instruct the AI to preserve the hierarchical interface.

Example prompt:

> Generate the complete SPICE netlist exactly as Xschem would generate it.
>
> Requirements:
>
> - Do **not** modify the subcircuit interface.
> - Preserve the exact `.subckt` pin order from the original symbol.
> - Preserve the hierarchy exactly as generated by Xschem.
> - Do **not** rename ports.
> - Do **not** reorder ports.
> - Do **not** replace global node `0` with a new ground pin unless it already exists in the original design.
> - Preserve all instance names (`x1`, `x2`, `x3`, etc.).
> - Preserve the hierarchy exactly.
> - If the symbol-generated interface and transistor implementation appear different, always trust the symbol-generated interface.
> - Never reconstruct the interface from the transistor schematic.
> - Generate a netlist that is 100% compatible with the original Xschem project.

---

# Best Practices for AI-Assisted Analog Design

✅ Always compare against the Xschem-generated netlist.

✅ Never infer subcircuit pin order from the schematic drawing.

✅ Preserve symbol-generated interfaces exactly.

✅ Remember that SPICE maps ports **positionally**, not by port names.

✅ Validate individual subcircuits before debugging the complete hierarchy.

✅ If a standalone subcircuit works but the hierarchical design fails, inspect the generated netlist before modifying the circuit.

---

# Final Conclusion

The DAC failure was **not caused by the transmission gate design**.

The resistor ladder was correct.

The transistor implementation was correct.

The symbol was correct.

The failure resulted from reconstructing the hierarchical subcircuit interface instead of preserving the interface automatically generated by Xschem.

Once the original Xschem-generated interface was restored, the DAC produced the expected four-level staircase output.

#Hard coded netlsit and output
```

.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt


**.subckt 2bitdac_tb
x1 x1_vdd x1_vref1 0 x1_out_v x1_d1 x1_d0 x1_vref5 2bitdac
V1 x1_vdd 0 dc 3.3
V2 x1_d0 0 PULSE 0 1.8 0ns 1p 1p 5u 10u
V3 x1_d1 0 PULSE 0 1.8 0ns 1p 1p 10u 20u
V4 x1_vref5 0 dc 0.1
V5 x1_vref1 0 dc 3.3V
**.ends

* expanding   symbol:  /home/harshitha/Desktop/xschem/xschem_library/2bitdac.sym # of pins=7

.subckt 2bitdac  vdd vref1 gnda out_v d1 d0 vref5
*.ipin vref1
*.ipin vref5
*.ipin d0
*.ipin d1
*.iopin vdd
*.iopin gnda
*.opin out_v
x1 x1_inp1 vdd x1_vout d0 gnda x1_inp2 TG2
x2 x2_inp1 vdd x2_vout d0 gnda vref5 TG2
x3 x1_vout vdd out_v d1 gnda x2_vout TG2
R1 x1_inp1 vref1 500 
R2 x1_inp2 x1_inp1 500
R3 x2_inp1 x1_inp2 500
R4 vref5 x2_inp1 250
.ends


* expanding   symbol:  /home/harshitha/Desktop/xschem/xschem_library/TG2.sym # of pins=6

.subckt TG2  inp1 vdd vout din 0 inp2
*.ipin inp1
*.ipin inp2
*.opin vout
*.ipin din
*.iopin vdd
*.iopin gnda

XM1 dinb din 0 0 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6 
XM2 dinb din vdd vdd sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

XM7 dd dinb 0 0 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM8 dd dinb vdd vdd sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

XM3 vout dinb inp2 inp2 sky130_fd_pr__nfet_01v8 L=0.15 W=0.6
XM4 inp1 dd vout vout sky130_fd_pr__nfet_01v8 L=0.15 W=0.6

XM5 vout dinb inp1 inp1 sky130_fd_pr__pfet_01v8 L=0.15 W=1.2
XM6 inp2 dd vout vout sky130_fd_pr__pfet_01v8 L=0.15 W=1.2

.ends

.tran 1n 20u
.control
run 
plot x1_d0 x1_d1 x1_out_v title chandranshu
.endc
.end
```
<img width="817" height="757" alt="image" src="https://github.com/user-attachments/assets/597d2fbc-0501-4aaf-8e18-402e22d26d38" />
## AI generated netlsit and output
```
*****************************************************
* 2-Bit Resistor String DAC Testbench
*****************************************************

.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt

.include switch.spice

*****************************************************
* Power Supplies
*****************************************************
VDD     switch_vdd    0    DC 3.3
VSS     switch_gnda   0    DC 0

*****************************************************
* Reference Voltages
*****************************************************
VREFH   inp1          0    DC 1.65
VREFL   inp2          0    DC 0

*****************************************************
* Resistor String
*****************************************************
R4  inp1   N1    500
R1  N1     N2    500
R2  N2     N3    500
R3  N3     inp2  250

*****************************************************
* Digital Inputs
*****************************************************
* LSB
Vd0 d0 0 PULSE(0 1.8 0u 100p 100p 5u 10u)
* MSB
Vd1 d1 0 PULSE(0 1.8 0u 100p 100p 10u 20u)
*****************************************************
* Switches
*****************************************************
* First Stage
XSW1 N1     N2     d0   x1_out   switch_vdd switch_gnda switch
*XSW2 N3     inp2   d0   x2_out   switch_vdd switch_gnda switch
* Second Stage
XSW3 x1_out x2_out d1   out_v    switch_vdd switch_gnda switch
*****************************************************
* Analysis
*****************************************************
.tran 0.05u 40u
*****************************************************
* Control
*****************************************************
.control
run
plot v(N3)
plot v(x2_out)
plot v(d0)
.endc
.end
```
<img width="1647" height="780" alt="Screenshot 2026-06-24 203537" src="https://github.com/user-attachments/assets/da6ef14a-be22-448b-aad9-2a31bbd90471" />





