# 4S Lithium Battery System — Engineering Design & Build Reference

**Scalable Smart BMS Node · USB-C PD Charger Board**  
Primary application: Bluetooth Speaker / TAS5825M Amplifier Platform  
Architecture: Distributed modular BMS — single node now, stackable via CAN later  
*Revision 6.0 · June 2026 · Arik Nel*

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [BMS Node Board](#2-bms-node-board)
3. [CAN Scalability — Designed-In Optionality](#3-can-scalability--designed-in-optionality)
4. [Charger Board](#4-charger-board)
5. [PCB Layer Count Decision](#5-pcb-layer-count-decision)
6. [Thermal Management & Heatsink](#6-thermal-management--heatsink)
7. [ESP32-S3 Firmware Architecture](#7-esp32-s3-firmware-architecture)
8. [Host Communication Protocol](#8-host-communication-protocol)
9. [PCB Ordering & Build Procedure](#9-pcb-ordering--build-procedure)
10. [Test & Verification Procedure](#10-test--verification-procedure)
11. [System Integration & Multi-Node Scaling](#11-system-integration--multi-node-scaling)
12. [Key Design Decisions Reference](#12-key-design-decisions-reference)

---

## 1. System Overview

### 1.1 Architecture Philosophy — The Smart Node

This is not a BMS for one speaker. It is a **self-contained, open-source, programmable 4S2P battery management node** that happens to power a speaker first.

Each node handles its own:
- Hardware cell protection (BQ76920 — operates even with no firmware)
- Active cell balancing (ESP32-driven discrete inductive balancer)
- SOC tracking (coulomb counting via BQ76920 current sense)
- Telemetry (UART host interface + WiFi webapp)
- Optional networking (CAN bus with galvanic isolation — footprints designed in, populated only when scaling)

**Scaling is done by stacking nodes, not redesigning them.** Need a 16S4P powerwall? Stack four nodes in series, populate the CAN/isolation section (~€5 of parts per board), and they join a shared bus reporting to a master aggregator. Same PCB, same firmware, no redesign.

For the speaker build: CAN section unpopulated. Node runs standalone with UART to the amp board ESP32.

### 1.2 Power Architecture Summary

| Stage | Function | Key Spec |
|---|---|---|
| USB-C PD Input | Negotiate 5–20V from any adapter via FUSB302 | PD2.0 / PD3.0 / QC3.0, sink and source |
| BQ25713B Buck-Boost | Charge: any PD voltage → 16.8V CC/CV. OTG: pack → 5V/9V on VBUS | 4-switch, 4× external FETs, 1 inductor, I2C |
| 4S2P Li-ion Pack | 14.8V nom / 16.8V full / 12.0V cutoff | 8× 18650, 2 per series group |
| BQ76920 | All measurement + hardware protection | ±1mV cell voltage, pack current, 2× NTC |
| ESP32-S3 (bare SoC) | Intelligence: balancing, SOC, comms, webapp | Deep sleep ~10µA, native USB pads |
| Active Balancer | 3× discrete FET pairs + inductors, ESP32-driven | Inductive, any-to-any, ~88% efficient |
| AP2112K-3.3 LDO | 3.3V for ESP32-S3 + BQ76920 logic | 600mA, low dropout, SOT-23-5 |
| TAS5825M Amp | PVDD from BMS P+/P− | 16.8V → ~2×25W @ 4Ω |

### 1.3 Cell Configuration: 4S2P with 18650

Parallel groups self-balance physically — cells sharing terminals equalise through internal resistance continuously. The BMS sees exactly 4 group voltages regardless of P count.

| Parameter | 2P (Samsung 30Q) |
|---|---|
| Capacity | 6000 mAh |
| Max cont. discharge | 30A cells / ~12A actual (TAS5825M limit) |
| Recommended charge current | 4A cells / 2.5A programmed (0.42C — conservative) |
| Runtime @ 6A average | ~60 min |

### 1.4 Why Two Boards?

- **Noise:** BQ25713B switches at 800kHz; BQ76920 resolves ±1mV. Physical separation is the cleanest solution.
- **Thermal:** BQ25713B + FETs dissipate 2–4W at full charge. Separate from cells and precision analog.
- **Modularity:** BMS is a standalone reusable product. Charger is application-specific. A powerwall built from these nodes uses a different charge source entirely.
- **Fault isolation:** independent testing, independent failure, independent replacement.

### 1.5 Why Two FETs on the BMS — Not Four, Six, or Eight

Commercial BMS boards with 4–8 FETs are paralleling devices per switch for high-current applications. The topology is always exactly **two switches: one CHG FET, one DSG FET**, back-to-back on the negative rail.

At this node's 12A continuous ceiling, two AON6354 at 5.2mΩ each @ Vgs=4.5V: P = 12² × 0.0104 = **1.5W total**. DFN5×6 exposed pad bonds directly to PCB copper pour → θJA ~30°C/W → ΔT = 45°C → ~70°C at 25°C ambient. Correctly sized. More FETs only needed above ~20A continuous.

---

## 2. BMS Node Board

### 2.1 Protection & Measurement IC: BQ76920

TI BQ76920 (3–5S analog front-end): the sole measurement and hardware-protection IC on the node. The ESP32 reads it over I2C; it protects the pack independently of firmware.

**Hardware protection (always active, firmware-independent):**

Power-on defaults protect the pack before any I2C configuration. An unprogrammed, crashed, or sleeping ESP32 never leaves the pack unprotected.

- Per-cell-group OVP / UVP — independent comparators per VC input, ±1mV accuracy
- Overcurrent discharge (OCD) — configurable threshold + delay via register
- Short circuit discharge (SCD) — ~400µs response, **latching**
- Overcurrent charge (OCC) — separate configurable threshold
- Dual NTC temperature protection (TS1, TS2 pins)
- Independent CHG and DSG FET gate drives — asymmetric fault response
- ALERT pin — hardware interrupt on any fault, wired to ESP32 wake-capable GPIO

**What the ESP32 adds over I2C (address 0x08):**
SOC coulomb counting · configurable threshold adjustment · fault logging with timestamps · active balancing decisions · host comms · graceful pre-shutdown warning · webapp.

### 2.2 Complete Protection Feature Map

| Fault | Detection | Default Threshold | Response | Latching? | Recovery |
|---|---|---|---|---|---|
| OVP per group | VC1–VC4 | 4.20V configurable | CHG FET off; DSG unaffected; ALERT low | No | Auto below OV_CLEAR |
| UVP per group | VC1–VC4 | 2.80V configurable | DSG FET off; CHG unaffected; ALERT low | No | Auto above UV_CLEAR |
| OCD | SRP/SRN shunt | 56mV / 3mΩ ≈ 18.7A | DSG off after configurable delay; ALERT low | No | Auto when current drops |
| SCD | SRP/SRN spike | Fast detect | DSG off ~400µs; ALERT low | **Yes** | Load disconnect required |
| OCC | SRP/SRN (charge dir.) | Configurable | CHG off; ALERT low | No | Auto |
| OTP | TS1/TS2 NTC | Configurable | CHG and/or DSG inhibit | No | Auto with hysteresis |

> **SCD latches deliberately** — a dead short must require human intervention. **OVP/UVP asymmetry is deliberate** — over-charged pack still discharges; over-discharged pack still accepts recovery charge.

> **Hardware defaults matter:** if ESP32 has not booted, the BQ76920 uses power-on register defaults. Pack is always protected.

### 2.3 Active Cell Balancing — Discrete, ESP32-Driven

#### The BQ76920 is not involved in balancing

The BQ76920 has built-in passive balancing (BALx pins). **This design does not use it.** CELLBAL registers stay 0x00, BALx pins unconnected. The BQ76920's only balancing role is voltage measurement.

True active balancing requires external hardware the BQ76920 cannot drive: complementary FET pairs switching an inductor at 50kHz. The ESP32 drives that hardware directly via LEDC peripheral. The BQ76920 measures; the ESP32 and discrete FETs act.

```
1. ESP32 reads VC1–VC4 from BQ76920 over I2C    ← BQ76920's only involvement
2. ESP32 computes deltas, picks transfer path
3. ESP32 drives TC4427 → FET gates at 50kHz      ← BQ76920 not involved
4. Inductor shuttles charge between cell groups
```

#### Balancing runs continuously — not only during charging

The balancer connects to cell tap nodes and transfers charge whenever the ESP32 wakes and finds a delta — during charge, discharge, or at rest. Same routine every wake cycle:

- Wake every 60s (configurable via NVS / `SET_BALANCE_INTERVAL` UART command)
- Read 4 group voltages from BQ76920
- If max−min > 30mV: run relevant balancer instance(s) for up to 30s
- Re-check voltages, sleep

60s is conservative: at 2A transfer current a 100mAh imbalance resolves over ~5–6 wake cycles. Longer intervals (120–300s) work equally well; shorter gains nothing. Interval is NVS-persisted.

#### Why no balancer IC

Every autonomous active balancer IC has sourcing problems: ETA3000 (unavailable Mouser/LCSC), LT8584 (custom flyback transformer per cell), LTC3300 (SPI host + coupled inductors). TI's own E2E forum confirms no standalone single-chip active balancer exists. Discrete circuit — Si2302 pairs + TC4427 + standard shielded inductor — uses commodity parts available everywhere. Stronger portfolio piece: circuit is designed, not bought.

#### Circuit topology (3 instances: C1↔C2, C2↔C3, C3↔C4)

```
Group N+  ──[Q_H drain]
               [Q_H source]──[L 4.7µH]──[Q_L source]
                                              [Q_L drain]──Group N−

Q_H / Q_L gates ← TC4427 dual driver ← ESP32 LEDC PWM
(complementary signals, firmware-enforced dead-time)
```

Q_H on: inductor charges from upper group. Q_L on: inductor discharges into lower group. Reverse pattern = reverse direction. **Any-to-any:** ESP32 runs multiple instances simultaneously in chosen directions for non-adjacent transfers.

### 2.4 ESP32-S3 Integration — Bare SoC Design

Bare ESP32-S3 SoC (QFN-56), 8MB NOR flash (W25Q64JVSSIQ, WSON-8), 40MHz crystal, PCB trace antenna. Full RF layout responsibility on the PCB — the main reason the BMS board is 4-layer (see §5).

#### Core subsystem components

| Ref | Component | Value / Part | Notes |
|---|---|---|---|
| U2 | ESP32-S3 SoC | QFN-56 | Native USB, TWAI, LEDC, WiFi/BT. No module — full RF control. |
| U_FL | NOR Flash | W25Q64JVSSIQ 8MB WSON-8 | QSPI to FSPI pins. 100nF + 1µF at VCC. |
| X1 | Crystal | 40MHz ±10ppm, 3225 | Epson FA-238 or equiv. Load caps per CL spec — typically 2×15pF at CL=10pF. |
| U3 | 3.3V LDO | **AP2112K-3.3TRG1** | 600mA, 300mV dropout, SOT-23-5. Powers ESP32-S3 + BQ76920 logic. Low quiescent — critical for sleep budget. |

#### Crystal layout rules

- Traces XTAL_P / XTAL_N: ≤5mm, no vias
- Load caps symmetrically placed at crystal pads
- Ground guard ring around crystal cell on top layer, stitched to layer 2 plane every 1mm
- Solid ground pour all layers under crystal
- No switching traces within 3mm

#### NOR flash layout rules

- All QSPI traces under 15mm, roughly equal length
- 100nF directly at flash VCC pin + 1µF on same node
- Route on inner layer if top layer is congested near SoC

#### PCB trace antenna — highest-priority placement rule

The antenna is a tuned 50Ω microstrip. **No copper on any layer** (top, inner planes, bottom) within the keepout zone — inner plane void must be explicitly designed in KiCad and verified in gerbers before ordering.

- Antenna at board corner or edge, facing outward
- All switching sources (balancer FETs, power polygon, XT30 connectors) on **opposite end of board** — maximum physical separation
- Antenna trace width from JLC7628 stackup: layer 1→layer 2 prepreg ~0.21mm, εr ~4.6 → **~0.38–0.42mm for 50Ω** — calculate with KiCad microstrip tool or Saturn PCB Toolkit using actual JLC values before routing
- RF matching network (π-match) on LNA_IN: use Espressif hardware design guide reference values as starting point

#### ESP32-S3 power domains (all require independent decoupling)

| Pin | Supply | Decoupling |
|---|---|---|
| VDD3P3 | 3.3V | 10µF + 100nF |
| VDD3P3_RTC | 3.3V | 100nF |
| VDD3P3_CPU | 3.3V | 10µF + 100nF |
| VDD_SDIO | 3.3V | 10µF + 100nF (FSPI to flash) |
| VDD3P3_USB | 3.3V | 100nF (USB PHY) |

Each cap within 0.5mm of its respective pin. Map datasheet pin table to physical QFN-56 pad locations before placement.

#### Power budget (60s wake interval)

| State | Current |
|---|---|
| Deep sleep | ~10µA |
| Active (balance / UART / fault) | ~80mA |
| WiFi webapp | ~120–180mA |

Average: (0.1s × 80mA + 59.9s × 0.01mA) / 60 ≈ **0.14mA** → ~0.2mAh/day from 6Ah pack. Negligible.

**Wake sources:** RTC timer · BQ76920 ALERT GPIO interrupt · UART RX · CAN RX GPIO edge (node mode only).

### 2.5 Programming Interface — Native USB Pads

No USB-C connector on BMS, no bridge IC. Four unpopulated 2.54mm THT pads from ESP32-S3 native USB:

```
[VBUS] [D−] [D+] [GND]
```

Cut any USB cable, clip/solder four wires. S3 enumerates as CDC serial + JTAG simultaneously — flash with `idf.py flash` or esptool, debug with OpenOCD, monitor with any terminal. All through the same four pads.

Also break out: **BOOT** pad (GPIO0 → GND for download mode) and **EN** pad (reset), each with optional tactile button footprint.

Route D+/D− as matched-length differential pair, no vias.

### 2.6 Host UART Port

4-pin JST-SH 1.0mm (GND / 3.3V / TX / RX). 115200 8N1, ASCII protocol (§8). Amp board ESP32 connects here. Doubles as dev serial monitor with CP2102 dongle. UART RX wakes BMS from sleep in ~5ms.

### 2.7 BMS Node Board — Full BOM

**Core (always populated):**

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | AFE / protection | TI BQ76920 | TSSOP-20 | 1 | ±1mV cell groups, I2C 0x08, OVP/UVP/OCD/SCD/OCC, CHG+DSG FET drive, ALERT. Mouser/Digikey stocked. |
| U2 | MCU | ESP32-S3 bare SoC | QFN-56 | 1 | Native USB, TWAI, LEDC, WiFi. Bare SoC — see §2.4 for full subsystem. |
| U3 | 3.3V LDO | AP2112K-3.3TRG1 | SOT-23-5 | 1 | 600mA, 300mV dropout. Powers ESP32-S3 + BQ76920. Low IQ — critical for sleep budget. |
| U_FL | NOR Flash | W25Q64JVSSIQ 8MB | WSON-8 | 1 | QSPI interface. 100nF + 1µF at VCC. |
| X1 | Crystal | 40MHz ±10ppm | 3225 | 1 | E.g. Epson FA-238. 2× load caps per CL spec. |
| Q1 | CHG FET | AON6354 30V 83A 5.2mΩ | DFN5×6 | 1 | Back-to-back on B− rail. Driven by BQ76920 CHG pin. Exposed pad → PCB copper pour for thermal spreading. |
| Q2 | DSG FET | AON6354 30V 83A 5.2mΩ | DFN5×6 | 1 | Driven by BQ76920 DSG pin. Body diode orientation critical — verify against BQ76920 ref schematic. |
| Q3 | Pre-discharge FET | 2N7002 | SOT-23 | 1 | Optional. Soft-starts PVDD caps via R2 — prevents inrush SCD trip. |
| Q_HA/Q_LA | Balancer FETs — inst. A | Si2302 20V 2A ×2 | SOT-23 | 2 | C1↔C2 instance. Low Vgs threshold, TC4427-driven. |
| Q_HB/Q_LB | Balancer FETs — inst. B | Si2302 20V 2A ×2 | SOT-23 | 2 | C2↔C3 instance. |
| Q_HC/Q_LC | Balancer FETs — inst. C | Si2302 20V 2A ×2 | SOT-23 | 2 | C3↔C4 instance. |
| DRV_A–C | Gate drivers | TC4427 dual ×3 | SOT-23-5 | 3 | One per balancer instance. 3.3V input compatible. Buffers ESP32 GPIO. |
| L_A–C | Balancer inductors | 4.7µH Isat ≥3A **shielded** ×3 | 4×4mm | 3 | **Shielded mandatory** — 50kHz unshielded core couples into BQ76920 VC lines. |
| R1 | Pack shunt | **3mΩ ±1% 2W** | 2512 | 1 | Kelvin to BQ76920 SRP/SRN. 56mV OCD max / 3mΩ = 18.7A trip. |
| R2 | Pre-discharge | 10Ω 1W | 2512 | 1 | With Q3. |
| R3, R4 | FET gate R | 1kΩ ×2 | 0402 | 2 | Q1/Q2 main protection FET gate damping. |
| R5 | Q3 pull-down | 100kΩ | 0402 | 1 | Holds Q3 off when control line floating. |
| R6–R9 | VC filter R | 100Ω ×4 | 0402 | 4 | With C6–C9: ~16kHz LPF per VC line. Rejects 50kHz balancer noise. |
| R10–R15 | Balancer gate R | 10Ω ×6 | 0402 | 6 | Switching edge damping per FET gate. |
| R16, R17 | I2C pull-ups | 4.7kΩ ×2 | 0402 | 2 | SDA/SCL to 3.3V, near BQ76920. |
| R18, R19 | BQ76920 REGSRC divider | Per datasheet Table 1 | 0402 | 2 | Sets BQ76920 internal LDO input from pack voltage. |
| C1, C2 | Pack bulk | 100µF 25V elec + 10µF X7R | SMD/0805 | 2 | Absorbs amp transients before sense comparators. |
| C3 | BQ76920 REGSRC | 10µF X7R | 0805 | 1 | Within 1mm of pin. |
| C4, C5 | BQ76920 decoupling | 100nF + 10nF | 0402 | 2 | VCC and DVDD pins. |
| C6–C9 | VC filter C | 0.1µF ×4 | 0402 | 4 | At VC pins to GND. With R6–R9: cutoff ~16kHz. |
| C10–C12 | Balancer bootstrap | 100nF ×3 | 0402 | 3 | One per balancer instance. |
| C13–C22 | ESP32-S3 decoupling | 10µF + 100nF per VDD domain | 0402/0805 | 10 | Five VDD domains: VDD3P3, VDD3P3_RTC, VDD3P3_CPU, VDD_SDIO, VDD3P3_USB. Within 0.5mm each. |
| C23 | LDO output | 10µF X7R | 0805 | 1 | AP2112K output stability. |
| C_X1, C_X2 | Crystal load caps | ~15pF (verify per CL) | 0402 | 2 | Symmetric placement at crystal pads. |
| NTC1 | Cell NTC | 100kΩ B=3950 | leaded | 1 | Epoxied to cell body → BQ76920 TS1. |
| NTC2 | FET NTC | 100kΩ B=3950 | 0402 | 1 | Near Q1/Q2 → BQ76920 TS2. |
| D1 | Reverse polarity | SS34 40V 3A | SMA | 1 | On B+ input from battery. |
| J1 | Cell taps | JST-XH 5-pin | THT | 1 | B−, B1, B2, B3, B+. Keyed. |
| J2 | Load output | XT30U-F right-angle | pads | 1 | P+/P− to amp PVDD. |
| J3 | Charger input | XT30U-F | pads | 1 | CHG+/CHG− from charger board. |
| J4 | Host UART | JST-SH 4-pin 1.0mm | SMD | 1 | GND / 3.3V / TX / RX. |
| J5 | USB programming pads | 4× 2.54mm THT | THT | 1 | VBUS / D− / D+ / GND. Unpopulated in production. |
| J6 | BOOT + EN | 2× tactile or pads | SMD | 2 | Download mode / reset. |
| LED1–4 | Status | G/B/R/Y 0402 ×4 | 0402 | 4 | CHG path / DSG path / ALERT fault / balance active. |

**CAN scaling section (footprints always on PCB — populate only when stacking nodes, ~€5):**

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U4 | Isolated CAN transceiver | TI ISO1050DUB | SOP-8 wide | 1 | Single chip: isolation + CAN transceiver. Logic side 3.3V (ESP32 TWAI), bus side 5V. |
| U5 | Isolated DC-DC | B0305S-1W (3.3V→5V, Mornsun) | SIP-4 | 1 | Powers ISO1050 bus side. Galvanic barrier between node logic and CAN bus ground. |
| C_CAN | Isolation decoupling | 10µF + 100nF ×2 sides | 0402/0805 | 4 | Per ISO1050 + B0305S datasheets. |
| R_TERM | CAN termination | 120Ω | 0805 | 1 | With solder jumper SJ1 — closed only on the two physical end nodes of the bus. |
| SJ1 | Termination jumper | 2-pad solder jumper | — | 1 | — |
| SW1 | Node address | 4-pos DIP switch | SMD | 1 | 16 node IDs, read by ESP32 at boot → CAN ID. |
| J7, J8 | CAN daisy-chain | JST-XH 4-pin ×2 | THT | 2 | CANH / CANL / FAULT / ISO_GND. Ladder-chain node to node. |
| OC1, OC2 | Fault line optos | PC817 ×2 | SMD-4 | 2 | Hardware fault interlock: any node fault trips all nodes faster than CAN. |
| D2 | CAN bus ESD | PESD1CAN | SOT-23 | 1 | CANH/CANL transient protection at connectors. |

### 2.8 Shunt Value Justification

BQ76920 OCD register maximum threshold: 56mV. Therefore:

```
R_sense = 56mV / I_trip_target
I_trip target = ~150% of max continuous = 12A × 1.5 = 18A
R_sense = 56mV / 18.7A = 3mΩ (standard value)
Power at trip: 18.7² × 0.003 = 1.05W → 2W rated 2512
```

Kelvin 4-wire connection to BQ76920 SRP/SRN pins is mandatory — see layout §2.9.

### 2.9 BMS Layout Rules

**High-current path** (`B− → D1 → Q1 → shared node → R1 → Q2 → P−`): polygon pours ≥3.5mm at 2oz. Cluster 6–8 vias (0.4mm drill) in parallel if layer change unavoidable. No single vias on 12A+ paths.

**Kelvin sensing on R1 — most critical rule:**

```
WRONG:  sense taps anywhere on power pour near R1
CORRECT: sense taps DIRECTLY at shunt end pads,
         0.15mm dedicated traces to BQ76920 SRP/SRN pins only
```

At 3mΩ nominal: 0.5mΩ trace error = 17% OCP threshold shift. Not a refinement — a functional requirement.

**VC sense lines:** JST tap → 100Ω → VC pin, 0.1µF to GND at pin. Minimum 3mm from any balancer inductor. Cross power polygon only perpendicular, opposite layer.

**Balancer placement:** each instance adjacent to its two cell-tap nodes. Inductor between FET pair SW node and taps — minimal loop area. TC4427 near gates, not near ESP32. Shielded inductors only.

**PCB trace antenna — first-priority placement:**
- Antenna at board corner or edge facing outward
- Keepout void on ALL layers including inner plane — specify explicitly in KiCad, verify in gerber viewer
- All switching sources on opposite end of board
- Antenna trace width ~0.38–0.42mm (verify from actual JLC7628 stackup with calculator)

**Crystal:** guard ring stitched to layer 2 every 1mm. No switching traces within 3mm. XTAL_P/N traces ≤5mm, no vias.

**CAN isolation gap:** no copper crosses the ISO1050/B0305S barrier on any layer. ≥4mm creepage. Bus-side ground pour is an isolated island fed only by B0305S output.

**Ground plane:** continuous layer 2 (inner GND plane), no split — except antenna keepout void and CAN isolation island. BQ76920 AGND and DGND both to same plane per datasheet.

**USB D+/D−:** matched-length differential pair to J5 pads, no vias.

---

## 3. CAN Scalability — Designed-In Optionality

### 3.1 The Concept

```
16S4P powerwall = 4× this node in series, CAN sections populated

  Node 4 (48V level)  ──┐
  Node 3 (32V level)  ──┤  isolated CAN bus ──→ Master aggregator
  Node 2 (16V level)  ──┤  (twisted pair)        (Raspberry Pi Zero 2 W)
  Node 1 (0V level)   ──┘
```

Each node treats its own 4S bank as its entire universe. Series current is common — any node opening its FETs breaks the entire string. Local autonomous protection = global protection. Master coordinates globally but no node depends on it for safety.

### 3.2 Why Galvanic Isolation Is Non-Negotiable

Node 2's battery ground sits 14.8V above Node 1's; Node 4's sits ~44V above Node 1's. Shared CAN ground without isolation = dead short through the communication wiring. The ISO1050 + B0305S create a floating data network spanning the full stack potential — the master sees only data packets, never pack voltages.

**Single-node speaker build:** leave U4, U5, optos, DIP, J7/J8 unpopulated. Footprints cost zero.

### 3.3 Bus Design Rules

- Bitrate: 500kbps
- Topology: linear daisy-chain via J7/J8 — no stubs, no star
- Termination: 120Ω at the two physical end nodes only (SJ1)
- Addressing: 4-bit DIP per node → CAN ID at boot
- Wiring: twisted pair CANH/CANL + FAULT + ISO_GND in same 4-pin cable
- Babbling node: master watchdogs per-node message rates; hardware FAULT line provides safety independent of CAN health

### 3.4 Hardware Fault Line

Open-drain FAULT line through daisy cable, opto-coupled at every node (PC817 ×2: assert + sense). Any node fault pulls line low → every other node's ESP32 wakes immediately and opens FETs. Hardware-speed, firmware-independent.

### 3.5 Balancing Authority in a Stack

Each node balances its own 4 groups autonomously. Inter-node balance is the master's job — it observes all nodes over CAN and adjusts per-node charge limits. Nodes never fight each other because none acts on data outside its own bank without a master command.

---

## 4. Charger Board

### 4.1 Architecture Overview

The charger board uses two ICs with a clean separation of responsibility:

- **FUSB302:** USB-C PD physical layer. Detects what is plugged in (source or sink via CC pin monitoring). Negotiates PD voltage as sink. Advertises capabilities as source. Communicates everything to the amp ESP32 over I2C.
- **BQ25713B:** 4-switch buck-boost charger controller. Receives instructions from amp ESP32 over I2C. Charges the 4S pack in sink mode. Outputs 5V or 9V on VBUS in OTG mode. Never makes autonomous role decisions — the ESP32 is always the decision maker.

The amp ESP32 is the arbiter:

```
Adapter plugged in:
  FUSB302 detects Rp on CC (source) → interrupt → ESP32 reads status
  ESP32 negotiates PD voltage via FUSB302 → VBUS = 20V (or best available)
  ESP32 writes BQ25713B charge registers (Ichg, Vchg, input current limit)
  BQ25713B charges pack

Phone/device plugged in:
  FUSB302 detects Rd on CC (sink) → interrupt → ESP32 reads status
  ESP32 writes BQ25713B OTG enable register + OTG voltage (5V or 9V)
  BQ25713B outputs on VBUS
  FUSB302 advertises Source_Capabilities if device supports PD
  Device negotiates → ESP32 responds via FUSB302

Nothing plugged in:
  FUSB302 idle, BQ25713B idle, ESP32 sleeping
```

**Role conflict safety:** the ESP32 never enables both charge and OTG simultaneously. OTG enable is a register bit written only after FUSB302 confirms sink role. Charge path enabled only after source role confirmed. The hardware interlock: BQ25713B has an ACOK pin that indicates valid input present — OTG mode is only entered when ACOK is low (no adapter present).

### 4.2 FUSB302 — USB-C PD Physical Layer

The FUSB302 (onsemi, FUSB302B variant) handles all USB-C physical layer operation:

- CC1/CC2 monitoring — detects Rp (source attached) or Rd (sink attached)
- DRP toggling — for connecting to other DRP devices (laptops)
- PD message physical layer (BMC encoding/decoding on CC lines)
- I2C interface to ESP32 (address 0x22)
- Interrupt line to ESP32 GPIO — fires on any CC event, attachment, detachment, PD message
- VCONN switch control for cable e-marker chips

The ESP32 runs a PD policy engine firmware layer on top — open-source implementations (pd_buddy, fusb302-esp) handle the PD state machine. The FUSB302 handles bits; the firmware handles protocol.

**FUSB302 availability:** onsemi part, Mouser/Digikey consistently stocked, ~$1.50.

### 4.3 BQ25713B — 4-Switch Buck-Boost Charger Controller

TI BQ25713B: synchronous NVDC buck-boost battery charge controller, 1–4S, I2C.

**Why BQ25713B:**
- 3.5–24V input range — accepts all PD voltages (5V, 9V, 12V, 15V, 20V) plus legacy adapters and barrel jacks. Works with any charger, not just USB-PD.
- Automatic buck/boost/buck-boost mode transition without host control — seamless across all input voltages
- 4× external N-channel FETs — allows FET optimisation for this voltage/current range
- Single inductor topology
- I2C (SMBus compatible) — amp ESP32 sets charge current, input current limit, reads VBUS, IBAT, VBAT, fault status
- OTG output: 4.48V–20.8V adjustable, up to 3A from battery — 5V/3A = 15W, 9V/2.2A = 20W
- Integrated ADC: VBUS, IBAT, VBAT, VSYS all readable over I2C — useful for the BMS webapp display
- ACOK pin: indicates valid input present — used as hardware interlock for OTG enable
- CHRG_OK, PROCHOT, IBAT monitor outputs
- ~$3.20 at Digikey 100 units, 228 units in stock (verify at order time; TI part with regular restocking)
- 32-QFN (4×4mm) — ENIG + stencil, same class as other ICs in this design

**External FETs — 4× AO4406 (or equivalent):**
- 30V Vds, 10A Id, Rds(on) ~10mΩ @ Vgs=4.5V, SO-8
- At 2.5A charge current: P = 2.5² × 0.04 (all 4 FETs) = 0.25W total — trivial
- Gate drive from BQ25713B integrated gate drivers — no external driver needed

**Single inductor:**
- Value: 2.2–4.7µH (per BQ25713B datasheet recommendation for 4S)
- Isat: ≥5A (charge current 2.5A + ripple)
- Shielded, DCR <50mΩ
- Recommend: Bourns SRR6028-4R7Y or Sunlord SWPA6028S4R7MT

### 4.4 OTG Operation and Safety

BQ25713B OTG output is explicitly controlled by the ESP32 via I2C register (OTG_CONFIG bits). The safety chain:

1. FUSB302 fires interrupt — sink device detected on CC
2. ESP32 reads FUSB302: Rd confirmed, role = source
3. ESP32 checks BQ25713B ACOK: must be LOW (no adapter present) — if HIGH (adapter present), OTG is blocked
4. ESP32 writes OTG_VOLTAGE and OTG_CURRENT registers
5. ESP32 sets OTG_CONFIG enable bit
6. BQ25713B activates OTG path
7. FUSB302 advertises 5V/3A (or 9V/2.2A if PD device) on CC

On adapter insertion during OTG: ACOK goes HIGH → ESP32 detects via interrupt → immediately clears OTG enable bit → BQ25713B switches to charge mode. No power contention.

### 4.5 Charger Board — Full BOM

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | PD physical layer | FUSB302BMPX (onsemi) | QFN-16 | 1 | CC1/CC2 detect, DRP, PD BMC, I2C 0x22. Mouser/Digikey stocked ~$1.50. |
| U2 | Buck-boost charger | BQ25713B (TI) | QFN-32 (4×4) | 1 | 3.5–24V in, 1–4S CC/CV out, OTG 4.48–20.8V, I2C. ~$3.20/100 Digikey. |
| Q1–Q4 | Buck-boost FETs | AON6354 30V 83A 5.2mΩ ×4 | DFN5×6 | 4 | 4-switch stage. Gate drive from BQ25713B integrated drivers. Exposed pad soldered to PCB copper pour. At 2.5A charge: P_total = 0.13W — negligible. Same part as BMS FETs — single FET across entire BOM. |
| L1 | Main inductor | 4.7µH Isat ≥5A DCR<50mΩ shielded | 6×6mm | 1 | Single inductor for 4-switch stage. Shielded. Bourns SRR6028-4R7Y. |
| C1, C2 | VBUS bulk | 22µF 25V X7R ×2 | 1210 | 2 | Input rail. Two in parallel for low ESR. 25V rated for 20V PD margin. |
| C3, C4 | Battery output bulk | 47µF 25V X7R ×2 | 1210 | 2 | Battery output. Charge ripple reduction. |
| C5–C10 | IC decoupling | 100nF ×6 + 10nF ×2 | 0402 | 8 | Per supply pin, within 0.5mm. |
| C11 | Bootstrap cap | 100nF X7R | 0402 | 1 | Per BQ25713B datasheet bootstrap requirements. |
| C12, C13 | FUSB302 decoupling | 100nF + 1µF | 0402/0805 | 2 | VDD and VCONN supply pins. |
| R_ISET | Charge current | Per BQ25713B Table — or I2C only | 0402 | 1 | Optional: sets minimum charge current floor. Primary current set via I2C. |
| R_SNS | Charge current shunt | 10mΩ ±1% 1W | 2512 | 1 | BQ25713B IADPT/IBAT sensing. Kelvin connections. |
| R1, R2 | I2C pull-ups | 4.7kΩ ×2 | 0402 | 2 | SDA/SCL for shared I2C bus (FUSB302 + BQ25713B). |
| R3 | FUSB302 VCONN R | 1kΩ | 0402 | 1 | VCONN current limit per FUSB302 application circuit. |
| NTC1 | Battery NTC | 100kΩ B=3950 | 0402 | 1 | BQ25713B TS pin. Charge derating above 40°C, inhibit above 50°C. |
| D1 | VBUS TVS | SMAJ20A 400W | SMA | 1 | Clamps VBUS transients. Between USB-C VBUS and BQ25713B input. |
| F1 | Polyfuse | 2.5A hold / ~5A trip | 1812 | 1 | VBUS overcurrent before IC. Self-resetting. |
| J1 | USB-C receptacle | HRO TYPE-C-31-M-12 or equiv — **16-pin** | SMD mid-mount | 1 | CC1/CC2 to FUSB302. VBUS/GND full copper fill, thermal relief OFF. 4-pin USB-C has no CC lines and cannot do PD. |
| J2 | Cell tap pass-through | JST-XH 5-pin | THT | 1 | B1/B2/B3 mid-taps from charger side to BMS. |
| J3 | Power output | XT30U-M | pads | 1 | CHG+/CHG− to BMS J3. |
| LED1 | Charging | Green 0402 | 0402 | 1 | BQ25713B CHRG_OK pin. |
| LED2 | OTG active | Blue 0402 | 0402 | 1 | Driven by ESP32 GPIO when OTG enabled. |
| LED3 | Fault | Red 0402 | 0402 | 1 | BQ25713B FAULT or ACOK status. |

### 4.6 Charger Layout Rules

**BQ25713B QFN-32 thermal pad:** 9 minimum vias (3×3), 0.3mm drill, tented bottom. 15×15mm 2oz bottom pour. On 4-layer board, inner GND plane provides primary thermal mass.

**Inductor placement:** as close as possible to BQ25713B SW1/SW2 pins. Every mm of trace = parasitic inductance = EMI + efficiency loss. Input caps between USB-C and IC; output caps between IC and output connector.

**FET placement:** 4× AO4407 arranged to minimise switching loop area. Gate resistors close to FET gates. Body diode orientation per BQ25713B reference schematic — verify before routing.

**FUSB302:** CC1/CC2 traces 0.15mm, direct to IC pins, minimal length. No routing under the IC that could couple into CC lines. VBUS trace wide (≥2mm) from connector to TVS to BQ25713B input.

**USB-C connector:** VBUS/GND pads — solid copper fill, thermal relief OFF (5A through these pads). CC1/CC2 — 0.15mm signal traces.

**R_SNS Kelvin:** same rule as BMS R1 — sense taps directly at resistor end pads, dedicated 0.15mm traces to BQ25713B IADPT+/IADPT− pins.

---

## 5. PCB Layer Count Decision

Both boards are 4-layer. Same JLCPCB process, same stackup (JLC7628, 1.6mm), one order.

| Board | Layers | Copper | Reasoning |
|---|---|---|---|
| BMS node | **4** | 2oz outer / 1oz inner | Bare ESP32-S3 SoC PCB trace antenna requires solid inner GND plane (layer 2) for microstrip impedance reference. Antenna keepout void must clear ALL layers including inner planes — impossible to implement correctly on 2-layer. 50kHz balancer FETs beside ±1mV sense lines benefit from plane separation. SoC QFN-56 thermal via stack. Crystal guard ring. Module's internal ground plane and shield can would handle this; bare SoC on 2-layer does not. |
| Charger | **4** | 2oz outer / 1oz inner | BQ25713B QFN at 2–4W needs inner GND plane (drops θJA ~30–40%); 800kHz 4-switch switching nodes need reference plane 0.2mm below; FUSB302 CC lines beside 5A VBUS benefit from shielding. Without 4-layer, high-power OTG pushes toward thermal derating. |

**Stackup:** Signal top (2oz) / GND plane (1oz) / Power or signal (1oz) / Signal bottom (2oz). JLC7628 standard prepreg.

> **Antenna keepout on BMS board:** inner GND plane (layer 2) must have copper void matching the top-layer antenna keepout boundary. Specify this void in KiCad inner layer and verify in 3D viewer + DRC before ordering. If this void is missing, the antenna microstrip impedance is undefined and RF performance is unpredictable.

---

## 6. Thermal Management

No heatsink required on either board. All components operate well within safe temperature limits.

### 6.1 BMS Board — Full Thermal Analysis

| Component | Dissipation | Package / θJA | ΔT @ 25°C ambient | T_junction |
|---|---|---|---|---|
| Q1 + Q2 (AON6354 ×2) | 12² × 0.0104 = **1.5W total** | DFN5×6, exposed pad to PCB pour ~30°C/W | 45°C | ~70°C ✅ |
| R1 shunt (3mΩ) | 12² × 0.003 = **0.43W** | 2512 SMD, self-limiting | ~30°C | ~55°C ✅ |
| Balancer FETs (Si2302 ×6) | ~0.1W per pair × 3 = **0.3W** (when active) | SOT-23, low duty cycle | ~10°C | ~35°C ✅ |
| BQ76920 | ~0.05W (µA quiescent) | TSSOP-20 | negligible | ~27°C ✅ |
| ESP32-S3 | ~0.26W average active | QFN-56, copper pour | ~15°C | ~40°C ✅ |

Total board dissipation: ~2.3W maximum. No single component stressed. DFN5×6 exposed pad is critical — it must be soldered to a solid copper pour (not just the DFN land pattern). Add thermal vias under the exposed pad to the inner ground plane for maximum spreading.

### 6.2 Charger Board — Full Thermal Analysis

| Component | Dissipation | Package / θJA | ΔT @ 25°C ambient | T_junction |
|---|---|---|---|---|
| Q1–Q4 (AON6354 ×4) | 2.5² × 0.021 = **0.13W total** (conduction) + ~0.4W switching @ 800kHz = ~**0.5W total** | DFN5×6 ×4, exposed pads to pours | ~15°C per FET | ~40°C ✅ |
| BQ25713B controller | ~0.3–0.5W (gate drive + logic only — no switching losses in controller) | QFN-32 4×4, 4-layer inner plane ~35°C/W | ~18°C | ~43°C ✅ |
| FUSB302 | ~0.05W | QFN-16 | negligible | ~27°C ✅ |

Total board dissipation: ~1W at full charge. Switching losses in the FETs (not the controller IC) are spread across 4× DFN5×6 packages with exposed pads — excellent thermal distribution.

**Why no heatsink is needed:** the controller topology (BQ25713B) separates switching losses from the controller IC. The AON6354's ultra-low Rds(on) of 5.2mΩ means conduction losses are trivial. Switching losses at 2.5A are modest. All distributed across 4 packages each with exposed thermal pads. A heatsink would add mechanical complexity with zero measurable benefit.

### 6.3 DFN5×6 Exposed Pad — Layout Requirement

The AON6354 DFN5×6 exposed bottom pad is the drain connection and the primary thermal path. On both boards:
- Connect exposed pad to the drain net copper pour
- Add 4× thermal vias (0.3mm drill) under each pad to the inner GND/power plane
- Solid 2oz copper pour minimum 8×8mm around each device
- Do **not** use SO-8 footprint — DFN5×6 land pattern is different. Use the correct AOS DFN5×6 footprint from KiCad library or create from datasheet dimensions

---

## 7. ESP32-S3 Firmware Architecture

### 7.1 State Machine

```
                    DEEP SLEEP (~10µA)
        ┌──────────────┬──────────────┬──────────────┐
     RTC timer      ALERT GPIO     UART RX       CAN RX*
     (60s, NVS-     (BQ76920       (host cmd)    (*node mode)
     configurable)   fault)
        │              │              │              │
        ▼              ▼              ▼              ▼
   BALANCE CYCLE   FAULT HANDLER  HOST COMMS    CAN HANDLER
        └──────────────┴──────┬───────┴──────────────┘
                              ▼
            Read BQ76920 (VC1–4, current, temps, faults)
            Update coulomb count + SOC (RTC memory)
            Balance if Δ>30mV (≤30s, LEDC 50kHz, dead-time)
            Log / respond / broadcast as applicable
            Update LEDs → return to DEEP SLEEP
```

RTC memory (8KB, survives sleep): SOC accumulator, coulomb count, fault history, cycle count, balance interval setting.

### 7.2 Balancing Algorithm

Every wake: read 4 voltages → find max/min → if Δ>30mV select transfer path (direct adjacent or chained for non-adjacent) → drive instance(s) for ≤30s via LEDC 50kHz with firmware dead-time → re-read → sleep. Runs identically charging, discharging, or at rest. SOC drift-corrects against OCV table after ≥5min rest.

### 7.3 WiFi Webapp Mode

No UART host for >60s after boot → join configured WiFi (NVS) or AP mode `BMS_[MAC4]`. LittleFS-served dashboard at `192.168.4.1`: live cell group voltages, SOC%, pack current (charge/discharge), temperatures, fault log, balance status. Exits on first UART command.

### 7.4 Node Mode (CAN populated)

DIP-read node ID at boot → periodic CAN frames (voltages, SOC, current, temp, status) → obeys master commands → asserts/reacts to hardware FAULT line independent of CAN bus health. TWAI peripheral + ISO1050, wake-on-CAN via RX GPIO edge.

---

## 8. Host Communication Protocol

UART 115200 8N1, ASCII + `\n`. Send `PING` first to wake BMS (~5ms response).

| Command | Response | Description |
|---|---|---|
| `PING` | `PONG` | Wake + connectivity check |
| `PACK_VOLTAGE` | `14823` | Total pack voltage in mV |
| `PACK_CURRENT` | `-2340` | mA, negative = discharging |
| `PACK_SOC` | `73` | State of charge % |
| `PACK_CAPACITY` | `5840` | Remaining capacity mAh |
| `PACK_TEMP` | `31` | Cell NTC temperature °C |
| `CELL_1` … `CELL_4` | `3721` | Individual group voltage mV |
| `CELL_ALL` | `3721,3718,3722,3719` | All 4 groups mV, comma-separated |
| `BALANCE_STATE` | `ACTIVE:1-2` / `IDLE` | Current balancer state |
| `FAULT_STATE` | `NONE` / `OVP:CELL3` | Current fault |
| `FAULT_LOG` | last 10 timestamped events | — |
| `STATUS` | `CHARGING/DISCHARGING/IDLE/FAULT` | Pack state |
| `CYCLE_COUNT` | `47` | Charge cycles logged |
| `SET_BALANCE_INTERVAL:120` | `ACK` | Set interval seconds, NVS-persisted |
| `VERSION` | `BMS_FW_1.0.0` | Firmware version |

**Unsolicited fault alert:** `FAULT:OVP:CELL3\n` within ~10ms of ALERT pin firing.

**Graceful shutdown warnings:**
- `WARN:LOW_SOC:10\n` at 10% SOC
- `WARN:LOW_SOC:5\n` at 5% SOC

These arrive before UVP hardware cutoff — the amp ESP32 fades audio and mutes TAS5825M cleanly. No mid-song hard cut, no pop.

**Companion library (`BMS_Client`, to be written):**

```cpp
#include "BMS_Client.h"
BMS_Client bms(Serial1);
bms.begin(115200);
bms.ping();
int soc      = bms.getSOC();
int voltage  = bms.getPackVoltage();   // mV
int current  = bms.getPackCurrent();   // mA signed
CellData c   = bms.getAllCells();      // c.cell[0..3] mV
String fault = bms.getFaultState();
bms.onFault(myFaultCallback);          // unsolicited FAULT: handler
bms.onLowSOC(myShutdownCallback);      // WARN:LOW_SOC: handler
```

Library handles wake-on-PING, timeout/retry, response parsing, and unsolicited message callbacks.

---

## 9. PCB Ordering & Build Procedure

### 9.1 JLCPCB Order Settings

| Parameter | BMS Node | Charger Board |
|---|---|---|
| Layers / copper | **4 / 2oz outer, 1oz inner** | **4 / 2oz outer, 1oz inner** |
| Surface finish | ENIG | ENIG |
| Via tenting | Open (inner plane antenna keepout void in gerbers) | Tented bottom under BQ25713B (fab note) |
| Stencil | Frameless, yes | Frameless, yes |
| Thickness | 1.6mm | 1.6mm |
| Silkscreen | Both sides | Both sides |

Stencil is mandatory for QFN-56, QFN-32, QFN-16, TSSOP-20. Frameless ~$2–5 at order time.

Tented bottom on charger: add fab note "tent vias under U2 QFN thermal pad on bottom side."

Antenna keepout: add note "inner copper layer 2 void in antenna keepout zone as shown in gerber — do not fill."

### 9.2 Reflow Profile — SAC305

| Zone | Rate / Temp | Notes |
|---|---|---|
| Preheat | 25→150°C @ 2°C/s | Activates flux, drives off paste volatiles |
| Soak | 150→200°C @ 1°C/s, hold ~90s | Thermal equalization — critical for large QFN center pads |
| Reflow | Peak 245°C, 20–30s above 217°C | SAC305 liquidus = 217°C |
| Cooling | <3°C/s to 100°C, then free-air | Fine-grain microstructure, stronger joints |

Post-reflow: probe continuity from QFN center pad thermal via to GND net. No continuity = center pad did not wet — apply flux, hot-air at 250°C until package settles (surface tension pull-down). Inspect TSSOP-20 under 10× magnification.

### 9.3 Assembly Order

1. Stencil paste top side. Inspect aperture fill under magnification.
2. 0402 passives (caps, resistors, crystal load caps).
3. ICs: BQ76920 (TSSOP-20), ESP32-S3 (QFN-56), BQ25713B (QFN-32), FUSB302 (QFN-16), TC4427 ×3, Si2302 ×6, AP2112K, AO4406 ×4.
4. Crystal (3225) — heat-sensitive, place carefully.
5. SO-8 protection FETs (AO4407A), shunt resistors (2512).
6. Shielded inductors — bulky, place last.
7. Reflow top side.
8. Inspect, touch up, IPA clean.
9. Flip — hand-solder THT (JST-XH, JST-SH, XT30 pads, USB programming pads, tactile buttons).
10. Continuity checks (§10.1).
11. **Flash ESP32-S3 firmware via USB pads before any battery connection.**

> ⛔ Never connect battery to an unverified board. First power-on always through a current-limited bench PSU at 100mA.

---

## 10. Test & Verification Procedure

### 10.1 Pre-Power Checks

- Visual under magnification: bridges on QFN and TSSOP packages, cold joints on 2512 shunts, FET orientation per schematic
- B+ to P−: >1MΩ (both FETs off, no gate drive)
- R1 shunt: 4-wire Kelvin = 3mΩ ±1%. Reading >50mΩ = cold joint.
- NTCs at 25°C ambient: 97–103kΩ
- Crystal load cap footprint: verify correct value soldered (common to mix 10pF and 15pF)
- USB D+/D− pads: continuity to SoC pins, no short to GND

### 10.2 ESP32-S3 Flash and Boot

Cut USB cable to J5 pads → pull BOOT low + pulse EN → flash firmware → serial monitor shows boot message, BQ76920 I2C OK (`0x08` found), initial VC reads reasonable.

### 10.3 BMS Protection Verification (4× bench PSU @ 3.700V as cell groups)

- **Nominal:** P+−P− = 14.8V, ALERT pin high, both FETs conducting
- **OVP:** raise one cell >4.21V → CHG FET off, DSG unaffected, ALERT low, ESP32 wakes ≤10ms, fault logged. Drop to 4.10V → auto-recovery.
- **UVP:** lower one cell <2.79V → DSG off, CHG path alive. Raise >2.90V → auto-recovery.
- **OCD:** load toward 18.7A → DSG off after register-set delay. Non-latching — auto-recovers when current drops.
- **SCD:** momentary 0.1Ω short → DSG off within 400µs. **Verify latch** — load reconnect does not re-enable DSG until fully removed and re-engaged.
- **ALERT→ESP32:** confirm wake within 10ms of ALERT falling edge in each test above.

### 10.4 Balancer Verification

Set adjacent simulated cells to 3.800V and 3.650V (150mV delta, above 30mV threshold).
- Oscilloscope on Q_H/Q_L gate pair: confirm 50kHz complementary drive with dead-time gap
- Inline ammeter on higher-voltage cell PSU: confirms current flowing out (transferring to lower cell)
- Delta closes over 2–3 minutes

Repeat for each of the 3 instances. Test a non-adjacent transfer (C1 and C4 offset) — confirm both intermediate instances activate as needed.

### 10.5 UART Protocol Verification

Connect CP2102 to J4 → send `PING` → `PONG` within 10ms (awake) or 50ms (from sleep) → send `CELL_ALL` → four mV values matching PSU set points ±5mV → trigger OVP → confirm `FAULT:OVP:CELL1` arrives unsolicited within 100ms → `SET_BALANCE_INTERVAL:120` → `ACK` → power cycle → verify interval persisted in NVS.

### 10.6 Charger Board Verification

- FUSB302 I2C: ESP32 reads device ID register 0x01 = 0x91 (confirmed FUSB302B)
- Adapter connection: FUSB302 interrupt fires → ESP32 logs CC attach, negotiates 20V → measure VBUS = 20V ±0.2V
- 22Ω dummy load on BQ25713B output: verify CC/CV regulation to 16.8V ±0.1V
- Charge current: measure ~2.5A ±10% via ammeter
- OTG test: disconnect adapter, connect phone → FUSB302 detects Rd → ESP32 enables OTG → VBUS = 5V, phone begins charging
- ACOK interlock: with OTG active, plug in adapter → OTG must disable automatically within one ESP32 interrupt cycle

### 10.7 Full System Integration

- All boards connected, real or simulated pack, TAS5825M or 22Ω dummy on P+/P−
- Amp ESP32 sends `PING` → `PONG`; `CELL_ALL` returns valid group voltages
- Charge cycle: adapter in, BQ25713B charges, BQ76920 reports increasing VBAT over I2C
- Discharge/playback: adapter out, load draws power, SOC decrements
- Loud bass audio: no OCD nuisance trips. If trips occur: increase OCD delay register in BQ76920 and retest.
- Low SOC test: run pack to ~12% → confirm `WARN:LOW_SOC:10` arrives on UART → amp ESP32 mutes TAS5825M → `WARN:LOW_SOC:5` → graceful shutdown before UVP hardware cutoff
- Thermal 30 minutes full load: FETs <65°C, BQ25713B controller <70°C, cells <40°C

---

## 11. System Integration & Multi-Node Scaling

### 11.1 Speaker Integration (Primary Use)

- **Cells:** 8× same-batch Samsung 30Q (or 35E / NCR18650GA). Spot-welded nickel strip preferred. 14AWG mains <200mm. 26AWG taps away from power leads.
- **Connectors:** XT30 power (male on board, female on cable). JST-XH cell taps. JST-SH UART. No dupont on any current-carrying path.
- **PVDD decoupling at TAS5825M:** 2×470µF low-ESR + 4×10µF X7R (1210) + 4×100nF 0402. Absorbs bass transients before they propagate back through BMS shunt.
- **Mounting:** M3 nylon standoffs ≥10mm. ≥10mm clearance between charger board and audio signal traces. Antenna edge of BMS board faces outward (toward speaker vent or clear enclosure wall).

### 11.2 Scaling Checklist (Future)

1. Populate CAN section on each node (~€5 each), set unique DIP IDs
2. Stack packs in series (Node1 P+ → Node2 P−, etc.)
3. Isolated DC-DC per node (B0305S) powers each CAN bus side independently
4. Daisy-chain 4-wire CAN cable (CANH/CANL twisted, FAULT, ISO_GND)
5. 120Ω termination solder jumper SJ1: closed only at the two physical end nodes
6. Master aggregator (Pi Zero 2W) on the floating CAN bus: logging, GUI, inter-node balancing authority
7. Verify FAULT line trips entire stack from every node before first real charge

### 11.3 Portfolio / Curriculum Summary

This project demonstrates: multi-cell Li-ion protection and management · 4-switch synchronous buck-boost with external FET design · USB-C PD negotiation (sink and source) via FUSB302 + firmware policy engine · inductive active cell balancing · bare ESP32-S3 SoC RF layout (PCB trace antenna, microstrip impedance, crystal design) · BQ76920 analog front-end integration with Kelvin sensing · deep-sleep embedded firmware architecture with interrupt-driven wake · I2C multi-device bus management · UART command protocol design and companion library architecture · distributed CAN bus architecture with galvanic isolation · hardware fault interlock design · 4-layer PCB with thermal via stack, inner GND plane, and antenna keepout · multi-stage test methodology with threshold measurement and verification.

---

## 12. Key Design Decisions Reference

| Decision | Choice | Rationale |
|---|---|---|
| Protection IC | BQ76920 | ±1mV accuracy, I2C to ESP32, configurable thresholds, hardware defaults active before/without firmware. Replaces S-8254AA entirely — no redundant sense lines. Mouser/Digikey stocked. |
| Active balancing | Discrete Si2302 ×6 + TC4427 ×3 + 4.7µH ×3, ESP32 LEDC 50kHz | No sourceable autonomous balancer IC exists. Commodity parts, any-to-any transfer, full firmware control. BQ76920 BALx passive path unused. Runs every 60s wake regardless of charge/discharge state. |
| BMS LDO | **AP2112K-3.3TRG1** | 600mA, 300mV dropout, SOT-23-5. Low quiescent current — critical for deep sleep power budget. Well stocked. |
| Shunt | 3mΩ 1% 2W | BQ76920 OCD max register = 56mV. At 8mΩ max trip = 7A (wrong). At 3mΩ: 18.7A trip (correct for this load). |
| MCU | ESP32-S3 bare SoC + W25Q64 + 40MHz crystal | Full RF layout control, smaller footprint vs module. Requires 4-layer board for antenna keepout and microstrip reference plane. Native USB = flash + JTAG + serial via 4 pads, no bridge IC. |
| BMS board layers | **4-layer** (revised from 2-layer) | Bare SoC PCB trace antenna needs inner GND plane for microstrip impedance and all-layer keepout void. Module would allow 2-layer; bare SoC does not. |
| Charger board layers | **4-layer** | BQ25713B + FET switching EMI, CC line integrity beside 5A VBUS, inner GND plane thermal mass. |
| Charger IC | **BQ25713B** | 3.5–24V input (all PD voltages + legacy chargers), 4S native, I2C, external FETs, OTG 5V/3A or 9V/2.2A, ~$3.20 Digikey 100 units. Sourcing confirmed. |
| PD negotiation | **FUSB302** (already on hand) | Handles CC detection, DRP toggling, PD BMC. ESP32 runs PD policy engine (open-source pd_buddy/fusb302-esp). Clean separation: FUSB302 handles bits, firmware handles protocol. |
| OTG | Enabled via BQ25713B register | FUSB302 detects sink, ESP32 confirms ACOK=LOW (no adapter), writes OTG register. Hardware interlock prevents simultaneous charge + OTG. Scraps separate USB-A + AP62300 boost — OTG comes free with BQ25713B. |
| FETs (all positions) | **AON6354 DFN5×6** | 30V, 83A, 5.2mΩ @ Vgs=4.5V. Single part across entire BOM: BMS CHG/DSG + charger 4-switch stage. Exposed drain pad bonds directly to PCB copper pour — better thermal spreading than SO-8. No heatsink needed on either board. |
| Heatsink | **None required** | AON6354 ultra-low Rds distributes losses trivially. BQ25713B controller dissipates ~0.5W (no switching losses in controller IC). Total charger board dissipation ~1W. No single component stressed. |
| Inductor (charger) | 4.7µH Isat ≥5A shielded, single | BQ25713B uses single-inductor 4-switch topology. Isat ≥5A covers 2.5A charge + ripple margin. |
| Scaling model | Stack identical nodes over isolated CAN | One PCB design: speaker → powerwall. ISO1050 + B0305S-1W + dual daisy ports + DIP ID + 120Ω jumper as ~€5 unpopulated optionality. |
| USB-A output | **Removed** | BQ25713B OTG provides 5V/3A (15W) or 9V/2.2A (20W) from same USB-C port. Separate USB-A + AP62300 + inductor eliminated. |

---

*End of document — Rev 6.0*
