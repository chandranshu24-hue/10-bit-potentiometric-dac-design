# Sky130 CMOS Transmission Gate Simulation & Optimization

This repository documents the iterative design, debugging, and sizing optimization of a CMOS Transmission Gate (TG) using the **SkyWater 130nm PDK** and **ngspice**. 

---

## 📋 Project Journey Overview

| Step | Objective / Issue Encountered | Root Cause | Resolution |
| :--- | :--- | :--- | :--- |
| **1** | Initialize `.spice` netlist & run simulation | `Error: unknown subckt: x1.xm1` | Corrected instance prefix from `XM` to `X` (Sky130 devices are subcircuits). |
| **2** | Handle missing model libraries | `Error: Could not find include file...` | Restored the stable `.lib` wrapper path matching the design environment. |
| **3** | Characterize $R_{on}$ (First Success) | Massive $12\text{ k}\Omega$ resistance bottleneck at $1.0\text{V}$ | Re-sized NMOS and PMOS widths to balance mobility differences and flatten $R_{on}$. |

---

## 🛠 Step 1: Initial Implementation & Netlist Syntax Fix

### The Prompt
> *How do I simulate a Sky130 CMOS Transmission Gate testbench in ngspice?*

### The Initial Netlist (`new_circuit.spice`)
```spice
* SKY130 CMOS Transmission Gate
.subckt tg_switch in out sel sel_b vdd vss
XM1 out sel in vss sky130_fd_pr__nfet_01v8 w=1 l=0.15
XM2 in sel_b out vdd sky130_fd_pr__pfet_01v8 w=2 l=0.15
.ends

* Testbench
Vdd vdd 0 dc 1.8V
Vss vss 0 dc 0V
Vsel sel 0 dc 1.8V
Vsel_b sel_b 0 dc 0V
Vin in 0 dc 0V
Iload out 0 1uA

X1 in out sel sel_b vdd vss tg_switch

.control
dc Vin 0 3.3 0.01
let ron = (v(in) - v(out)) / 1u
plot ron
.endc
```

###
The ProblemRunning the file threw the following fatal error:BashError: unknown subckt: x1.xm1 out sel in vss sky130_fd_pr__nfet_01v8 w=1 l=0.15
Simulation interrupted due to error!
The FixIn standard SPICE, M denotes a raw MOSFET card, while X denotes a subcircuit call. Because SkyWater 130nm models transistors as complex subcircuits (including parasitic diodes and geometry configurations), they must be instantiated using the X prefix. Combining them into XM causes the parser to fail.📚 Step 2: Library Path & Component MappingThe PromptThe parser error is gone, but now ngspice throws a model warning and can't find sky130_fd_pr__nfet_01v8.The ProblemAttempting to include the raw .model.spice files directly led to missing file references because the internal files rely on relative paths mapped inside the master wrapper:BashError: Could not find include file /.../sky130_fd_pr__nfet_01v8.model.spice
The FixReverted to using the unified .lib file with the specific corner indicator (tt), ensuring the subcircuit calls inside tg_switch matched the definitions provided by the PDK wrapper.📈 Step 3: $R_{on}$ Characteristic Peak AnalysisThe PromptThe simulation successfully executed, but the $R_{on}$ plot shows a sharp peak of $12\text{ k}\Omega$ around $1.0\text{V}$. How do I flatten the curve?The ProblemAt a mid-rail voltage ($\sim 1.0\text{V}$), both the NMOS and PMOS enter weak inversion/turn off because their gate-to-source voltages ($V_{gs}$) approach their respective threshold voltages ($V_{th}$). Since holes have much lower mobility than electrons ($\mu_n \approx 2.5\times \mu_p$), the initial sizing ($W_p = 2$, $W_n = 1$) was insufficient to bridge the resistance bottleneck.The Final, Optimized Netlist (new_circuit.spice)By scaling up the device widths—especially the PMOS—the overall parallel resistance drops significantly, yielding a highly efficient, flatter analog switch profile.Code snippet* SKY130 CMOS Transmission Gate - Optimized Sizing
```
.subckt tg_switch in out sel sel_b vdd vss
* Upsized widths to minimize the parallel resistance bottleneck
X1 out sel in vss sky130_fd_pr__nfet_01v8 w=5 l=0.15
X2 in sel_b out vdd sky130_fd_pr__pfet_01v8 w=15 l=0.15
.ends

* Testbench Configuration
Vdd vdd 0 dc 1.8
Vss vss 0 dc 0
Vsel sel 0 dc 1.8
Vsel_b sel_b 0 dc 0
Vin in 0 dc 0
Iload out 0 1u

* Instance
X1 in out sel sel_b vdd vss tg_switch

* PDK Model Library Wrapper
.lib "/workspaces/vsd-7nm/avsddac_3v3_sky130_v1/sky130_fd_pr/models/sky130.lib.spice" tt

* Analysis Control Block
.control
dc Vin 0 1.8 0.01
let ron = (v(in) - v(out)) / 1u
plot ron
.endc
.end
```
🏁 ResultsBefore Optimization: $R_{on}$ peak value $\approx 12\text{ k}\Omega$ at $1.0\text{V}$.After Optimization: The upsized $W_n=5\mu\text{m} / W_p=15\mu\text{m}$ configuration successfully flattens the parallel resistance across the full $0\text{V}$ to $1.8\text{V}$ operating range, stabilizing performance for analog-to-digital data conversions.
