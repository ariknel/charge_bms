# 4S Lithium Battery System — Engineering Design & Build Reference

**Scalable Smart BMS Node · USB-C PD Charger Board**  
Primary application: Bluetooth Speaker / TAS5825M Amplifier Platform  
Architecture: Distributed modular BMS — single node now, stackable via CAN later  
*Revision 6.2 — Dual Balancing Mode · June 2026 · Arik Nel*

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
13. [Pre-Schematic Design Review — Findings & Required Configuration](#13-pre-schematic-design-review--findings--required-configuration)
14. [Datasheet Library — Schematic Reference PDFs](#14-datasheet-library--schematic-reference-pdfs)

---

## 1. System Overview

### 1.1 Architecture Philosophy — The Smart Node

This is not a BMS for one speaker. It is a **self-contained, open-source, programmable 4S2P battery management node** that happens to power a speaker first.

Each node handles its own:
- Hardware cell protection (BQ76920 — operates even with no firmware)
- Cell balancing — **passive** (BQ76920 BALx + bleed resistors, firmware-controlled, runs during deep sleep) **or active** (discrete inductive half-bridges, ESP32 LEDC 15kHz) — firmware-selected, mutually exclusive
- SOC tracking (coulomb counting via BQ76920 current sense)
- Telemetry (UART host interface + WiFi webapp)
- Optional networking (CAN bus with galvanic isolation — footprints designed in, populated only when scaling)

**Scaling is done by stacking nodes, not redesigning them.** Need a 16S4P powerwall? Stack four nodes in series, populate the CAN/isolation section (~€5 of parts per board), and they join a shared bus reporting to a master aggregator. Same PCB, same firmware, no redesign.

For the speaker build: CAN section unpopulated. Node runs standalone — the BMS ESP32 manages everything including charger board communication.

### 1.2 Power Architecture Summary

| Stage | Function | Key Spec |
|---|---|---|
| USB-C PD Input | Negotiate 5–20V from any adapter via FUSB302 | PD2.0 / PD3.0 / QC3.0, sink and source. FUSB302 controlled by BMS ESP32. |
| BQ25713B Buck-Boost | Charge: any PD voltage → 16.8V CC/CV. OTG: pack → 5V/9V on VBUS | 4-switch, 4× external FETs, 1 inductor, I2C to BMS ESP32 |
| 4S2P Li-ion Pack | 14.8V nom / 16.8V full / 12.0V cutoff | 8× 18650, 2 per series group |
| BQ76920 | All measurement + hardware protection | ±1mV cell voltage, pack current, 2× NTC |
| ESP32-S3 (bare SoC) | Intelligence: balancing, SOC, comms, webapp | Deep sleep ~10µA, native USB pads |
| Active Balancer | 3× discrete FET pairs + inductors, ESP32-driven | Inductive, any-to-any, ~88% efficient |
| AP63203 Buck | 3.3V fixed from B+ (12–16.8V) for all logic | Synchronous buck, SOT-23-6, ~50µA IQ. Powers ESP32-S3 + BQ76920 + FUSB302 + BQ25713B logic. Sourced from B+ so ESP32 stays alive regardless of protection FET state. |
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

#### Both passive and active balancing implemented — firmware selects

The BQ76920 has built-in passive balancing via its BALx pins — external bleed resistors on each BALx output dissipate energy from the highest cell as heat. **This design uses it** as the default balancing mode. The discrete half-bridge circuit provides active (inductive) balancing as an alternative mode. Both share the same decision logic; only the actuation differs. The two modes are **mutually exclusive** — running both simultaneously would burn energy the active circuit just transferred.

**Mode selection:** stored in NVS, changeable at runtime via `SET_BALANCE_MODE:PASSIVE` or `SET_BALANCE_MODE:ACTIVE` over UART. Default: `PASSIVE` — simpler, no audible switching noise inside the speaker enclosure, adequate for a 6Ah pack cycling daily. Switch to `ACTIVE` once the discrete circuit is verified on the bench.

**Key difference in behaviour during deep sleep:**
- Passive: BQ76920 latches the CELLBAL bit in hardware — **bleeding continues during deep sleep** without the ESP32 awake. The ESP32 sets the bit and sleeps; it clears it on the next wake when delta closes.
- Active: requires the ESP32 awake and LEDC running — **stops during deep sleep**.

True active balancing requires external hardware the BQ76920 cannot drive: complementary FET half-bridges switching an inductor at 15kHz. The ESP32 drives that hardware directly via LEDC + discrete level shifters. The BQ76920 measures; the ESP32 and discrete FETs act.

```
Passive mode (default):
1. ESP32 reads VC1–VC4 from BQ76920 over I2C     ← measurement
2. ESP32 picks highest cell, writes CELLBAL1 bit  ← actuation
3. BQ76920 BALx pin pulls down → bleed resistor dissipates energy from that cell
4. ESP32 enters deep sleep — BQ76920 latches CELLBAL, bleeding continues autonomously
5. Next wake: re-read voltages → if delta < 10mV, clear CELLBAL → stop

Active mode:
1. ESP32 reads VC1–VC4 from BQ76920 over I2C     ← measurement
2. ESP32 computes transfer path (source cell → destination cell)
3. ESP32 drives level shifters → Si2301/Si2302 gates at 15kHz ← BQ76920 not involved
4. 22µH inductor shuttles charge between cell groups (~1A average)
5. ESP32 must stay awake — active mode stops when ESP32 sleeps
```

#### Balancing runs every wake cycle — both modes

The balancer decision logic runs every wake cycle regardless of charge/discharge state:

- Wake every 60s (configurable via NVS / `SET_BALANCE_INTERVAL` UART command)
- Read 4 group voltages from BQ76920
- If max−min > 30mV: actuate the selected mode
  - Passive: write CELLBAL1, sleep — BQ76920 bleeds until next wake clears it
  - Active: run relevant instance(s) for up to 30s, stop, re-read
- If max−min < 10mV (hysteresis): clear CELLBAL (passive) or stop LEDC (active)

Interval is NVS-persisted. At ~80mA passive bleed current, a 100mAh imbalance resolves in ~75 minutes of cumulative bleed time across multiple sleep cycles. At ~1A active transfer the same resolves in ~6 minutes of active run time.

#### Why no balancer IC

Every autonomous active balancer IC has sourcing problems: ETA3000 (unavailable Mouser/LCSC), LT8584 (custom flyback transformer per cell), LTC3300 (SPI host + coupled inductors). TI's own E2E forum confirms no standalone single-chip active balancer exists. Discrete circuit — Si2301/Si2302 half-bridges + 2N7002/BSS84 level shifters + standard shielded inductors — uses commodity parts available everywhere. Stronger portfolio piece: circuit is designed, not bought.

#### Circuit topology — corrected for floating gate drive (3 instances: C1↔C2, C2↔C3, C3↔C4)

Each instance is a half-bridge across its two stacked cell groups (nodes BOT / MID / TOP), inductor from the switch node to MID:

```
TOP (upper group +) ──[Si2301 P-ch, source=TOP]──┐
                                                 SW ──[L 22µH]── MID (junction)
BOT (lower group −) ──[Si2302 N-ch, source=BOT]──┘
```

**Why not a ground-referenced gate driver:** only instance A's low-side FET sits at ESP32 ground. Every other gate floats 3.7–16.8V above it — a TC4427-style driver (which also requires a ≥4.5V supply) physically cannot drive them. The correct discrete solution is ground-referenced open-drain level shifters:

- **High-side (Si2301):** gate pulled to TOP via 10kΩ (default off). 2N7002 (source = board GND, gate = ESP32 GPIO_H via 100Ω) pulls the gate down through 1kΩ to turn on. BZT52C6V2 zener TOP→gate clamps |Vgs| ≤ 6.2V.
- **Low-side (Si2302):** gate pulled to BOT via 10kΩ (default off). BSS84 (source = MID) pulls the gate up to MID (+3.7V Vgs) to turn on. The BSS84's own gate is pulled to MID via 10kΩ and pulled down by a second 2N7002 (source = GND, gate = ESP32 GPIO_L) through 1kΩ, with a BZT52C6V2 zener MID→BSS84 gate.
- **Switching frequency: 15kHz** (LEDC). Resistor-pullup level shifters are too slow for 50kHz; at 15kHz the 10kΩ/Ciss time constants comfortably fit the 66µs period with firmware dead-time ≥2µs.
- **Inductor: 22µH, Isat ≥3A, shielded** — sized for ~2.8A pk-pk ripple at 15kHz → ~1A average balance current.
- Net transfer direction set by duty cycle around 50%; ESP32 runs instances simultaneously for non-adjacent (chained) transfers.

Per-instance parts: Si2301 ×1, Si2302 ×1, 2N7002 ×2, BSS84 ×1, BZT52C6V2 ×2, 10kΩ ×3, 1kΩ ×2, 100Ω ×2, 22µH ×1, 100nF ×1. At ~1A, a 100mAh imbalance resolves in ~6 minutes of cumulative balancing — entirely adequate at a 60s wake interval.

### 2.4 ESP32-S3 Integration — Bare SoC Design

Bare ESP32-S3 SoC (QFN-56), 8MB NOR flash (W25Q64JV, SOIC-8 or WSON-8), 40MHz crystal, 1206 ceramic chip antenna. Full RF layout responsibility on the PCB.

#### Core subsystem components

| Ref | Component | Value / Part | Notes |
|---|---|---|---|
| U2 | ESP32-S3 SoC | QFN-56 | Native USB, TWAI, LEDC, WiFi/BT. No module — full RF control. |
| U_FL | NOR Flash | W25Q64JVSSIQ (SOIC-8) or W25Q64JVZPIQ (WSON-8) | QSPI to FSPI pins. 100nF + 1µF at VCC. Suffix sets package — SSIQ = SOIC-8, ZPIQ = WSON-8. |
| X1 | Crystal | 40MHz ±10ppm, 3225 | Epson FA-238 or equiv. Load caps per CL spec — typically 2×15pF at CL=10pF. |
| U3 | 3.3V Buck | **AP63203** (Diodes Inc.) | TSOT-26. 3.8–32V input, 3.3V fixed output, **2A**. Sourced from B+ — powers all logic regardless of protection FET state. ~22µA IQ. 2A covers worst case (WiFi TX ~350mA + all logic <600mA total). |
| ANT1 | Chip antenna | Johanson 2450AT18A100E or 2450AT18B100E | 1206, 2.4GHz. Board edge placement, radiating end outward. Top-layer clearance only — no inner layer keepout. ~€0.30. |
| L_RFC | RF matching — series | 0Ω placeholder or tuned value | Series element of π-match on LNA_IN. Use Espressif chip antenna reference values (different from trace antenna table). |
| C_RFC1, C_RFC2 | RF matching — shunt | Per Espressif chip antenna ref schematic | Shunt caps in π-match. Typically 1–2pF. |

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

#### 1206 Chip Antenna — replaces PCB trace antenna

A 1206 ceramic chip antenna replaces the PCB trace antenna. This is the correct choice for an ESP32-S3 inside a speaker enclosure:

- **No all-layer keepout void** — inner GND plane (layer 2) runs continuously everywhere except a small pad clearance under the antenna element itself. This eliminates the most complex layout constraint of the trace antenna approach.
- **Top-layer clearance only** — ~3mm in front of the antenna tip, ~1mm either side, on the top copper layer only. No inner layer or bottom layer restriction.
- **Better enclosure performance** — ceramic element is less sensitive to nearby copper pours and battery cells than a trace antenna. Penetrates wood/MDF/plastic speaker enclosures easily at 5–10m range.
- **Hand-solderable** — 1206 package is comfortable with a standard iron. No reflow required if preferred.

**Recommended part:** Johanson Technology 2450AT18A100E or 2450AT18B100E (1206, 2.4GHz — both Johanson parts, interchangeable footprint; referenced in Espressif antenna design guides). Both Mouser/Digikey stocked, ~€0.30.

**Placement:**
- At board edge, antenna element flush with or slightly overhanging the edge — radiating end faces outward into free space
- All high-current components (FETs, balancer inductors, power polygons) on the opposite end of the board
- BQ76920 VC sense lines maintain ≥3mm clearance from antenna element

**Matching network (π-match on LNA_IN):**
Same as trace antenna — two shunt components + one series component between LNA_IN and antenna feed. Use Espressif's ESP32-S3 hardware design guide reference values for the chip antenna variant (separate table from trace antenna values — verify the correct section). 0Ω placeholder on the series element for tuning if needed.

**What this removes from the design vs trace antenna:**
- All-layer copper keepout void on BMS board (was the most complex gerber requirement)
- Inner GND plane void specification in JLCPCB fab notes
- Microstrip width calculation and trace routing
- Antenna keepout DRC rule in KiCad

#### ESP32-S3 power domains (all require independent decoupling)

| Pin | Supply | Decoupling |
|---|---|---|
| VDD3P3 | 3.3V | 10µF + 100nF |
| VDD3P3_RTC | 3.3V | 100nF |
| VDD3P3_CPU | 3.3V | 10µF + 100nF |
| VDD_SDIO | 3.3V | 10µF + 100nF (FSPI to flash) |
| VDD3P3_USB | 3.3V | 100nF (USB PHY) |

Each cap within 0.5mm of its respective pin. Map datasheet pin table to physical QFN-56 pad locations before placement.

#### Power budget — deep sleep current breakdown

All figures from B+ rail (pack voltage), accounting for AP63203 buck efficiency (~70% at ultra-light load).

| Component | Deep sleep (FUSB302 active) | Deep sleep (FUSB302 low-power) | Notes |
|---|---|---|---|
| AP63203 buck IQ | ~50µA | ~50µA | Quiescent at light load |
| ESP32-S3 | ~10µA | ~10µA | RTC timer + GPIO wake active only |
| BQ76920 | ~3µA | ~3µA | Cell monitoring + protection always running, drawn from REGSRC not 3.3V rail |
| FUSB302 | ~125µA | **~10µA** | DRP toggling vs low-power unattached mode |
| BQ25713B | ~50µA | ~50µA | Idle register state |
| All other (leakage, pull-downs) | ~2µA | ~2µA | — |
| **3.3V rail total** | **~240µA** | **~125µA** | — |
| **From B+ (÷ buck η ~70%)** | **~340µA** | **~180µA** | Actual pack drain |

**The FUSB302 in DRP toggling mode is the dominant consumer** — more than all other components combined. Writing `CONTROL2` register to disable DRP toggling before entering deep sleep drops it to ~10µA. The FUSB302 INT pin remains active at this quiescent level and still fires on cable attach/detach, so wake capability is fully preserved.

**Pre-sleep sequence (firmware):**
1. Write FUSB302 `CONTROL2`: disable DRP toggling → unattached low-power mode
2. Write BQ25713B: disable ADC continuous conversion → idle mode
3. Configure ESP32 wake sources (RTC timer, ALERT GPIO, UART RX, FUSB302 INT)
4. Enter deep sleep

**On wake from FUSB302 INT:** immediately re-enable DRP toggling in FUSB302 before reading CC status, so role detection is fresh.

**Pack drain per day:**

| Scenario | Daily drain | Runtime on 6Ah (self-discharge aside) |
|---|---|---|
| FUSB302 active (no sleep optimisation) | 340µA × 24h = **8.2mAh/day** | ~730 days |
| FUSB302 low-power (recommended) | 180µA × 24h = **4.3mAh/day** | ~1400 days |

Both are well below cell self-discharge (~60–180mAh/month on a 6Ah pack). The FUSB302 low-power mode is a one-register write — implement it.

**Average current across a 60s balance cycle (active 100ms, sleep 59.9s, FUSB302 low-power):**

```
(0.1s × 80mA + 59.9s × 0.18mA) / 60s = (8mAs + 10.78mAs) / 60s ≈ 0.31mA average
→ ~7.5mAh/day total — still negligible from a 6Ah pack
```

**Wake sources:** RTC timer (balance interval, NVS-configurable) · BQ76920 ALERT GPIO (any protection fault, immediate) · UART RX (host command) · FUSB302 INT GPIO (USB-C cable attach/detach) · CAN RX GPIO edge (node mode only).

### 2.5 Programming Interface — Native USB Pads

No USB-C connector on BMS, no bridge IC. Four unpopulated 2.54mm THT pads from ESP32-S3 native USB:

```
[VBUS] [D−] [D+] [GND]
```

Cut any USB cable, clip/solder four wires. S3 enumerates as CDC serial + JTAG simultaneously — flash with `idf.py flash` or esptool, debug with OpenOCD, monitor with any terminal. All through the same four pads.

Also break out: **BOOT** pad (GPIO0 → GND for download mode) and **EN** pad (reset), each with optional tactile button footprint.

Route D+/D− as matched-length differential pair, no vias.

### 2.6 Host UART Port

4-pin JST-SH 1.0mm (GND / 3.3V / TX / RX). 115200 8N1, ASCII protocol (§8). Amp board ESP32 connects here. Doubles as dev serial monitor with CP2102 dongle. **Wake mechanics:** the ESP32-S3 cannot wake from deep sleep on UART data — only light sleep supports UART wake. Instead the RX pin (an RTC-capable GPIO 0–21) is configured as an EXT wake source on falling edge: the start bit of the host's `PING` wakes the chip (that byte is lost; full reboot with RTC RAM intact, ~150–300ms). The host library sends `PING`, waits up to 500ms, retries ×3.

### 2.7 BMS Node Board — Full BOM

**Core (always populated):**

| Ref | Component | Part / Value | Package | Qty | Notes |
|---|---|---|---|---|---|
| U1 | AFE / protection | TI BQ76920 | TSSOP-20 | 1 | ±1mV cell groups, I2C 0x08, OVP/UVP/OCD/SCD/OCC, CHG+DSG FET drive, ALERT. Mouser/Digikey stocked. |
| U2 | MCU | ESP32-S3 bare SoC | QFN-56 | 1 | Native USB, TWAI, LEDC, WiFi. Bare SoC — see §2.4 for full subsystem. |
| U3 | 3.3V Buck | AP63203 (Diodes Inc.) | TSOT-26 | 1 | 3.8–32V input, 3.3V fixed output, **2A**, synchronous buck. Sourced from B+ — ESP32 stays powered regardless of protection FET state. ~50µA IQ at light load — negligible sleep impact. Add 10µF input + 22µF output caps + shielded inductor per datasheet. |
| U_FL | NOR Flash | W25Q64JVSSIQ 8MB | SOIC-8 | 1 | QSPI interface. 100nF + 1µF at VCC. (WSON-8 variant = W25Q64JVZPIQ.) |
| X1 | Crystal | 40MHz ±10ppm | 3225 | 1 | E.g. Epson FA-238. 2× load caps per CL spec. |
| Q1 | CHG FET | AON6354 30V 83A 5.2mΩ | DFN5×6 | 1 | Back-to-back on B− rail. Driven by BQ76920 CHG pin. Exposed pad → PCB copper pour for thermal spreading. |
| Q2 | DSG FET | AON6354 30V 83A 5.2mΩ | DFN5×6 | 1 | Driven by BQ76920 DSG pin. Body diode orientation critical — verify against BQ76920 ref schematic. |
| Q3 | Pre-charge FET | Si2302 20V 2A | SOT-23 | 1 | Soft-starts amplifier PVDD bulk caps via R2 for ~200–300ms before Q2 closes — prevents inrush SCD trip. (2N7002 at 300mA was undersized for the 0.76A peak.) |
| Q_HA–C | Balancer high-side FETs | Si2301 P-ch 20V ×3 | SOT-23 | 3 | One per instance, source at TOP node. Gate via 10k pull-up + 2N7002 level shifter + 6.2V zener clamp. |
| Q_LA–C | Balancer low-side FETs | Si2302 N-ch 20V ×3 | SOT-23 | 3 | One per instance, source at BOT node. Gate driven to MID via BSS84 + level shifter. |
| Q_LS1–6 | Level shifters (pull-down) | 2N7002 ×6 | SOT-23 | 6 | Ground-referenced open-drain shifters: 2 per instance (one per power FET gate path). |
| Q_LS7–9 | Level shifters (pull-up) | BSS84 P-ch ×3 | SOT-23 | 3 | One per instance: pulls low-side N gate up to MID node. |
| D_Z1–6 | Gate zener clamps | BZT52C6V2 6.2V ×6 | SOD-123 | 6 | Two per instance — clamp Vgs of Si2301 and BSS84 within rating. |
| L_A–C | Balancer inductors | **22µH** Isat ≥3A **shielded** ×3 | 5×5mm | 3 | Sized for 15kHz: ~2.8A pk-pk ripple, ~1A avg transfer. Shielded mandatory — couples into VC lines otherwise. |
| R1 | Pack shunt | **3mΩ ±1% 2W** | 2512 | 1 | Kelvin to BQ76920 SRP/SRN. 56mV OCD max / 3mΩ = 18.7A trip. |
| R2 | Pre-charge resistor | 22Ω 1W | 2512 | 1 | With Q3. Peak inrush 16.8V/22Ω = 0.76A. |
| R3, R4 | FET gate R | 1kΩ ×2 | 0402 | 2 | Q1/Q2 main protection FET gate damping. |
| R5 | Q3 pull-down | 100kΩ | 0402 | 1 | Holds Q3 off when control line floating. |
| R_BAL1–4 | Passive balance bleed resistors | 68Ω ±5% 0.25W ×4 | 0402 | 4 | One per BALx pin (BAL1–BAL4). BALx pin sinks current through resistor to cell negative — ~54mA bleed per cell at 3.7V. Value sets bleed current: I = Vcell / R. 68Ω chosen for ~55mA (conservative, low heat). Footprint on BALx pin → cell tap node. |
| R10–R15 | Balancer gate R | 10Ω ×6 | 0402 | 6 | Switching edge damping per FET gate. |
| R16, R17 | I2C pull-ups | 4.7kΩ ×2 | 0402 | 2 | SDA/SCL to 3.3V, near BQ76920. |
| R18, R19 | BQ76920 REGSRC divider | Per datasheet Table 1 | 0402 | 2 | Sets BQ76920 internal LDO input from pack voltage. |
| C1, C2 | Pack bulk | 100µF 25V elec + 10µF X7R | SMD/0805 | 2 | Absorbs amp transients before sense comparators. |
| C3 | BQ76920 REGSRC | 10µF X7R | 0805 | 1 | Within 1mm of pin. |
| C4, C5 | BQ76920 decoupling | 100nF + 10nF | 0402 | 2 | VCC and DVDD pins. |
| C6–C9 | VC filter C | 0.1µF ×4 | 0402 | 4 | At VC pins to GND. With R6–R9: cutoff ~16kHz. |
| C10–C12 | Balancer local decoupling | 100nF ×3 | 0402 | 3 | One per instance, across TOP–BOT near the half-bridge. |
| C13–C22 | ESP32-S3 decoupling | 10µF + 100nF per VDD domain | 0402/0805 | 10 | Five VDD domains: VDD3P3, VDD3P3_RTC, VDD3P3_CPU, VDD_SDIO, VDD3P3_USB. Within 0.5mm each. |
| C23 | Buck input cap | 10µF 25V X7R | 0805 | 1 | AP63203 VIN decoupling. Place within 1mm of VIN pin. |
| C24 | Buck output cap | 22µF 10V X7R | 0805 | 1 | AP63203 output. Per datasheet recommendation for 3.3V output stability. |
| L_PWR | Buck inductor | 3.3µH or 4.7µH Isat ≥1A shielded | 4×4mm | 1 | AP63203 power inductor. Shielded — place away from ESP32 antenna and BQ76920 VC sense lines. |
| C_X1, C_X2 | Crystal load caps | ~15pF (verify per CL) | 0402 | 2 | Symmetric placement at crystal pads. |
| NTC1 | Cell NTC | 100kΩ B=3950 | leaded | 1 | Epoxied to cell body → BQ76920 TS1. |
| NTC2 | FET NTC | 100kΩ B=3950 | 0402 | 1 | Near Q1/Q2 → BQ76920 TS2. |
| R_RS, D_RS | BQ76920 REGSRC protection | 1kΩ + 5.1V zener | 0402/SOD-123 | 2 | Series R + zener on REGSRC supply line per BQ76920 datasheet. (Series Schottky on the main B+ path deleted — a 3A diode on a 12A path would burn and waste ~5W; reverse protection is provided by the keyed JST-XH cell connector.) |
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
| U4 | Isolated CAN transceiver | TI **ISO1042BD** | SOIC-8/16 | 1 | Single chip: isolation + CAN transceiver. VIO logic side 1.71–5.5V (true 3.3V TWAI compatibility — the ISO1050 requires 4.5–5.5V on its logic side and was replaced for this reason). Bus side 5V from B0305S. |
| U5 | Isolated DC-DC | B0305S-1W (3.3V→5V, Mornsun) | SIP-4 | 1 | Powers ISO1042 bus side. Galvanic barrier between node logic and CAN bus ground. |
| C_CAN | Isolation decoupling | 10µF + 100nF ×2 sides | 0402/0805 | 4 | Per ISO1042 + B0305S datasheets. |
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

**AP63203 buck converter:** place inductor (L_PWR) and output cap (C24) as close as possible to the IC SW and output pins — standard buck layout rules. Shielded inductor mandatory. Keep the switching loop (VIN → IC → L_PWR → C24 → GND) away from the antenna keepout zone and away from BQ76920 VC sense lines — the 3.3V buck switches at ~1.5MHz, which is within the range that can couple into VC traces if carelessly placed.

**High-current path** (`B− → D1 → Q1 → shared node → R1 → Q2 → P−`): polygon pours ≥3.5mm at 2oz. Cluster 6–8 vias (0.4mm drill) in parallel if layer change unavoidable. No single vias on 12A+ paths.

**Kelvin sensing on R1 — most critical rule:**

```
WRONG:  sense taps anywhere on power pour near R1
CORRECT: sense taps DIRECTLY at shunt end pads,
         0.15mm dedicated traces to BQ76920 SRP/SRN pins only
```

At 3mΩ nominal: 0.5mΩ trace error = 17% OCP threshold shift. Not a refinement — a functional requirement.

**VC sense lines:** JST tap → 100Ω → VC pin, 0.1µF to GND at pin. Minimum 3mm from any balancer inductor. Cross power polygon only perpendicular, opposite layer.

**Balancer placement:** each instance adjacent to its two cell-tap nodes. Inductor between half-bridge SW node and MID tap — minimal loop area. Level-shifter transistors near the gates they drive, not near the ESP32. Shielded inductors only.

**1206 chip antenna — placement:**
- Board edge, element overhanging or flush, radiating end outward
- Top layer only: ~3mm clearance in front, ~1mm sides — no copper/traces/vias in this zone on top layer only
- **Inner layers and bottom: no keepout** — layer 2 GND plane continues normally right up to the antenna pad boundary
- All switching sources (FETs, balancer inductors, power polygon) on the opposite end of the board from the antenna
- VC sense lines ≥3mm from antenna element
- π-match directly at LNA_IN — use Espressif chip antenna reference values (distinct table from trace antenna in the hardware design guide)

**AON6354 DFN5×6 thermal vias:** minimum **3×3 grid (9 vias)**, 0.3mm drill, 1.0mm pitch under each exposed drain pad. Tented on top side (solder mask over via entry — prevents paste from wicking into via during stencil print), open on bottom side (allows solder to partially fill via during reflow, improving thermal conductivity). Connect via landings to a solid 2oz copper pour minimum 8×8mm around each device on the bottom layer, stitched to the inner GND plane.

**Crystal:** guard ring stitched to layer 2 every 1mm. No switching traces within 3mm. XTAL_P/N traces ≤5mm, no vias.

**CAN isolation gap:** no copper crosses the ISO1042/B0305S barrier on any layer. ≥4mm creepage. Bus-side ground pour is an isolated island fed only by B0305S output.

**Ground plane:** continuous layer 2 (inner GND plane), no split — except CAN isolation island. BQ76920 AGND and DGND both to same plane per datasheet. With chip antenna there is no inner layer keepout void — layer 2 is fully continuous.

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

Node 2's battery ground sits 14.8V above Node 1's; Node 4's sits ~44V above Node 1's. Shared CAN ground without isolation = dead short through the communication wiring. The ISO1042 + B0305S create a floating data network spanning the full stack potential — the master sees only data packets, never pack voltages.

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

- **FUSB302:** USB-C PD physical layer. Detects what is plugged in (source or sink via CC pin monitoring). Negotiates PD voltage as sink. Advertises capabilities as source. Communicates over I2C to the **BMS ESP32**.
- **BQ25713B:** 4-switch buck-boost charger controller. Receives instructions from the **BMS ESP32** over I2C. Charges the 4S pack in sink mode. Outputs 5V or 9V on VBUS in OTG mode. Never makes autonomous role decisions — the BMS ESP32 is always the decision maker.

The charger board connects to the BMS board via:
- **I2C:** shared bus carrying FUSB302 (0x22) and BQ25713B (0x6B) — routed to BMS ESP32 SDA/SCL via a dedicated connector
- **XT30:** CHG+/CHG− power path to BMS J3
- **JST-XH:** cell tap pass-through for balancer reference during charge

**The BMS ESP32 is the sole arbiter** — this is the correct architecture because it already knows SOC, cell voltages, temperature, and current. Charge and OTG decisions are battery-state-aware:

```
Adapter plugged in:
  FUSB302 fires interrupt → BMS ESP32 wakes (ALERT or dedicated IRQ GPIO)
  BMS ESP32 reads FUSB302: Rp on CC → source attached
  BMS ESP32 negotiates PD voltage (requests 20V or best available)
  BMS ESP32 writes BQ25713B: Ichg, Vchg, input current limit
  BQ25713B charges pack
  BMS ESP32 adjusts Ichg dynamically based on cell temp / SOC state

Phone/device plugged in:
  FUSB302 fires interrupt → BMS ESP32 wakes
  BMS ESP32 reads FUSB302: Rd on CC → sink attached
  BMS ESP32 checks SOC — enforces power limit based on pack state:
    SOC > 50%: advertise 9V/2.2A (20W)
    SOC 20–50%: advertise 5V/3A (15W)
    SOC < 20%: advertise 5V/0.9A (4.5W) — protect pack from deep discharge
  BMS ESP32 writes BQ25713B OTG voltage + current registers
  BMS ESP32 sets OTG_CONFIG enable bit
  BQ25713B outputs on VBUS
  FUSB302 advertises chosen Source_Capabilities
  Device negotiates → BMS ESP32 responds via FUSB302

Nothing plugged in:
  FUSB302 idle, BQ25713B idle, BMS ESP32 returns to deep sleep
```

**SOC-aware OTG** is a key benefit of this architecture: the BMS ESP32 knows exactly how much capacity remains and can throttle or cut OTG output to protect the pack — something the amp ESP32 could never do because it has no direct knowledge of battery state.

**Role conflict safety:** BMS ESP32 never enables charge and OTG simultaneously. OTG enable written only after FUSB302 confirms sink role. Charge enabled only after source role confirmed. Hardware interlock: BQ25713B ACOK pin — OTG only entered when ACOK is low (no adapter present).

**Standalone operation:** the charger board is fully functional with only the BMS board connected — no amp board needed. This makes the BMS node a viable standalone charging solution for any project, completely independent of the speaker or any host application.

### 4.2 FUSB302 — USB-C PD Physical Layer

The FUSB302 (onsemi, FUSB302B variant) handles all USB-C physical layer operation:

- CC1/CC2 monitoring — detects Rp (source attached) or Rd (sink attached)
- DRP toggling — for connecting to other DRP devices (laptops)
- PD message physical layer (BMC encoding/decoding on CC lines)
- I2C interface to **BMS ESP32** (address 0x22) — on the same I2C bus as BQ76920 and BQ25713B
- Interrupt line to **BMS ESP32** GPIO — fires on any CC event, attachment, detachment, PD message — wakes BMS ESP32 from deep sleep
- VCONN switch control for cable e-marker chips

The BMS ESP32 runs a PD policy engine firmware layer on top — open-source implementations (pd_buddy, fusb302-esp) handle the PD state machine. The FUSB302 handles bits; the BMS firmware handles protocol. All charging and OTG decisions are made by the same firmware that tracks SOC, cell voltages, and temperature — enabling fully battery-aware power management.

**FUSB302 availability:** onsemi part, Mouser/Digikey consistently stocked, ~$1.50.

### 4.3 BQ25713B — 4-Switch Buck-Boost Charger Controller

TI BQ25713B: synchronous NVDC buck-boost battery charge controller, 1–4S, I2C.

**Why BQ25713B:**
- 3.5–24V input range — accepts all PD voltages (5V, 9V, 12V, 15V, 20V) plus legacy adapters and barrel jacks. Works with any charger, not just USB-PD.
- Automatic buck/boost/buck-boost mode transition without host control — seamless across all input voltages
- 4× external N-channel FETs — allows FET optimisation for this voltage/current range
- Single inductor topology
- I2C (SMBus compatible) — **BMS ESP32** sets charge current, input current limit, reads VBUS, IBAT, VBAT, fault status
- OTG output: 4.48V–20.8V adjustable, up to 3A from battery — 5V/3A = 15W, 9V/2.2A = 20W
- Integrated ADC: VBUS, IBAT, VBAT, VSYS all readable over I2C — useful for the BMS webapp display
- ACOK pin: indicates valid input present — used as hardware interlock for OTG enable
- CHRG_OK, PROCHOT, IBAT monitor outputs
- ~$3.20 at Digikey 100 units, 228 units in stock (verify at order time; TI part with regular restocking)
- 32-QFN (4×4mm) — ENIG + stencil, same class as other ICs in this design

**External FETs — 4× AON6354:**
- 30V Vds, 83A Id, Rds(on) ~5.2mΩ @ Vgs=4.5V, DFN5×6
- At 2.5A charge current: P_conduction = 2.5² × (4 × 0.0052) = 0.13W total — trivial
- Switching losses at 800kHz: ~0.4W total across 4 FETs
- Gate drive from BQ25713B integrated gate drivers — no external driver needed
- Same part as BMS protection FETs — single AON6354 across entire project BOM

**Single inductor:**
- Value: 2.2–4.7µH (per BQ25713B datasheet recommendation for 4S)
- Isat: ≥5A (charge current 2.5A + ripple)
- Shielded, DCR <50mΩ
- Recommend: Bourns SRR6028-4R7Y or Sunlord SWPA6028S4R7MT

### 4.4 OTG Operation and Safety

BQ25713B OTG output is explicitly controlled by the **BMS ESP32** via I2C register (OTG_CONFIG bits). The safety chain:

1. FUSB302 fires interrupt — sink device detected on CC
2. **BMS ESP32** wakes (FUSB302 IRQ pin wired to ESP32 GPIO)
3. **BMS ESP32** reads FUSB302: Rd confirmed, role = source
4. **BMS ESP32** checks BQ25713B ACOK: must be LOW (no adapter) — if HIGH, OTG is blocked
5. **BMS ESP32** reads SOC from BQ76920 — determines OTG power limit:
   - SOC > 50%: OTG_VOLTAGE = 9V, OTG_CURRENT = 2.2A (20W)
   - SOC 20–50%: OTG_VOLTAGE = 5V, OTG_CURRENT = 3A (15W)
   - SOC < 20%: OTG_VOLTAGE = 5V, OTG_CURRENT = 0.9A (4.5W)
   - SOC < 10%: OTG refused entirely — protect pack from deep discharge
6. **BMS ESP32** writes OTG_VOLTAGE and OTG_CURRENT registers
7. **BMS ESP32** sets OTG_CONFIG enable bit
8. BQ25713B activates OTG path
9. FUSB302 advertises Source_Capabilities matching the chosen limits

On adapter insertion during OTG: ACOK goes HIGH → **BMS ESP32** detects via interrupt → immediately clears OTG enable bit → BQ25713B switches to charge mode. No power contention.

**This SOC-aware OTG logic is only possible because the BMS ESP32 owns both the battery monitoring (BQ76920) and the charger control (BQ25713B + FUSB302) on the same I2C bus.** The amp ESP32 has no role in this decision chain.

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
| C11, C11b | Bootstrap caps BTST1/BTST2 | 100nF X7R ×2 | 0402 | 2 | One per high-side gate driver — the 4-switch stage has two high-side FETs, both need bootstrap. |
| C12, C13 | FUSB302 decoupling | 100nF + 1µF | 0402/0805 | 2 | VDD and VCONN supply pins. |
| R_ISET | Charge current | Per BQ25713B Table — or I2C only | 0402 | 1 | Optional: sets minimum charge current floor. Primary current set via I2C. |
| R_AC | Input current sense | 10mΩ ±1% 1W | 2512 | 1 | Between ACP/ACN — adapter/input current sensing. Kelvin connections. **The BQ25713 requires two sense resistors.** |
| R_SR | Charge current sense | 10mΩ ±1% 1W | 2512 | 1 | Between SRP/SRN — battery/charge current sensing. Kelvin connections. |
| R1, R2 | I2C pull-ups | 4.7kΩ ×2 | 0402 | 2 | SDA/SCL for shared I2C bus (FUSB302 + BQ25713B). |
| R3 | FUSB302 VCONN R | 1kΩ | 0402 | 1 | VCONN current limit per FUSB302 application circuit. |
| — | ~~Battery NTC~~ | — | — | 0 | Removed: the BQ25713 has no TS/NTC input pin. Temperature-based charge derating is done in firmware — the BMS ESP32 reads the cell NTCs via BQ76920 and writes reduced Ichg over I2C (§7.2). |
| D1 | VBUS TVS | SMAJ20A 400W | SMA | 1 | Clamps VBUS transients. Between USB-C VBUS and BQ25713B input. |
| F1 | Polyfuse | 2.5A hold / ~5A trip | 1812 | 1 | VBUS overcurrent before IC. Self-resetting. |
| J1 | USB-C receptacle | HRO TYPE-C-31-M-12 or equiv — **16-pin** | SMD mid-mount | 1 | CC1/CC2 to FUSB302. VBUS/GND full copper fill, thermal relief OFF. 4-pin USB-C has no CC lines and cannot do PD. |
| J2 | I2C + INT to BMS ESP32 | JST-SH **5-pin** 1.0mm | SMD | 1 | GND / 3.3V / SDA / SCL / INT. I2C bus (FUSB302 0x22 + BQ25713B 0x6B) plus FUSB302 INT line to a BMS ESP32 RTC-capable GPIO. Pull-ups live on the BMS board only — one set per bus. |
| J3 | Cell tap pass-through | JST-XH 5-pin | THT | 1 | B1/B2/B3 mid-taps from charger side to BMS. |
| J4 | Power output | XT30U-M | pads | 1 | CHG+/CHG− to BMS J3. |
| LED1 | Charging | Green 0402 | 0402 | 1 | BQ25713B CHRG_OK pin. |
| LED2 | OTG active | Blue 0402 | 0402 | 1 | Driven by BMS ESP32 GPIO via I2C cable — or directly from BQ25713B OTG status pin if available. |
| LED3 | Fault | Red 0402 | 0402 | 1 | BQ25713B FAULT or ACOK status. |

### 4.6 Charger Layout Rules

**BQ25713B QFN-32 thermal pad:** 9 minimum vias (3×3), 0.3mm drill, tented bottom. 15×15mm 2oz bottom pour. On 4-layer board, inner GND plane provides primary thermal mass.

**Inductor placement:** as close as possible to BQ25713B SW1/SW2 pins. Every mm of trace = parasitic inductance = EMI + efficiency loss. Input caps between USB-C and IC; output caps between IC and output connector.

**FET placement:** 4× AON6354 arranged to minimise switching loop area. Gate resistors close to FET gates. Exposed pads connected to drain copper pours. Body diode orientation per BQ25713B reference schematic — verify before routing.

**FUSB302:** CC1/CC2 traces 0.15mm, direct to IC pins, minimal length. No routing under the IC that could couple into CC lines. VBUS trace wide (≥2mm) from connector to TVS to BQ25713B input.

**USB-C connector:** VBUS/GND pads — solid copper fill, thermal relief OFF (5A through these pads). CC1/CC2 — 0.15mm signal traces.

**R_AC / R_SR Kelvin:** same rule as BMS R1 — sense taps directly at each resistor's end pads, dedicated 0.15mm traces to the BQ25713B ACP/ACN and SRP/SRN pins respectively.

---

## 5. PCB Layer Count Decision

Both boards are 4-layer. Same JLCPCB process, same stackup (JLC7628, 1.6mm), one order.

| Board | Layers | Copper | Reasoning |
|---|---|---|---|
| BMS node | **4** | 2oz outer / 1oz inner | Bare ESP32-S3 SoC requires solid inner GND plane (layer 2) as reference plane for chip antenna π-match and crystal guard ring via stitching. 15kHz balancer half-bridges beside ±1mV BQ76920 sense lines benefit from plane separation and clean return current paths. SoC QFN-56 thermal via stack to inner plane. With chip antenna the inner plane is fully continuous (no keepout void) — simpler than trace antenna while retaining all thermal and signal integrity benefits. |
| Charger | **4** | 2oz outer / 1oz inner | BQ25713B QFN at 2–4W needs inner GND plane (drops θJA ~30–40%); 800kHz 4-switch switching nodes need reference plane 0.2mm below; FUSB302 CC lines beside 5A VBUS benefit from shielding. Without 4-layer, high-power OTG pushes toward thermal derating. |

**Stackup:** Signal top (2oz) / GND plane (1oz) / Power or signal (1oz) / Signal bottom (2oz). JLC7628 standard prepreg.

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

Total board dissipation: ~2.3W maximum. No single component stressed. DFN5×6 exposed pad is critical — it must be soldered to a solid copper pour (not just the DFN land pattern). Add a **minimum 3×3 grid (9 vias)**, 0.3mm drill, 1.0mm pitch under each exposed pad, tented top side / open bottom side, connecting to the inner GND plane. Minimum 8×8mm 2oz copper pour around each device on the bottom layer.

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
                    DEEP SLEEP (~180µA from B+)
                    FUSB302 in low-power unattached mode
                    BQ25713B ADC disabled
        ┌──────────────┬──────────────┬──────────────┬─────────────┐
     RTC timer      ALERT GPIO     UART RX       FUSB302 INT   CAN RX*
     (60s, NVS-     (BQ76920       (host cmd)    (USB-C attach/ (*node mode)
     configurable)   fault)                       detach event)
        │              │              │              │              │
        ▼              ▼              ▼              ▼              ▼
   BALANCE CYCLE   FAULT HANDLER  HOST COMMS   CHARGER/OTG    CAN HANDLER
        └──────────────┴──────┬───────┴──────────────┴─────────────┘
                              ▼
            Read BQ76920 (VC1–4, current, temps, faults)
            Re-enable FUSB302 DRP if woken by FUSB302 INT
            Read FUSB302 status if IRQ wake (role, PD negotiation)
            Read/write BQ25713B if charge/OTG state changed
            Update coulomb count + SOC (RTC memory)
            Balance if Δ>30mV (≤30s, LEDC 15kHz, dead-time)
            Adjust Ichg based on temp/SOC if charging
            Log / respond / broadcast as applicable
            Update LEDs
                              ▼
            PRE-SLEEP SEQUENCE:
            1. Write FUSB302 CONTROL2 → disable DRP → low-power unattached (~10µA)
            2. Write BQ25713B → disable continuous ADC → idle (~50µA)
            3. Configure wake pins (RTC timer, ALERT, UART RX, FUSB302 INT)
            4. Enter deep sleep
```

RTC memory (8KB, survives sleep): SOC accumulator, coulomb count, fault history, cycle count, balance interval setting, last OTG power level, charger state, FUSB302 last known role.

### 7.2 Charger and OTG Management

The BMS ESP32 owns the entire charging and OTG decision chain. The FUSB302 IRQ pin is wired to a wake-capable ESP32 GPIO — any USB-C attach or detach event wakes the ESP32 immediately from deep sleep regardless of the balance timer interval.

**I2C bus on BMS ESP32:** BQ76920 (0x08) + FUSB302 (0x22) + BQ25713B (0x6B) all share the same I2C bus. The BMS ESP32 reads cell state and makes charger decisions in the same firmware context — no inter-board communication delay, no data staleness.

**Charge current management:** the BMS ESP32 adjusts Ichg dynamically during the charge cycle:
- Cell temperature >40°C: derate charge current by 50%
- Cell temperature >50°C: inhibit charging entirely
- Any cell group >4.15V: reduce current for top-balance convergence
- All groups within 10mV: resume full current

**OTG power limits by SOC:** see §4.4 for the full decision table. These thresholds are NVS-configurable.

### 7.3 Balancing Algorithm

### 7.3 Balancing Algorithm

Every wake: read 4 voltages → find max/min → evaluate delta:

**Passive mode (default):** Δ > 30mV → write CELLBAL1 for highest cell → sleep. BQ76920 bleeds ~55mA autonomously. Next wake: Δ < 10mV → clear CELLBAL. Passive bleed persists across sleep cycles uninterrupted.

**Active mode:** Δ > 30mV → select transfer path (direct adjacent or chained for non-adjacent) → drive instance(s) for ≤30s via LEDC 15kHz with firmware dead-time → re-read → if Δ < 10mV stop. Active only runs while ESP32 is awake.

Both modes: SOC drift-corrects against OCV table after ≥5min rest. Mode loaded from NVS on boot.

### 7.3 WiFi Webapp Mode

No UART host for >60s after boot → join configured WiFi (NVS) or AP mode `BMS_[MAC4]`. LittleFS-served dashboard at `192.168.4.1`: live cell group voltages, SOC%, pack current (charge/discharge), temperatures, fault log, balance status. Exits on first UART command.

### 7.4 Node Mode (CAN populated)

DIP-read node ID at boot → periodic CAN frames (voltages, SOC, current, temp, status) → obeys master commands → asserts/reacts to hardware FAULT line independent of CAN bus health. TWAI peripheral + ISO1042, wake-on-CAN via RX GPIO edge.

---

## 8. Host Communication Protocol

UART 115200 8N1, ASCII + `\n`. Send `PING` first to wake BMS (~5ms response).

| Command | Response | Description |
|---|---|---|
| `PING` | `PONG` | Wake + connectivity check. First `PING` after sleep wakes the BMS (its start bit is the wake edge and is consumed); reply arrives ~150–300ms later. Library retries automatically. |
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
| `STATUS` | `CHARGING/DISCHARGING/OTG/IDLE/FAULT` | Pack state including charger mode |
| `CHARGER_STATUS` | `CHARGING:20V:2.5A` / `OTG:5V:0.9A` / `IDLE` | Current charger/OTG state with voltage and current |
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
| Via tenting | Open (no inner plane antenna keepout void required — chip antenna, top-layer clearance only) | Tented bottom under BQ25713B (fab note) |
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
3. ICs: BQ76920 (TSSOP-20), ESP32-S3 (QFN-56), BQ25713B (QFN-32), FUSB302 (QFN-16), Si2301 ×3, Si2302 ×4 (3 balancer + Q3), 2N7002 ×6, BSS84 ×3, AP63203, AON6354 ×6.
4. Crystal (3225) — heat-sensitive, place carefully.
5. Protection FETs Q1/Q2 (AON6354 DFN5×6), shunt resistors (2512).
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
**Passive mode test:** set adjacent simulated cells to 3.800V and 3.650V → confirm `BALANCE_STATE` UART response shows `ACTIVE:n` → measure BALx pin pulled low on correct channel → confirm 55mA bleed current on that cell PSU → delta closes over multiple wake cycles.

**Active mode test:** send `SET_BALANCE_MODE:ACTIVE` → set same 150mV delta → oscilloscope on Q_H/Q_L gate pair: confirm 15kHz complementary drive with dead-time gap; verify gate swings (Si2301: TOP→TOP−6.2V max; Si2302: BOT→MID) → inline ammeter confirms current transfer → delta closes over 2–3 minutes.

Repeat active test for each of the 3 instances. Test a non-adjacent transfer (C1 and C4 offset) — confirm both intermediate instances activate as needed. Send `SET_BALANCE_MODE:PASSIVE` → confirm mode persists across power cycle.

### 10.5 UART Protocol Verification

Connect CP2102 to J4 → send `PING` → `PONG` within 10ms (awake) or ~150–300ms (from deep sleep; first PING consumed as wake edge — retry) → send `CELL_ALL` → four mV values matching PSU set points ±5mV → trigger OVP → confirm `FAULT:OVP:CELL1` arrives unsolicited within 100ms → `SET_BALANCE_INTERVAL:120` → `ACK` → power cycle → verify interval persisted in NVS.

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
- **Mounting:** M3 nylon standoffs ≥10mm. ≥10mm clearance between charger board and audio signal traces. BMS board oriented so chip antenna edge faces outward (toward speaker vent or clear enclosure wall, away from battery cells).

### 11.2 Scaling Checklist (Future)

1. Populate CAN section on each node (~€5 each), set unique DIP IDs
2. Stack packs in series (Node1 P+ → Node2 P−, etc.)
3. Isolated DC-DC per node (B0305S) powers each CAN bus side independently
4. Daisy-chain 4-wire CAN cable (CANH/CANL twisted, FAULT, ISO_GND)
5. 120Ω termination solder jumper SJ1: closed only at the two physical end nodes
6. Master aggregator (Pi Zero 2W) on the floating CAN bus: logging, GUI, inter-node balancing authority
7. Verify FAULT line trips entire stack from every node before first real charge

### 11.3 Portfolio / Curriculum Summary

This project demonstrates: multi-cell Li-ion protection and management · 4-switch synchronous buck-boost with external FET design · USB-C PD negotiation (sink and source) via FUSB302 + firmware policy engine · inductive active cell balancing · bare ESP32-S3 SoC RF layout (1206 chip antenna matching network, crystal design) · BQ76920 analog front-end integration with Kelvin sensing · deep-sleep embedded firmware architecture with interrupt-driven wake and pre-sleep peripheral management · I2C multi-device bus management · UART command protocol design and companion library architecture · distributed CAN bus architecture with galvanic isolation · hardware fault interlock design · 4-layer PCB with thermal via stack and inner GND plane · multi-stage test methodology with threshold measurement and verification.

---

## 12. Key Design Decisions Reference

| Decision | Choice | Rationale |
|---|---|---|
| Protection IC | BQ76920 | ±1mV accuracy, I2C to ESP32, configurable thresholds, hardware defaults active before/without firmware. Replaces S-8254AA entirely — no redundant sense lines. Mouser/Digikey stocked. |
| Balancing | **Both passive and active — firmware selects** | Passive: BQ76920 BALx + 68Ω bleed resistors (~55mA), continues during deep sleep, default mode. Active: Si2301/Si2302 half-bridges + 2N7002/BSS84 level shifters + 22µH ×3, ESP32 LEDC 15kHz, ~1A transfer, stops during deep sleep. Mutually exclusive — mode stored in NVS, switchable via UART `SET_BALANCE_MODE`. TC4427 rejected: 4.5V min supply + floating gate problem. |
| 3.3V supply | **AP63203 synchronous buck** | Direct B+ input (12–16.8V) → 3.3V. No LDO needed — ESP32-S3 tolerates switching supply; BQ76920 has internal analog LDO. Sourced from B+ so ESP32 stays powered through any protection FET event. ~50µA IQ. Shielded inductor placed away from antenna and VC sense lines. |
| Shunt | 3mΩ 1% 2W | BQ76920 OCD max register = 56mV. At 8mΩ max trip = 7A (wrong). At 3mΩ: 18.7A trip (correct for this load). |
| MCU | ESP32-S3 bare SoC + W25Q64 + 40MHz crystal + 1206 chip antenna | Full RF control, smaller footprint vs module. 1206 chip antenna replaces trace antenna — top-layer clearance only, inner plane continuous, better enclosure performance. Native USB = flash + JTAG + serial via 4 pads. |
| BMS board layers | **4-layer** | Bare SoC needs inner GND plane for chip antenna π-match reference and crystal guard ring stitching. Balancer FET switching beside ±1mV sense lines benefits from plane separation. Inner plane fully continuous with chip antenna (no keepout void). |
| Antenna | **1206 ceramic chip antenna** (Johanson 2450AT18A100E) | Replaces PCB trace antenna. Top-layer clearance only (~3mm front, ~1mm sides). Inner GND plane continuous — no all-layer keepout void. Better performance inside speaker enclosure. Hand-solderable. ~€0.30. |
| Charger board layers | **4-layer** | BQ25713B + FET switching EMI, CC line integrity beside 5A VBUS, inner GND plane thermal mass. |
| Charger IC | **BQ25713B** | 3.5–24V input (all PD voltages + legacy chargers), 4S native, I2C, external FETs, OTG 5V/3A or 9V/2.2A, ~$3.20 Digikey 100 units. Sourcing confirmed. |
| Charger comms ownership | **BMS ESP32** | BMS ESP32 owns BQ76920 + FUSB302 + BQ25713B on the same I2C bus. Charging and OTG decisions are battery-state-aware (SOC, temp, cell voltage) in the same firmware context. Amp ESP32 is completely uninvolved — charger board + BMS board is a standalone charging system usable in any project. |
| SOC-aware OTG | OTG power limit scaled by SOC | >50%: 20W. 20–50%: 15W. <20%: 4.5W. <10%: refused. Possible only because BMS ESP32 owns both measurement and charger control simultaneously. |
| OTG | Enabled via BQ25713B register | FUSB302 detects sink, ESP32 confirms ACOK=LOW (no adapter), writes OTG register. Hardware interlock prevents simultaneous charge + OTG. Scraps separate USB-A + AP62300 boost — OTG comes free with BQ25713B. |
| FETs (all positions) | **AON6354 DFN5×6** | 30V, 83A, 5.2mΩ @ Vgs=4.5V. Single part across entire BOM: BMS CHG/DSG + charger 4-switch stage. Exposed drain pad bonds directly to PCB copper pour — better thermal spreading than SO-8. No heatsink needed on either board. |
| Heatsink | **None required** | AON6354 ultra-low Rds distributes losses trivially. BQ25713B controller dissipates ~0.5W (no switching losses in controller IC). Total charger board dissipation ~1W. No single component stressed. |
| Inductor (charger) | 4.7µH Isat ≥5A shielded, single | BQ25713B uses single-inductor 4-switch topology. Isat ≥5A covers 2.5A charge + ripple margin. |
| Scaling model | Stack identical nodes over isolated CAN | One PCB design: speaker → powerwall. ISO1042 + B0305S-1W + dual daisy ports + DIP ID + 120Ω jumper as ~€5 unpopulated optionality. |
| USB-A output | **Removed** | BQ25713B OTG provides 5V/3A (15W) or 9V/2.2A (20W) from same USB-C port. Separate USB-A + AP62300 + inductor eliminated. |


---

## 13. Pre-Schematic Design Review — Findings & Required Configuration

This section documents the full design review performed before schematic entry (Rev 6.0 → 6.1). Items marked ✅ are already corrected in this document; items marked ⚠ are configuration or verification steps to perform during schematic/firmware work.

### 13.1 Critical corrections applied (✅)

1. **✅ Balancer gate drive redesigned + dual mode implemented.** The original TC4427-driven design had two fatal flaws: ≥4.5V supply requirement and floating gates unreachable by any ground-referenced driver. Corrected to P-ch/N-ch half-bridges with 2N7002/BSS84 open-drain level shifters and zener gate clamps (§2.3). Switching frequency 50kHz → **15kHz**, inductors 4.7µH → **22µH**, ~1A transfer current. **Both passive (BQ76920 BALx + 68Ω bleed resistors) and active (discrete half-bridges) are implemented.** Firmware selects via NVS-stored mode enum. Default: PASSIVE. Passive continues during deep sleep; active stops when ESP32 sleeps. Modes are mutually exclusive.
2. **✅ Series Schottky (SS34) removed from the main B+ path.** A 3A diode in a 12A path would overheat and fail, while wasting ~5W. Reverse protection is the keyed JST-XH connector; the BQ76920 REGSRC line gets its own 1kΩ + 5.1V zener protection per datasheet.
3. **✅ UART deep-sleep wake corrected.** The ESP32-S3 cannot wake from deep sleep on UART data (light-sleep only feature). The RX pin is now specified as an EXT falling-edge wake source — the host's first `PING` start bit wakes the chip (~150–300ms full reboot, RTC RAM intact); the companion library retries automatically.
4. **✅ BQ25713B requires two sense resistors** (R_AC between ACP/ACN for input current, R_SR between SRP/SRN for charge current) **and two bootstrap caps** (BTST1/BTST2 — one per high-side FET). BOM corrected.
5. **✅ ISO1050 → ISO1042.** The ISO1050's logic side requires 4.5–5.5V — incompatible with 3.3V TWAI. The ISO1042's VIO pin accepts 1.71–5.5V. Direct swap, same isolation architecture.
6. **✅ Pre-charge FET upsized.** 2N7002 (300mA) replaced with Si2302 + 22Ω (0.76A peak inrush, ~250ms PVDD soft-start).
7. **✅ Charger-board NTC removed** — the BQ25713 has no TS pin. Thermal charge derating is firmware: BMS ESP32 reads cell NTCs via BQ76920 and reduces Ichg over I2C.
8. **✅ AP63203 corrected to 2A** (not 3A; family: AP632xx = 2A, AP633xx = 3A), IQ ~22µA, TSOT-26. 2A covers worst-case load with margin.
9. **✅ Flash package fixed** — W25Q64JVSSIQ is SOIC-8; the WSON-8 variant is W25Q64JVZPIQ.
10. **✅ Antenna vendor corrected** — 2450AT18A100E is a Johanson Technology part (not Molex).
11. **✅ Charger J2 → 5-pin JST-SH** (GND/3.3V/SDA/SCL/INT). I2C pull-ups on the BMS side only — one set per bus.

### 13.2 BQ76920 — required configuration & behavioral notes (⚠)

- **⚠ FETs are OFF at power-up.** CHG_ON / DSG_ON bits (SYS_CTRL2) default to 0 — the host must enable them. An unprogrammed board outputs nothing (safe). Once enabled, all protections act **autonomously in hardware** even if the ESP32 crashes.
- **⚠ OCD/SCD faults latch in SYS_STAT** — the corresponding FET stays off until the host clears the status bit over I2C. This is not auto-recovery; the firmware fault handler must clear flags after evaluating the cause. (OV/UV release with hysteresis but also set status bits to clear.)
- **⚠ Set RSNS = 1** in PROTECT1 — the 56mV OCD / upper threshold table assumes the high-range setting.
- **⚠ SCD threshold must be ≥ 89mV** (= 29.7A at 3mΩ). The minimum SCD setting (44mV = 14.7A) would sit *below* the 18.7A OCD trip and fire first. Recommended: SCD = 89mV or 111mV.
- **⚠ OV/UV trip registers** (OV_TRIP, UV_TRIP) must be written at first boot — program 4.20V / 2.80V (or chosen values) and verify with the bench-PSU test (§10.3) after configuration, not before.
- **⚠ Boot circuit on TS1:** the BQ76920 boots from SHIP mode when TS1 is pulled high momentarily. Provide: the NTC network per datasheet + a small boot pushbutton footprint + an ESP32 GPIO via diode to TS1 so firmware can wake the AFE programmatically.

### 13.3 BQ25713B — required configuration (⚠)

- **⚠ I2C watchdog:** ChargeOption0 enables a watchdog (~175s default) that suspends charging if the host goes silent. Either disable the WDT at init, or keep the ESP32 awake / on a short wake interval while ACOK is high (recommended — power budget is irrelevant while an adapter is connected).
- **⚠ MinSystemVoltage / charge voltage registers:** program for 4S (16.8V max charge, MinSysVoltage ≈ 12.0V) at init; verify output on a dummy load before connecting the pack (§10.6).
- **⚠ Follow the datasheet §Typical Application exactly** — including the optional input ACFET/RBFET back-to-back N-FET pair (ACDRV-driven) for input reverse blocking and OTG port isolation. Budget 2× additional AON6354 positions on the schematic; populate per the reference design if OTG is used.
- **⚠ ILIM_HIZ and PROCHOT pins** need their strap/divider networks per datasheet even if unused.

### 13.4 ESP32-S3 bare-SoC checklist (⚠)

- **EN pin RC:** 10kΩ pull-up + 1µF to GND (power-on reset timing).
- **Strapping pins:** GPIO0 (BOOT button, pulled up), GPIO46 (pulled down/floating per guidelines), GPIO45 (sets VDD_SPI voltage — leave per 3.3V flash), GPIO3 — review the strapping table in the hardware design guidelines before assigning any of these to balancer/level-shifter duty.
- **Wake-capable pins:** ALERT, FUSB302 INT, UART RX, and CAN RX must all land on **RTC-capable GPIOs (GPIO0–21)** for EXT wake to function.
- **Native USB:** D− = GPIO19, D+ = GPIO20 — reserved, routed only to J5 pads.
- **Flash connects to the dedicated SPI pins** (SPICS0/SPIQ/SPID/SPICLK/SPIHD/SPIWP) — not arbitrary GPIO.

### 13.5 System-level verifications (⚠)

- Charger output → BMS CHG path inrush: small (converter soft-starts); the pre-charge circuit covers the discharge side only — confirmed adequate.
- When the BQ25713B is unpowered, its battery-side node remains connected to the converter FETs' body diodes — system-level disconnect is provided by the **BMS CHG FET**, which the BQ76920/ESP32 controls. No additional isolation FET required on the charger board.
- UART 3.3V pin on J4: optional supply to a host dongle — leave unconnected in the speaker cable to avoid back-feeding the amp board's own 3.3V rail.
- JLCPCB 4-layer with 2oz outer copper is a supported (non-default) option — select explicitly at order time.

---

## 14. Datasheet Library — Schematic Reference PDFs

Direct links for every IC and key component. TI `/lit/ds/symlink/` links always redirect to the latest datasheet revision. If any other link rots, search `<part number> datasheet pdf` — all are current production parts.

| Part | Function | Datasheet |
|---|---|---|
| BQ76920 | BMS AFE / protection | https://www.ti.com/lit/ds/symlink/bq76920.pdf |
| BQ25713 / BQ25713B | Buck-boost charger controller | https://www.ti.com/lit/ds/symlink/bq25713.pdf |
| FUSB302B | USB-C PD PHY | https://www.onsemi.com/pdf/datasheet/fusb302b-d.pdf |
| ESP32-S3 (SoC) | MCU | https://www.espressif.com/sites/default/files/documentation/esp32-s3_datasheet_en.pdf |
| ESP32-S3 HW Design Guidelines | RF / strapping / layout bible | https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/ |
| W25Q64JV | 8MB NOR flash | https://www.winbond.com/hq/product/code-storage-flash-memory/serial-nor-flash/?__locale=en&partNo=W25Q64JV |
| AON6354 | Power N-FET (all positions) | https://aosmd.com/res/data_sheets/AON6354.pdf |
| AP63203 | 3.3V 2A buck | https://www.diodes.com/assets/Datasheets/AP63200-AP63201-AP63203-AP63205.pdf |
| Si2301 | P-ch FET (balancer high-side) | https://www.vishay.com/docs/70628/si2301bds.pdf |
| Si2302 | N-ch FET (balancer low-side, pre-charge) | https://www.vishay.com/en/product/63597/ |
| 2N7002 | Level shifter N-FET | https://assets.nexperia.com/documents/data-sheet/2N7002.pdf |
| BSS84 | Level shifter P-FET | https://assets.nexperia.com/documents/data-sheet/BSS84.pdf |
| BZT52C6V2 | 6.2V gate-clamp zener | https://www.diodes.com/assets/Datasheets/ds18004.pdf |
| ISO1042 | Isolated CAN transceiver | https://www.ti.com/lit/ds/symlink/iso1042.pdf |
| B0305S-1WR2 | Isolated 3.3V→5V DC-DC | https://www.mornsun-power.com/html/products/681/B0305S-1WR2.html |
| PESD1CAN | CAN bus ESD | https://assets.nexperia.com/documents/data-sheet/PESD1CAN.pdf |
| PC817 | Fault-line optocoupler | https://global.sharp/products/device/lineup/data/pdf/datasheet/PC817XxNSZ1B_e.pdf |
| 2450AT18A100E | 1206 chip antenna + matching app note | https://www.johansontechnology.com/datasheets/2450AT18A100/2450AT18A100.pdf |
| TAS5825M | Amplifier (system context) | https://www.ti.com/lit/ds/symlink/tas5825m.pdf |
| FA-238 (40MHz) | Crystal | https://www5.epsondevice.com/en/products/crystal_unit/fa238v.html |

**Schematic-day workflow suggestion:** open BQ76920 §"Typical Application," BQ25713 §"Application and Implementation," FUSB302B application schematic, and the ESP32-S3 hardware design guidelines side by side — those four reference circuits, plus §2.3's balancer topology and §13's configuration notes, are the complete blueprint. Draw power stages first (charger → BMS FET path → buck), then the AFE and sense networks, then the MCU block, then comms/CAN last.

---

*End of document — Rev 6.2*
