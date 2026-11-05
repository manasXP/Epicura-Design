---
created: 2026-03-21
modified: 2026-11-05
version: 2.3
status: Review Complete — dispositioned against design v3.x
schematic_revision: v1.1 (20-03-2026); dispositions against Unified_v1.3 KiCad and design doc v3.1 (2026-11-04)
---

# Schematic Review Report — Unified PCB v1 rev1.1

**Date:** 2026-03-21 (review); 2026-11-04 (v3.x disposition)
**Schematic reviewed:** rev 1.1 of what is now `Unified_v1.3/` (4 sheets, KiCad 9.0.4)
**Reviewer:** Claude (automated analysis)
**Scope:** Full electrical review of all 4 schematic sheets (Root, Controller, Driver, Connector)

> [!note] How to read this document
> §3 to §8 are the original rev 1.1 review and are kept verbatim as the record. The board has since been redesigned on paper as the single-board v3.x (design doc [[01-Unified-PCB-Design]] v3.1: Luckfox Pico Zero slot, Dispenser Servo Block, two CAN ports, 2S battery, STM32G0B1), and the KiCad project already carries the feedback-divider fixes and the G0B1 swap. §0 below gives the current status of every finding so the next schematic pass starts from a clean list.

---

## 0. Disposition against v3.x (2026-11-04)

Status key: **Fixed** = corrected in `Unified_v1.3/` (commit `6630e48` or later); **Resolved by design** = the v3.x design doc specifies the fix, to be drawn in the next KiCad pass; **Superseded** = the circuit no longer exists in v3.x; **Open** = still to do in the v3.x schematic or layout; **Accepted** = no change needed.

### 0.1 Critical

| ID | Finding (rev 1.1) | Status | v3.x note |
|----|-------------------|--------|-----------|
| C-01 | MP1584EN #2 feedback gave 3.8 V, not 6.5 V | **Fixed** | R36 71.5K → 14K; U8 now 6.51 V. The 6.5 V rail is the switched DSB servo rail in v3.x (§4.3 of the design doc) |
| C-02 | MT3608 boost could put 6.7 V on the 5 V bus | **Superseded** | MT3608, DW01A, FS8205A and TP4056 leave the board; the 2S pack feeds the TPS54531 through the VSYS diode-OR (design doc §4.4, §4.6). Delete the micro-UPS block in the next pass |
| C-03 | PCF8574AT addresses 0x38/0x39 vs firmware 0x20/0x21 | **Fixed** (value) / verify BOM | Both expander instances now carry Value `PCF8574T` on the `PCF8574AT` library symbol. v3.x keeps only PCF8574 #2 at 0x21 (P-ASD expander #1 removed); confirm the MPN in `BoM/` is the non-A part |
| C-04 | C34 470 µF rated 25 V on the 24 V rail | **Fixed** | C34 is now `470uF@35V` |
| C-05 | TPS54531 output unverified (6.5 V by calculation) | **Fixed** | R26 37.4K → 52.3K; U5 now 4.98 V. In v3.x U5 is fed from VSYS (24 V or battery), not from a 12 V UPS input |

### 0.2 High

| ID | Finding (rev 1.1) | Status | v3.x note |
|----|-------------------|--------|-----------|
| H-01 | Root label `PB6_FAN1_PWM` for a PA6 signal | **Fixed** | Root sheet reads `PA6_FAN1_PWM` |
| H-02 | AMS1117 dropout on battery-only 5 V | **Resolved by design** | 5V_MERGED is always the TPS54531 output (VSYS from 24 V or the 2S pack), so the LDO sees ≥ 4.9 V; the MT3608 path that could droop is gone |
| H-03 | INA219 10 mΩ shunt, poor resolution at 2–5 A | **Accepted** (firmware) | v3.1 keeps 10 mΩ (design doc §8); the 24 V peak is now 3 A, so run the INA219 at PGA ÷1 (±40 mV, 4 A full scale, 10 µV LSB). No hardware change; the shunt also serves the DSB stall window |
| H-04 | No common-mode choke on CAN at the connector | **Fixed** (heater bus) | A `Common_Mode_Choke_Epicura` part is placed on the Connector sheet for the isolated bus. The v3.x regular stir-motor bus (SN65HVD230, J_CAN_STIR) is short and internal; no choke planned |
| H-05 | No bypass near J_SLD1 3.3 V (HX711 supply) | **Open** (layout) | J_SLD and the external HX711 boards remain in v3.x (design doc §6.4); add 100 nF at the connector pin during layout |
| H-06 | TP4056 TEMP pin bypassed with a fixed 100K | **Superseded** | TP4056 is replaced by a TP5100 with a real 10 kΩ NTC on J_BAT pin 3 (design doc §4.6.2); the interim R92 fix becomes moot once the block is deleted |
| H-07 | J_PASD1 12 V pin at the DF11 3 A limit | **Superseded** | P-ASD pneumatics removed; each DSB servo has its own J_DSB header on the PTC-protected 6.5 V rail |
| H-08 | Schematic PC8 vs design doc PC5 for SLD water PWM | **Open** (pin map) | Reconcile in the G0B1 pin map: the architecture doc still lists the pumps on PC3–PC6 while the schematic uses PC8; pick the timer-capable pin on the G0 and update both |

### 0.3 Moderate

| ID | Finding (rev 1.1) | Status | v3.x note |
|----|-------------------|--------|-----------|
| M-01 | HSE load capacitors vs crystal CL | **Open** (verify) | 8 MHz HSE stays on the G0B1 (PF0/PF1); check Y1's CL and C11/C12 before fabrication |
| M-02 | CAN 120 Ω possible double termination | **Resolved by design** | Design doc §7: isolated bus terminated at J_CAN_HEAT plus 120 Ω inside the module (both on-board nodes are a short stub); stir bus terminated at J_CAN_STIR plus the motor. Confirm the module's and motor's built-in terminations |
| M-03 | BOOT0 pull-down not visible | **Resolved by design** | SW_BOOT to 3.3 V with 10k pull-down, plus R_AP_BOOT (1k) from J_AP GPIO 32. On the G0B1 BOOT0 is PA14 (shared with SWCLK) and needs nBOOT_SEL cleared in option bytes; the swapped U3 carries that note |
| M-04 | No series resistor on PB3 IRQ to the CM5 | **Superseded** | CM5 and J_CM5 removed; the host link is UART2 on J_AP (service/bootloader only) |
| M-05 | TB6612 STBY must be pulled high | **Resolved by design** | Design doc §6.2: STBY ← 3.3 V via 10k; the net exists on the Driver sheet |
| M-06 | DRV8876 PMODE mode select | **Superseded** | DRV8876 removed with the CID linear actuator; delete U11 in the next pass |
| M-07 | R37 dissipation at fault current | **Accepted** | 3 A peak on the 24 V rail (§4.11); F1 5 A PTC upstream |
| M-08 | J_CID1 stepper 24 V pin at 3 A | **Superseded** | No stepper; the 24 V CAN stir motor gets its own XT30 (J_STIR_PWR) |
| M-09 | No RC snubbers on inductive loads | **Accepted** (prototype) | Still applies to the SLD solenoids and pumps; add snubbers only if EMC testing requires |
| M-10 | No polyfuse on 5 V to the CM5 | **Open** | Now the Pico Zero on J_AP (0.5 A peak). Add a 1.5 A PTC on the J_AP 5 V pins so a Pico Zero fault cannot drop the STM32 |
| M-11 | WS2812B data at 3.3 V below the 5 V threshold | **Resolved by design** | Ring driven from STM32 PB14 through a 74AHCT1G125 level shifter and 33 R (design doc §6.6) |
| M-12 | I2C bus capacitance with 5 devices | **Accepted** (improved) | v3.x I2C1 has three on-board devices only (PCA9685 0x41, PCF8574 #2 0x21, INA219 0x40); the external MLX90614 and ADS1015 are gone, so bus capacitance drops to roughly 50 pF |

### 0.4 Sections superseded by the v3.x design

- **§5 pin allocation audit** describes the STM32G474RE map. The board is now the STM32G0B1RET6 (design doc §3): I2C1_SCL on PB6, VBUS sense PA0, battery sense PA4, buzzer PA8/TIM1_CH1, BOOT0 on PA14, FDCAN1 on PB8/PB9 and FDCAN2 on PB12/PB13; the SPI2 CM5 bridge, BLDC PWM/EN/DIR, CID actuator and P-ASD pins are retired.
- **§6 power budget** assumed a stepper and a BLDC on 24 V plus a 12 V UPS. The v3.1 budget (design doc §4.11) is 2.1 A typical / 3.0 A peak on the LRS-75-24 with the CAN stir motor and the DSB serialised in firmware; LRS-100-24 is the fallback if the measured peak exceeds 3 A. The 12 V rail carries fans, pumps, solenoids and the TP5100 charger (1.4 / 3.0 A); 5 V carries the Pico Zero, LCD, LDO, B0505S and LED ring (0.7 / 2.2 A).
- **§4 design strengths**: the 12 V input protection chain, the TP4056/DW01A/FS8205A battery chain and the XT30 12 V input no longer exist; the 24 V input chain, USB, CAN isolation and signal-protection points still hold.
- **§8 recommended changes for rev 1.2**: items 1–4, 6 and 7 are done or superseded as tabled above; item 5 (level shifter) and 9 (documentation) are specified in v3.x; item 8 (PSU) is the LRS-100-24 fallback.

### 0.5 Carried into the v3.x schematic pass

1. Delete the superseded blocks: MT3608/DW01A/FS8205A/TP4056 micro-UPS, J_12V_UPS, DRV8876 (U11), PCF8574 #1 (U9), J_PASD, J_CID, J_PRES, J_BLDC, J_IR_LED, J_CM5.
2. Draw the v3.x blocks: J_AP slot and J_LCD, MCP2518FD + ISO1050DUB #2 for the RV1106 CAN node, PCA9685 + Q_DSB + J_DSB1-4 + D_DSB4, SN65HVD230 + J_CAN_STIR + Q_STIR + J_STIR_PWR, TP5100 + J_BAT + VSYS diode-OR, 74AHCT1G125 on J_LED.
3. Close the open items: H-05 (J_SLD 3.3 V bypass), H-08 (SLD PWM pins on the G0 map), M-01 (crystal CL), M-10 (J_AP 5 V PTC).
4. Confirm the PCF8574 MPN in the BOM (C-03) and the module/motor CAN terminations (M-02).
5. Re-run ERC and a fresh review once the above is drawn; this document then retires in favour of a v3.x review.

---

---

## Executive Summary

The Unified PCB v1 rev1.1 schematic is a well-structured hierarchical design with strong fundamentals — proper input protection chains, galvanic CAN isolation, USB ESD protection, and comprehensive decoupling. However, this review identified **5 critical**, **8 high**, and **12 moderate** issues that should be addressed before PCB fabrication.

| Severity | Count | Key Theme |
|----------|-------|-----------|
| **Critical** | 5 | Voltage regulator feedback errors, I2C address mismatch, capacitor voltage rating |
| **High** | 8 | Missing protection, power margin, net naming errors |
| **Moderate** | 12 | Documentation mismatches, thermal concerns, signal integrity |
| **Low/Info** | 8 | Cosmetic, future improvements |

---

## 1. Schematic Structure Overview

```
Unified_v1.kicad_sch (Root — Sheet 1/4)
├── Control.kicad_sch (Sheet 2/4) — STM32G474RETx + LDO + crystals + debounce
├── Connector.kicad_sch (Sheet 3/4) — All external connectors + input protection
└── Driver.kicad_sch (Sheet 4/4) — Power conversion + motor drivers + I/O expanders + CAN
```

**Total Components:** ~160 unique parts (88 BOM lines)
**Hierarchical Pins:** ~161 inter-sheet signals
**Power Rails:** +24V, +12V, +6.5V, +5V_Merged, +5V_USB, +5V_LED_RING, +3.3V, VBAT, +5V_Isolated

---

## 2. Component Summary

### 2.1 Active ICs

| Ref | Part | Function | Sheet |
|-----|------|----------|-------|
| U1 | USBLC6-2SC6 | USB ESD protection | Connector |
| U2 | AMS1117-3.3 | 5V → 3.3V LDO (1A) | Controller |
| U3 | STM32G474RETx | Main MCU (Cortex-M4F, LQFP-64) | Controller |
| U4 | MP1584EN | 24V → 12V buck (3A) | Driver |
| U5 | TPS54531DDA | 12V → 5V buck (5A) | Driver |
| U6 | ISO1050DUB | Isolated CAN transceiver | Driver |
| U7 | INA219AxD | I2C current/power monitor | Driver |
| U8 | MP1584EN | 24V → 6.5V buck (3A) | Driver |
| U9 | PCF8574AT | I2C GPIO expander #1 (P-ASD solenoids) | Driver |
| U10 | PCF8574AT | I2C GPIO expander #2 (SLD solenoids, LEDs) | Driver |
| U11 | DRV8876PWPR | CID-2 linear actuator H-bridge (3.5A) | Driver |
| U12 | TB6612FNG | SLD dual peristaltic pump driver | Driver |
| U13 | B0505S-1WR3 | 5V → 5V isolated DC-DC (CAN) | Driver |
| U14 | MT3608 | Battery 3.7V → 5V boost | Driver |
| U15 | DW01A | Li-ion protection (OVP/UVP/OCP) | Driver |
| U16 | TP4056-42-ESOP8 | Li-ion charger (USB-C) | Driver |

### 2.2 Connectors

| Ref | Type | Pins | Interface |
|-----|------|------|-----------|
| J_CM5_1 | 2x20 Pin Socket (2.54mm) | 40 | CM5 IO Board (SPI, GPIO, 5V) |
| J_24_VIN1 | XT30PW-M | 2 | 24V DC input |
| J_12V_UPS1 | XT30PW-M | 2 | 12V UPS input |
| J_USB1 | USB-C 16P | 16 | USB 2.0 (charging + data) |
| J_BAT1 | B2B-PH-K-S | 2 | Li-ion battery |
| J_CAN1 | B4B-XH-A | 4 | CAN bus (microwave module) |
| J_BLDC1 | DF11-6DP-2DS | 6 | BLDC stirring motor |
| J_IR_LED1 | DF11-8DP-2DS | 8 | IR sensor + LED ring |
| J_CID1 | DF11-14DP-2DS | 14 | Stepper + linear actuator |
| J_PASD1 | DF11-16DP-2DS | 16 | Pneumatic seasoning dispenser |
| J_SLD1 | DF11-16DP-2DS | 16 | Liquid dispenser + load cells |
| J_PRES1 | DF11-4DP-2DS | 4 | Pressure sensor (ADS1015) |
| J_FAN1 | B4B-XH-A | 4 | Exhaust fans |
| J_SAFE1 | B4B-XH-A | 4 | Safety relay + reed switch + e-stop |
| J_BUZZ1 | B2B-PH-K-S | 2 | Piezo buzzer |
| J_SWD1 | Tag-Connect TC2030 | 6 | SWD debug |

### 2.3 Power Conversion

| Converter | Input | Output | Max Current | Topology |
|-----------|-------|--------|-------------|----------|
| U4 (MP1584EN) | 24V | 12V | 3A | Asynchronous buck |
| U8 (MP1584EN) | 24V | 6.5V | 3A | Asynchronous buck |
| U5 (TPS54531) | 12V (UPS) | 5V | 5A | Synchronous buck |
| U14 (MT3608) | 3.7V (battery) | 5V | 2A | Boost |
| U2 (AMS1117) | 5V | 3.3V | 1A | LDO |
| U13 (B0505S) | 5V | 5V isolated | 200mA | Isolated DC-DC |

---

## 3. Issues Found

### 3.1 CRITICAL Issues (Must Fix Before Fabrication)

#### C-01: MP1584EN #2 (U8) Feedback Resistors — Output Voltage Mismatch

**Location:** Driver sheet, "24V to 6.5V conversion" section
**Components:** R24 = 140K (top), R26 = 37.4K (bottom)
**Expected Output:** 6.5V (per section label and design doc)
**Calculated Output:** Vout = 0.8V × (1 + 140K/37.4K) = **3.79V**

The feedback divider produces ~3.8V, not 6.5V. For 6.5V output:
- R_bot should be **19.6K** (closest E96: 19.6K) with R_top = 140K
- Or R_top should be **267K** (closest E96: 267K) with R_bot = 37.4K

**Impact:** Downstream circuits expecting 6.5V will receive only 3.8V. If this rail feeds the INA219 bus voltage sensing or other 6.5V-dependent circuits, they will malfunction.

**Action:** Verify intended output voltage and correct feedback resistors.

---

#### C-02: MT3608 (U14) Feedback Resistors — Battery Boost Output Voltage

**Location:** Driver sheet, "Battery to +5V converter" section
**Components:** R64 = 732K (top), feedback-to-GND resistor (verify value)
**Expected Output:** 5V (per design doc)

If R64 = 732K and the bottom resistor is 71.5K (R36):
- Vout = 0.6V × (1 + 732K/71.5K) = **6.74V**

This output feeds through a Schottky OR-diode (~0.5V drop) into +5V_MERGED, producing ~6.2V on the 5V bus. This **exceeds the absolute maximum rating** of many 5V-rated components.

**Impact:** Potential damage to USB peripherals, TB6612FNG (VCC max 5.5V), and any 5V-rated ICs on the merged bus.

**Action:** Verify which resistors form the MT3608 feedback divider. For 5V output: R_bot should be ~100K with R_top = 732K (gives Vout = 0.6 × (1 + 7.32) = 4.99V). Alternatively, reduce R64 to ~524K with R_bot = 71.5K.

> **Note:** If R36 (71.5K) is actually the TPS54531 feedback resistor rather than the MT3608, this issue may not apply. Cross-check the schematic to confirm feedback resistor assignments for U5, U8, and U14.

---

#### C-03: PCF8574AT I2C Address vs Documentation Mismatch

**Location:** Driver sheet, U9 and U10
**Part:** PCF8574**A**T (BOM line 80, MPN: PCF8574AT)

The PCF8574 (non-A) uses base address **0x20-0x27**, while the PCF8574**A** uses base address **0x38-0x3F**. The "T" suffix indicates TSSOP package.

| Variant | Address Range | A0=0,A1=0,A2=0 | A0=1,A1=0,A2=0 |
|---------|--------------|-----------------|-----------------|
| PCF8574 | 0x20–0x27 | 0x20 | 0x21 |
| PCF8574A | 0x38–0x3F | 0x38 | 0x39 |

**Design doc states:** addresses 0x20 and 0x21
**Actual addresses (if PCF8574A):** 0x38 and 0x39

**Impact:** Firmware using addresses 0x20/0x21 will get NACKs. All P-ASD solenoids, SLD solenoids, status LED, and LED ring power will be non-functional.

**Action:** Either:
1. Change BOM part to PCF8574T (non-A variant) to match firmware addresses 0x20/0x21, **or**
2. Update firmware to use addresses 0x38/0x39 matching the PCF8574AT.

---

#### C-04: C34 Voltage Rating — Insufficient Margin on 24V Rail

**Location:** Driver sheet, 24V bulk capacitor
**Component:** C34 = 470uF @ **25V** electrolytic (10x12.5mm package)

On a nominal 24V rail, this provides only **4% voltage margin**. The Mean Well LRS-75-24 PSU output can reach 28.8V at high line (per spec). Additionally, load dump transients and hot-plug events on the 24V connector can produce spikes exceeding 25V.

**Impact:** Exceeding rated voltage degrades electrolytic capacitor lifespan (gas generation, ESR rise) and risks catastrophic failure (venting/short).

**Action:** Replace with **470uF @ 35V** (or 50V). Verify footprint compatibility — a 35V cap in the same 10x12.5mm case may be available from Nichicon/Panasonic.

---

#### C-05: TPS54531 (U5) Feedback Verification Needed

**Location:** Driver sheet, "UPS_INPUT" section
**Expected Output:** 5V (per design doc — 12V UPS → 5V conversion)

The TPS54531DDA VSENSE reference is 0.8V. The feedback resistor values need verification against the schematic to confirm 5V output. If R36 = 71.5K (top) and R25 = 10K (bottom):
- Vout = 0.8V × (1 + 71.5K/10K) = **6.52V** (not 5V!)

If this is intentional (producing 6.5V to match the "+6.5V_OUT" rail), then the section label "UPS_INPUT" and design doc description ("12V → 5V") are wrong.

**Impact:** If TPS54531 outputs 6.5V instead of 5V, the 5V_MAIN Schottky-OR bus may see higher-than-expected voltage.

**Action:** Verify the actual feedback resistor connections on U5 in the schematic. Confirm intended output voltage and update documentation accordingly.

---

### 3.2 HIGH Issues (Fix Before First Power-On)

#### H-01: Root Page Net Label Typo — `PB6_FAN1_PWM`

**Location:** Root schematic (page 1), label connecting Controller `PA6_FAN1_PWM` to Connector
**Issue:** The root page label reads **`PB6_FAN1_PWM`** but the signal originates from **PA6** (pin 20), not PB6 (pin 59). PB6 is `CID1_HOME`.

In KiCad, the net will be named `PB6_FAN1_PWM` in the netlist, which is misleading during PCB layout and debugging. The electrical connection via the hierarchical pin wire is correct, but the net name in the PCB editor will show the wrong pin reference.

**Action:** Rename root label from `PB6_FAN1_PWM` to `PA6_FAN1_PWM`.

---

#### H-02: AMS1117-3.3 (U2) Load Capacity — Single LDO for All 3.3V

**Location:** Controller sheet, U2
**Capacity:** 1A max (with adequate heatsinking on SOT-223 pad)

**3.3V Load Estimate:**

| Consumer | Current |
|----------|---------|
| STM32G474 core + peripherals | 80–150 mA |
| I2C pull-ups (2× 4.7K to 3.3V) | ~1.4 mA |
| External sensors (MLX90614, ADS1015) via connectors | ~20 mA |
| PCF8574AT × 2 | ~2 mA |
| INA219 | ~1 mA |
| Misc (LEDs, resistor dividers) | ~10 mA |
| **Total** | **~115–185 mA** |

The AMS1117 is adequate for this load. However, the dropout voltage is 1.1V typ (1.3V max), requiring minimum 4.6V input. If the 5V_MERGED bus droops during battery-only operation (after Schottky drop), the 3.3V output may droop.

**Action:** Verify 5V_MERGED minimum voltage under battery backup. If < 4.6V, consider replacing AMS1117 with a low-dropout regulator (e.g., AP2112K-3.3, 150mV dropout).

---

#### H-03: INA219 (U7) Shunt Resistor — Poor Resolution at Expected Current

**Location:** Driver sheet, INA219 section
**Component:** R37 = 10mΩ (2512 package)

With the 24V bus expected to draw 2–5A and INA219 default PGA at ±320mV:
- At 3A: shunt voltage = 30mV (only 9.4% of full scale → poor ADC resolution)
- Power dissipation at 5A: 250mW (acceptable for 2512)

For better resolution at the expected current range (1–5A):
- **50mΩ** would give 250mV at 5A (78% of ±320mV range, 625mW → needs 2512)
- **33mΩ** would give 165mV at 5A (51% range, 825mW → adequate compromise)

**Action:** Consider increasing R37 to 33mΩ or 50mΩ for better measurement resolution. Verify power dissipation fits 2512 footprint (1W rated).

---

#### H-04: No Surge Protection on CAN Bus Lines at Connector

**Location:** Connector sheet, J_CAN1

The CAN bus connects to an 1,800W microwave induction module which generates significant EMI. While ISO1050DUB provides galvanic isolation (5kV RMS) and D25 (CDSOT23-SM712) provides ESD/TVS protection on the connector, the SM712 is designed for ESD (±30V clamp) not for high-energy induction transients.

**Action:** Consider adding a common-mode choke (e.g., ACT45B-510-2P) in series with CANH/CANL before the connector for additional EMI filtering. Verify CAN bus cable shielding requirements.

---

#### H-05: Missing Bypass Capacitor Near J_SLD1 3.3V Output (Pin 11)

**Location:** Connector sheet, J_SLD1 pin 11 (+3.3V for external HX711 boards)

The 3.3V supply to external HX711 load cell amplifiers exits through J_SLD1 pin 11. There is no visible bypass capacitor near this connector pin. Long cables to external HX711 boards will pick up switching noise from the 12V solenoid/pump drivers also on this connector.

**Action:** Add 100nF ceramic bypass cap close to J_SLD1 pin 11 on the PCB layout.

---

#### H-06: TP4056 TEMP Pin — Temperature Monitoring Bypassed

**Location:** Driver sheet, U16 (TP4056-42-ESOP8)
**Component:** R65 = 100K on TEMP pin

A fixed 100K resistor on the TEMP pin simulates a 25°C NTC thermistor, permanently disabling thermal protection. The TP4056 will continue charging even if the Li-ion cell overheats.

**Impact:** Safety risk for unattended operation. Acceptable for prototype; must fix for production.

**Action:** For prototype: document as known limitation. For production: replace R65 with an NTC thermistor (10K @ 25°C, B=3950) mounted near the battery.

---

#### H-07: J_PASD1 Pin 1 (+12V) Current Margin Tight

**Location:** Connector sheet, J_PASD1 pin 1
**Connector Rating:** Hirose DF11 — 3A per pin maximum

Worst-case P-ASD load: 6× solenoids (0.5A each) + 1× pump (0.8A) = **3.8A** if all energized simultaneously. The firmware limits to 1 solenoid + pump at a time (~1.3A sustained), but inrush transients or firmware bugs could exceed 3A.

**Action:** Either:
1. Add firmware hard-limit: never energize >1 solenoid simultaneously, **or**
2. Parallel pin 1 with pin 2 for 12V delivery (requires PCB footprint change), **or**
3. Add a 3A PTC fuse in series with J_PASD1 pin 1 to protect the connector.

---

#### H-08: PC8 vs PC5 Pin Assignment Discrepancy

**Location:** Controller sheet — `PC8_SLD_WATER_PWM`
**Design Doc says:** PC5 = SLD Pump 2 PWM (line 331 of 01-Unified-PCB-Design.md)
**Schematic shows:** PC8 = SLD_WATER_PWM (PC5 is no-connect)

The schematic has moved the water pump PWM from PC5 to PC8. This is electrically valid (PC8 supports TIM8_CH3 for PWM), but the design document pin map table is outdated.

**Impact:** Firmware developed from the design doc will configure the wrong pin.

**Action:** Update the pin allocation table in `01-Unified-PCB-Design.md` to reflect PC8 assignment.

---

### 3.3 MODERATE Issues

#### M-01: HSE Crystal Load Capacitor Verification

**Location:** Controller sheet, Y1 (8MHz, HC49-SD)
**Components:** C11 = 18pF, C12 = 18pF

Effective load capacitance: CL = (18×18)/(18+18) + Cstray = 9 + ~3pF = **12pF**

This is correct if the crystal specifies CL = 12pF (common for HC49-SD). However, if the crystal has CL = 18pF or 20pF, the caps should be ~33pF.

**Action:** Verify against the actual Y1 crystal datasheet CL specification. Adjust C11/C12 accordingly using formula: C = 2 × (CL - Cstray).

---

#### M-02: 120R CAN Termination — Potential Double Termination

**Location:** Driver sheet, R40 = 120R between CANH and CANL

If the microwave induction module also has a 120R termination resistor, the effective bus termination becomes 60R. Standard CAN requires 120R at each end of the bus (2 nodes = 60R total impedance, which is correct for a 2-node bus). However, if there are intermediate stubs, this could cause reflections.

**Action:** Confirm with the microwave module manufacturer whether their CAN port has internal termination. If yes, the 120R on the Unified PCB is correct (2-node bus). If no, the Unified PCB termination alone may be insufficient. Consider adding a jumper or 0R placeholder for optional termination.

---

#### M-03: BOOT0 Pull-Down Verification

**Location:** Controller sheet, SW1 (BOOT_Switch)

The schematic shows SW1 connected between BOOT0 and +3.3V. The design doc states a 10K pull-down resistor holds BOOT0 LOW during normal operation. This pull-down is not clearly visible in the schematic PDF.

**Action:** Verify R4 (10K) or similar pull-down resistor exists between BOOT0 and GND. Without it, BOOT0 may float HIGH on power-up, entering bootloader mode instead of running user firmware.

---

#### M-04: No Series Resistor on IRQ Line (PB3 → J_CM5 Pin 7)

**Location:** Connector sheet, PB3 → GPIO4 on CM5 connector

The STM32 PB3 (IRQ to CM5) is connected directly to J_CM5 pin 7 without a series resistor. All other signal lines to external connectors have 100R series protection.

**Action:** Add 100R series resistor between PB3 and J_CM5 pin 7 for consistency and ESD protection.

---

#### M-05: TB6612FNG STBY Pin Configuration

**Location:** Driver sheet, U12 TB6612FNG

The STBY pin (pin 19) must be pulled HIGH for normal operation. If connected to GND or left floating, both motor channels are disabled.

**Action:** Verify STBY pin 19 is connected to +3.3V or +5V via a pull-up resistor. If driven by a GPIO, ensure the firmware initializes it HIGH before attempting motor control.

---

#### M-06: DRV8876 PMODE Pin Configuration

**Location:** Driver sheet, U11 DRV8876PWPR

Pin 16 (PMODE) selects the H-bridge control mode:
- PMODE = LOW or float → PH/EN mode (phase/enable)
- PMODE = HIGH → PWM mode (IN1/IN2)

The design doc implies PH/EN mode (PB5 = EN, PC2 = PH/DIR). Verify PMODE is tied LOW or floating.

**Action:** Confirm PMODE pin 16 connection matches the intended control mode.

---

#### M-07: R37 (10mΩ) Power Dissipation Footprint

**Location:** Driver sheet, INA219 shunt resistor
**Package:** R_2512_6332Metric (rated ~1W)

At maximum expected current (5A): P = 5² × 0.01 = 250mW (well within 1W rating). However, if a fault condition drives 10A through the shunt: P = 1W, which is the package limit.

**Action:** Acceptable for prototype. For production, verify upstream fuse (F_IN1, 5A PTC) limits current before the shunt reaches thermal limit.

---

#### M-08: J_CID1 Pin 14 (+24V_Step) Current Capacity

**Location:** Connector sheet, J_CID1 pin 14
**Load:** External NEMA 23 stepper driver (DM542/TB6600) — draws 2-3A

Hirose DF11 is rated 3A per pin. A running stepper motor at full current is right at the limit.

**Action:** Monitor actual current draw on prototype. If sustained >2.5A, consider routing stepper power through a separate XT30 connector.

---

#### M-09: Missing RC Snubbers on Inductive Loads

**Location:** Driver sheet, MOSFET switches Q_SW1–Q_SW13

All inductive loads (solenoids, pumps, fans) have Schottky flyback diodes (SS14/SS34). However, no RC snubber circuits are present. Fast-switching solenoids can generate high-frequency ringing even with flyback diodes.

**Action:** Acceptable for prototype. If EMI testing shows radiated emissions issues, add 100R + 10nF snubber across each solenoid coil at the connector interface.

---

#### M-10: 5V Power Delivery to CM5 — No Polyfuse

**Location:** Connector sheet, J_CM5 pins 2/4 (+5V_Merged)

The CM5 receives 5V directly from TPS54531 output (via Schottky-OR). If the CM5 develops a short or draws excessive current, there's no dedicated current limiting between the 5V bus and J_CM5.

**Action:** Consider adding a 3A PTC fuse between +5V_Merged and J_CM5 pins 2/4 to protect the power bus from a CM5 failure.

---

#### M-11: LED Ring Data Signal — No Level Shifter

**Location:** Connector sheet, J_IR_LED1 pin 6 (GPIO18 from CM5)

WS2812B LEDs require a data signal with VIH > 0.7 × VDD. At 5V supply: VIH > 3.5V. The CM5 GPIO18 outputs 3.3V logic, which is **below** the WS2812B threshold (3.5V).

**Action:** Add a level shifter (e.g., SN74LVC1T45 or simple MOSFET level shifter) between GPIO18 and the LED data line. Alternatively, use a 74HCT125 buffer powered from 5V (HCT family has 3.3V-compatible input threshold).

---

#### M-12: Multiple I2C Devices on Single Bus — Bus Capacitance Verified

**Severity:** Low/Informational (downgraded from Moderate — verified within spec)

**Location:** Driver sheet + Connector sheet, I2C1 bus (PA15/PB7)

Five devices share I2C1:

| Device | Address | Location |
|--------|---------|----------|
| PCF8574AT #1 (U9) | 0x38* or 0x20 | On-board |
| PCF8574AT #2 (U10) | 0x39* or 0x21 | On-board |
| INA219 (U7) | 0x40 (A0=A1=GND) | On-board |
| MLX90614 | 0x5A (fixed) | External (J_IR_LED) |
| ADS1015 | 0x48 (ADDR=GND) | External (J_PRES) |

*See C-03 for PCF8574AT address ambiguity.

No address conflicts exist if addresses are confirmed.

**Bus Capacitance Verification:**

| Source | Capacitance | Notes |
|--------|-------------|-------|
| PCF8574AT × 2 (U9, U10) | 20 pF | 10 pF each (on-board) |
| INA219 (U7) | 5 pF | On-board |
| MLX90614 | 10 pF | External via J_IR_LED |
| ADS1015 | 5 pF | External via J_PRES |
| STM32G474 I2C1 pins | 8 pF | ~4 pF per pin |
| PCB traces (~80 mm on-board) | 10 pF | ~1.2 pF/cm for 4-layer FR4 |
| J_IR_LED cable (~200 mm) | 40 pF | ~20 pF/10cm shielded cable |
| J_PRES cable (~200 mm) | 40 pF | ~20 pF/10cm shielded cable |
| Connector pin capacitance (3×) | 1 pF | Hirose DF11, negligible |
| **Total** | **139 pF** | **35% of 400 pF limit** |

**Rise time with 4.7K pull-ups:** t_r = 0.8473 × R_p × C_bus = 0.8473 × 4700 × 139 pF ≈ **551 ns** (55% of 1000 ns standard-mode limit)

**Key findings:**
- Total bus capacitance (139 pF) is well within the 400 pF standard-mode limit with **65% margin**
- Rise time (551 ns) is within the 1000 ns standard-mode limit with **45% margin**
- External cables dominate capacitance (58% of total) — keep cable runs under 300 mm
- Fast mode (400 kHz, 300 ns rise time limit) would NOT be supported with current 4.7K pull-ups (would need ≤2.2K)

**Status:** Verified — no action needed for standard mode (100 kHz)

---

## 4. Design Strengths

The following aspects of the schematic demonstrate good design practice:

| Area | Implementation | Assessment |
|------|----------------|------------|
| **24V Input Protection** | 5A PTC (F_IN1) → SS54 reverse polarity → SMBJ24A TVS | Excellent |
| **12V Input Protection** | 5A PTC (F_IN2) → SS54 reverse polarity → SMBJ24A TVS | Excellent |
| **USB ESD** | USBLC6-2SC6 bidirectional TVS array | Industry standard |
| **USB-C UFP ID** | R13/R14 = 5.1K to GND on CC1/CC2 | Per USB-C spec |
| **USB Series R** | R15/R16 = 22R on D-/D+ | Correct for USB 2.0 FS |
| **CAN Isolation** | ISO1050DUB (5kV RMS) + B0505S-1WR3 isolated DC-DC | Excellent |
| **CAN TVS** | D25 CDSOT23-SM712 on J_CAN1 | Good |
| **CAN Termination** | R40 = 120R between CANH/CANL | Present |
| **VDDA Filtering** | FB1 (600R@100MHz) + C9 (100nF) + C10 (1uF) | Proper analog supply filtering |
| **STM32 Decoupling** | 6× 100nF (C4-C9) + C44 (10uF) bulk | Good coverage |
| **I2C Pull-ups** | R2/R3 = 4.7K to 3.3V | Standard for 100kHz |
| **Switch Debounce** | R87-R89 (100R) + C41-C43 (100nF) on limit/safety inputs | Proper RC debounce |
| **Signal Protection** | 100R series on all external signal lines | Consistent |
| **Motor Flyback** | SS14/SS34 Schottky diodes on all inductive loads | Complete coverage |
| **Battery Protection** | TP4056 charger + DW01A OVP/UVP + FS8205A dual MOSFET | Full protection chain |
| **Connector Choice** | Hirose DF11 (locking, polarized) for subsystems | Reliable, vibration-resistant |
| **Power Input Connectors** | XT30PW-M (rated 30A) for 24V/12V | Massive margin |

---

## 5. STM32G474RE Pin Allocation Audit

### 5.1 Used Pins (from schematic)

| Pin | STM32 Function | Signal | Subsystem |
|-----|---------------|--------|-----------|
| PA0 | TIM2_CH1 | PUMP_PWM | P-ASD |
| PA1 | COMP1_INP | 24V_Fail_Detection | Power |
| PA3 | ADC1_IN4 | ADC_BUS_Detection | USB VBUS |
| PA4 | GPIO | MOT_EN | BLDC |
| PA5 | GPIO | MOT_DIR | BLDC |
| PA6 | TIM3_CH1 | FAN1_PWM | Exhaust |
| PA8 | TIM1_CH1 | MOT_PWM | BLDC |
| PA10 | TIM1_CH3 | CID1_Actuator_EN (Stepper PUL) | CID |
| PA11 | USB_DM | USB D- | USB |
| PA12 | USB_DP | USB D+ | USB |
| PA13 | SWDIO | Debug | SWD |
| PA14 | SWCLK | Debug | SWD |
| PA15 | I2C1_SCL | Sensors/Expanders | I2C |
| PB0 | GPIO | SAFETY_RELAY | Safety |
| PB1 | GPIO | REED_SWITCH | Safety |
| PB2 | GPIO/EXTI | E-STOP | Safety |
| PB3 | GPIO | IRQ_CM5_IO4 | Comms |
| PB4 | GPIO | CID1_Actuator_DIR | CID |
| PB5 | GPIO | CID2_Actuator_EN | CID |
| PB6 | GPIO/EXTI | CID1_HOME | CID |
| PB7 | I2C1_SDA | Sensors/Expanders | I2C |
| PB8 | FDCAN1_RX | MW_FDCAN1_RX | CAN |
| PB9 | FDCAN1_TX | MW_FDCAN1_TX | CAN |
| PB10 | TIM2_CH3 | FAN2_PWM | Exhaust |
| PB12 | SPI2_NSS | CM5 CE0 | SPI |
| PB13 | SPI2_SCK | CM5 SCLK | SPI |
| PB14 | SPI2_MISO | CM5 MISO | SPI |
| PB15 | SPI2_MOSI | CM5 MOSI | SPI |
| PC2 | GPIO | CID2_Actuator_DIR | CID |
| PC3 | GPIO | SLD_OIL_PWM | SLD |
| PC4 | GPIO | SLD_OIL_DIR | SLD |
| PC6 | GPIO | SLD_WATER_DIR | SLD |
| PC7 | TIM8_CH2 | BUZZER_PWM | Audio |
| PC8 | GPIO | SLD_WATER_PWM | SLD |
| PC9 | GPIO | HX711_SCK (water) | SLD |
| PC10 | GPIO | HX711_DOUT (water) | SLD |
| PC11 | GPIO | HX711_SCK (oil) | SLD |
| PC12 | GPIO | HX711_DOUT (oil) | SLD |
| PC14 | OSC32_IN | LSE 32.768kHz | Clock |
| PC15 | OSC32_OUT | LSE 32.768kHz | Clock |
| PD2 | GPIO/EXTI | CID2_HOME | CID |
| PF0 | OSC_IN | HSE 8MHz | Clock |
| PF1 | OSC_OUT | HSE 8MHz | Clock |

### 5.2 Unused Pins (No-Connect)

| Pin | Notes |
|-----|-------|
| PA2 | Available — freed from prior design (TIM2_CH3, USART2_TX, ADC1_IN3) |
| PA7 | Available — freed from prior design (TIM3_CH2, SPI1_MOSI) |
| PA9 | Available — freed from prior design (TIM1_CH2, USART1_TX) |
| PB11 | Available — freed from CID-1 extend limit |
| PC0, PC1 | Available — freed from pot load cells |
| PC5 | Available — SLD_WATER_PWM moved to PC8 |
| PG10 | Available |

**7 GPIO pins available** for future expansion.

---

## 6. Power Budget Analysis

### 6.1 24V Rail

| Load | Current (typ) | Current (peak) |
|------|---------------|----------------|
| BLDC Motor (via J_BLDC) | 1.5A | 3.0A |
| CID-1 Stepper (via J_CID) | 1.5A | 3.0A |
| MP1584EN #1 input (24V→12V) | 1.5A | 2.0A |
| MP1584EN #2 input (24V→6.5V) | 0.1A | 0.3A |
| **Total** | **4.6A** | **8.3A** |

**PSU:** Mean Well LRS-75-24 = 3.2A continuous.

**Issue:** Peak 24V demand (8.3A) exceeds PSU capacity (3.2A) by **2.6×**. Even typical demand (4.6A) exceeds it by 1.4×.

**Mitigation:** BLDC motor and stepper never run simultaneously at full torque. Actual sustained draw is likely ~3A with proper firmware sequencing. However, this should be validated.

**Action:** Consider upgrading to LRS-150-24 (6.5A) for adequate headroom.

### 6.2 12V Rail (from MP1584EN #1, 3A max)

| Load | Current (typ) |
|------|---------------|
| P-ASD solenoids (1 active) | 0.5A |
| P-ASD pump | 0.8A |
| SLD solenoids (1 active) | 0.5A |
| Exhaust fans × 2 | 1.0A |
| **Total** | **2.8A** |

Within 3A limit. Tight during simultaneous fan + dispense operations.

### 6.3 5V Rail (from TPS54531, 5A max)

| Load | Current (typ) |
|------|---------------|
| CM5 + peripherals | 2.5A |
| LED ring | 0.5A |
| AMS1117 input (3.3V load) | 0.2A |
| **Total** | **3.2A** |

Adequate margin.

---

## 7. Action Item Summary

### Must Fix (Before Fabrication)

| ID | Issue | Action |
|----|-------|--------|
| C-01 | MP1584EN #2 feedback gives 3.8V not 6.5V | Correct R24/R26 values |
| C-02 | MT3608 feedback may give 6.7V not 5V | Verify and correct feedback divider |
| C-03 | PCF8574AT addresses 0x38/0x39, not 0x20/0x21 | Change part or update firmware |
| C-04 | C34 470uF@25V on 24V rail (4% margin) | Replace with 35V or 50V rated cap |
| C-05 | TPS54531 output voltage unverified | Confirm feedback resistors and output |

### Must Fix (Before First Power-On)

| ID | Issue | Action |
|----|-------|--------|
| H-01 | Root label `PB6_FAN1_PWM` should be `PA6_FAN1_PWM` | Rename label |
| H-02 | AMS1117 dropout margin on battery backup | Verify min input voltage |
| H-03 | INA219 shunt 10mΩ — poor resolution | Consider 33-50mΩ |
| H-04 | No common-mode choke on CAN lines | Add choke if EMI issues |
| H-05 | No bypass cap near J_SLD1 3.3V output | Add 100nF at connector |
| H-06 | TP4056 TEMP pin bypassed (100K fixed) | Document; fix for production |
| H-07 | J_PASD1 12V current margin tight (3A limit) | Add firmware limit or parallel pins |
| H-08 | Design doc PC5 vs schematic PC8 mismatch | Update design doc |

### Should Fix (Before Production)

| ID | Issue | Action |
|----|-------|--------|
| M-01 | Verify HSE crystal load caps vs datasheet | Check Y1 CL spec |
| M-02 | CAN 120R — verify no double termination | Confirm with module vendor |
| M-03 | BOOT0 pull-down resistor verification | Verify in schematic |
| M-04 | PB3 IRQ to CM5 lacks series resistor | Add 100R |
| M-05 | TB6612 STBY pin configuration | Verify pull-up to VCC |
| M-06 | DRV8876 PMODE pin configuration | Verify matches PH/EN mode |
| M-07 | R37 power dissipation at fault current | Protected by upstream fuse |
| M-08 | J_CID1 24V stepper pin at 3A limit | Monitor on prototype |
| M-09 | No RC snubbers on solenoids | Add if EMI issues arise |
| M-10 | No polyfuse on 5V to CM5 | Add 3A PTC |
| M-11 | LED data 3.3V vs WS2812B 5V threshold | Add level shifter |
| M-12 | I2C bus capacitance with 5 devices | Verified OK — 139 pF, 65% margin |

---

## 8. Recommended Changes for Rev 1.2

Based on this review, the following changes are recommended for the next revision:

1. **Correct all voltage regulator feedback dividers** (C-01, C-02, C-05) — verify every output voltage with actual resistor values
2. **Standardize PCF8574 variant** (C-03) — use PCF8574T (non-A) if firmware uses 0x20/0x21
3. **Upgrade C34 to 35V rating** (C-04)
4. **Fix root page net labels** (H-01)
5. **Add WS2812B level shifter** (M-11)
6. **Add NTC thermistor on TP4056** (H-06) for battery safety
7. **Add 100R on PB3 → CM5 IRQ** (M-04)
8. **Consider PSU upgrade** to LRS-150-24 for 24V headroom
9. **Update design documentation** to match actual schematic pin assignments

---

*Report generated 2026-03-21. Review based on schematic PDF rev1.1 (20-03-2026), BOM export (20-03-2026), and KiCad source files. §0 added 2026-11-04 against `Unified_v1.3/` and design doc v3.1.*

## 9. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 2.3 | 2026-11-05 | Manas Pradhan | 2026-11-05 addendum extended: the heating-unit tap was withdrawn the same day — CSC v1.6 runs from its own Mean Well LRS-35-12 on J_12V_IN and K_SAFE is back on the unit's AC inlet; findings unchanged |
| 2.2 | 2026-11-05 | Manas Pradhan | Added the 2026-11-05 addendum: CSC v1.5 board power from the heating unit's 12 V (J_HU_PWR), MP1584EN #1 dropped, J_24V_IN → J_DU_IN (DU feed only), K_SAFE on the element loom; findings unchanged |
| 2.1 | 2026-11-04 | Manas Pradhan | Added the 2026-11-04 addendum: dispositions that named the SN65HVD230 stir port, Q_STIR / J_STIR_PWR and the J_SU harness are superseded by CSC v1.3 (ISO1050DUB #3 + B0505S #2, K_SU, 24V_SU / GND_SU); findings themselves unchanged |
| 2.0 | 2026-11-04 | Manas Pradhan | Added §0: every finding dispositioned against the Unified_v1.3 KiCad state (C-01, C-04, C-05, H-01, H-04 fixed; C-03 value corrected) and the v3.x design (C-02, H-06, H-07, M-04, M-06, M-08 superseded; H-02, M-02, M-03, M-05, M-11 resolved by design); open list and v3.x schematic-pass checklist; §5/§6 marked as superseded by the G0B1 pin map and the v3.1 power budget; original review kept verbatim |
| 1.0 | 2026-03-21 | Claude (automated) | Initial review of schematic rev 1.1: 5 critical, 8 high, 12 moderate findings |

---

> [!note] Addendum 2026-11-04 — superseded by CSC v1.3
> The dispositions above that refer to the v3.x stir-motor port are out of date since [[02-Cooking-Surface-Controller|Cooking Surface Controller]] v1.3 (both CAN ports isolated, Stirring Unit on its own PSU): the **SN65HVD230** regular port (H-04, M-02, §0 item 2) is now **ISO1050DUB #3 + B0505S #2** (isolated CAN port #2, 5V_ISO2 / GND_SU, second isolation slot); **Q_STIR** / F_STIR and the switched 24 V (M-08, §0 item 2) are replaced by the pilot relay **K_SU** cutting the AC input of a second Mean Well LRS-75-24 (#2) whose floating output enters the CSC on J_SU_IN and leaves on J_STIR_PWR (24V_SU / GND_SU); the **J_SU** harness pinout is now 24V_SU, GND_SU, CAN_H, CAN_L, SHIELD (GND_SU at the CSC end), with nothing referencing the CS ground; the §6 budget note's 2.1 / 3.0 A on one LRS-75-24 becomes ~1.3 / ~2.3 A on LRS-75-24 #1 with the motor on #2. The original findings are kept verbatim.

> [!note] Addendum 2026-11-05 — superseded by CSC v1.5 (board power from the heating unit), revised by CSC v1.6 the same day
> Since [[02-Cooking-Surface-Controller|Cooking Surface Controller]] v1.5 the CSC, both G5V-2 relay coils and the exhaust fans are powered from **12V_HU**, a 12 V SELV tap off the heating unit's own control-board supply brought in on the new **J_HU_PWR** (F1 2 A, SS34, SMBJ12A, 470 µF); neither LRS-75-24 powers the board. The 24 → 12 V **MP1584EN #1** (§6.2 "12V Rail (from MP1584EN #1)", C-01's sibling buck) is dropped on the CS side — the one remaining MP1584EN makes 5 V from 12 V — and the 24 V input chain **J_24V_IN** becomes **J_DU_IN** (F_DU 3 A, SS54, SMBJ24A → 24V_DU), carrying LRS-75-24 #1 to Q_DOCK and the dock feed only, so the §6.1 24 V budget on LRS #1 falls further to ~0.6 / 1.6 A. The heater safety relay **K_SAFE** is wired on the heating **element** loom (downstream of the OEM board's own element relay), not on the unit's AC inlet, since the OEM board now feeds the CSC. The tap was withdrawn the same day: since CSC v1.6 the CSC, both relay coils and the fans run from a dedicated Mean Well LRS-35-12 on **J_12V_IN** (12V_IN; F1 2 A, SS34, SMBJ12A, 100 µF), nothing is tapped from the heating unit, and K_SAFE is back on the unit's AC inlet loom (J_DU_IN and the MP1584EN #1 removal stand). The original findings are kept verbatim.
