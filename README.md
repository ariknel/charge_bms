# 4S Lithium Battery System — Engineering Design & Build Reference

**BMS Protection Board · USB-C PD Charger Board**  
For: Bluetooth Speaker / TAS5825M Amplifier Platform  
*Revision 3.0 · June 2026 · Arik Nel*

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [BMS Protection Board](#2-bms-protection-board)
3. [Charger Board](#3-charger-board)
4. [PCB Layer Count Decision](#4-pcb-layer-count-decision)
5. [Thermal Management & Heatsink](#5-thermal-management--heatsink)
6. [PCB Ordering & Build Procedure](#6-pcb-ordering--build-procedure)
7. [Test & Verification Procedure](#7-test--verification-procedure)
8. [System Integration Notes](#8-system-integration-notes)
9. [Key Design Decisions Reference](#9-key-design-decisions-reference)

---

## 1. System Overview

### 1.1 Power Architecture Summary

| Stage | Function | Key Spec |
|---|---|---|
| USB-C PD Input | Negotiate 9V / 12V / 15V / 20V from any adapter | 5–20V input, PD2.0 / PD3.0 / QC3.0 / QC3+ |
| IP2368 Buck-Boost | Convert PD voltage → 16.8V CC/CV charge, or discharge pack → USB-C PD output | Up to 65W bidirectional, 4-switch synchronous BB |
| xP4S Li-ion Pack | 14.8V nom. / 16.8V full / 12.0V cutoff | 18650 cells, ≥2P per series group |
| S-8254AA BMS | Cell-group OVP, UVP, OCP, SCP, temp cutoff | 4S, ±20mV accuracy, hardware-only, no MCU |
| 3× ETA3000 Balancer | Inductive active balancing between adjacent series groups | Autonomous, up to 2A, 92% efficiency |
| Si2301 Isolation FET | Prevents battery back-feed into IP2368 when IC is unpowered | P-ch SOT-23, gate driven by IC VCC rail |
| TAS5825M Amp | Receives protected PVDD from BMS P+/P− | 16.8V → ~2×25W @ 4Ω, >90% efficiency |

### 1.2 TAS5825M Power Output at 4S

PVDD range: 4.5V–26.4V. At 4S full charge (16.8V PVDD):

- **BTL stereo, 4Ω load:** ~2×25W continuous (THD+N <1%), peaks to ~2×32W on transients
- **BTL stereo, 8Ω load:** ~2×17W continuous (THD+N <1%)
- **PBTL mono, 4Ω:** ~50W peak (both channels bridged to one driver)
- **Total system peak draw:** ~55–60W (both channels + DSP quiescent + BMS overhead)

> **Note:** TI rates 2×38W at 24V/8Ω. At 16.8V on 4Ω you get ~2×25W usable — excellent for a portable sealed enclosure. A dedicated boost stage to 20–24V PVDD is possible but significantly complicates the architecture.

### 1.3 Cell Configuration: xP4S with 18650

You are using **18650 cells, minimum 2 parallel per series group** — written as **2P4S** (or 3P4S, 4P4S etc. as you scale). This has important implications throughout the design:

**Parallel groups self-balance physically.** Within a single parallel group, both cells share the exact same positive and negative terminal. Any voltage difference between them causes an immediate equalisation current through their internal resistance — no balancing circuit needed within a group. The group behaves as a single high-capacity cell from every IC's perspective.

**The BMS sees exactly 4 voltages regardless of P count.** The S-8254AA VM1–VM4 sense lines connect to the four series node tap points. Whether there are 2, 4, or 8 cells in each group, the IC measures one group voltage per input. Checking each individual cell in a parallel group is unnecessary and architecturally wrong.

**Current capacity scales with P.** Each Samsung 30Q 18650 has a 15A continuous discharge rating and ~2A recommended charge current:

| Parameter | 1P | 2P | 3P | 4P |
|---|---|---|---|---|
| Capacity (30Q) | 3000 mAh | 6000 mAh | 9000 mAh | 12000 mAh |
| Max cont. discharge | 15A | 30A | 45A | 60A |
| Rec. charge current | 2A | 4A | 6A | 8A |
| Runtime @ 6A avg load | ~30 min | ~60 min | ~90 min | ~120 min |

For a speaker drawing ~3–6A average, 2P4S gives ~1+ hour. 3P4S or 4P4S extend this proportionally with no circuit complexity change.

**Sense resistor and OCP:** the load current bottleneck is always the TAS5825M (~12A peak), not the cells regardless of P count. The sense resistor value therefore stays constant across all P configurations. See §2.5.

### 1.4 Why Two Boards?

- **Switching noise isolation:** The IP2368 switches at 250kHz. That switching noise coupled into the BMS VM1–VM4 sense lines would cause false OVP/UVP trips — the S-8254AA comparators resolve ±20mV. Physical separation is the cleanest fix.
- **Thermal isolation:** IP2368 dissipates 2–5W during full-power operation. Keeping that heat source away from the battery cells and BMS analog circuitry improves protection accuracy and cell lifetime.
- **Modularity:** BMS stays permanently in the speaker. Charger connects only when charging — protection is always active regardless of charger state. A charger board fault cannot directly damage the BMS.
- **Independent fault domains:** Simpler debugging. Each board can be tested and replaced independently.

### 1.5 Why Two FETs — Not Four, Six, or Eight

Commercial BMS boards often use 4, 6, or 8 FETs. This confuses people. The complete explanation:

**The topology is always exactly two switches: one CHG FET, one DSG FET, back-to-back on the negative rail.** This fundamental BMS architecture never changes regardless of total FET count.

**Multiple FETs per switch = FETs in parallel to share current.** A 40A BMS uses 2× FETs per switch (4 total) because at 40A a single SO-8 device would overheat. Paralleling two 20mΩ FETs gives an effective 10mΩ and halves the thermal load per device.

**This design — calculated:**

- TAS5825M peak draw: ~12A continuous from pack
- AO4407A Rds(on) = 8mΩ max @ Vgs = 4.5V
- P_dissipated = I² × Rds_total = 144 × 0.016 = **2.3W total across both FETs**
- SO-8 with PCB copper pour, θJA ≈ 40°C/W per device
- Each FET dissipates ~1.15W → ΔT = 1.15 × 40 = **46°C rise** → FETs at ~71°C at 25°C ambient ✅

At 3P4S or 4P4S the available current rises, but the amplifier is still the bottleneck. Two FETs remains correct.

**When you would need more:** loads drawing >20A continuously — power tools, e-bikes, high-power inverters. For a Bluetooth speaker: two FETs is the right, correctly-sized answer.

---

## 2. BMS Protection Board

### 2.1 IC Selection: ABLIC S-8254AA

The S-8254AA is the correct IC for a custom 4S BMS at this power level. Full hardware protection, no firmware, 16-pin TSSOP package.

**Feature set:**

- Native 3S/4S support via SEL pin — tie high (VDD) for 4S, no external logic needed
- Per-cell-group independent voltage monitoring on VM1–VM4 — each input is its own comparator, not averaged
- **Separate COUT and DOUT gate drive outputs** — CHG and DSG FETs controlled independently. Over-discharge cuts discharge path only, leaving charge path available for recovery. Over-charge cuts charge path only, leaving discharge available. This asymmetric response is critical for safe recovery from fault conditions.
- **Three-tier overcurrent protection:**
  - OCD1: soft overcurrent, delay programmed by external capacitor (CTD pin) — accommodates speaker transients
  - OCD2: hard overcurrent, fixed ~8ms internal delay — catches sustained overload
  - SCD: short-circuit detection, fixed ~200µs response, latching — immediate protection, requires load disconnect to reset
- **NTC temperature input (TM pin):** inhibits charging independently of voltage state when cell temperature exceeds threshold
- **±20mV OVP accuracy** (AA suffix) vs ±50mV on the original A suffix — important for tight cell protection at 4.2V top-of-charge
- ~3µA standby quiescent current drawn from pack — negligible self-discharge

### 2.2 Complete Protection Feature Map

Every protection event, its source, which FET it cuts, and its recovery mechanism:

| Fault | Detection | Threshold | Response | Latching? | Recovery |
|---|---|---|---|---|---|
| Cell group overvoltage (OVP) | VM1–VM4 vs internal ref | 4.275V ±20mV | COUT low → CHG FET off | No | Auto when cell drops to 4.175V (100mV hysteresis) |
| Cell group undervoltage (UVP) | VM1–VM4 vs internal ref | 2.500V ±50mV | DOUT low → DSG FET off | No | Auto when cell recovers above 2.700V (200mV hysteresis) |
| Soft overcurrent OCD1 | CS+/CS− across R1 shunt | ~18A @ 8mΩ | DOUT low after C3 delay (~16ms) | No | Auto when current drops below threshold |
| Hard overcurrent OCD2 | CS+/CS− across R1 shunt | ~18A sustained | DOUT low after fixed ~8ms | No | Auto when current drops below threshold |
| Short circuit SCD | CS+/CS− spike | >5× Vcs (~90A spike) | DOUT low within ~200µs | **Yes** | Must disconnect load to reset — intentional |
| Overtemperature (OTP) | NTC thermistor on TM pin | ~45°C (R_NTC divider sets threshold) | COUT low → charge inhibit | No | Auto when temp drops 5°C below threshold |
| Discharge overtemp | NTC on TM pin | ~60°C configurable | DOUT low → discharge inhibit | No | Auto recovery |

> **Note on SCD latching:** Short-circuit detection is deliberately latching. A dead short across the output (e.g. shorted XT30 connector, wiring fault) could cause continuous high-current arcing if auto-recovery were allowed. The latch forces a deliberate human intervention — disconnect the load, investigate the fault, reconnect.

> **Note on OVP/UVP asymmetry:** OVP cuts CHG only — you can still use the pack normally while a cell is slightly over-voltage, the charger just stops. UVP cuts DSG only — a deeply discharged pack cannot power the amplifier but can still receive a recovery charge. This is the correct safe behaviour in both cases.

### 2.3 Active Balancing: ETA3000 (×3)

#### Why Active Balancing Over Passive

Passive balancing burns excess cell energy as heat through a bleed resistor. At 100mV imbalance on a 3Ah cell group that represents ~70mAh of wasted energy per cycle, the bleed resistor dissipates ~300mW during balancing, and passive balancing only acts at top-of-charge — it cannot redistribute charge during discharge.

Active (inductive) balancing uses an inductor as an energy shuttle. The higher-voltage cell charges the inductor; that stored energy is then delivered to the lower-voltage cell. Net result: charge moves between cells with 85–92% transfer efficiency, nothing is wasted as heat, and balancing acts continuously — both during charging and discharging.

For an 18650 xP4S pack this matters because:
- 18650 cells from different lots develop capacity drift over cycles
- A well-balanced pack holds rated capacity significantly longer — Arrhenius equation: every 10°C of unnecessary heat halves component life; same principle applies to cycle life under electrical imbalance
- The TAS5825M draws large transient currents, causing uneven voltage sag across groups if capacities differ
- Active balancing is the technically superior, more complex choice — appropriate for a portfolio project

#### ETA3000 — Autonomous Inductive Balancer

The ETA3000 requires no MCU. Three instances are used, one spanning each adjacent series-group pair:

```
ETA3000 #1:  B− node  ↔  B1 tap   →  balances Group 1 ↔ Group 2
ETA3000 #2:  B1 tap   ↔  B2 tap   →  balances Group 2 ↔ Group 3
ETA3000 #3:  B2 tap   ↔  B3/B+ tap →  balances Group 3 ↔ Group 4
```

Each instance operates fully independently and simultaneously.

**Key specs:**
- Bidirectional inductive switching at 1MHz
- Up to 2A balancing current (set by R_ISET: 27kΩ → ~2A, 51kΩ → ~1A)
- 92% charge transfer efficiency
- Activates automatically when ΔV between its two cell groups exceeds ~30mV
- Settles to within 30mV accuracy
- 2µA standby per IC — three instances draw 6µA total at rest
- Precondition mode: reduces balance current when one cell is deeply undervoltage to avoid stress
- SOT23-6 package — hand solderable

**Why not LT8584 or LTC3300?** The LT8584 requires one custom flyback transformer per cell and an LTC680x host IC with SPI. The LTC3300 is bidirectional but requires external SPI control from an MCU and also uses coupled inductors. Both are automotive-grade overkill that add substantial BOM cost, PCB area, and firmware complexity for no meaningful benefit at 4S with a moderate charge rate. The ETA3000 is the correct tool for this scale.

### 2.4 MOSFET Selection: AO4407A

Two AO4407A devices, back-to-back on the negative rail (B−/P− path), source-to-source.

- **Vds:** 30V — above all 4S operational voltages including charger transients
- **Rds(on):** 8mΩ max @ Vgs = 4.5V — very low for this current level
- **Id:** 15A continuous, 60A pulsed
- **Package:** SO-8 with exposed drain pad for PCB copper thermal spreading
- **Gate drive:** directly from S-8254AA COUT/DOUT pins — no external gate driver needed at ≤12A

**Back-to-back orientation (source-to-source):** body diodes face outward in both directions. When both FETs are on, current flows through the channel (low resistance). When either FET is off, its body diode blocks the relevant current direction. This gives independent CHG/DSG control from a single pair of N-channel devices — the universal BMS topology.

> ⛔ **CRITICAL — Body diode orientation:** Q1 (CHG FET): drain toward battery B−, source toward shared node. Q2 (DSG FET): drain toward load P−, source toward shared node. Inverting either FET defeats the protection entirely — the body diode conducts in the wrong direction and the switch does nothing. Verify against the S-8254AA reference schematic before routing a single trace.

### 2.5 BMS Board — Full BOM

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | BMS Protection IC | ABLIC S-8254AA | 16-TSSOP | 1 | SEL → VDD = 4S. Per-cell-group OVP/UVP/OCP/SCP. Independent COUT + DOUT. ±20mV accuracy. |
| U2–U4 | Active Balancer IC | ETA3000 ×3 | SOT23-6 | 3 | One per adjacent cell-group pair. Autonomous 1MHz inductive balancing. LCSC: C7465544. |
| Q1 | Charge FET | AO4407A 30V 15A 8mΩ | SO-8 | 1 | CHG switch. Drain → B−, source → shared node. Gate from COUT via R3. |
| Q2 | Discharge FET | AO4407A 30V 15A 8mΩ | SO-8 | 1 | DSG switch. Drain → P−, source → shared node. Gate from DOUT via R4. |
| Q3 | Pre-discharge FET | 2N7002 60V 300mA | SOT-23 | 1 | Optional. Soft-starts PVDD bulk caps via R2 before Q2 fully opens. Prevents inrush SCP nuisance trip. |
| R1 | Discharge sense shunt | 8mΩ ±1% 3W | 2512 | 1 | OCP sense. Kelvin 4-wire to CS+/CS−. 8mΩ × 18A = 144mV trip. Constant value regardless of P count — amp is current bottleneck. |
| R2 | Pre-discharge resistor | 10Ω 1W | 2512 | 1 | In series with Q3. Limits inrush during pre-discharge phase. |
| R3, R4 | Gate resistors Q1/Q2 | 1kΩ | 0402 | 2 | Damp gate drive edges. Prevent dV/dt ringing on FET transitions. |
| R5 | Q3 gate pull-down | 100kΩ | 0402 | 1 | Ensures Q3 is off when control line is floating or unpowered. |
| R6–R9 | Cell sense RC — resistors | 100Ω each | 0402 | 4 | One per VM1–VM4. Forms LPF with C6–C9. Rejects 250kHz–1MHz switching noise from charger and ETA3000. |
| R10–R12 | ETA3000 R_ISET | 27kΩ each | 0402 | 3 | One per ETA3000. Sets ~2A balance current. Increase to 51kΩ for ~1A if thermal concerns arise. |
| C1 | CCT delay cap | 0.1µF X7R | 0402 | 1 | OVP detection delay ~1s. See S-8254AA Table 4 for exact timing. |
| C2 | CDT delay cap | 0.1µF X7R | 0402 | 1 | UVP detection delay. |
| C3 | CTD delay cap | 0.22µF X7R | 0402 | 1 | OCD1 delay ~16ms. Accommodates TAS5825M bass transients. Increase to 0.47µF if nuisance trips occur. |
| C4 | VBAT bulk bypass | 100µF 25V electrolytic | SMD 6.3×5.4 | 1 | Decouples B+ rail during amplifier load transients. Reduces dynamic voltage sag reaching BMS comparators. |
| C5 | VDD bypass | 100nF X7R | 0402 | 1 | IC VDD local decoupling. Place within 1mm of pin. |
| C6–C9 | Cell sense RC — caps | 0.1µF X7R each | 0402 | 4 | One per VM tap. With R6–R9: LPF cutoff ~16kHz, rejects PWM from both charger (250kHz) and ETA3000 (1MHz). |
| C10 | Gate drive bypass | 10nF X7R | 0402 | 1 | Local bypass on COUT/DOUT lines. Stabilises gate drive voltage at switching edges. |
| C11–C13 | ETA3000 bypass | 100nF + 10nF per IC | 0402 | 6 | BATP and BIAS pin decoupling per ETA3000 datasheet. Place within 0.5mm of pins. |
| L1–L3 | ETA3000 inductors | 4.7µH Isat ≥3A **shielded** | 4×4mm SMD | 3 | One per ETA3000. **Shielded mandatory** — unshielded core at 1MHz radiates directly into adjacent VM sense lines and causes false protection trips. |
| NTC1 | Battery NTC | 100kΩ B=3950 (Vishay NTCLE100E3104) | 0402 / leaded | 1 | Glued to cell body with thermal epoxy. Routes to TM pin. Inhibits charge above ~45°C, discharge above ~60°C. |
| NTC2 | Ambient/FET NTC | 100kΩ B=3950 | 0402 | 1 | Optional. Mounted near Q1/Q2. Distinguishes FET thermal event from cell thermal event during fault analysis. |
| D1 | Reverse polarity Schottky | SS34 40V 3A | SMA | 1 | On B+ line from battery. Blocks reverse pack connection. Prevents charger back-feed to battery negative during unusual fault states. |
| J1 | Battery tap connector | 5-pin JST-XH 2.54mm | THT/SMD | 1 | Pins: B−, B1, B2, B3, B+. Keyed — physically prevents reversed insertion. |
| J2 | Load output | XT30U female right-angle | PCB pads | 1 | P+/P− to TAS5825M PVDD. 30A rated gold contacts. |
| J3 | Charger input | XT30U female | PCB pads | 1 | CHG+/CHG− from charger board. Separate connector from load. |
| J4 | Debug header | 4-pin 2.54mm | THT | 1 | FAULT / TM_OUT / SDA / GND. Bench monitoring during development. |
| LED1 | Charge path indicator | Green 0402 | 0402 | 1 | Via 1kΩ series R from COUT. On = CHG FET active. Off = OVP or OTP tripped. |
| LED2 | Fault indicator | Red 0402 | 0402 | 1 | Combined COUT∧DOUT = LOW state. On = any protection event active. |

### 2.6 Sense Resistor — Value and Justification

The S-8254AA OCD2 threshold is a fixed internal voltage across the CS pins. For the standard variant this is Vcs = 0.150V (150mV). Therefore:

```
R1 = Vcs / I_trip = 0.150V / I_trip
I_trip target = ~150% of maximum expected continuous current
Maximum continuous current = ~12A (TAS5825M bottleneck, all P configs)
I_trip = 12A × 1.5 = 18A
R1 = 0.150 / 18 = 8.3mΩ → use 8mΩ standard value
```

Power at trip current: P = I² × R = 324 × 0.008 = 2.6W → 3W rated 2512 shunt required.

This value is constant across 2P4S, 3P4S, and 4P4S because the amplifier, not the cells, sets the current ceiling.

### 2.7 Protection Thresholds — Configuration Summary

| Parameter | Threshold | Hysteresis | Config method |
|---|---|---|---|
| OVP per cell group | 4.275V ±20mV | 100mV | Fixed internal — AA suffix |
| UVP per cell group | 2.500V ±50mV | 200mV | Fixed internal |
| OCD1 delay | ~16ms at 0.22µF | — | C3 value on CTD pin |
| OCD2 delay | ~8ms | — | Fixed internal |
| SCD response | ~200µs | — | Fixed internal, latching |
| OTP charge inhibit | ~45°C | 5°C | NTC1 resistor divider + TM pin |
| OTP discharge inhibit | ~60°C | 5°C | NTC1 resistor divider + TM pin |

### 2.8 BMS Layout Rules

#### High-Current Path

The path `B− → D1 → Q1 drain → Q1/Q2 source node → R1 shunt → Q2 drain → P−` carries full pack current at up to 12A continuous, 18A trip point.

- **Trace/pour width:** minimum 5mm at 1oz, 3.5mm at 2oz for 12A. Use copper polygon pours, not routed traces. Wider is always better — there is no penalty.
- **2oz copper:** order it. Halves trace resistance. On JLCPCB adds ~$3. Non-negotiable.
- **Vias on high-current path:** avoid if possible. If needed, cluster 6–8 vias (0.4mm drill) in parallel. Single vias on a 12A path are a thermal fuse.
- **Shunt R1:** verify cold-joint-free solder on both end pads. A cold joint on an 8mΩ shunt adds measurable parasitic resistance that throws off the OCP threshold directly.

#### Kelvin Sensing on R1 — The Most Common BMS Layout Mistake

```
WRONG:
  B− ──[fat pour]──[R1]──[fat pour]── Q2
                              └── CS+ (tapped partway along power trace)
              └── CS− (tapped partway along power trace)

CORRECT:
  B− ──[fat pour]──[R1 pad A]──[R1 pad B]──[fat pour]── Q2
                        │              │
                    CS− (thin)     CS+ (thin)
              (tapped directly at resistor end pads, nowhere else)
```

Every millimetre of fat power trace between the resistor pad and the sense tap adds ~0.5mΩ/mm of parasitic resistance to the sensed voltage. At 8mΩ nominal, a 5mm error is a 31% OCP threshold shift. This is not a refinement — it is a functional requirement.

Sense traces must be thin (0.15mm) signal-class routes going directly from the resistor pad to the IC CS pins. No power fills, no junctions, no shared paths.

#### Cell Sense Lines (VM1–VM4)

- Route the series RC filter close to the IC. Connection order: JST tap pin → R (100Ω) → IC VM pin, with C (0.1µF) from that VM pin directly to GND.
- Keep all VM traces away from ETA3000 inductors — minimum 3mm clearance. The ETA3000 switches at 1MHz; the inductor is the source of that field.
- Crossing a VM trace over the high-current power polygon is acceptable only if perpendicular (minimum coupling) and on the opposite layer with ground copper between.

#### ETA3000 Placement

- Place each instance physically adjacent to the two cell-tap nodes it spans.
- The balancing inductor goes directly between the IC SW pins and cell tap nodes — short, minimum-loop-area connections.
- Shielded inductors only. An unshielded 1MHz core is an antenna pointed at your precision sense lines.
- Minimum 3mm between any ETA3000 inductor body and any VM sense trace.

#### Ground Plane

- Single continuous ground plane on the bottom layer (or inner layer 3 on a 4-layer board). Do not split it. The S-8254AA analog ground and power return are the same net — splitting creates ground impedance that directly shifts comparator thresholds.
- All high-current return current flows through this plane. The plane provides the low-impedance return path that prevents ground bounce affecting the sense comparators.

---

## 3. Charger Board

### 3.1 IC Selection: Injoinic IP2368

The IP2368 is a fully bidirectional USB-C power management IC — it charges the battery from any PD adapter and discharges the battery to power any USB-C device (OTG). Both functions use the same connector, the same IC, and the same buck-boost converter running in opposite directions.

**Why this IC:**
- Integrated PD2.0/PD3.0/QC3.0/QC3+/PPS/FCP/AFC/SCP negotiation — CC1/CC2 lines connect directly to IC, no external FUSB302 needed
- 4-switch synchronous buck-boost with two external inductors — single topology for all input voltages 5V–20V to 16.8V output, and 16.8V → 5–20V in discharge direction
- Up to 65W charging from a 65W PD adapter (20V × 3.25A)
- Up to 65W OTG discharge output — can charge a laptop from the speaker pack
- Configurable 2–6S via VSET resistor divider — no firmware
- Built-in thermal derating, NTC input, UVLO, charge termination, coulometer, SOC display outputs
- **Try.SRC / DRP mode:** automatically negotiates its role (power source vs power sink) based on what is connected. Connect a PD adapter: it charges. Connect a phone: it discharges. No manual switching required.
- 250kHz switching frequency
- ~$1–2 from LCSC

> **IP2366 upgrade path:** If IP2368 is out of stock at order time, the IP2366 (PD3.1, 140W, 2–6S) is the same Injoinic family and pin-compatible for 4S with minor resistor changes. Strictly better, slightly more expensive.

### 3.2 Bidirectional Operation and OTG

The IP2368 handles role negotiation autonomously via the USB-C CC pins:

- **Adapter plugged in:** IC detects a power source, requests the highest available PD voltage, charges the battery
- **Phone/device plugged in:** IC detects a power sink, advertises available voltages as a PD source, discharges the battery to power the device
- **Nothing connected:** IC idles, <1mA standby, waiting

This means the speaker's USB-C port doubles as a power bank output. Plug your phone in while the speaker is sitting on a table — it charges. No extra circuitry, no user setting, it just works.

**What is explicitly forbidden:** connecting two bidirectional IP2368 boards together (e.g. speaker to another power bank). Both devices attempt to assert their role simultaneously, causing repeated negotiation conflict and potential damage to both. This is not a normal usage scenario for a speaker.

**COUT pin — discharge enable/disable:** The IP2368 has a COUT pin that enables or disables the USB-C discharge output function. Tying it high disables OTG output entirely. Tying to a solder jumper or GPIO gives user control. For a speaker application, leaving OTG enabled is a useful feature with no downside.

### 3.3 Reverse Current Protection: Si2301 Isolation FET

The specific failure mode that "blows up the IP2368" on poor designs:

The IP2368's internal 4-switch buck-boost has body diodes on all four switches. When the IC is unpowered (no USB-C connected, IC in deep sleep), those body diodes form a passive path from the battery BAT pin back through the converter to the VBUS pin. Under normal circumstances this is harmless because VBUS is floating. But under fault conditions — a cable with VBUS shorted to ground, a mis-connected device feeding 5V backwards, or even just a charged cable capacitance discharge — current can flow through those body diodes with no IC gate control. The IC is not designed to handle that.

**Fix: P-channel isolation FET between battery B+ and IC BAT pin.**

```
Battery B+ ──[Si2301 P-ch FET]── IP2368 BAT+
                  │
              Gate control:
              Gate tied to IC VCC via R (10kΩ)
              Gate pulled to source via R (100kΩ)

When IC powered:  VCC present → gate pulled low below source → FET ON  → normal operation
When IC off:      VCC = 0V → gate = source (via 100kΩ) → Vgs = 0V → FET OFF → battery isolated
```

The Si2301 (P-ch, SOT-23, 20V, 3A, Rds = 65mΩ) adds 65mΩ × 3.25A² = 0.69W at max charge current — acceptable, less than 1°C rise. It turns on automatically whenever the IC is powered and off whenever it is not. No user intervention, no firmware.

> **Why not a diode?** A Schottky diode would work but drops 0.3–0.5V at 3A, dissipating 1–1.5W continuously during charging. That heat is wasted and reduces charging efficiency. The FET solution has ~20× lower loss.

### 3.4 PD Voltage Negotiation and Efficiency

The IP2368 requests the highest available PD voltage. Efficiency improves as input voltage approaches the output (16.8V = 4S full):

| PD Voltage | Input Current | Power | Converter Mode | Efficiency | P_loss |
|---|---|---|---|---|---|
| 5V | ~5A (limited) | ~25W | Hard boost (1:3.4) | ~91% | ~2.3W |
| 9V | ~3A | ~27W | Moderate boost (1:1.9) | ~93% | ~1.9W |
| 12V | ~3A | ~36W | Light boost (1:1.4) | ~94% | ~2.2W |
| 15V | ~3A | ~45W | Near-unity (1:1.1) | ~95% | ~2.25W |
| 20V | ~3.25A | ~65W | Light buck (0.84:1) | **~96%** | **~2.6W** |

At 20V input the converter is in light buck mode — its most efficient operating point. This is why a 65W adapter gives better efficiency than a 12V brick even though 12V is closer to the nominal pack voltage.

### 3.5 Charger Board — Full BOM

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | Charger / Discharge IC | Injoinic IP2368 | QFN-28 | 1 | Integrated PD negotiator. Bidirectional buck-boost. 2–6S. 65W max both directions. LCSC: verify current stock. |
| Q_iso | Battery isolation FET | Si2301 P-ch 20V 3A 65mΩ | SOT-23 | 1 | Between B+ and IC BAT pin. Gate driven by IC VCC — auto on/off. Prevents back-feed when IC unpowered. See §3.3. |
| L1 | Buck phase inductor | 4.7µH Isat **≥8A** DCR <30mΩ shielded | 6.7×6.7mm SMD | 1 | First inductor of the 4-switch stage. **Isat ≥8A required** — at 5V input (hard boost) peak inductor current reaches 8–10A. 5A Isat saturates. Sunlord SWPA6028S4R7MT or Bourns SRR6028-4R7Y. |
| L2 | Boost phase inductor | 4.7µH Isat **≥8A** DCR <30mΩ shielded | 6.7×6.7mm SMD | 1 | Second inductor. Identical to L1. **Both inductors required** — 4-switch buck-boost uses two. Shielded mandatory. |
| C1, C2 | Input VBUS bulk caps | 22µF 25V X7R ×2 | 1210 | 2 | VBUS input rail. Two in parallel for lower ESR. 25V rated for margin above 20V PD. |
| C3, C4 | Output bulk caps | 47µF 25V X7R ×2 | 1210 | 2 | Battery output rail. Reduces charge ripple. X7R maintains capacitance at 16.8V DC bias. |
| C5–C10 | IC decoupling | 100nF X7R ×6, 10nF X7R ×2 | 0402 | 8 | Per supply pin. Each within 0.5mm of its pin. |
| C11 | Bootstrap cap BST | **470nF** X7R | 0402 | 1 | High-side gate driver bootstrap. **470nF specifically** — 220nF causes high-side gate dropout at peak power. Not a generic decoupling cap value. |
| R_VSET | Charge voltage resistors | Per IP2368 Table 5 | 0402 | 2 | Programs CC/CV termination. For 4S Li-ion (16.8V) read exact RSET1/RSET2 values from datasheet. Verify output with meter before first charge. |
| R_ISET | Charge current set | 12kΩ → ~2.5A | 0402 | 1 | CC charging current. 12kΩ = ~2.5A. For 2P pack this is 0.42C — conservative and safe. Scale to 8kΩ for 3P or 4P packs (still <0.5C). |
| R_iso1 | Isolation FET gate drive | 10kΩ | 0402 | 1 | From IC VCC to Q_iso gate. Controls turn-on speed. |
| R_iso2 | Isolation FET gate pull-up | 100kΩ | 0402 | 1 | From Q_iso gate to source. Ensures FET is off when IC VCC is absent. |
| R_SENSE | Charge current shunt | 10mΩ ±1% 2W | 2512 | 1 | In-line charge current for CC regulation. Kelvin connections — same rules as BMS R1. |
| NTC1 | Battery NTC | 100kΩ B=3950 | 0402 | 1 | TBAT pin. Derates charge above 40°C, inhibits above 50°C. |
| NTC2 | IC thermal NTC | 100kΩ B=3950 | 0402 | 1 | Mounted near IC. Monitors IC body temperature alongside internal OTP. |
| J1 | USB-C receptacle | HRO TYPE-C-31-M-12 or equiv | SMD mid-mount 16-pin | 1 | **Must be 16-pin** — CC1/CC2 pins required for PD. A 4-pin USB-C looks identical but negotiates nothing above 5V. VBUS/GND pads: full copper fill, thermal relief OFF. |
| J2 | Cell tap pass-through | 5-pin JST-XH 2.54mm | THT/SMD | 1 | Mates with BMS J1. Carries B1/B2/B3 mid-taps for ETA3000 balancer feed during charge. |
| J3 | Charger power output | XT30U male | PCB pads | 1 | Mates with BMS J3 female. CHG+/CHG−. |
| D1 | VBUS TVS clamp | SMAJ20A 20V 400W transient | SMA | 1 | Clamps VBUS transients from cable inductance and adapter glitches. Place between USB-C VBUS pin and IC input. |
| F1 | Polyfuse PTC | MF-MSMF250/16X 2.5A hold / ~5A trip | 1812 | 1 | VBUS overcurrent before IC. Self-resetting. Catches cable fault conditions. |
| LED1 | Charging indicator | Green 0402 | 0402 | 1 | CHRG pin. On = CC or CV phase. |
| LED2 | Charge complete | Blue 0402 | 0402 | 1 | STDBY pin. On = charge complete / standby. |
| LED3 | Fault indicator | Red 0402 | 0402 | 1 | FAULT pin. On = OTP, input UV, or negotiation failure. |

### 3.6 Charger Layout Rules

#### Inductors — The Single Most Important Placement Decision

L1 and L2 must be as close as physically possible to the IP2368 SW1/SW2 switching node pins. Every extra millimetre of trace to the switching node is a parasitic inductance in the switching loop, causing:
- Increased voltage overshoot at switching transitions — potential damage to internal FET gate oxides
- Increased EMI radiation — coupling into nearby signal lines
- Reduced converter efficiency

Both inductors on the top layer. Orient them parallel but separated by ≥3× their body height to prevent mutual inductance between them. Input caps (C1, C2) physically between USB-C connector and IC. Output caps (C3, C4) between IC and output connectors. The physical layout should mirror the signal flow.

#### Thermal Management of IP2368 QFN-28

The QFN-28 exposed center pad is the only thermal path out of the IC. It must be properly soldered to a copper pour with thermal vias — this is not optional at 65W.

- **Minimum 16 vias** (4×4 grid) under the center pad, 0.3mm drill, 0.5mm annular ring
- **Tented vias** on the bottom side — cap each via opening with solder mask so solder cannot wick down through the via during reflow and starve the pad joint
- **Bottom copper pour** directly under the IC: minimum 20×20mm polygon, 2oz
- On a 4-layer board: the inner ground plane (layer 3) directly under the IC provides a massive thermal mass — this is one of the significant thermal advantages of 4-layer (see §4)
- Place a copper keepout window on the bottom layer under L1 and L2 to prevent switching node EMI coupling into the ground plane

#### USB-C Connector

VBUS and GND pads carry up to 5A. Solid copper fill, thermal relief OFF. Thin traces here = resistive heating at the joint = connector degradation over time. CC1 and CC2 are signal lines only — 0.15mm trace directly to IC CC pins.

> ⚠️ Verify the USB-C part number before ordering. A 4-pin USB-C connector is physically identical to a 16-pin but has no CC lines and cannot negotiate any PD voltage. This is a common and expensive mistake.

---

## 4. PCB Layer Count Decision

### 4.1 The Question

Both boards could be 2-layer or 4-layer. The amplifier board is already 4-layer. JLCPCB pricing difference at prototype quantities (5 boards):

- 2-layer 100×100mm: ~$5–8
- 4-layer 100×100mm: ~$25–40

That's a real cost difference at prototype scale, roughly $30–60 more for both boards combined. Complexity of KiCad 4-layer setup is trivial — you already have a 4-layer board as reference. The question is whether the performance benefits justify the cost.

### 4.2 What a 4-Layer Stackup Gives You

Standard JLCPCB 4-layer stackup: **Signal (top) / GND / Power / Signal (bottom)**

| Benefit | 2-Layer | 4-Layer | Impact |
|---|---|---|---|
| Dedicated ground plane | Partial — GND fills on bottom layer with gaps | Full — Layer 2 is 100% unbroken GND copper | Massive. Every signal has a continuous reference plane beneath it. |
| EMI containment | Poor — signals route on both layers, limited shielding | Good — all signals sandwiched between GND and power planes | Significant for the charger board specifically |
| Thermal mass for IP2368 | Bottom pour only, thin | Layer 2 GND plane sits 0.2mm from top layer — enormous thermal capacitor | Reduces θJA by ~30–40% vs 2-layer |
| Routing density | Limited — GND fills compete with signal routing | High — inner planes free both outer layers for uncompromised signal routing | Easier layout, especially around QFN-28 |
| Power distribution | Single trace or pour | Dedicated inner power plane, near-zero resistance distribution | Cleaner PVDD rail for BMS board |
| Return current paths | Must be manually managed | Automatic — return currents hug their signal trace through the solid GND plane underneath | Fundamentally better signal integrity |

### 4.3 Per-Board Analysis

#### BMS Board — 2-Layer is Sufficient

The BMS board has:
- Low-frequency signals (cell voltage sensing, gate drive, NTC — all DC to a few kHz)
- One high-current polygon (12A max, well below where return current management becomes complex)
- No high-frequency switching on-board (ETA3000 switches at 1MHz but the ETA3000 inductors are shielded and the IC is small)
- Generous board area available — routing is not dense

**Verdict: 2-layer is correct for the BMS board.** The switching noise concern is handled by physical separation (two-board architecture) and the RC filters on sense lines. Spending $15–20 extra per prototype set on 4-layer BMS boards buys nothing measurable. Keep it 2-layer, use 2oz copper, spend the layout effort on Kelvin sensing and ground plane continuity.

#### Charger Board — 4-Layer is the Right Choice

The charger board has:
- A QFN-28 switching at 250kHz dissipating up to 4.2W — needs every thermal advantage available
- A 4-switch buck-boost with two inductors and four switching nodes — high di/dt loops that need tight return current paths
- USB-C CC signal lines (noise-sensitive, sub-picofarad coupling causes PD negotiation failures) immediately adjacent to high-current VBUS pads
- Input and output capacitors that create local switching current loops — these loops must be physically small and have ground reference beneath them

**Without 4-layer:** The charger board can be made to work on 2-layer with very careful layout — commercial IP2368 reference designs exist on 2-layer. However:
- The IP2368 thermal scenario at 65W discharge (4.2W dissipated) hits thermal shutdown on a 2-layer board with θJA = 30°C/W (see §5)
- With 4-layer, the inner GND plane drops θJA to approximately **18–22°C/W**, bringing 65W discharge to a safe 100°C junction
- The switching loop areas are inherently tighter with a dedicated ground reference plane 0.2mm below the top layer
- CC line integrity is better — the ground plane beneath provides controlled impedance and shields from VBUS adjacent coupling

**Verdict: 4-layer for the charger board.** Given the amp is already 4-layer and the complexity difference is zero, the thermal and EMI benefits justify the ~$15–20 premium per prototype run. The heatsink (see §5) further reduces junction temperature on top of this.

### 4.4 Final Layer Decision

| Board | Layer Count | Reason |
|---|---|---|
| BMS Protection Board | **2-layer, 2oz** | Low-frequency signals, simple routing, 2oz copper handles current. 4-layer adds cost with no measurable benefit. |
| Charger Board | **4-layer, 2oz outer / 1oz inner** | Thermal critical (IP2368 4.2W at 65W), switching noise management, CC line integrity, tight return current loops. |

JLCPCB stackup for 4-layer: default JLC7628 (1.6mm total, prepreg between layers). Outer layers 2oz, inner layers 1oz (standard, sufficient for planes).

---

## 5. Thermal Management & Heatsink

### 5.1 Thermal Analysis Framework

All junction temperature calculations follow:

```
T_junction = T_ambient + P_dissipated × θJA_effective
P_dissipated = P_input × (1 − η)
```

IP2368 junction temperature limit: **125°C** (absolute maximum). Thermal shutdown triggers at ~120°C internally.

### 5.2 θJA by Implementation

| Implementation | θJA (°C/W) | Notes |
|---|---|---|
| No thermal vias, 1oz, 2-layer | ~55–65 | Cheap module quality. Thermal shutdown at sustained >30W. |
| 9 vias, 2oz, 2-layer, no heatsink | ~28–32 | Minimum viable. Safe at 65W charge. Marginal at 65W discharge. |
| 16 vias, 2oz, 2-layer, no heatsink | ~24–28 | Good. Safe at 65W charge. Borderline at 65W OTG discharge. |
| 16 vias, 2oz, **4-layer** (inner GND plane), no heatsink | ~18–22 | **Recommended baseline.** Inner GND plane acts as massive heat spreader. |
| 16 vias, 2oz, 4-layer + **20×20×8mm heatsink** | **~10–14** | Full solution. Safe at 65W in both directions with margin. |

### 5.3 How the Heatsink Works With This Board

A 20×20×8mm aluminum heatsink attached to the **bottom of the PCB** directly under the IP2368 is a valid and effective approach. The thermal path is:

```
IC junction
    ↓  θJC ~2°C/W  (internal package)
QFN center pad (soldered to top copper)
    ↓  θpad ~1°C/W (solder + copper spread)
Top copper pour
    ↓  θvia ~3–5°C/W  (16 thermal vias through 1.6mm PCB)
Bottom copper pour
    ↓  θTIM ~1–2°C/W  (thermal pad or paste)
Heatsink base
    ↓  θHS ~8–12°C/W  (20×20×8mm aluminum, natural convection)
Ambient air
```

Total chain: ~15–22°C/W depending on TIM quality and via implementation.

**The critical insight:** thermal vias work well here because QFN packages have no topside thermal path — the only heat exit is downward through the exposed pad. Thermal vias are the only source to dissipate heat from QFN/DFN packages, as the top copper does not have maximum space due to pin allocation. For exposed-pad packages, the maximum heat dissipation happens through the vias to the bottom layer of the PCB. Your heatsink sits at the end of that exact path.

**Via plating matters.** Standard JLCPCB vias are plated to ~25µm copper wall thickness (not solid-filled). If via walls are only plated rather than solid-filled, their thermal conductivity is dramatically less than assumed. This is accounted for in the θvia estimate above. Solid-filled copper vias (available as an upgrade on JLCPCB) would improve this but are unnecessary given the heatsink is already in the chain.

### 5.4 Complete Thermal Scenarios — With Heatsink (θJA = 12°C/W)

| Scenario | Power | η | P_loss | T_junction at 25°C ambient | Status |
|---|---|---|---|---|---|
| Charging 65W (20V PD) | 65W | 96% | 2.6W | 25 + (2.6 × 12) = **56°C** | ✅ Excellent |
| Charging 36W (12V) | 36W | 94% | 2.2W | 25 + (2.2 × 12) = **51°C** | ✅ Excellent |
| Charging 25W (9V) | 25W | 93% | 1.75W | 25 + (1.75 × 12) = **46°C** | ✅ Excellent |
| Discharging 65W OTG | 65W | 93% | 4.2W | 25 + (4.2 × 12) = **75°C** | ✅ Comfortable |
| Discharging 45W OTG | 45W | 93% | 3.15W | 25 + (3.15 × 12) = **63°C** | ✅ Very comfortable |
| Discharging 20W OTG (phone) | 20W | 94% | 1.2W | 25 + (1.2 × 12) = **39°C** | ✅ Barely warm |

Compare with no heatsink, 4-layer only (θJA = 20°C/W):

| Scenario | T_junction | Status |
|---|---|---|
| Charging 65W | 25 + (2.6 × 20) = **77°C** | ✅ Fine |
| Discharging 65W OTG | 25 + (4.2 × 20) = **109°C** | ⚠️ Hot, near limit |

**The heatsink takes every scenario from borderline to comfortable.** Given you have 20×20×8mm heatsinks available, use one on the charger board. It costs nothing extra and the thermal margin improvement is substantial.

### 5.5 Heatsink Mounting — Practical Implementation

**Step 1 — Via implementation (PCB design phase):**
- 4×4 grid (16 vias) under QFN-28 center pad
- 0.3mm drill, 0.5mm annular ring, 1.0mm pitch
- Tented on bottom side (solder mask over via holes) — prevents solder wicking during reflow from starving the QFN thermal pad joint
- Bottom copper pour directly connected to via landings: minimum 20×20mm, 2oz

**Step 2 — Thermal Interface Material (TIM):**

Two options, in order of preference:

| TIM Type | Thermal conductivity | Ease | Notes |
|---|---|---|---|
| Silicone thermal pad (Bergquist GP3000 or equiv.) | 3.0 W/(m·K) | Cut to 20×20mm, peel and stick | Best choice. Electrically isolating, handles PCB surface unevenness well, no mess. |
| Thermal paste (Arctic MX-4 or equiv.) | 8.5 W/(m·K) | Apply thin layer, mechanical retention needed | Better conductivity but needs physical clamping pressure to maintain contact. Without pressure, paste migrates. |
| Thermal adhesive tape | 1.5–2.0 W/(m·K) | Peel and stick, permanent | Lowest conductivity. Use only if mechanical mounting is impossible. |

**Recommendation: silicone thermal pad.** The gap-filling property handles any PCB surface roughness or slight bow. Cut to exactly 20×20mm. Clean both surfaces with IPA before application.

> ⚠️ **The bottom PCB copper under the IC contains ground plane copper.** If you use thermal paste (not a silicone pad), the paste itself must be electrically isolating — standard thermal paste is electrically isolating (ceramic or oxide filler) but double-check. Do NOT use silver-filled thermal compounds on an exposed copper surface with mixed voltages on adjacent traces — risk of conduction bridges.

**Step 3 — Heatsink attachment:**

- Place TIM on bottom PCB copper pour under IC
- Place heatsink flat face on TIM
- **Mechanical retention is required** — thermal paste and silicone pads both require contact pressure to maintain optimal conductivity. Options:
  - Two M2 screws through PCB corners into tapped heatsink holes — clean, professional, removable
  - Thermal adhesive in addition to pad — permanent, simpler
  - Small PCB clip bracket — if heatsink has holes
- Orient heatsink fins (if any) for natural convection airflow — fins vertical, pointing downward or toward any enclosure vent

**Step 4 — Enclosure clearance:**
- Heatsink extends 8mm below board
- Account for this in speaker enclosure standoff height — nylon M3 standoffs must be ≥10mm to clear heatsink
- If space is constrained, a 20×20×5mm heatsink still provides meaningful thermal benefit (θHS ≈ 14°C/W) and only adds 5mm

### 5.6 BMS Board Thermal — No Heatsink Needed

The BMS board dissipates:
- Q1 + Q2 FETs combined: ~1.15W at 12A → θJA ~40°C/W per SO-8 → ~23°C rise → FETs at ~48°C. Fine.
- R1 shunt: P = I² × R = 144 × 0.008 = 1.15W → 2512 package dissipates this easily, stays below 60°C
- ETA3000 ×3: each instance at 2A balancing dissipates (1−0.92) × Vcell × I = 0.08 × 3.7 × 2 = 0.59W per IC → trivial in SOT-23
- S-8254AA: µA-level quiescent, negligible

Total BMS board dissipation at full load: ~3.5W, spread across multiple components. No single component is thermally stressed. No heatsink required on the BMS board.

---

## 6. PCB Ordering & Build Procedure

### 6.1 JLCPCB Order Settings

| Parameter | BMS Board | Charger Board | Reason |
|---|---|---|---|
| Layers | **2** | **4** | See §4 — charger needs inner GND plane for thermal and EMI |
| Copper weight | **2oz outer** | **2oz outer / 1oz inner** | 2oz on current-carrying layers. Inner planes at 1oz is standard and adequate for planes. |
| Surface finish | **ENIG** | **ENIG** | Required for QFN-28. Improves TSSOP solderability. Prevents oxidation on USB-C pads and connector contacts. |
| Solder mask | Both sides | Both sides | Standard |
| Silkscreen | Both sides | Both sides | Reference designators for assembly |
| PCB thickness | 1.6mm | 1.6mm | Standard. Thermal via aspect ratio (1.6mm / 0.3mm drill = 5.3:1) is within reliable plating range. |
| Via tenting | Open | **Tented bottom** | QFN thermal vias must be tented on bottom — prevents solder wicking during reflow. Specify in fabrication notes. |
| Stencil | **Yes, frameless** | **Yes, frameless** | Frameless ~$2–5. Mandatory for QFN-28 and TSSOP-16. Without stencil, paste on fine-pitch pads is uneven and causes bridges. |

### 6.2 Reflow Profile — SAC305 Lead-Free

| Zone | Temperature / Rate | Notes |
|---|---|---|
| Preheat | 25°C → 150°C @ 2°C/s | Gentle ramp. Activates flux, drives off paste volatiles. |
| Soak | 150°C → 200°C @ 1°C/s, hold ~90s | Thermal equalization across board. Critical for QFN — ensures center pad reaches liquidus. |
| Reflow | Peak 245°C, 20–30s above 217°C | SAC305 liquidus = 217°C. Full dwell for complete wetting of all pads. |
| Cooling | <3°C/s down to 100°C, then free-air | Controlled fast cool → fine grain microstructure → stronger joints. No quench below 100°C. |

> **QFN-28 (IP2368):** After reflow, probe continuity between the QFN center pad thermal via and GND net. No continuity = center pad did not reflow. Apply flux, hover hot-air at 250°C until device settles slightly (surface tension pull-down indicates reflow). For TSSOP fine-pitch (S-8254AA): inspect under 10× magnification, solder wick bridges with flux.

### 6.3 Assembly Order

1. Apply paste via stencil. Inspect aperture fill under magnification before placing any component.
2. Place 0402 passives (caps, resistors). Blow-off check.
3. Place ICs: S-8254AA (TSSOP-16), IP2368 (QFN-28), ETA3000 ×3 (SOT23-6), Si2301 (SOT-23). Align pin-1 markers carefully.
4. Place MOSFETs (SO-8) and shunt resistors (2512).
5. Place inductors — bulky, last before reflow.
6. Reflow top side.
7. Inspect, touch up, clean with IPA + stiff brush.
8. Flip, hand-solder bottom-side THT connectors (JST-XH, headers, XT30 pads).
9. Continuity checks on all critical nodes.
10. Attach heatsink to charger board bottom with TIM pad + M2 screws.

> ⛔ Do not connect battery until all bench tests (§7) are complete. First power-on through a current-limited bench PSU at 100mA.

---

## 7. Test & Verification Procedure

### 7.1 Pre-Power Checks (Board Unpowered, No Battery)

- **Visual:** bridges on QFN and TSSOP, cold joints on 2512 shunts, SO-8 FET orientation matches schematic
- **B+ to P− resistance:** >1MΩ — both FETs off with no gate drive
- **R1 shunt:** 4-wire Kelvin measurement should read 8mΩ ±1%. >50mΩ indicates cold joint.
- **NTC resistance at 25°C ambient:** 97–103kΩ — confirms thermistor is not open
- **Si2301 isolation FET gate:** with IC unpowered, Vgs should equal 0V (gate = source via 100kΩ) — FET must be off

### 7.2 BMS Board — Protection Threshold Verification

Use 4× bench PSU channels simulating cell groups at 3.700V each. Connect to B−, B1, B2, B3, B+.

**Nominal:** P+ to P− = 14.8V. Both FETs conducting.

**OVP:** Raise one simulated cell above 4.28V → COUT falls low → CHG FET off. P+ still present (DSG path unaffected). Drop cell to 4.17V → COUT returns high. Hysteresis confirmed.

**UVP:** Lower one simulated cell below 2.45V → DOUT falls low → P+ = 0V. Raise cell above 2.70V → DOUT returns high. Recovery confirmed.

**OCD1/OCD2:** Apply resistive load to approach 18A trip threshold. OCD1 should trip after ~16ms delay, OCD2 after ~8ms sustained. Non-latching — both reset when current drops.

**SCD:** Apply a near-short (0.1Ω load) momentarily → DOUT cuts within 200µs. Verify latch — disconnecting and reconnecting load without resetting should keep DOUT low until load is fully removed and reconnected cleanly.

**ETA3000 verification:** Set two adjacent simulated cells to 3.800V and 3.650V (150mV delta — well above 30mV activation). Confirm ETA3000 instance spanning them draws current from higher cell, delivers to lower. Confirm delta closes over time.

**OTP:** Heat NTC1 to ~45°C → COUT falls low (charge inhibit). DOUT remains high (discharge allowed). Cool below 40°C → COUT recovers.

### 7.3 Charger Board — First Power-On

- Measure heatsink bottom surface before connecting anything — establishes baseline for thermal test.
- Connect 65W PD adapter via USB-C.
- Measure VBUS pin: starts 5V, rises to negotiated PD voltage within 2–3 seconds.
- Measure Si2301 drain (BAT pin): VCC must be present for FET to conduct — confirm IC VCC rail first.
- Measure CHG+/CHG− output into 22Ω dummy load: expect 16.8V ±0.1V.
- Measure charge current: should match R_ISET programmed value (~2.5A).
- **5-minute sustained charge test:** measure IC surface temperature with IR thermometer. With heatsink and 4-layer PCB, should remain below 65°C case temperature at 65W input.
- Verify LED sequence: green = charging, blue = done/standby, red = no fault.
- **OTG test:** disconnect adapter, connect a phone or USB-C load. IP2368 should negotiate as power source, output should appear on CHG+/CHG− in reverse direction.

### 7.4 Full System Integration Test

- Connect charger XT30 to BMS J3. Connect cell pack to BMS J1. Connect TAS5825M board (or 22Ω dummy) to BMS J2.
- **Charging:** adapter connected, charge current flows, ETA3000 instances balance as needed.
- **Discharging/playback:** adapter disconnected, load draws power from pack through BMS.
- **Thermal 30min full load:** FET packages <65°C, IP2368 case <70°C with heatsink, cell body <40°C.
- **Nuisance OCD1 trip test:** play audio at high volume with heavy bass. If OCD1 trips on kick drum transients, increase C3 (CTD) from 0.22µF to 0.47µF and retest.
- **Heatsink contact verification:** after thermal test, remove heatsink and inspect TIM pad for even contact impression. Uneven impression = heatsink not fully flat — re-seat with fresh TIM.

---

## 8. System Integration Notes

### 8.1 Cell Selection — 18650 Recommendations

| Cell | Capacity | CDR | Rec. Charge | Notes |
|---|---|---|---|---|
| Samsung 30Q | 3000 mAh | 15A | 1.5A | High CDR for its capacity. Good choice for 2P4S. Proven reliability. |
| Samsung 35E | 3500 mAh | 8A | 1.75A | Higher capacity, lower CDR. Fine for 2P+ where CDR doubles. |
| Sanyo/Panasonic NCR18650GA | 3500 mAh | 10A | 1.75A | Excellent cycle life. Slightly lower CDR but long-term longevity. |
| Molicel P26A | 2600 mAh | 35A | 2.6A | Overkill CDR. Use only if scaling to a much higher power system. |

Match cells from the same batch. Never mix chemistries or capacities in the same pack.

### 8.2 Connector Guidance

- **XT30** on all high-current paths. 30A rated, gold-plated, locking. Male on board, female on cable.
- **JST-XH** on all cell tap connections. Keyed — prevents reversed insertion. 26AWG silicone wire.
- **No dupont / header pins** on any current-carrying path. <1A per contact, will arc under transients.

### 8.3 Wiring the Pack

- **Cell interconnects:** spot-welded nickel strip preferred. If soldering: 60W+ iron, pre-tinned surfaces, ≤2 second contact per pad.
- **Main leads (B+, B−):** 14AWG silicone, <200mm length.
- **Cell tap leads:** 26AWG silicone, routed away from main power leads and inductor fields.

> ⛔ Lithium cells must never be shorted. Kapton tape on all unconnected terminals during assembly. No metal tools near exposed cell terminals.

### 8.4 PVDD Decoupling on Amplifier Board

The TAS5825M draws large transient currents on bass. Without adequate local decoupling, these spikes propagate back through the BMS shunt and nuisance-trip OCD1.

At TAS5825M PVDD input, as close to IC as possible:
- 2× 470µF 25V low-ESR electrolytic
- 4× 10µF 25V X7R ceramic (1210)
- 4× 100nF X7R ceramic (0402, right at PVDD pins)

If nuisance OCD1 trips still occur, increase C3 (CTD) on BMS board as described in §7.4.

### 8.5 Board Mounting

- M3 nylon standoffs, minimum 10mm height on charger board (heatsink clearance).
- Minimum 10mm clearance between charger board and any audio signal trace.
- Heatsink fins oriented for natural convection — vertical fins, not horizontal. Sealed enclosure: add small ventilation slots near heatsink location if possible.

### 8.6 Portfolio / Curriculum Context

This project demonstrates:

- **Power electronics:** multi-cell Li-ion management, synchronous 4-switch buck-boost, USB-PD bidirectional protocol negotiation
- **Analog design:** precision Kelvin sensing, RC noise filtering, threshold configuration by passive components, NTC temperature limiting
- **Active circuit design:** inductive active cell balancing, autonomous bidirectional energy transfer
- **PCB layer stack decision:** documented reasoning for mixed 2-layer/4-layer choice based on functional requirements not just cost
- **Thermal engineering:** IC junction temperature analysis across operating scenarios, heatsink selection and sizing, thermal via implementation, TIM selection
- **Protection architecture:** multi-tier fault response, asymmetric FET control, latching vs non-latching protection design, back-feed isolation
- **Process:** stencil reflow, QFN soldering, thermal via tenting, multi-stage test with threshold measurement

Document KiCad schematic DRC-clean, export Gerbers + interactive BOM (IBoM plugin). Write a one-page test report with measured threshold values vs. expected — that converts a build into a professional engineering document suitable for a portfolio.

---

## 9. Key Design Decisions Reference

| Decision | Choice | Rationale |
|---|---|---|
| Cell format | 18650, xP4S | Common availability, parallel groups self-balance within group, BMS always sees exactly 4 voltages |
| BMS IC | ABLIC S-8254AA | Native 4S, per-cell-group sensing, hardware-only, ±20mV accuracy, independent CHG/DSG control |
| Balancing type | Active — ETA3000 ×3 | 92% efficiency inductive transfer, no MCU, autonomous, 2µA standby, correct scale for 4S |
| Protection FET count | 2 FETs (1 CHG + 1 DSG) | 12A bottleneck from TAS5825M. Two AO4407A dissipate <1.2W combined. More FETs = parallel for >20A loads only. |
| FET topology | Back-to-back SO-8, source-to-source | Universal BMS topology. Body diodes block in both directions for independent CHG/DSG control. |
| Sense resistor | 8mΩ 1% 3W | OCP trip at 18A. Constant across all P configs — amp is always the current bottleneck. |
| Charger IC | Injoinic IP2368 | Integrated PD negotiation. Bidirectional OTG. 4-switch BB for all PD voltages. 65W. No external PD controller needed. |
| OTG discharge | Enabled via IP2368 | Same IC, same port, same converter reversed. Speaker doubles as power bank. No extra cost. |
| Reverse current protection | Si2301 P-ch isolation FET | FET solution: ~0.7W loss at max current vs 1.5W+ for Schottky. Auto on/off from IC VCC rail. |
| Charger topology | 4-switch synchronous buck-boost | Single topology for all PD voltages 5–20V. No mode gaps, no minimum input constraint. Bidirectional. |
| Inductor Isat | ≥8A for charger inductors | At 5V input (hard boost mode) peak inductor current reaches 8–10A. 5A Isat saturates the core. |
| Board layers | BMS: 2-layer / Charger: 4-layer | BMS: low-frequency, simple — 2-layer + 2oz is sufficient. Charger: thermal-critical QFN at 65W, switching EMI — 4-layer inner GND plane drops θJA by ~30–40%. |
| Copper weight | 2oz outer layers both boards | Halves trace resistance. ~$3–5 premium on JLCPCB. Non-negotiable for current-carrying reliability. |
| Heatsink | 20×20×8mm on charger board bottom | Drops θJA from ~20°C/W to ~12°C/W. 65W OTG discharge: 75°C junction vs 109°C without. Silicone pad TIM + M2 screws. |
| Via tenting | Tented bottom on charger QFN vias | Prevents solder wicking down through thermal vias during QFN reflow, which would starve the thermal pad joint. |
| High-current connectors | XT30 + JST-XH | XT30: 30A, gold, locking. JST-XH: keyed cell taps. No dupont on any current path. |
| USB-C connector | 16-pin SMD mid-mount | CC1/CC2 required for PD. 4-pin USB-C is physically identical but useless for PD negotiation. |
| OCD1 delay | 0.22µF CTD cap (~16ms) | Accommodates TAS5825M bass transients without nuisance trips. Tunable to 0.47µF. |

---

*End of document — Rev 3.0*
