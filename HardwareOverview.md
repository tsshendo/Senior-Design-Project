# Three-Phase Active Rectifier

A custom 4-layer PCB and embedded firmware that replaces a passive three-phase diode bridge with a synchronous (active) rectifier for the Pack Motorsports Formula SAE internal-combustion car. Designed to recover the alternator's charge deficit without modifying the engine, alternator, or battery pack.

**Role:** Power-stage hardware lead: schematic capture, MOSFET selection, gate drive design, layout, and PFC firmware on Infineon TLE9893. <br>
**Tools:** Altium Designer, Infineon Config Wizard, embedded C, oscilloscope / electronic load for bring-up.

---

## Project Snapshot

| Spec | Value |
|---|---|
| Topology | 3-phase synchronous active rectifier (6 N-MOSFETs, 3 half-bridges) |
| AC Input | 60 Vrms line–line (170 Vp-p), ±3.5 A per phase |
| DC Output | ~85 V, 4.8 A max (≈400 W) |
| Target Efficiency | 97% (vs. ~75–80% on the OEM passive bridge) |
| Control MCU | Infineon TLE98932QKW62SXUMA1 (ARM Cortex-M3, "Embedded Power") |
| Gate Drivers | 3× LTC4444 half-bridge bootstrap drivers |
| PCB | 4-layer, mixed-signal, Altium Designer |
| Comms | CAN @ 1 Mbps, SWD debug |

---

## Problem (Situation & Task)

The ICE car ran a charge deficit during endurance events. The alternator is integrated into the engine (not replaceable) and the battery couldn't grow due to packaging constraints. Investigation traced the bottleneck to the off-the-shelf passive rectifier that measured efficiency of only 75–80%. Commercial high-efficiency replacements existed but were outside the team's budget.

**Goal:** Design a custom 3-phase active rectifier delivering ~400 W at 97% efficiency, in a sealed, vibration-tolerant package suitable for an FSAE car, using only PCB-level components.

---

## Hardware Design

### Topology

Three half-bridges (one per alternator phase), each built from a high-side and low-side N-channel MOSFET. The MCU drives complementary PWM signals through bootstrap gate drivers, synchronously commutating each phase to its output rail and using the MOSFET channel resistance instead of a diode drop. This is what unlocks the efficiency gain by replacing two diode V_F drops per cycle (~1.4 V × I) with two channel I·R drops (~2 × 2.1 mΩ × I).

### MOSFET Selection

I built a comparison spreadsheet that estimated total per-device power loss across six loss mechanisms: conduction, switching (turn-on + turn-off), reverse recovery, dead-time conduction, gate-charge, and C_oss loss. Inputs were datasheet values for V_DS, R_DS(on), Q_g, t_r/t_f, Q_rr, V_F, C_oss. I evaluated five candidates spanning Infineon, onsemi, and Alpha & Omega:

| Part | R_DS(on) | Q_g | Total Loss @ 5 A | Cost |
|---|---|---|---|---|
| Infineon IAUT300N10S5N014ATMA1 | 1.4 mΩ | 166 nC | 90.9 mW | $6.09 |
| onsemi NVBLS1D5N10MCTXG | 1.5 mΩ | 131 nC | 133.6 mW | $6.57 |
| Infineon IRF100P218AKMA1 (TH) | 1.28 mΩ | 330 nC | 151.4 mW | $9.21 |
| Alpha & Omega AOTL66912Q | 1.7 mΩ | 155 nC | 119.4 mW | $8.58 |
| **Infineon IAUCN10S7N021ATMA1** | **2.1 mΩ** | **81 nC** | **85.8 mW** | **$3.52** |

**Chosen: IAUCN10S7N021ATMA1** (OptiMOS 5, 100 V, 220 A, TDSON-8 SMD). It won on both total loss and cost, the lower R_DS(on) of the alternatives was outweighed by their much higher Q_g and switching losses at the operating point. Lower Q_g also reduces gate-driver current demand and bootstrap cap size.

### Gate Driver Design

Each phase uses an **Analog Devices LTC4444EMS8E** half-bridge bootstrap driver in MSOP-8:
- 2.5 A peak gate current: fast edges into ~10 nC of gate charge
- High-side floats up to ~80 V above ground via bootstrap rail
- Bootstrap diode: **SBR1U150SA-13** Schottky (150 V, low V_F, fast recovery): chosen for low forward drop to maximize V_GS headroom on the high-side
- Bootstrap cap C_BOOT = 100 nF, sized per the LTC4444 application rule **C_BOOT ≥ 10 · Q_g / (V_CC − V_diode)**
- Gate resistors **RGH/RGL = 10 Ω** (Vishay Dale RCA0805, 0.5 W) on each gate: tunable knob to trade switching loss for ringing/EMI during bring-up
- 1 µF ceramic right at the driver V_CC pin for instantaneous gate charge supply
- Driver V_CC fed from a filtered "VS" rail derived from 12 V through a BAS5202 Schottky + RC filter, isolating the driver supply from MCU/sensor noise

### MCU & Control

**Infineon TLE98932QKW62SXUMA1** ("Embedded Power" SoC): ARM Cortex-M3 with integrated 5 V LDO (VDDP), 1.5 V LDO (VDDC), CAN PHY, and dedicated motor-control PWM peripheral. Selected because the integrated rails dramatically simplified the BoM and the CCU7 PWM unit natively generates complementary, dead-time-controlled outputs across three channel pairs, exactly what a 3-phase bridge needs.

- 16 MHz crystal (NDK NX3225GA) with 12 pF load caps + 100 Ω damping resistor per Infineon's HW guideline
- PWM: `CCU7.CC7[0/1/2]` → high-side, `CCU7.COUT7[0/1/2]` → complementary low-side, with hardware dead-time insertion
- External SPI ADC (TI ADC122S051, 12-bit) on the rectified-output and output-current channels for headroom beyond the SDADC inputs
- SWD debug header for in-circuit programming via XMC Link
- The internal CSA was bypassed (the ±2 V common-mode limit didn't suit either high- or low-side shunt placement at our bus voltage), so SL is tied to GND and external isolated current sensors are used instead

### Sensing Subsystem

**Phase voltage sense (×3) + V_OUT sense:** ADA4807-4 quad op-amp configured as a difference amplifier with a 180 kΩ / 5 kΩ input divider biased at 2.5 V (LM4040 shunt reference). Maps ±85 V phase swing → 0–5 V single-ended for the MCU SDADC. RC roll-off set for ~160 kHz cutoff to suppress switching-edge pickup without phase-shifting the fundamental.

**Current sense (3× phase + 1× output):** TI **TMCS1107A4BQDRQ1** (AEC-Q100, ±420 V isolated Hall-effect, 50 mV/A) on each phase, plus a TMCS1107A4UQDR on the output rail. Chosen over shunts to keep the current loop fully isolated from the high-current power path and avoid burning a sense resistor.

**Output filter:** 4× **United Chemi-Con EMHS 110 µF / 100 V** aluminum electrolytics in parallel, sized from the standard 3-phase ripple equation V_ripple = I_load / (6π · f · C) for the worst-case alternator electrical frequency.

### Protection & Power Tree

- 12 V (vehicle rail) → TPS7B6950 automotive LDO → 5 V analog/sensor rail
- 12 V → BAS5202 Schottky + LC filter → VS rail (gate drivers, MCU power input): protects against reverse polarity and decouples switching transients from the analog supplies
- LM4040 2.5 V shunt reference → midpoint bias for the bipolar voltage-sense front end
- TDK ACT1210 common-mode choke + 4.7 nF cap on the CAN bus for EMC, with DNP-able 60.4 Ω + 60.4 Ω split termination
- Power-rail status LEDs on 12 V, 5 V, and "5 V good" diagnostic for bring-up

### Power Budget

Worked the rail-by-rail current draw to size the LDOs and validate thermal headroom:

| Rail | Total Current | Total Power |
|---|---|---|
| 12 V (input) | 145.8 mA | 1.75 W |
| 5 V (sensors + MCU analog) | 17.4 mA | 163.5 mW |

The TPS7B6950 (150 mA-class) carries 35 mA expected vs. 17 mA budgeted.

---

## PCB Layout (4-Layer)

**Stackup (top → bottom):**

| Layer | Function | Notes |
|---|---|---|
| L1: Top (signal + components) | Component placement, high-current power routing, sense front end | All ICs and the three half-bridges are on this layer. MCU on the right, three vertically-stacked half-bridges on the left, output bulk caps and connectors on the far left. |
| L2: Solid GND plane | Continuous reference for every signal on L1 and L4 | Kept as an uninterrupted pour except for a differential CAN pair routed through |
| L3: Power / mid signal | Phase pours, V_DH bus, local power routing for the half-bridges | Carries the high-current rectified output and per-phase nodes between the bulk caps and the half-bridges, freeing L1 for component-side routing. |
| L4: Bottom (signal) | PWM and low-voltage signal fan-out from MCU to gate drivers | All six PWM gate-drive signals (PWM_H1..3, PWM_L1..3) are routed here as a bus from the MCU on the right to the three drivers on the left, sandwiched between the L2 GND plane and the bottom-side reference. |

This stackup gives every fast-edge signal like PWM gate drives on L4, sense lines on L1, an unbroken ground reference one layer away, which keeps loop area (and therefore radiated/conducted EMI) low.

Layout decisions worth calling out:

- **Tight gate-drive loops.** Each LTC4444 sits directly adjacent to its half-bridge MOSFET pair on L1, with V_CC bypass cap (1 µF) and bootstrap cap (100 nF) placed within a few mm of the driver. The HS_Gate / LS_Gate nets are kept on the top layer and short gate-loop inductance is the main driver of V_GS ringing and overshoot, so this is the highest-leverage layout decision.
- **PWM signal bus on bottom layer.** The six PWM gate-drive signals run from the MCU on the right to the three gate drivers on the left as a clean parallel bus on L4, fully referenced to the L2 GND plane directly above them. Visible in the bottom-layer image as the diagonal trace bundle.
- **Three half-bridges identically laid out.** Each phase uses the same MOSFET-pair → driver → bootstrap geometry (visible as three identical motifs stacked vertically on L1 / L3). Repeating the layout makes per phase parasitics match, which matters for the 3-phase PFC loop convergence.
- **Power loop minimization.** High-side drain → phase node → low-side source loop kept short and wide. The IAUCN10S7N021's TDSON-8 footprint exposes 5 drain pins and 4 source pins, both flooded into copper pours sized for the steady state RMS current with thermal headroom, with stitching vias down to the L3 power pour.
- **Solid ground reference (L2).** L2 is intentionally kept as an unbroken plane, only one CAN Differential pair is routed on it. This prevents return-current discontinuities.
- **Bulk caps near the bridge.** The four 110 µF / 100 V output caps are placed at the left edge between the harness connector and the half bridges so the high-current commutation loop from each MOSFET pair into the bulk capacitance is short.
- **Sense-line routing.** Voltage- and current-sense traces routed as short differential runs over the GND plane, kept away from the switching phase pours, with the 2.5 V LM4040 reference buffered locally at the sense front end.
- **Test-point coverage.** Every critical net (each phase, V_DH, each I_sense, each gate-drive output, every ground-return island) has a Keystone 5019 turret test point on top.
- **Mechanical.** Square outline with M4 mounting holes in all four corners (visible as the cyan annular pads), sealed Amphenol AT13 12-pin harness connector and a 10-pin Samtec FTSH SWD header on the far left.

---

## Firmware: Power Factor Correction

I wrote bare-metal C for the TLE9893 to implement classic FOC-style PFC on the rectifier:

1. **Sample** phase voltages and currents synchronously with PWM zero-crossings (SDADC + SPI ADC).
2. **Clarke transform**: collapse the 3-phase currents to stationary α-β.
3. **Park transform**: rotate α-β into the d-q frame aligned with the voltage vector. The d-component represents real (in-phase) current, q represents reactive.
4. **Two PI controllers** drive q → 0 (unity power factor) and d → reference (commanded power).
5. **Inverse Park + Clarke** → reference back to 3-phase voltages.
6. **Space-Vector PWM (SVPWM)** generates the gate signals for CCU7, with hardware dead-time inserted between complementary outputs.

The transform-based loop runs in the PWM ISR. PI gains were tuned bench-side using current step responses captured on the test points.

---

## Bring-Up & Test Plan

1. **Power-on without bridge**: populate only the LDO, MCU, and sense circuitry; verify 5 V, 2.5 V ref, MCU programs over SWD.
2. **Gate-drive verification**: populate one half-bridge, drive PWM at low-voltage bench supply (12 V VS, no DC bus), confirm clean V_GS edges and dead-time on a scope.
3. **Single-phase soak**: bench AC source on one phase into a resistive load, validate sense-channel scaling and rectification.
4. **3-phase low-power**: bench 3-phase source, ramp output power, confirm PI loops converge.
5. **Vehicle integration**: install on car, instrumented run on dyno before track use.

---

## Outcome

A working prototype 3-phase active rectifier that addressed the FSAE car's charge deficit at a fraction of the cost of a COTS solution with no mechanical changes to the engine, alternator, or battery. The hardware was designed to be sealed, vibration tolerant, and serviceable, with extensive test-point access for bring-up and field diagnostics.
