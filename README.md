# 4S Lithium Battery System — Engineering Design & Build Reference

**Scalable Smart BMS Node · USB-C PD Charger Board**  
Primary application: Bluetooth Speaker / TAS5825M Amplifier Platform  
Architecture: Distributed modular BMS — one node now, stackable via CAN later  
*Revision 5.0 · June 2026 · Arik Nel*

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

**Scaling is not done inside the node — it is done by stacking nodes.** Need a 16S4P powerwall later? Stack four nodes in series, populate the CAN/isolation section (~€5 of parts per board), and they join a shared bus reporting to a master aggregator. Same PCB, same firmware, no redesign. The node treats its own 4S bank as its entire universe: if its bank is unsafe, it acts locally and immediately — opening its FETs breaks the entire series string's current path.

For the speaker build: don't populate the CAN section. The node runs standalone with UART to the amp board's ESP32.

### 1.2 Power Architecture Summary

| Stage | Function | Key Spec |
|---|---|---|
| USB-C PD Input | Negotiate 5–20V from any adapter | PD2.0 / PD3.0 / QC3.0, bidirectional |
| IP2368 Buck-Boost | Charge: PD → 16.8V CC/CV. Discharge: pack → USB-C OTG | Up to 65W both directions |
| 4S2P Li-ion Pack | 14.8V nom / 16.8V full / 12.0V cutoff | 8× 18650, 2 per series group |
| BQ76920 | All measurement + hardware protection | ±1mV cell voltage, pack current, 2× NTC, OVP/UVP/OCD/SCD/OCC |
| ESP32-S3 | Intelligence: balancing, SOC, comms, webapp | Deep sleep ~10µA, native USB |
| Active Balancer | 3× discrete FET pairs + inductors, ESP32-driven | Inductive transfer, any-to-any, ~88% efficient |
| Si2301 Isolation FET | Blocks back-feed into IP2368 when unpowered | P-ch, auto on/off from IC VCC |
| TAS5825M Amp | PVDD from BMS P+/P− | 16.8V → ~2×25W @ 4Ω |

### 1.3 Measurement — BQ76920 Covers Everything, No Extra ADC

An important simplification: **the BQ76920 is the complete measurement system for this node.** It provides:

- All 4 series group voltages at ±1mV accuracy (VC1–VC4 inputs)
- Pack current via the 3mΩ shunt (integrated current sense amplifier, 16-bit coulomb counter support)
- Two NTC temperature channels (TS1, TS2)

In a 2P4S pack, the two cells within each parallel group share terminals and are at identical voltage by physics — there is nothing additional to measure per group. Per-cell current monitoring inside parallel groups (shunt-per-cell + amplifier trees) was evaluated and deliberately rejected: it adds BOM and wiring complexity to detect a failure mode (single-cell divergence inside a 2P group) that matched same-batch cells rarely exhibit within product lifetime, and scaling is handled at the node level instead. **Zero additional ADC components needed.**

### 1.4 Cell Configuration: 4S2P with 18650

| Parameter | Value (Samsung 30Q, 2P) |
|---|---|
| Capacity | 6000 mAh |
| Max continuous discharge | 30A (cells) — but TAS5825M limits actual draw to ~12A peak |
| Recommended charge current | 4A (charger set to 2.5A = 0.42C, conservative) |
| Runtime @ 6A average load | ~60 min |

Parallel cells self-balance within a group through their direct terminal connection — any voltage delta drives immediate equalisation current through cell internal resistance. The BMS sees and manages exactly 4 voltages.

### 1.5 Why Two FETs — Not Four, Six, or Eight

Commercial BMS boards with 4–8 FETs are paralleling devices per switch to share current at >20A. The topology is always exactly **two switches: one CHG FET, one DSG FET**, back-to-back on the negative rail.

At this node's 12A continuous ceiling:
- 2× AO4407A @ 8mΩ each: P = 12² × 0.016 = **2.3W total**
- SO-8 with copper pour, θJA ≈ 40°C/W per device → ~46°C rise → ~71°C at 25°C ambient ✅

Two FETs is the correctly-sized professional answer for this load. Parallel additional FETs only when a node variant needs >20A continuous.

### 1.6 Why Two Boards?

- **Noise:** IP2368 switches at 250kHz; BQ76920 resolves ±1mV. Physical separation is the cleanest isolation.
- **Thermal:** IP2368 dissipates up to 4.2W; keep it away from cells and precision analog.
- **Modularity:** the node is a standalone product. The charger is application-specific (speaker). A powerwall built from these nodes would use a different charge source entirely — the node doesn't care.
- **Fault domains:** independent testing, independent failure, independent replacement.

---

## 2. BMS Node Board

### 2.1 Protection & Measurement IC: BQ76920

TI's BQ76920 (3–5S analog front-end) is the single measurement and hardware-protection IC on the node.

**Hardware protection (always active, firmware-independent):**

The BQ76920 powers up with hardwired default protection thresholds before any I2C configuration occurs. An unprogrammed, crashed, or sleeping ESP32 never leaves the pack unprotected. This is the fundamental layering: **hardware safety floor, firmware intelligence above it.**

- Per-cell-group OVP / UVP — independent comparators per VC input
- Overcurrent discharge (OCD), configurable threshold + delay
- Short circuit discharge (SCD) — ~400µs response, latching
- Overcurrent charge (OCC)
- Dual NTC temperature protection
- Independent CHG and DSG FET gate drives — asymmetric fault response (OVP cuts charge only; UVP cuts discharge only, recovery charge stays possible)
- ALERT pin — hardware interrupt on any fault, wired directly to ESP32 wake-capable GPIO

**Measurement (read by ESP32 over I2C, address 0x08):**
- VC1–VC4: ±1mV cell group voltages
- SRP/SRN: pack current through 3mΩ shunt, signed (charge/discharge)
- TS1/TS2: NTC temperatures
- Fault status registers

**What the ESP32 layer adds:** SOC coulomb counting, configurable thresholds, fault logging with timestamps, balancing decisions, host comms, graceful pre-shutdown warnings to the amp, webapp. If the firmware has a bug, the worst case is smart features stop — hardware protection never stops.

### 2.2 Complete Protection Feature Map

| Fault | Detection | Default Threshold | Response | Latching? | Recovery |
|---|---|---|---|---|---|
| OVP per group | VC1–VC4 | 4.20V (configurable) | CHG FET off; DSG unaffected; ALERT low | No | Auto below OV_CLEAR hysteresis |
| UVP per group | VC1–VC4 | 2.80V (configurable) | DSG FET off; CHG unaffected; ALERT low | No | Auto above UV_CLEAR |
| OCD | Shunt SRP/SRN | 56mV / 3mΩ ≈ 18.7A | DSG off after configurable delay; ALERT low | No | Auto when current drops |
| SCD | Shunt spike | Fast detect | DSG off ~400µs; ALERT low | **Yes** | Load disconnect required |
| OCC | Shunt (charge dir.) | Configurable | CHG off; ALERT low | No | Auto |
| OTP | TS1/TS2 NTC | Configurable | CHG and/or DSG inhibit | No | Auto with hysteresis |

> **SCD latching is deliberate** — a dead short must require human intervention before discharge re-enables. **OVP/UVP asymmetry is deliberate** — an over-charged pack can still play music; an over-discharged pack can still accept recovery charge.

### 2.3 Active Cell Balancing — Discrete, ESP32-Driven

#### Correct framing: the BQ76920 is bypassed for balancing

The BQ76920 has built-in *passive* balancing (BALx pins driving bleed resistors). **This design does not use it.** The CELLBAL registers stay 0x00, the BALx pins are unconnected. The BQ76920's role in balancing is measurement only.

True active balancing — moving charge between cells instead of burning it — requires external hardware the BQ76920 cannot drive: complementary FET pairs switching an inductor at 50kHz. The ESP32 drives that hardware directly. The signal chain:

```
1. ESP32 reads VC1–VC4 from BQ76920 over I2C       ← BQ76920's only involvement
2. ESP32 computes deltas, picks transfer path
3. ESP32 drives TC4427 → FET gates at 50kHz (LEDC)  ← BQ76920 not involved
4. Inductor shuttles charge between adjacent groups
```

#### Balancing runs continuously — not only during charging

The balancer connects directly to cell tap nodes. It transfers charge whenever the ESP32 wakes and finds a delta — during charge, during discharge, at rest. There is no "balancing mode"; it is the same routine every wake cycle:

- Wake every 60s (interval configurable via NVS / UART command)
- Read 4 group voltages
- If max−min > 30mV: run the relevant balancer instance(s) for up to 30s
- Re-check, sleep

60s is conservative: at 2A transfer current, a significant 100mAh imbalance resolves across ~5–6 wake cycles (minutes of wall time) on a pack that drifts over hours. Longer intervals (120s–300s) would also work; shorter intervals buy nothing.

#### Circuit topology per instance (3 instances: C1↔C2, C2↔C3, C3↔C4)

```
Group N+  ──┬──[Q_H drain]
            │      [Q_H source]──[L 4.7µH]──[Q_L source]
            │                                    │
Group N−  ──┴──────────────────────────[Q_L drain]

Q_H / Q_L gates ← TC4427 dual driver ← ESP32 LEDC PWM (complementary, dead-time in firmware)
```

Q_H on: inductor charges from upper group. Q_L on: inductor discharges into lower group. Reverse the pattern to reverse direction. ~85–90% transfer efficiency. **Any-to-any balancing:** the ESP32 runs multiple instances simultaneously in chosen directions to move charge between non-adjacent groups (C1→C4 via the chain), deciding the optimal path from measured voltages.

#### Why discrete instead of a balancer IC

Autonomous active balancer ICs are a sourcing dead end: ETA3000 (unreliable LCSC/Mouser stock), LT8584 (custom flyback transformer per cell), LTC3300 (SPI host + coupled inductors, automotive overkill). TI's own E2E forum confirms no standalone single-chip active balancer exists in their catalog. The discrete circuit — Si2302 pairs + TC4427 + standard shielded inductor — is commodity parts available everywhere, and a stronger portfolio piece: the balancer is designed, not bought.

### 2.4 ESP32-S3 Integration

**Why ESP32-S3:** native USB (flash/JTAG/serial via D+/D− — no bridge IC), deep sleep ~10µA, TWAI (CAN) controller on-chip (only a transceiver is external when scaling), LEDC hardware PWM for balancer drive, WiFi for webapp, 8MB flash for LittleFS frontend. Use the **ESP32-S3-MINI-1** module — antenna, flash, shielding included.

**Power budget (60s wake interval):**

| State | Current |
|---|---|
| Deep sleep | ~10µA |
| Active (balance cycle / UART reply / fault handling) | ~80mA |
| WiFi webapp mode | ~120–180mA |

Average: (0.1s × 80mA + 59.9s × 0.01mA)/60 ≈ **0.14mA** → ~0.2mAh/day from a 6Ah pack. Negligible — below cell self-discharge.

**Wake sources:** RTC timer (balance interval) · BQ76920 ALERT GPIO interrupt (faults, immediate) · UART RX (host command) · CAN RX GPIO edge (node mode only).

### 2.5 Programming Interface — Native USB Pads

No USB-C connector, no bridge IC. Four unpopulated 2.54mm through-hole pads from the ESP32-S3 native USB:

```
[VBUS] [D−] [D+] [GND]
```

Cut any USB cable, clip/solder the four wires (red/white/green/black), and the S3 enumerates as CDC serial + JTAG simultaneously. Flash with `idf.py flash` or esptool; debug with OpenOCD; monitor with any terminal — all through the same four pads. Plus **BOOT** and **EN** pads (with optional tactile buttons) for download-mode entry. Production boards: pads unpopulated, zero cost.

Route D+/D− as a matched-length differential pair, no vias if possible.

### 2.6 Host UART Port

4-pin JST-SH (GND / 3.3V / TX / RX), 115200 8N1, ASCII protocol (§8). The amp board's ESP32 connects here. Doubles as the dev serial monitor port with a CP2102 dongle. UART RX wakes the BMS from deep sleep in ~5ms; the host library sends `PING` first.

### 2.7 BMS Node Board — Full BOM

**Core (always populated):**

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | AFE / protection | TI BQ76920 | TSSOP-20 | 1 | All measurement + hardware protection. I2C 0x08. Mouser/LCSC stocked. |
| U2 | MCU | ESP32-S3-MINI-1 | Module | 1 | Native USB, TWAI, LEDC, WiFi. |
| U3 | 3.3V LDO | XC6210B332 | SOT-89 | 1 | Low IQ (~55µA) — critical for sleep budget. Not AMS1117 (5mA IQ would dominate). |
| Q1, Q2 | CHG / DSG FETs | AO4407A 30V 15A 8mΩ ×2 | SO-8 | 2 | Back-to-back source-to-source on B− rail. Driven by BQ76920 CHG/DSG pins. |
| Q3 | Pre-discharge FET | 2N7002 | SOT-23 | 1 | Optional. Soft-starts PVDD bulk caps via R2 — prevents inrush SCD trip. |
| Q_H/Q_L ×3 pairs | Balancer FETs | Si2302 20V 2A ×6 | SOT-23 | 6 | Low Vgs threshold, TC4427-driven. |
| DRV1–3 | Gate drivers | TC4427 dual ×3 | SOT-23-5 / SOIC-8 | 3 | One per balancer instance. 3.3V-logic compatible. |
| Q_iso | Reverse-feed isolation | Si2301 P-ch | SOT-23 | 1 | B+ rail; gate from 3.3V rail — auto off when board unpowered. |
| L1–L3 | Balancer inductors | 4.7µH Isat ≥3A **shielded** | 4×4mm | 3 | Shielded mandatory — 50kHz switching beside ±1mV sense lines. |
| R1 | Pack shunt | **3mΩ ±1% 2W** | 2512 | 1 | Kelvin to BQ76920 SRP/SRN. 56mV max OCD register / 3mΩ = 18.7A trip. |
| R2 | Pre-discharge | 10Ω 1W | 2512 | 1 | With Q3. |
| R3–R5 | Gate / pull-down | 1kΩ ×2, 100kΩ | 0402 | 3 | Q1/Q2 gates, Q3 pull-down. |
| R6–R9 | VC filter R | 100Ω ×4 | 0402 | 4 | With C6–C9: ~16kHz LPF per VC line. |
| R10–R15 | Balancer gate R | 10Ω ×6 | 0402 | 6 | Damping. |
| R16, R17 | I2C pull-ups | 4.7kΩ ×2 | 0402 | 2 | SDA/SCL, near BQ76920. |
| R18, R19 | Q_iso gate network | 10kΩ + 100kΩ | 0402 | 2 | — |
| C1, C2 | Pack bulk | 100µF 25V elec + 10µF X7R | SMD/0805 | 2 | Absorbs amp transients before sense comparators see them. |
| C3 | BQ76920 REGSRC | 10µF X7R | 0805 | 1 | Within 1mm of pin, per datasheet. |
| C4, C5 | BQ76920 decoupling | 100nF + 10nF | 0402 | 2 | — |
| C6–C9 | VC filter C | 0.1µF ×4 | 0402 | 4 | At VC pins to GND. |
| C10–C12 | Balancer bootstrap | 100nF ×3 | 0402 | 3 | — |
| C13–C18 | ESP32 decoupling | 10µF + 100nF per rail | 0402/0805 | 6 | Per module datasheet. |
| C19 | LDO output | 10µF X7R | 0805 | 1 | — |
| NTC1 | Cell NTC | 100kΩ B=3950 | leaded | 1 | Epoxied to cell body → TS1. |
| NTC2 | FET NTC | 100kΩ B=3950 | 0402 | 1 | Near Q1/Q2 → TS2. |
| D1 | Reverse polarity | SS34 | SMA | 1 | B+ input. |
| J1 | Cell taps | JST-XH 5-pin | THT | 1 | B−, B1, B2, B3, B+. Keyed. |
| J2 | Load out | XT30U-F right-angle | pads | 1 | P+/P− to amp PVDD. |
| J3 | Charger in | XT30U-F | pads | 1 | CHG+/CHG−. |
| J4 | Host UART | JST-SH 4-pin | SMD | 1 | GND/3.3V/TX/RX. |
| J5 | USB programming | 4× 2.54mm pads | THT | 1 | VBUS/D−/D+/GND, unpopulated. |
| J6 | BOOT + EN | 2× tactile or pads | SMD | 2 | Download mode / reset. |
| LED1–4 | Status | G/B/R/Y 0402 | 0402 | 4 | CHG path / DSG path / fault (ALERT) / balance active. |

**CAN scaling section (footprints always on PCB — populate only when stacking nodes, ~€5):**

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U4 | Isolated CAN transceiver | TI **ISO1050DUB** | SOP-8 wide | 1 | Single chip = digital isolation **+** CAN transceiver. Replaces separate isolator + SN65HVD230. Logic side 3.3V (ESP32 TWAI TX/RX), bus side 5V isolated. |
| U5 | Isolated DC-DC | **B0305S-1W** (3.3V→5V, 1W, Mornsun) | SIP-4 | 1 | Powers ISO1050 bus side + bus reference. Galvanic barrier: node logic ground ≠ bus ground. |
| C20–C23 | Isolation-side decoupling | 10µF + 100nF ×2 sides | 0402/0805 | 4 | Per ISO1050 + B0305S datasheets. |
| R20 | CAN termination | 120Ω | 0805 | 1 | With solder jumper SJ1 — **enabled only on the two end nodes** of the bus. |
| SJ1 | Termination jumper | 2-pad solder jumper | — | 1 | — |
| SW1 | Node address | 4-pos DIP switch | SMD | 1 | 16 node IDs, read by ESP32 GPIO at boot → CAN ID. |
| J7, J8 | CAN daisy-chain | JST-XH 4-pin ×2 | THT | 2 | CANH / CANL / FAULT / ISO_GND. Two ports = ladder-style chaining node to node. |
| OC1, OC2 | Fault line optos | PC817 ×2 | DIP/SMD-4 | 2 | Hardware fault interlock: one opto asserts the shared FAULT line, one senses it. Any node's fault hardware-trips all nodes — faster than CAN messages. |
| D2, D3 | CAN bus ESD | PESD1CAN | SOT-23 | 1 | CANH/CANL transient protection at connectors. |

### 2.8 BMS Layout Rules

**High-current path** (`B− → D1 → Q1 → source node → R1 → Q2 → P−`): polygon pours ≥3.5mm at 2oz, no single vias (cluster 6–8 if a layer change is unavoidable).

**Kelvin sensing on R1 — the most consequential rule on the board:** SRP/SRN traces tap **directly at the shunt end pads**, thin 0.15mm dedicated routes to the BQ76920. At 3mΩ nominal, 0.5mΩ of trace error inside the sense loop = 17% OCP threshold shift.

**VC lines:** JST tap → 100Ω → VC pin, 0.1µF at the pin. ≥3mm from any balancer inductor. Cross power polygons perpendicular, opposite layer only.

**Balancer instances:** each placed adjacent to its two cell-tap nodes; inductor loop area minimised; TC4427 near the gates, not near the ESP32.

**Isolation barrier (if CAN populated):** the ISO1050 + B0305S straddle a physical gap in the PCB copper — **no copper, no traces, no planes cross the barrier** except through the isolator and DC-DC packages. ≥4mm creepage. Bus-side ground pour is its own island fed only by B0305S output.

**Ground:** one continuous plane for the node logic/power (no analog/digital split — BQ76920 datasheet recommends unified ground). The isolated CAN island is the only separate ground, by design.

**USB D+/D−:** matched differential pair to J5 pads.

---

## 3. CAN Scalability — Designed-In Optionality

### 3.1 The Concept

Scaling = stacking identical nodes, not redesigning the node:

```
16S4P powerwall = 4× this node in series, CAN sections populated

  Node 4 (48V level)  ──┐
  Node 3 (32V level)  ──┤  isolated CAN bus  ──→  Master aggregator
  Node 2 (16V level)  ──┤  (twisted pair)         (Raspberry Pi Zero 2 W:
  Node 1 (0V level)   ──┘                          logging, GUI, global logic)
```

Each node only sees its own 4 cells — and that's sufficient. Series current is common to the whole string, so **any node opening its FETs breaks the entire pack's current path.** Local autonomous protection is global protection. The master coordinates (balancing authority, logging, display) but no node depends on it for safety — the "soldier–commander" hierarchy with no single point of failure.

### 3.2 Why Galvanic Isolation Is Non-Negotiable

When nodes stack in series, **Node 2's battery ground sits 14.8V above Node 1's; Node 4's sits ~44V above Node 1's.** If their logic grounds were tied together through a shared CAN ground, the series stack would short through the communication wiring — destroying ESP32s, transceivers, or worse.

The isolation chain per node:

```
ESP32-S3 TWAI (TX/RX, 3.3V, node logic ground)
        │
   ISO1050 ── galvanic barrier (capacitive isolation, 2.5kVrms)
        │
CAN bus side (CANH/CANL, 5V from B0305S isolated DC-DC, floating bus ground)
```

The CAN bus becomes a floating data network spanning the full stack potential without any conductive path between node grounds. The master connects to the same floating bus — it never sees the pack voltages at all.

**Single-node speaker use needs none of this** — leave U4, U5, optos, DIP, J7/J8 unpopulated. The footprints cost zero.

### 3.3 Bus Design Rules

- **Bitrate:** 500kbps (automotive standard; fine to several metres of bus)
- **Topology:** linear daisy-chain via the dual J7/J8 ports — no stubs, no star wiring
- **Termination:** 120Ω at the two physical ends of the bus only (SJ1 solder jumper)
- **Addressing:** 4-bit DIP per node → CAN ID base; read at boot
- **Wiring:** twisted pair for CANH/CANL; FAULT and ISO_GND alongside in the same 4-pin cable
- **"Babbling node" note:** a faulty node spamming the bus is a real failure mode at larger scales; the master should watchdog per-node message rates and the FAULT hardware line exists precisely because safety must not depend on the CAN bus being healthy

### 3.4 Hardware Fault Line

A shared open-drain FAULT line runs through the daisy cable, referenced to the isolated bus ground, opto-coupled at every node (PC817 ×2: assert + sense). Any node detecting a critical fault pulls the line low; every other node's ESP32 sees it as a wake interrupt and opens its FETs immediately. Hardware-speed, firmware-independent of CAN, works even if the bus controller is wedged.

### 3.5 Balancing Authority in a Stack

Standalone: each node balances its own 4 groups autonomously (§2.3). In a stack, node-level balancing continues unchanged, but **inter-node balance** (Node 1's average vs Node 3's average) is the master's job — it observes all nodes over CAN and commands per-node charge limits or balancer bias. Nodes never fight each other because none acts on data outside its own bank without a master command.

### 3.6 What This Costs and What It Buys

~€5/node populated, €0 unpopulated. It converts a speaker BMS into an industrial-pattern distributed control node — CAN networking, galvanic isolation, hardware interlocks, master/node hierarchy — the exact architecture used in EV and grid storage. As a portfolio piece: "Distributed, CAN-based, galvanically isolated, modular BMS" is a hiring-grade systems-engineering project; the speaker is merely its first deployment.

---

## 4. Charger Board

*(Application-specific to the speaker build — a powerwall made of these nodes would use a different charge source. Unchanged from Rev 4.0 except framing.)*

### 4.1 IC: Injoinic IP2368 — Bidirectional USB-C PD

- Integrated PD2.0/3.0/QC negotiation — CC1/CC2 direct to IC, no FUSB302
- 4-switch synchronous buck-boost, two external inductors, 250kHz
- 65W charge **and** 65W OTG discharge (speaker doubles as a power bank; phone plugged in just works via Try.SRC role negotiation)
- 2–6S via VSET divider; ~€1–2 LCSC; IP2366 (PD3.1/140W) is the pin-family upgrade if out of stock
- COUT pin: solder jumper to disable OTG if ever unwanted
- **Never connect two bidirectional C-port modules to each other** — role-negotiation conflict can damage both

### 4.2 Reverse-Feed Protection: Si2301

Unpowered, the IP2368's four switch body-diodes form a passive battery→VBUS path; fault conditions on VBUS can then push uncontrolled current through the dead IC. A P-ch Si2301 between B+ and the IC BAT pin (gate to IC VCC via 10kΩ, 100kΩ pull-to-source) opens automatically whenever the IC is unpowered. 0.69W at full charge current — ~20× less loss than a Schottky.

### 4.3 PD Efficiency Table

| PD In | Power | Mode | η | P_loss |
|---|---|---|---|---|
| 5V | 25W | hard boost | ~91% | 2.3W |
| 9V | 27W | boost | ~93% | 1.9W |
| 12V | 36W | light boost | ~94% | 2.2W |
| 15V | 45W | near-unity | ~95% | 2.3W |
| 20V | 65W | light buck | **~96%** | 2.6W |
| 65W OTG out | 65W | reverse | ~93% | **4.2W** ← thermal worst case |

### 4.4 Charger BOM

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | PD buck-boost | IP2368 | QFN-28 | 1 | Verify LCSC stock at order time. |
| Q_iso | Isolation FET | Si2301 | SOT-23 | 1 | §4.2. |
| L1, L2 | Inductors | 4.7µH **Isat ≥8A** DCR<30mΩ shielded ×2 | 6.7×6.7mm | 2 | **Both required** (4-switch = 2 inductors). 5V-input hard boost peaks 8–10A — 5A Isat saturates; paralleling small inductors does **not** raise Isat. Sunlord SWPA6028S4R7MT / Bourns SRR6028-4R7Y. |
| C1, C2 | VBUS bulk | 22µF 25V X7R ×2 | 1210 | 2 | — |
| C3, C4 | Output bulk | 47µF 25V X7R ×2 | 1210 | 2 | — |
| C5–C10 | Decoupling | 100nF ×6 + 10nF ×2 | 0402 | 8 | — |
| C11 | Bootstrap | **470nF** X7R | 0402 | 1 | Specifically 470nF — 220nF drops out the high-side gate at peak power. |
| R_VSET | 4S voltage set | per datasheet Table 5 | 0402 | 2 | Verify 16.8V with meter before first pack charge. |
| R_ISET | Charge current | 12kΩ → 2.5A | 0402 | 1 | 0.42C on 2P. |
| R_iso1/2 | Q_iso gate | 10kΩ + 100kΩ | 0402 | 2 | — |
| R_SENSE | Charge shunt | 10mΩ 1% 2W | 2512 | 1 | Kelvin. |
| NTC1/2 | Battery / IC NTC | 100kΩ B=3950 ×2 | 0402 | 2 | TBAT derating + IC monitor. |
| J1 | USB-C | **16-pin** mid-mount | SMD | 1 | 4-pin USB-C has no CC pins = no PD, ever. VBUS/GND pads: full fill, thermal relief OFF. |
| J2, J3 | Cell taps / power out | JST-XH 5p + XT30U-M | THT/pads | 2 | — |
| D1 | VBUS TVS | SMAJ20A | SMA | 1 | — |
| F1 | Polyfuse | 2.5A hold | 1812 | 1 | — |
| LED1–3 | CHG / done / fault | G/B/R | 0402 | 3 | CHRG / STDBY / FAULT pins. |

### 4.5 Layout

Inductors tight to SW pins, parallel, ≥3× body-height apart; caps placed along the physical power flow; 16 tented thermal vias under the QFN pad to a 20×20mm 2oz bottom pour; CC lines 0.15mm direct; Kelvin on R_SENSE.

---

## 5. PCB Layer Count Decision

| Board | Layers | Copper | Reasoning |
|---|---|---|---|
| BMS node | **2** | 2oz | DC-to-kHz signals, one 12A polygon, 50kHz shielded balancer inductors, no thermally critical IC. 4-layer adds ~€15–20/run for nothing measurable. Spend the effort on Kelvin routing and the isolation gap instead. |
| Charger | **4** | 2oz outer / 1oz inner | QFN dissipating 4.2W (inner GND plane cuts θJA ~30–40%); 250kHz × 4 switching nodes need a reference plane 0.2mm below; CC lines beside 5A VBUS need shielding. Without it, 65W OTG hits thermal shutdown (§6). |

JLCPCB 4-layer stackup: JLC7628 default, 1.6mm.

---

## 6. Thermal Management & Heatsink

### 6.1 θJA by Implementation (IP2368, QFN-28)

| Implementation | θJA |
|---|---|
| No vias, 1oz, 2-layer | ~60°C/W |
| 16 vias, 2oz, 2-layer | ~26°C/W |
| 16 vias, 2oz, **4-layer** | ~20°C/W |
| 4-layer + **20×20×8mm bottom heatsink** | **~12°C/W** |

Thermal chain with heatsink: junction →(θJC 2)→ pad →(vias 3–5)→ bottom pour →(TIM 1–2)→ heatsink →(8–12)→ ambient.

### 6.2 Scenarios @ 25°C Ambient

| Scenario | P_loss | T_j (heatsink, 12°C/W) | T_j (no heatsink, 20°C/W) |
|---|---|---|---|
| Charge 65W | 2.6W | **56°C** ✅ | 77°C ✅ |
| Charge 36W | 2.2W | **51°C** ✅ | 69°C ✅ |
| OTG 65W | 4.2W | **75°C** ✅ | **109°C** ⚠️ derates |
| OTG 20W (phone) | 1.2W | **39°C** ✅ | 49°C ✅ |

The 20×20×8mm heatsink (on hand) is what makes full 65W OTG sustainable. QFN has no topside metal — the PCB-bottom heatsink via the thermal-via stack is the correct and only attachment.

### 6.3 Implementation

16 vias (4×4, 0.3mm drill, 1.0mm pitch, **tented bottom** so reflow solder can't wick away from the pad) → 20×20mm 2oz bottom pour → **silicone thermal pad** (Bergquist GP3000 class, 3 W/m·K — gap-filling, electrically isolating, no clamp-pressure dependency; not silver paste over live copper) → heatsink, retained with 2× M2 screws. Standoffs ≥10mm for the 8mm sink; fins vertical.

**BMS node:** no heatsink — ~3.5W spread across FETs (1.15W), shunt (~1W), balancer FETs (~0.5W/instance); nothing stressed.

---

## 7. ESP32-S3 Firmware Architecture

### 7.1 State Machine

```
                    DEEP SLEEP (~10µA)
        ┌──────────────┬──────────────┬──────────────┐
     RTC timer      ALERT GPIO     UART RX      CAN RX edge*
     (60s, NVS-     (BQ76920       (host cmd)   (*node mode)
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
            Update LEDs → sleep
```

RTC memory persists SOC, coulomb accumulator, fault history, and cycle count through sleep.

### 7.2 Balancing Algorithm

Every wake: read 4 group voltages → identify max/min → if Δ>30mV choose the transfer path (adjacent direct, or chained instances for non-adjacent) → drive the instance(s) ≤30s → re-read → sleep. Same routine charging, discharging, or at rest — the balancer is electrically independent of pack current flow. SOC drift-corrects against an OCV table after ≥5min rest.

### 7.3 WiFi Webapp Mode

If no UART host appears for 60s after boot: join configured WiFi (NVS) or fall back to AP `BMS_[MAC4]`. LittleFS-served dashboard at `192.168.4.1`: live group voltages, SOC, current, temps, fault log, balance indicator. Exits on first UART command.

### 7.4 Node Mode (CAN populated)

DIP-read node ID at boot → periodic CAN frames (voltages, SOC, current, temp, status) → obeys master commands (charge-limit, balance bias, e-stop) → asserts/reacts to the hardware FAULT line independent of CAN. TWAI peripheral + ISO1050; wake-on-CAN via RX GPIO edge.

---

## 8. Host Communication Protocol

UART 115200 8N1, ASCII + `\n`. `PING` first to wake (~5ms).

| Command | Response | Notes |
|---|---|---|
| `PING` | `PONG` | Wake + connectivity |
| `PACK_VOLTAGE` | `14823` | mV |
| `PACK_CURRENT` | `-2340` | mA, − = discharge |
| `PACK_SOC` | `73` | % |
| `PACK_CAPACITY` | `5840` | mAh remaining |
| `PACK_TEMP` | `31` | °C (TS1) |
| `CELL_1` … `CELL_4` | `3721` | mV per group |
| `CELL_ALL` | `3721,3718,3722,3719` | mV ×4 |
| `BALANCE_STATE` | `ACTIVE:1-2` / `IDLE` | — |
| `FAULT_STATE` | `NONE` / `OVP:CELL3` / … | — |
| `FAULT_LOG` | last 10 events | timestamped |
| `STATUS` | `CHARGING/DISCHARGING/IDLE/FAULT` | — |
| `CYCLE_COUNT` | `47` | — |
| `SET_BALANCE_INTERVAL:120` | `ACK` | seconds, NVS-persisted |
| `VERSION` | `BMS_FW_1.0.0` | — |

**Unsolicited messages:** `FAULT:<type>[:CELLn]` on any ALERT; `WARN:LOW_SOC:10` and `WARN:LOW_SOC:5` ahead of UVP so the amp can fade out and mute the TAS5825M before hardware cutoff — no pop, no mid-song death.

**Companion library (`BMS_Client`, to be written):** `begin() / ping() / getSOC() / getPackVoltage() / getPackCurrent() / getAllCells() / getFaultState()` + fault callback; handles wake, timeout/retry, and parsing.

---

## 9. PCB Ordering & Build Procedure

### 9.1 JLCPCB Settings

| | BMS node | Charger |
|---|---|---|
| Layers / copper | 2 / 2oz | 4 / 2oz-1oz-1oz-2oz |
| Finish | ENIG | ENIG |
| Vias | open | **tented bottom under U1** (fab note) |
| Stencil | frameless, yes | frameless, yes — mandatory for QFN-28/TSSOP-20 |
| Thickness | 1.6mm | 1.6mm |

### 9.2 Reflow (SAC305)

Preheat 25→150°C @2°C/s · soak 150→200°C ~90s · peak 245°C, 20–30s above 217°C · cool <3°C/s to 100°C. Post-reflow: continuity from QFN center-pad via to GND (none = pad didn't wet → flux + hot air until the package settles); 10× inspection on TSSOP-20.

### 9.3 Assembly Order

Stencil → 0402s → ICs (BQ76920, IP2368, TC4427s, FETs, LDO) → ESP32-S3 module → SO-8s + 2512 shunts → inductors → reflow → inspect/IPA → flip, hand-solder THT (JST, XT30, pads, buttons) → continuity → heatsink (TIM + M2) → **flash firmware via USB pads before any battery contact**.

> ⛔ First power-on always via current-limited bench PSU (100mA). Never connect cells to an unverified board.

---

## 10. Test & Verification Procedure

### 10.1 Pre-Power
Bridges/orientation under magnification · B+→P− >1MΩ · R1 = 3mΩ ±1% (4-wire) · NTCs 97–103kΩ @25°C · Q_iso Vgs = 0V unpowered · D+/D− continuity, no GND short.

### 10.2 Flash & Boot
Cut-cable to J5 → BOOT low + EN pulse → flash → serial monitor shows boot, BQ76920 I2C OK, initial VC reads.

### 10.3 Protection (4× bench PSU channels @3.700V as cell groups)
- Nominal: P+−P− = 14.8V, ALERT high
- OVP: one channel >4.21V → CHG FET off, DSG unaffected, ALERT low, ESP32 wakes ≤10ms, fault logged; recovers on hysteresis
- UVP: one channel <2.79V → DSG off, CHG path alive; recovers >UV_CLEAR
- OCD: load toward 18.7A → DSG off after configured delay, auto-recovers
- SCD: momentary ~0.1Ω → DSG off ≤400µs, **stays latched** until load removed
- OTP: heat NTC1 to ~45°C → charge inhibit, discharge alive

### 10.4 Balancer
Adjacent channels 3.800/3.650V → scope 50kHz complementary gates with dead-time on that instance → ammeter confirms high→low transfer → Δ closes over minutes. Repeat per instance; verify a non-adjacent (chained) transfer too.

### 10.5 UART
`PING`→`PONG` · `CELL_ALL` matches PSU ±5mV · forced OVP produces unsolicited `FAULT:OVP:CELLn` ≤100ms · `SET_BALANCE_INTERVAL` persists across power cycle.

### 10.6 Charger
PD adapter → VBUS negotiates max within 3s · 22Ω dummy: 16.8V ±0.1V, I = 2.5A · 5min @65W: case <65°C with heatsink · OTG: phone negotiates and charges · LED sequence correct.

### 10.7 System
All boards + pack + amp (or dummy): charge path, discharge path, loud bass with no OCD nuisance trips (raise OCD delay register if any), `WARN:LOW_SOC` arrives before UVP, 30-min thermal: FETs <65°C, IP2368 <70°C, cells <40°C.

### 10.8 CAN (only when populated)
Verify **no continuity** across the isolation barrier (logic GND ↔ ISO_GND) · two nodes + master on bench: enumeration by DIP ID, telemetry frames, FAULT line trips all nodes from any node, termination only at the two ends.

---

## 11. System Integration & Multi-Node Scaling

### 11.1 Speaker Integration (Node #1's First Job)

- **Cells:** 8× same-batch Samsung 30Q (or 35E/GA); spot-welded nickel preferred; 14AWG mains <200mm; 26AWG taps routed clear of power leads and inductors. Kapton every exposed terminal during assembly.
- **Connectors:** XT30 power (male on board), JST-XH taps, JST-SH UART. No dupont on current paths.
- **Amp PVDD decoupling** (prevents OCD nuisance trips): 2×470µF low-ESR + 4×10µF X7R + 4×100nF at the TAS5825M PVDD pins.
- **Mounting:** nylon M3 standoffs (≥10mm under charger for heatsink); ≥10mm between charger and audio traces; heatsink fins vertical; vent slots if the enclosure allows.

### 11.2 Scaling Checklist (When the Day Comes)

1. Populate CAN section on every node (~€5 each); set unique DIP IDs
2. Stack packs in series (Node1 + → Node2 −, …)
3. **Isolated DC-DC per node** powers each bus side — never bridge node logic grounds
4. Daisy-chain the 4-wire bus (CANH/CANL twisted, FAULT, ISO_GND); 120Ω jumpers closed on the two end nodes only
5. Master (Pi Zero 2 W class) joins the floating bus: aggregation, logging, GUI, inter-node balancing authority, e-stop
6. Verify the FAULT interlock trips the full stack from every node before first real charge

Each node still protects itself with zero dependence on master or bus — opening any node's FETs opens the whole series string.

### 11.3 Portfolio Framing

Power electronics (PD buck-boost, inductive active balancing) · mixed-signal (Kelvin sensing, ±1mV AFE integration, RC noise design) · embedded (deep-sleep architecture, interrupt-driven wake, LEDC, protocol + library design) · systems (hardware/firmware layered safety, galvanic isolation, distributed CAN architecture, fault interlocks, graceful degradation) · process (QFN reflow, thermal-via design, staged verification). Deliverables: DRC-clean KiCad, Gerbers + iBOM, firmware repo, protocol doc, and a measured-vs-expected threshold test report.

---

## 12. Key Design Decisions Reference

| Decision | Choice | Rationale |
|---|---|---|
| Measurement | **BQ76920 only — no extra ADC** | AFE already provides ±1mV ×4 groups + pack current + 2 NTC. Per-cell-in-parallel monitoring rejected: parallel physics self-equalises; scaling moved to node level. €0 added BOM. |
| Protection | BQ76920 hardware defaults | Pack protected before/without firmware. ESP32 adds intelligence above an un-defeatable hardware floor. |
| Balancing | **Active, discrete** (Si2302 ×6 + TC4427 ×3 + 4.7µH ×3), ESP32 LEDC 50kHz | BQ76920 BALx/passive path deliberately unused. No sourceable autonomous balancer IC exists; discrete = commodity parts + any-to-any transfer + portfolio value. Runs every wake regardless of charge/discharge state; 60s interval (configurable) is amply fast. |
| Scaling model | **Stack nodes over isolated CAN** — not bigger boards | One PCB design for speaker → powerwall. ISO1050 + B0305S-1W + dual daisy ports + DIP ID + 120Ω jumper as unpopulated optionality (~€5 when needed). |
| Isolation | Galvanic, per node | Series stacking puts node grounds 14.8–44V apart; shared comm ground = dead short through the bus. Non-negotiable. |
| Fault interlock | Opto-coupled hardware FAULT line | Stack-wide e-stop independent of CAN/firmware health. |
| Shunt | 3mΩ 1% 2W | BQ76920 OCD register max 56mV → 18.7A trip; 8mΩ would cap trip at 7A. |
| MCU | ESP32-S3-MINI-1 | Native USB (4 pads = flash+JTAG+serial, no bridge IC, no USB-C), TWAI on-chip, 10µA sleep, WiFi webapp. |
| LDO | XC6210 (not AMS1117) | 55µA vs 5mA quiescent — AMS1117 would be 500× the sleep budget. |
| Charger | IP2368, bidirectional | 65W PD both ways; speaker is also a power bank. Si2301 blocks unpowered body-diode back-feed. |
| Charger inductors | 4.7µH **Isat ≥8A** ×2 | Hard-boost peaks 8–10A; 5A parts saturate; paralleling doesn't raise Isat. |
| Layers | BMS 2-layer / charger 4-layer | Match complexity to need; inner GND plane is the charger's thermal/EMI requirement, not the node's. |
| Heatsink | 20×20×8mm, PCB-bottom, silicone pad, M2 | θJA 20→12°C/W; 65W OTG: 109°C→75°C junction. |
| Host I/F | UART JST-SH, ASCII + library | Any MCU can host it; doubles as dev monitor; CAN is the scale-out path, UART the local one. |

---

*End of document — Rev 5.0*
