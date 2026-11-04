---
created: 2026-11-04
modified: 2026-11-04
version: 1.1
status: Draft
---

# Cooking Surface Controller (CSC) PCB Design

## 1. Overview

The **Cooking Surface Controller (CSC)** is the second custom board of Epicura and the only one in the **Cooking Surface (CS)**, the mains-powered base unit. It sits in the CS rear beside the Mean Well LRS-75-24 and the two exhaust fans, and carries the **STM32G0B1RET6** that was on the Unified PCB up to v3.x together with everything that belongs physically to the base: the electric heating unit's safety relay, the E-stop button, the pot reed switch, the exhaust fans, the 24 V feed and CAN port of the **Stirring Unit (SU)**, and the power + CAN side of the dock that the **Dispensing Unit (DU)** seats on.

One ~100 x 70 mm, 4-layer board carries:

- **STM32G0B1RET6** (Cortex-M0+ @ 64 MHz, 2x FDCAN), FreeRTOS, MISRA C, X-CUBE-CLASSB: safety monitor, heater off-switch, stir motor control, fans, dock interlock
- **Isolated CAN port** (FDCAN1 → ISO1050DUB #1, B0505S-1WR3): the 500 kbps bus shared by the STM32, the heating unit (J_CAN_HEAT) and the RV1106 in the DU (dock pins 3–6)
- **Regular CAN port** (FDCAN2 → SN65HVD230, J_CAN_STIR) and the switched 24 V (Q_STIR, J_STIR_PWR) for the CAN servo stirring motor; both bundled into the SU harness ending at J_SU on the right pillar
- **Dock interface (J_DOCK)**: switched, fused 24 V charge feed (Q_DOCK, F_DOCK) to the DU, the isolated bus and 5V_ISO over the pogo block, DOCK_SENSE_B interlock
- **Safety I/O**: Omron G5V-2 relay cutting AC to the heating unit, E-stop NC button, pot reed switch (J_SAFE), two 12 V exhaust fans (J_FAN)
- Power conversion (24 V in; 12 V, 5 V, 3.3 V), USB-C DFU + SWD + debug UART for the bench

The CSC has no I2C devices, no dispensers, no load cells, no battery and no display: those are on the Unified PCB v4.0 in the DU ([[01-Unified-PCB-Design|Unified PCB Design]]). It has no temperature sensor either: the heating unit runs its own loop on its centre (pot-bottom) sensor and reports over CAN (see [[../05-Subsystems/01-Electric-Heating|Electric Heating]]).

### 1.1 Board Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Cooking Surface Controller v1.0 (~100 x 70 mm, 4-layer)                 │
│                                                                          │
│   ┌───────────────────┐    ┌──────────────────────────────────────────┐  │
│   │ STM32G0B1RET6     │    │ Isolated CAN domain (5V_ISO / GND_ISO)   │  │
│   │ FreeRTOS, Class B │    │ B0505S ──► ISO1050 #1 ──┬──► J_CAN_HEAT  │  │
│   │                   │──FDCAN1──►                   │   (heater)     │  │
│   │                   │    │        R_TERM1 120 Ω    └──► J_DOCK 3-6  │  │
│   │                   │    └──────────────────────────────(DU RV1106)─┘  │
│   │                   │──FDCAN2──► SN65HVD230 ──► J_CAN_STIR ─┐          │
│   │                   │──PB4────► Q_STIR (24 V) ──► J_STIR_PWR ┴─► J_SU  │
│   │                   │──PB5────► Q_DOCK (24 V, F_DOCK 2 A) ──► J_DOCK 1 │
│   │                   │◄─PB3───── DOCK_SENSE_B ◄──────────────── J_DOCK 8│
│   │                   │──PB0────► Q_RLY ──► G5V-2 relay ──► J_SAFE 1     │
│   │                   │◄─PB1/PB2─ pot reed, E-stop ◄──────── J_SAFE 2/3  │
│   │                   │──PA6/PB10► fan MOSFETs ──► J_FAN (12 V)          │
│   └───────────────────┘                                                  │
│   J_USB (DFU), J_SWD, J_DBG, SW_BOOT, SW_RST, status LED                  │
└──────────────────────────────────────────────────────────────────────────┘
          ▲ J_24V_IN (Mean Well LRS-75-24)
```

### 1.2 Scope

| Item | Status | Notes |
|------|--------|-------|
| **Cooking Surface Controller** (this document) | Custom design required | STM32G0B1RE + both CAN ports + dock power/CAN + safety I/O + fans; ~100 x 70 mm, 4-layer |
| **Unified PCB v4.0** | Custom design — see [[01-Unified-PCB-Design\|Unified PCB Design]] | DU board: RV1106, battery, dispensers, RV1106 CAN node; meets this board through the dock |
| **Electric heating unit** | Commercial rice-cooker-class heating unit (resistive plate, own PID, CAN port; 1,300 W assumed) (see [[../05-Subsystems/01-Electric-Heating\|Electric Heating]]) | Isolated CAN port J_CAN_HEAT; AC cut by the safety relay (J_SAFE) |
| **Stirring motor** | 24 V integrated BLDC servo with CAN interface (see [[../05-Subsystems/02-Robotic-Arm\|Robotic Arm]]) | Regular CAN port J_CAN_STIR + 24 V on J_STIR_PWR, harness inside the right pillar to J_SU |
| **Dock pogo block** | Off-the-shelf 8-contact magnetic pogo pair (2.54 mm pitch), spring-pin half in the CS left pillar | Harness to J_DOCK (8-pin JST-XH) |
| **Mean Well LRS-75-24** | Off-the-shelf PSU in the CS rear | 24 V, 3.2 A to J_24V_IN |

### 1.3 What Moved Here from the Unified PCB v3.x

| Function | Unified PCB v3.x | CSC v1.0 |
|----------|------------------|----------|
| MCU | STM32G0B1RE, 8 MHz HSE, J_SWD, J_USB, SW_BOOT/SW_RST | Same parts, same clock; PCF8574 dropped (all outputs are direct GPIO) |
| Isolated CAN | ISO1050DUB #1 (STM32) + #2 (RV1106) + B0505S on one board, J_CAN_HEAT | ISO1050DUB #1 + B0505S here; the bus continues over J_DOCK pins 3–6 to ISO1050 #2 on the DU board |
| Regular CAN + motor power | SN65HVD230 on FDCAN2, Q_STIR from PCF8574 #2 P5, J_CAN_STIR, J_STIR_PWR | Same, Q_STIR now from STM32 PB4; both connectors feed the SU harness (J_SU) |
| Safety | Q_RLY on PB0, pot reed PB1, E-stop PB2, J_SAFE | Unchanged |
| Fans | PA6 (TIM3_CH1), PB10, J_FAN, 12 V | Unchanged (PB10 as software PWM on the G0) |
| 24 V input | J_24V_IN, F1 5 A, SS54, SMBJ24A, MP1584EN #1 → 12 V | Same chain; 12 V now feeds only the fans and the relay coil |
| 5 V / 3.3 V | TPS54531 from VSYS (battery-backed) | MP1584EN #2 24 → 5 V (no battery on this board), AMS1117-3.3, 5V_USB OR for the bench |
| Dock | — | New: Q_DOCK + F_DOCK → DOCK_24V, DOCK_SENSE_B interlock on PB3, J_DOCK 8-pin |
| Dropped from the STM32 side | PCF8574, INA219, PA1 24 V comparator, PA4 battery ADC, PCA9685 / TB6612 / HX711 / solenoids / buzzer / LED ring, USART2 service link to the RV1106 | On the DU board (RV1106-driven) or gone; USART2 is now a debug console on J_DBG |

---

## 2. Board-Level Block Diagram

```
J_24V_IN ─── F1 (5A) ─── SS54 ─── SMBJ24A ─── 24V Internal ──┬──────────────────────────────┬───────────────────────────┐
                                                             │                              │                           │
                                                             ▼                              ▼                           ▼
                                                     ┌──────────────┐               ┌──────────────┐            ┌──────────────┐
                                                     │ MP1584EN #1  │               │ Q_STIR       │            │ Q_DOCK       │
                                                     │ 24V → 12V 3A │               │ AO4407A      │            │ AO4407A      │
                                                     └──────┬───────┘               │ PB4 → 2N7002 │            │ PB5 → 2N7002 │
                                                            │ 12V                   └──────┬───────┘            └──────┬───────┘
                                                      ┌─────┴──────┐                F_STIR (5 A)               F_DOCK (2 A)
                                                      ▼            ▼                       │                           │
                                                   J_FAN      G5V-2 relay coil        24V_STIR ──► J_STIR_PWR     DOCK_24V ──► J_DOCK pin 1
                                                 (2x fans)    (Q_RLY, PB0)                 │  (XT30)                       (to the DU charger)
                                                                                            └──► J_SU (SU harness, right pillar)
24V Internal ──► MP1584EN #2 (24V → 5V 3A) ──► 5V_MAIN ──┬── D_OR1 (SS14) ──┐
                                                          │                  ├── 5V_MERGED ──► AMS1117-3.3 ──► 3.3V (STM32, SN65HVD230, ISO1050 #1 VCC1)
USB-C VBUS ─── F_USB (500 mA) ─── 5V_USB ── D_OR2 (SS14) ─┼──────────────────┘
                                                          └──► B0505S-1WR3 ──► 5V_ISO / GND_ISO ──┬──► ISO1050 #1 VCC2
                                                                                                   ├──► J_CAN_HEAT pin 4 (F_CAN 100 mA)
                                                                                                   └──► J_DOCK pin 6 (shared F_CAN)

                     ┌────────────────────────────────────────────────────────────────┐
                     │  STM32G0B1RET6 (LQFP-64), 8 MHz HSE                            │
                     │  FDCAN1 (PB8/PB9) ──► ISO1050DUB #1 ──► CAN_H/L ─┬─ R_TERM1 ─ J_CAN_HEAT (heating unit)
                     │                                                   └─────────── J_DOCK 3/4 (DU RV1106 node)
                     │  FDCAN2 (PB12/PB13) ──► SN65HVD230 ──► J_CAN_STIR (1 Mbps)     │
                     │  PB0 ──► Q_RLY ──► relay; PB1 ◄── pot reed; PB2 ◄── E-stop     │
                     │  PB3 ◄── DOCK_SENSE_B; PB4 ──► Q_STIR; PB5 ──► Q_DOCK           │
                     │  PA6 (TIM3_CH1), PB10 (SW PWM) ──► fan MOSFETs                 │
                     │  PC7 ──► status LED; PA0 ◄── VBUS detect                       │
                     │  USART2 (PA2/PA3) ──► J_DBG; USB (PA11/PA12) ──► J_USB; SWD ──► J_SWD
                     └────────────────────────────────────────────────────────────────┘
```

---

## 3. STM32G0B1RE Pin Allocation

### 3.1 Pin Map (LQFP-64)

```
STM32G0B1RET6 (LQFP-64) — CSC v1.0 Pin Assignment
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌─── FDCAN1: Isolated bus (heating unit + DU RV1106) ──────┐  │
│  │  PB8  (FDCAN1_RX, AF3) ◄── ISO1050DUB #1 RXD               │
│  │  PB9  (FDCAN1_TX, AF3) ──► ISO1050DUB #1 TXD               │
│  │  500 kbps classic CAN; nodes: STM32, RV1106 (via dock),   │
│  │  heating unit MCU; also carries FW_UPDATE 0x7E0/0x7E8     │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── FDCAN2: Regular bus to the stirring motor ────────────┐  │
│  │  PB12 (FDCAN2_RX, AF3) ◄── SN65HVD230 R                   │
│  │  PB13 (FDCAN2_TX, AF3) ──► SN65HVD230 D                   │
│  │  1 Mbps classic CAN (motor protocol), non-isolated        │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Power switches ───────────────────────────────────────┐  │
│  │  PB4  (GPIO)      ──► Q_STIR enable (2N7002 → AO4407A)    │
│  │  PB5  (GPIO)      ──► Q_DOCK enable (2N7002 → AO4407A)    │
│  │  PB3  (GPIO/EXTI) ◄── DOCK_SENSE_B (10k pull-up, low =    │
│  │                       DU present; gates Q_DOCK)           │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Exhaust Fans (2x 120mm) ──────────────────────────────┐  │
│  │  PA6  (TIM3_CH1, AF1) ──► Fan 1 PWM (25 kHz)              │
│  │  PB10 (GPIO, SW PWM)  ──► Fan 2 PWM                       │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── GPIO: Safety & Control ───────────────────────────────┐  │
│  │  PB0  (GPIO)      ──► Safety Relay (via Q_RLY 2N7002)     │
│  │  PB1  (GPIO)      ◄── Pot Detection (reed switch, backup) │
│  │  PB2  (GPIO/EXTI) ◄── E-Stop Button (NC, active-low)      │
│  │  PC7  (GPIO)      ──► Status LED                          │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── USB: Native USB 2.0 Full-Speed (bench DFU / CDC) ─────┐  │
│  │  PA11 (USB_DM) ◄─► 22R ── USBLC6-2SC6 ── J_USB D-         │
│  │  PA12 (USB_DP) ◄─► 22R ── USBLC6-2SC6 ── J_USB D+         │
│  │  PA0  (ADC_IN0) ◄── VBUS detect (100k/100k divider)       │
│  │  BOOT0 (PA14 on the G0) ── SW_BOOT to 3.3V + 10k pull-down│
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── USART2: Debug console ────────────────────────────────┐  │
│  │  PA2  (USART2_TX, AF1) ──► J_DBG pin 2                    │
│  │  PA3  (USART2_RX, AF1) ◄── J_DBG pin 3                    │
│  │  3.3 V, 115200 8N1; no host link (the RV1106 is in the DU)│
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  Debug: SWD via PA13 (SWDIO) / PA14 (SWCLK), J_SWD             │
│  Clock: 8 MHz HSE (PF0, PF1); LSE pads (PC14, PC15) optional   │
│  Reset: NRST with 100nF + SW_RST                               │
│  I2C1 pads PB6/PB7 unpopulated (no I2C device on this board)  │
│  Free: PA1, PA4, PA5, PA7, PA8, PA9, PA10, PA15, PB6, PB7,     │
│        PB11, PB14, PB15, PC0–PC6, PC8–PC12, PD2 (26 pins)      │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 Pin Summary Table

| Pin | Function | Peripheral | Direction | Destination | Subsystem |
|-----|----------|------------|-----------|-------------|-----------|
| PA0 | ADC_IN0 | USB VBUS detect | Input | 100k/100k divider from 5V_USB | USB |
| PA1 | — | Available | — | — | — |
| PA2 | USART2_TX | Debug console | Output | J_DBG pin 2 | Debug |
| PA3 | USART2_RX | Debug console | Input | J_DBG pin 3 | Debug |
| PA4 | — | Available | — | — | — |
| PA5 | — | Available | — | — | — |
| PA6 | TIM3_CH1 | Exhaust Fan 1 PWM | Output | IRLML6344 gate → J_FAN | Exhaust |
| PA7 | — | Available | — | — | — |
| PA8 | — | Available | — | — | — |
| PA9 | — | Available | — | — | — |
| PA10 | — | Available | — | — | — |
| PA11 | USB_DM | USB D- | Bidir | USBLC6-2SC6 → 22R → J_USB | USB |
| PA12 | USB_DP | USB D+ | Bidir | USBLC6-2SC6 → 22R → J_USB | USB |
| PA13 | SWDIO | SWD Debug | Bidir | J_SWD | Debug |
| PA14 | SWCLK / BOOT0 | SWD Debug, boot pin | Input | J_SWD; SW_BOOT | Debug |
| PA15 | — | Available | — | — | — |
| PB0 | GPIO | Safety Relay | Output | Q_RLY (2N7002) → G5V-2 coil → J_SAFE | Safety |
| PB1 | GPIO | Pot Detection (backup) | Input | Reed switch (10k pull-up) → J_SAFE | Safety |
| PB2 | GPIO (EXTI) | E-Stop | Input | NC button (10k pull-up, 100 nF) → J_SAFE | Safety |
| PB3 | GPIO (EXTI) | DOCK_SENSE_B | Input | J_DOCK pin 8 (10k pull-up; low = DU present) | Dock |
| PB4 | GPIO | Q_STIR enable | Output | 2N7002 → AO4407A (24V_STIR) | Stir |
| PB5 | GPIO | Q_DOCK enable | Output | 2N7002 → AO4407A (DOCK_24V) | Dock |
| PB6 | — | Available (I2C1_SCL pad, unpopulated) | — | — | — |
| PB7 | — | Available (I2C1_SDA pad, unpopulated) | — | — | — |
| PB8 | FDCAN1_RX | Isolated CAN RX | Input | ISO1050DUB #1 RXD | CAN (heater / DU) |
| PB9 | FDCAN1_TX | Isolated CAN TX | Output | ISO1050DUB #1 TXD | CAN (heater / DU) |
| PB10 | GPIO (SW PWM) | Exhaust Fan 2 PWM | Output | IRLML6344 gate → J_FAN | Exhaust |
| PB11 | — | Available | — | — | — |
| PB12 | FDCAN2_RX | Regular CAN RX | Input | SN65HVD230 R | CAN (stir) |
| PB13 | FDCAN2_TX | Regular CAN TX | Output | SN65HVD230 D | CAN (stir) |
| PB14 | — | Available | — | — | — |
| PB15 | — | Available | — | — | — |
| PC0–PC6 | — | Available | — | — | — |
| PC7 | GPIO | Status LED | Output | Green LED, 330R | — |
| PC8–PC12 | — | Available | — | — | — |
| PC13 | — | Available (tamper/wakeup pad) | — | — | — |
| PC14, PC15 | OSC32 | LSE pads (optional) | — | 32.768 kHz footprint, unpopulated | Clock |
| PD2 | — | Available | — | — | — |
| PF0, PF1 | OSC_IN/OUT | 8 MHz HSE | — | Y1 | Clock |

### 3.3 Pin Usage Summary

| Port | Used | Available |
|------|------|-----------|
| PA | PA0, PA2, PA3, PA6, PA11–PA14 (8) | PA1, PA4, PA5, PA7, PA8, PA9, PA10, PA15 (8) |
| PB | PB0–PB5, PB8, PB9, PB10, PB12, PB13 (11) | PB6, PB7, PB11, PB14, PB15 (5) |
| PC | PC7 (1) + LSE pads | PC0–PC6, PC8–PC12 (12) |
| PD | — | PD2 (1) |

Everything the STM32 used to drive over I2C (PCF8574, INA219, PCA9685) is gone from this unit, so every output is a direct GPIO and 26 pins stay free for the CS-side extras that may come later (fan tachometers, a second pot sensor, module interlocks).

---

## 4. Power Supply

### 4.1 24V Input Protection Chain

```
24V from LRS-75-24 (J_24V_IN) ──── F1 (5A Polyfuse) ──── D1 (SS54 Schottky) ────┬── D2 (SMBJ24A TVS) ──── 24V Internal
                                                                                │
                                                                               GND
```

| Component | Part | Function |
|-----------|------|----------|
| F1 | 5A resettable polyfuse (1812) | Overcurrent protection |
| D1 | SS54 (5A, 40V Schottky) | Reverse polarity protection (0.5 V drop) |
| D2 | SMBJ24A (24V TVS, 600W) | Transient suppression |

24 V Internal feeds the two bucks, Q_STIR and Q_DOCK. The CSC has no battery: when mains drops, the board, the fans, the relay and the dock 24 V all go down together, and the DU carries the session on its own pack.

### 4.2 24V → 12V Rail (MP1584EN #1)

| Parameter | Value |
|-----------|-------|
| Output | 12 V, 3 A max |
| Load | Exhaust fans (2x 0.5 A), G5V-2 relay coil (~40 mA) |
| Inductor | 33 µH, CDRH104R, Isat > 4 A |
| Output caps | 2x 22 µF MLCC (25 V) + 100 µF electrolytic |
| Feedback | 140k / 10k → 12.0 V |
| Protection | 2 A polyfuse + SMBJ12A on output |

### 4.3 24V → 5V Rail (MP1584EN #2)

| Parameter | Value |
|-----------|-------|
| Output | 5V_MAIN, 3 A max (loaded < 0.5 A) |
| Load | AMS1117-3.3 (~0.2 A), B0505S (~0.1 A in), status LED |
| Inductor | 22 µH, CDRH104R |
| Output caps | 2x 22 µF MLCC + 100 µF electrolytic |
| Feedback | 52.3k / 10k → 4.98 V |
| Protection | 1 A polyfuse + SMBJ5.0A on output |

The TPS54531 and the VSYS diode-OR of the Unified PCB are not needed here: there is no battery and no Pico Zero on this board, so a second MP1584EN is enough.

### 4.4 Two-Source 5V Schottky-OR → 3.3V Rail (AMS1117-3.3)

```
5V_MAIN (MP1584EN #2, 24 V) ─────────────────── D_OR1 (SS14) ──┐
                                                                ├── 5V_MERGED ──► AMS1117-3.3 ──► 3.3V
5V_USB  (USB-C VBUS, 500 mA polyfuse) ───────── D_OR2 (SS14) ──┘
```

The USB path powers the STM32 (3.3 V) for bench programming without the PSU; it does not back-feed 5V_MAIN or the B0505S (D_OR1 blocks it), so the isolated domain, the relay and the fans stay dead on USB alone. 3.3 V loads: STM32 (~0.15 A), SN65HVD230, ISO1050 #1 VCC1 — ~0.2 A.

### 4.5 Isolated 5V DC-DC (B0505S-1WR3)

The isolated CAN domain (5V_ISO / GND_ISO) is generated once, here, and shared by all three isolated-side transceivers on the bus: ISO1050 #1 on this board, the heating unit's node (if it takes bus power on J_CAN_HEAT pin 4) and ISO1050 #2 on the DU board (over J_DOCK pin 6).

| Parameter | Value |
|-----------|-------|
| IC | B0505S-1WR3 (SIP-4), 1 W, 3 kVDC isolation |
| Input | 5V_MAIN (before D_OR1, so it is never fed from USB); C_ISO_IN 10 µF |
| Output | 5V_ISO, 200 mA max; C_ISO_OUT 10 µF; loads: ISO1050 #1 VCC2 (~40 mA), ISO1050 #2 VCC2 on the DU (~40 mA, via the dock), optional J_CAN_HEAT pin 4 |
| Fuse | F_CAN 100 mA polyfuse on the exported 5V_ISO (J_CAN_HEAT pin 4 and J_DOCK pin 6 share it) |

### 4.6 Stirring Motor 24 V Switch (Q_STIR)

```
24V Internal ──► F_STIR (5A polyfuse) ──► Q_STIR: AO4407A (P-MOSFET, SO-8, -30 V, -12 A) ──► 24V_STIR ──► J_STIR_PWR pin 1
                                             Gate ── 10k to 24V (default OFF)
                                             Gate ── Q_STIR_N: 2N7002 drain; 2N7002 gate ── 100R ── PB4, 10k pull-down
                                          C_STIR: 100 µF (35 V) + 10 µF MLCC on 24V_STIR
```

The motor's own firmware stops it when CAN commands cease, but the switch gives the STM32 a command-independent way to de-energise the arm in the boot-safe state, on E-stop and on DU_UNDOCKED. Driven directly from PB4 (it was PCF8574 #2 P5 on the Unified PCB).

### 4.7 Dock 24 V Switch (Q_DOCK)

```
24V Internal ──► Q_DOCK: AO4407A (P-MOSFET, SO-8) ──► F_DOCK (2 A polyfuse) ──► DOCK_24V ──► J_DOCK pin 1
                    Gate ── 10k to 24V (default OFF)
                    Gate ── Q_DOCK_N: 2N7002 drain; 2N7002 gate ── 100R ── PB5, 10k pull-down
                 C_DOCK: 10 µF MLCC (50 V) on DOCK_24V; SMBJ24A at J_DOCK pin 1
```

| Parameter | Value |
|-----------|-------|
| Switch | AO4407A high-side, enable from STM32 PB5 via 2N7002 |
| Interlock | Firmware closes Q_DOCK only while DOCK_SENSE_B (PB3) is low, i.e. the DU is seated; opens it within one Safety Monitor tick (≤ 100 ms) when DOCK_SENSE_B goes high |
| Fuse | F_DOCK 2 A polyfuse (the pogo contacts are rated 2 A each) |
| Load | The DU's charger (≈ 0.6 A at 24 V) plus its logic; ≈ 1.6 A peak during a dispense on the dock |

The pads on the pillar are dead whenever the DU is lifted: no exposed 24 V, no arcing when the DU is seated or removed, and the DU's VSYS diode-OR sees a clean step.

### 4.8 Per-Rail Protection

| Rail | Polyfuse | TVS | Notes |
|------|----------|-----|-------|
| 24V input | 5 A (F1) | SMBJ24A | Reverse-polarity SS54 |
| 24V_STIR | 5 A (F_STIR) | — | Behind Q_STIR |
| DOCK_24V | 2 A (F_DOCK) | SMBJ24A at J_DOCK | Behind Q_DOCK; only on while DOCK_SENSE_B is low |
| 12V | 2 A | SMBJ12A | Fans, relay coil |
| 5V_MAIN | 1 A | SMBJ5.0A | LDO, B0505S |
| 5V_USB | 500 mA (F_USB) | USBLC6 on data | Bench only |
| 5V_ISO | 100 mA (F_CAN) | — | Exported on J_CAN_HEAT 4 and J_DOCK 6 |

### 4.9 Decoupling

- 100 nF MLCC within 5 mm of every STM32 VDD/VSS pair; 1 µF + ferrite bead on VDDA
- 10 µF at LDO input and output; 100 nF + 10 µF at each IC supply pin (SN65HVD230, ISO1050 #1 both sides, B0505S input and output)
- 10 µF + 100 nF on DOCK_24V and 5V_ISO at J_DOCK

### 4.10 Power Budget Summary

| Rail | Source | Typical (A) | Peak (A) | Notes |
|------|--------|-------------|----------|-------|
| 3.3V (0.8 A max) | 5V_MERGED → AMS1117 | 0.18 | 0.25 | STM32, SN65HVD230, ISO1050 #1 logic side |
| 5V_MAIN (3 A max) | 24V → MP1584EN #2 | 0.3 | 0.4 | LDO + B0505S |
| 5V_ISO (200 mA max) | 5V_MAIN → B0505S | 0.08 | 0.15 | ISO1050 #1 + #2 bus sides, optional module node |
| 12V (3 A max) | 24V → MP1584EN #1 | 1.0 | 1.1 | Two fans + relay coil |
| 24V_STIR | 24V via Q_STIR | 1.0 | 3.0 | CAN servo motor |
| DOCK_24V | 24V via Q_DOCK | 0.6 | 1.6 | DU charger (1 A at 8.4 V) + DU logic, + a DU servo move |
| **24V input** | **Mean Well LRS-75-24 (3.2 A)** | **~2.2** | **~3.2** | Peak = full stir torque + DU dispense + fans; the RV1106 sequences dispenses against stir torque (velocity limit while a carousel indexes) |

> [!warning]
> The 24 V budget is tight at peak, as it was on the single-board design. The stir velocity limit during a DSB move is now enforced on the RV1106 side (the dispense thread owns the servos and sends SET_STIR); the CSC's Safety Monitor still caps stir torque when the motor's reported current exceeds its limit. Move to the LRS-100-24 (4.2 A) if the measured peak exceeds 3 A.

---

## 5. Isolated CAN Port (FDCAN1)

The isolated 500 kbps bus is the system's host link and heater link in one: the STM32 (here), the heating unit (J_CAN_HEAT) and the RV1106 in the DU (through the dock) are its three nodes.

```
PB9 (FDCAN1_TX) ──► TXD ┐  ISO1050DUB #1  ┌── CANH ──┬── R_TERM1 (120R) ──┬── J_CAN_HEAT 1 (CANH), 2 (CANL)
PB8 (FDCAN1_RX) ◄── RXD ┘  VCC1 3.3V / GND │  CANL ───┤   PESD1CAN          │   J_CAN_HEAT 3 GND_ISO, 4 5V_ISO (F_CAN)
                            VCC2 5V_ISO / GND_ISO     │                     │
                                                      └─────────────────────┴── J_DOCK 3 (CAN_H), 4 (CAN_L), 5 GND_ISO, 6 5V_ISO
                  ─── 6 mm milled slot under ISO1050 #1 and the B0505S output; J_CAN_HEAT and J_DOCK 3–6 on the island ───
```

| | Isolated port |
|---|---|
| Connectors | J_CAN_HEAT (JST-XH 4-pin) to the heating unit; J_DOCK pins 3–6 to the pillar pogo block |
| STM32 controller | FDCAN1, PB8/PB9 (AF3), 64 MHz kernel clock |
| Transceiver | ISO1050DUB #1, 5 kV RMS; logic side 3.3 V / GND, bus side 5V_ISO / GND_ISO from the on-board B0505S |
| Nodes | STM32 (src_node 2, "STIR"), RV1106 ("AP", ISO1050 #2 + MCP2518FD on the DU board), heating unit MCU ("HEAT") |
| Bit rate | 500 kbps, classic CAN 2.0B, 11-bit IDs |
| Termination | R_TERM1 120 Ω here at J_CAN_HEAT and 120 Ω inside the heating unit. The dock → Unified PCB run is a stub ≤ 30 cm; the DU board has no termination |
| Ground | GND_ISO only on the island; the cable shield to the heating unit lands on GND_ISO at the PCB end |
| Traffic | Host commands/telemetry (RV1106 ↔ STM32), heater setpoint and status, heartbeat, E_STOP, STM32 firmware update (0x7E0/0x7E8) — ID map in [[../03-Software/02-Controller-Software-Architecture#7.2 Message Types\|Controller Software Architecture §7.2]] |

- The transceiver sits on the system-ground side of the slot with its bus pins crossing into the isolated island; the B0505S output crosses it too; no other copper crosses the boundary (creepage ≥ 6.4 mm, IEC 60335 reinforced).
- Decoupling: 100 nF on VCC1, 100 nF + 10 µF on VCC2.
- Shielded twisted pair ≤ 1 m to the heating unit; the dock harness to the pillar block ≤ 30 cm.
- Lifting the DU removes the RV1106 node from the bus: the STM32 keeps the heater off on keep-alive loss (≤ 1 s) and holds safe on DOCK_SENSE_B (§7.2); the bus itself stays terminated and keeps working between the STM32 and the heating unit.

---

## 6. Regular CAN Port (FDCAN2) and Stirring-Unit Harness

```
PB13 (FDCAN2_TX) ──► D  ┐ SN65HVD230 (SOIC-8, 3.3 V)  ┌── CANH ──┬── R_TERM2 (120R) ──┬── J_CAN_STIR 1 (CANH)
PB12 (FDCAN2_RX) ◄── R  ┘ Rs ── 10k to GND (slope ctl) └── CANL ──┘                    └── J_CAN_STIR 2 (CANL)
                                                                                           J_CAN_STIR 3 (GND)
PESD1CAN (ESD) across CANH/CANL; 100 nF on VCC
24V_STIR (Q_STIR, §4.6) ──► J_STIR_PWR 1; GND ──► J_STIR_PWR 2
```

| Parameter | Value |
|-----------|-------|
| Transceiver | SN65HVD230DR, 3.3 V, up to 1 Mbps, non-isolated (the motor shares the 24 V PSU ground) |
| Bit rate | 1 Mbps (motor protocol default), 29-bit IDs as the motor requires; FDCAN2 bit timing from the 64 MHz kernel clock |
| Termination | 120 Ω at J_CAN_STIR; 120 Ω in or at the motor (verify the motor's built-in termination) |
| Harness | J_CAN_STIR and J_STIR_PWR are bundled into one 5-way harness that runs inside the right rear pillar to **J_SU** (Molex Micro-Fit 3.0, 5-way: 24V_STIR, GND, CAN_H, CAN_L, SHIELD) at the pillar top, where the Stirring Unit plugs in. ≤ 1 m, twisted CAN pair with the 24 V pair, shield to GND at the CSC end |
| Fail-safe | Motor firmware command timeout (≤ 500 ms) must be configured so a silent bus stops the arm; Q_STIR removes 24 V independently |

The two CAN buses never share a transceiver, a termination or a connector; the regular bus is never routed adjacent to the isolated one.

---

## 7. Dock Interface (J_DOCK)

### 7.1 Pinout (CS view)

J_DOCK is an 8-pin JST-XH header; a short harness carries it to the spring-pin half of the magnetic pogo block in a keyed recess on top of the CS left pillar. The DU's flat gold pads (Unified PCB J_DOCK) meet it when the DU is seated.

| Pin | Signal | Direction | On the CSC |
|-----|--------|-----------|------------|
| 1 | DOCK_24V | CS → DU | Q_DOCK (AO4407A, PB5 via 2N7002) → F_DOCK 2 A; SMBJ24A + 10 µF at the header. Live only while DOCK_SENSE_B is low |
| 2 | GND | — | System GND; charge return. Ties the DU logic ground to the CS ground while docked |
| 3 | CAN_H | bus | Isolated bus, from the island (ISO1050 #1 bus side) |
| 4 | CAN_L | bus | |
| 5 | GND_ISO | — | Isolated bus ground |
| 6 | 5V_ISO | CS → DU | B0505S output through F_CAN 100 mA (shared with J_CAN_HEAT pin 4); powers the DU's ISO1050 #2 bus side |
| 7 | DOCK_SENSE_A | CS → DU | Tied to GND here (0R link); the DU pulls it up and reads low = docked |
| 8 | DOCK_SENSE_B | DU → CS | 10k pull-up to 3.3 V here → STM32 PB3 (EXTI), 100 nF; the DU ties it to GND; low = DU present |

Pins 1–2 and 7–8 are on the system-ground side; pins 3–6 are on the isolated island. The header is placed on the island boundary so the harness carries both domains without copper crossing the slot on the PCB; the harness itself keeps the two pairs apart (two 4-way sub-looms).

### 7.2 DOCK_SENSE_B Interlock

| Event | STM32 response |
|-------|----------------|
| DOCK_SENSE_B low (DU seated), HB_AP present | NORMAL: Q_DOCK closed (DOCK_24V live), bus traffic with the RV1106 |
| DOCK_SENSE_B goes high (DU lifted) | Immediate safe hold (EXTI, ≤ 10 ms): HEAT_ON_OFF = 0 to the heating unit, stir disable frame on FDCAN2, Q_STIR off, Q_DOCK off, STATUS (0x312) with fault code 0x17 DU_UNDOCKED (Info). Not an E_STOP: the relay stays closed, fans run per the thermal task |
| DOCK_SENSE_B low again and HB_AP returns | Back to NORMAL; Q_DOCK closes after a 100 ms debounce so the pogo contacts are seated before 24 V appears |
| DOCK_SENSE_B low but no HB_AP for 5 s | Existing heartbeat rule: safe state, Q_DOCK stays closed (the DU may be booting or charging) |

The heater also stops by itself ≤ 1 s after the RV1106's last keep-alive once the DU is off the bus, so the DU_UNDOCKED hold is a belt-and-braces measure, not the only one. Q_DOCK default is OFF (gate pull-up), so the pads are dead while the STM32 is in reset.

---

## 8. Safety and Auxiliary I/O

### 8.1 Relay Driver (Q_RLY)

PB0 → 330R → 2N7002 → Omron G5V-2 coil (12 V) with 1N4148 flyback; the relay's contacts cut AC to the heating unit through J_SAFE pin 1 (the AC wiring itself stays off the PCB, on the unit's inlet loom). NC wiring: relay de-energised = heater disconnected.

### 8.2 E-Stop Input

The hard E-stop button is on the CS front panel, wired to J_SAFE pin 3 → PB2 (EXTI) with 10k pull-up and 100 nF; pressed or wire-broken = LOW = stop: relay off, HEAT_ON_OFF = 0 and E_STOP (0x001) on FDCAN1, motor disable on FDCAN2, Q_STIR off, all within 100 ms. The DU's UI has a soft stop that broadcasts E_STOP over CAN while docked; the STM32 treats both the same.

### 8.3 Pot Detection Input (backup)

Reed switch on J_SAFE pin 2 → PB1 with 10k pull-up. Primary pot detection is the heating unit's own (spring-loaded centre sensor pressed by the pot), reported as `pot_detected` in HEAT_STATUS; the reed switch is an optional cross-check read by the Thermal/Fan task and reported in the TELEMETRY `pot_detected` flag.

### 8.4 Exhaust Fans (2x)

Two 120 mm 12 V fans at the CS rear behind the filter cartridge, low-side PWM via IRLML6344 + SS14 flyback: fan 1 on PA6 (TIM3_CH1, 25 kHz), fan 2 on PB10 (software PWM from the Thermal/Fan task at 1 Hz duty updates). Both independently controllable; stage and duty come from the RV1106 over CAN (SET_EXHAUST 0x260, GET_EXHAUST 0x261, EXHAUST_AUTO 0x262). Fan tachometers are optional (free pins PB14/PB15 reserved).

### 8.5 Status LED

PC7 → 330R → green LED, active-low: boot, NORMAL heartbeat blink, safe-hold pattern.

### 8.6 MOSFET Gate Safety

Every MOSFET gate and driver enable has a 100R series resistor and a 10k pull-down (or pull-up for the P-MOSFET switches Q_STIR and Q_DOCK) so that both fans, both 24 V switches and the relay are off while the STM32 is in reset or unpowered.

---

## 9. Programming and Debug

| Path | Interface | Use |
|------|-----------|-----|
| **CAN bootloader** (production) | FDCAN1, IDs 0x7E0 FW_UPDATE_REQ (AP → STIR: seq, op ERASE/WRITE/VERIFY/BOOT, 5 data bytes) and 0x7E8 FW_UPDATE_RSP (STIR → AP: seq, op, result, 4 data bytes) | The RV1106's swupdate handler streams the STM32 image over the isolated bus to an in-application, OpenBLT-style bootloader in the first flash sector. Update mode only, lowest priority; the application is only entered after VERIFY passes. There is no UART or BOOT0/NRST line from the RV1106 any more |
| **USB-C DFU / CDC** (bench) | J_USB, PA11/PA12, SW_BOOT on BOOT0 (PA14 on the G0; nBOOT_SEL cleared so the pin is sampled), SW_RST | ST system bootloader DFU with 24 V disconnected (USB powers the 3.3 V domain only, §4.4); CDC console once the application runs |
| **SWD** (factory, debug) | J_SWD 2x5 1.27 mm: VCC 3.3 V, SWDIO (PA13), SWCLK (PA14), SWO (n/a on the M0+, pin left for the standard pinout), NRST, GND | Initial bootloader flash, debugger |
| **Debug console** | J_DBG 3-pin (GND, TX PA2, RX PA3), 3.3 V, 115200 8N1 | FreeRTOS trace, Class B self-test log; no host link |

The USB-C port is 16-pin mid-mount with 5.1k CC pull-downs, USBLC6-2SC6 ESD, 22R series on D+/D−, 500 mA VBUS polyfuse → 5V_USB → D_OR2, VBUS detect divider → PA0.

---

## 10. Connector Definitions (10 total)

| Ref | Type | Function |
|-----|------|----------|
| J_24V_IN | XT30 2-pin | 24 V DC input from the LRS-75-24 |
| J_CAN_HEAT | JST-XH 2.5 mm 4-pin | Isolated CAN to the heating unit |
| J_CAN_STIR | JST-XH 2.5 mm 3-pin | Regular CAN to the stirring motor (SU harness) |
| J_STIR_PWR | XT30 2-pin | Switched 24 V to the stirring motor (SU harness) |
| J_DOCK | JST-XH 2.5 mm 8-pin | Harness to the pillar pogo block: DOCK_24V, GND, isolated CAN, 5V_ISO, sense |
| J_FAN | JST-XH 2.5 mm 4-pin | Two exhaust fans |
| J_SAFE | Hirose DF11-4DP-2DS (2x2) | Relay drive, pot detect, E-stop, GND |
| J_DBG | 1x3 header, 2.54 mm | Debug console (USART2) |
| J_SWD | 2x5 1.27 mm | Cortex debug |
| J_USB | USB-C 16-pin mid-mount | STM32 DFU / CDC (bench) |

### 10.1 J_24V_IN — 24 V DC Power Input (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | +24V | From Mean Well LRS-75-24 |
| 2 | GND | Power ground |

### 10.2 J_CAN_HEAT — Isolated CAN to the Heating Unit (JST-XH 2.5 mm, 4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | CAN_H | Isolated bus, R_TERM1 120 Ω here |
| 2 | CAN_L | |
| 3 | GND_ISO | Isolated ground (NOT system GND); cable shield lands here |
| 4 | 5V_ISO | Optional isolated 5 V for the heating unit's node, F_CAN 100 mA polyfuse (shared with J_DOCK pin 6) |

### 10.3 J_CAN_STIR — Regular CAN to the Stirring Motor (JST-XH 2.5 mm, 3-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | CAN_H | Non-isolated, 1 Mbps, R_TERM2 120 Ω |
| 2 | CAN_L | |
| 3 | GND | System ground reference for the motor's CAN transceiver |

### 10.4 J_STIR_PWR — Stirring Motor Power (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 24V_STIR | Switched by Q_STIR (PB4), F_STIR 5 A |
| 2 | GND | Power ground |

J_CAN_STIR + J_STIR_PWR → one harness inside the right pillar → **J_SU** (Molex Micro-Fit 3.0, 5-way) at the pillar top:

| J_SU pin | Signal | From |
|----------|--------|------|
| 1 | 24V_STIR | J_STIR_PWR 1 |
| 2 | GND | J_STIR_PWR 2 |
| 3 | CAN_H | J_CAN_STIR 1 |
| 4 | CAN_L | J_CAN_STIR 2 |
| 5 | SHIELD | J_CAN_STIR 3 (GND at the CSC end only) |

### 10.5 J_DOCK — see §7.1

### 10.6 J_FAN — Exhaust Fans (JST-XH 4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 12V_FAN1 | Fan 1 supply |
| 2 | FAN1_OUT | Fan 1 drain (PA6 PWM) |
| 3 | 12V_FAN2 | Fan 2 supply |
| 4 | FAN2_OUT | Fan 2 drain (PB10 PWM) |

### 10.7 J_SAFE — Safety I/O (Hirose DF11-4DP-2DS)

| Pin | Signal | STM32 Pin | Notes |
|-----|--------|-----------|-------|
| 1 | RELAY_DRV | PB0 via Q_RLY | Safety relay coil (12 V) |
| 2 | POT_DET | PB1 | Reed switch, 10k pull-up (backup) |
| 3 | E_STOP | PB2 | NC button on the CS front, 10k pull-up, RC debounce |
| 4 | GND | GND | |

### 10.8 J_DBG — Debug Console (1x3, 2.54 mm)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | GND | |
| 2 | TX | PA2 (USART2_TX), 3.3 V |
| 3 | RX | PA3 (USART2_RX), 3.3 V, 10k pull-up |

### 10.9 J_SWD — SWD Debug (2x5, 1.27 mm)

Standard Cortex debug pinout: VCC 3.3 V, SWDIO (PA13), SWCLK (PA14), SWO (unused), NRST, GND.

### 10.10 J_USB — USB-C 2.0 (16-pin mid-mount)

VBUS → F_USB → 5V_USB; CC1/CC2 5.1k to GND; D+/D− → USBLC6 → 22R → PA12/PA11. SuperSpeed and SBU pins unconnected.

---

## 11. 4-Layer Stackup and Zone Layout

### 11.1 Layer Stack

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (only break: CAN isolation slot) | 1 oz |
| L3 | Split power plane: 3.3 V / 5 V, 12 V, 24 V | 1 oz |
| L4 (Bottom) | Signal + power traces | 2 oz |

FR-4 Tg ≥ 150 °C, 1.6 mm, ENIG. Same stack and net classes as the Unified PCB ([[03-PCB-Design-Rules|PCB Design Rules]]).

### 11.2 Board Layout Zones

```
┌─────────────────────────── ~100 mm ───────────────────────────┐
│ ZONE A: MCU + DEBUG            │ ZONE C: POWER IN + SWITCHES   │
│ STM32G0B1RET6, Y1 (8 MHz)      │ J_24V_IN XT30, F1, SS54, TVS  │
│ AMS1117, D_OR1-2               │ MP1584EN #1 (12 V), #2 (5 V)  │
│ J_SWD  J_USB  J_DBG            │ Q_STIR + F_STIR, Q_DOCK + F_DOCK│ ~70 mm
│ SW_BOOT  SW_RST  status LED    │ Q_RLY + G5V-2 relay            │
├────────────────────────────────┼───────────────────────────────┤
│ ZONE B: REGULAR CAN            │ ZONE D: ISOLATED CAN           │
│ SN65HVD230, R_TERM2, PESD      │ ┌── 6 mm slot ───────────────┐│
│ J_CAN_STIR  J_STIR_PWR         │ │ B0505S │ ISO1050 #1 │ island ││
│ (SU harness exit, right)       │ └────────── J_CAN_HEAT  J_DOCK ┘│
│ fan MOSFETs  J_FAN  J_SAFE     │ (dock harness exit, left)      │
└────────────────────────────────┴───────────────────────────────┘
```

Placement rationale:
- **Zone D** at the left edge, nearest the left pillar: J_DOCK straddles the island boundary (pins 1–2, 7–8 outside, 3–6 inside); J_CAN_HEAT is on the island; the slot runs under ISO1050 #1 and the B0505S output.
- **Zone B** at the right edge, nearest the right pillar, so the SU harness (J_CAN_STIR + J_STIR_PWR) leaves the board as one bundle; ≥ 20 mm from the isolated island, never routed parallel to it.
- **Zone C**: input chain, both bucks and all three switches together; the relay with its 12 V coil drive here, its contact side wired off-board on the heating unit's AC loom (≥ 2.5 mm clearance around the contact pins, IEC 60335-1).
- **Zone A**: the STM32 ≥ 25 mm from the bucks; USB pair < 30 mm to J_USB.

### 11.3 Critical Layout Rules

| Rule | Specification | Rationale |
|------|---------------|-----------|
| CAN isolation | Creepage ≥ 6.4 mm GND ↔ GND_ISO; 6 mm milled slot under ISO1050 #1 and the B0505S output; no copper crossing; J_DOCK pins 2 ↔ 3 and 6 ↔ 7 spaced for ≥ 6.4 mm (skip a position or use two 4-pin headers if the pitch cannot) | IEC 60335 reinforced insulation, carried across the dock |
| Two CAN buses | Stub lengths < 30 mm, 120 Ω at each connector, never route the two buses adjacent | Signal integrity, no cross-talk |
| USB D+/D− | 90 Ω differential, matched ± 0.5 mm, < 30 mm | USB 2.0 FS |
| 24V_STIR | ≥ 2 mm, Q_STIR with ≥ 200 mm² pour | 3 A motor peak |
| DOCK_24V | ≥ 1.5 mm, Q_DOCK with ≥ 150 mm² pour | 2 A fused |
| Buck loops | Input cap → IC → inductor → output cap loop < 10 mm | EMI |
| Thermal vias | ≥ 9 under the STM32 pad, ≥ 6 under each buck IC | Heat to L2 |
| Mains relay | ≥ 2.5 mm clearance around relay contact pins; contact side on L1 only | IEC 60335-1 |

---

## 12. Protection Circuits

- **24 V input**: polyfuse → Schottky → TVS (§4.1)
- **ESD**: PRTR5V0U2X on the J_SAFE inputs and DOCK_SENSE_B; PESD1CAN on J_CAN_STIR and on the isolated side of J_CAN_HEAT / J_DOCK 3–4; USBLC6-2SC6 on USB D+/D−
- **Dock**: SMBJ24A + 10 µF on DOCK_24V at J_DOCK; F_DOCK 2 A; Q_DOCK default OFF and interlocked on DOCK_SENSE_B so the pogo contacts never make or break under load
- **Isolated domain**: F_CAN 100 mA on the exported 5V_ISO; B0505S input taken before D_OR1 so USB never feeds the island
- **MOSFET gates**: 100R series + 10k pull-down / pull-up on every gate (§8.6)
- **Relay**: 1N4148 flyback; NC wiring so loss of coil power disconnects the heater
- **Watchdog**: IWDG kicked by the Safety Monitor; X-CUBE-CLASSB self-tests at boot and periodically (CPU, flash CRC, RAM march, clock)

---

## 13. Thermal Analysis

| Component | Condition | Power (W) | Notes |
|-----------|-----------|-----------|-------|
| MP1584EN #1 (12 V) | 1.0 A continuous | ~1.4 | Fans run through the whole session |
| MP1584EN #2 (5 V) | 0.3 A | ~0.3 | |
| AMS1117-3.3 | 0.2 A | ~0.35 | |
| B0505S, ISO1050 #1, SN65HVD230 | — | ~0.4 | |
| SS54 (D1), Q_STIR, Q_DOCK, Q_RLY, fan MOSFETs | 3 A / 1.6 A / coil / 0.5 A each | ~1.6 | Diode drop dominates |
| G5V-2 coil | 12 V, 40 mA | ~0.5 | Energised while the heater is allowed |
| **Total** | | **~4.5 W typical** | In the CS rear airflow of the exhaust fans; every IC below 110 °C junction with the pours in §11.3 |

---

## 14. Firmware Integration Notes

### 14.1 FreeRTOS Task Allocation

| Task | Rate | Priority | Functions |
|------|------|----------|-----------|
| Motor Control | 50 Hz | Highest | FDCAN2: velocity/position commands to the stir motor, feedback parsing (rpm, current, temperature), stir patterns, ramps, stall detection from motor current |
| Safety Monitor | 10 Hz | Highest | HEAT_STATUS limits (surface/element over-temperature), E-stop, HB_AP heartbeat (safe state after 5 s), DOCK_SENSE_B hold, Q_STIR / Q_DOCK / relay authority, IWDG kick |
| CAN Comms (FDCAN1) | 20 Hz | High | Host commands, ACK/NAK, TELEMETRY / STATUS, HEAT_STATUS capture, heating unit 1 s keep-alive, FW_UPDATE handling into the bootloader |
| Thermal / Fan | 1 Hz | Medium | Fan duty (PA6 / PB10), optional tachometers, pot reed, exhaust stage (0x260–0x262) |

The Dispense and Sensor Poll tasks of the single-board design are gone: dispensing, load cells, battery and charger monitoring run on the RV1106.

### 14.2 STM32 Peripheral Allocation

| Peripheral | Pins | Function |
|------------|------|----------|
| FDCAN1 | PB8/PB9 | Isolated bus (heating unit + DU), 500 kbps, 11-bit IDs |
| FDCAN2 | PB12/PB13 | Stir motor bus, 1 Mbps, 29-bit IDs |
| USART2 | PA2/PA3 | Debug console (J_DBG) |
| TIM3_CH1 | PA6 | Fan 1, 25 kHz |
| GPIO | PB10 | Fan 2, software PWM |
| GPIO | PB0, PB1, PB2 | Relay, pot reed, E-stop (EXTI) |
| GPIO | PB3, PB4, PB5 | DOCK_SENSE_B (EXTI), Q_STIR, Q_DOCK |
| GPIO | PC7 | Status LED |
| ADC1 | PA0 | VBUS detect |
| USB FS | PA11/PA12 | DFU / CDC |
| IWDG, CRC, RAM | — | Class B |

### 14.3 CAN Behaviour Specific to the CSC

- **TELEMETRY (0x310) flags**: pot_detected, stir_enabled, docked, fan1_ok, fan2_ok (on_battery and charging are gone; the DU reports its own battery to the UI)
- **STATUS (0x312)** fault codes kept: 0x01–0x05, 0x10 STIR_STALL, 0x15 STIR_CAN_FAULT, 0x20; added 0x17 DU_UNDOCKED (Info); removed 0x11–0x14, 0x16 and 0x21 (no I2C on this board). `fault_detail` no longer carries batt_mV
- **DOCK_SENSE_B open** → safe hold (§7.2), not an E_STOP; back to NORMAL when DOCK_SENSE_B is low again and HB_AP returns
- **Mains return after an outage**: the CSC reboots in < 1 s, closes Q_DOCK once DOCK_SENSE_B reads low, resumes TELEMETRY; the RV1106 (still running on its pack) auto-resumes if the outage was < 5 min
- Full requirements: [[../03-Software/05-MCU-Functional-Specification|MCU Functional Specification]]; ID map: [[../03-Software/02-Controller-Software-Architecture|Controller Software Architecture]] §7.2

---

## 15. Design Verification Checklist

### 15.1 Pre-Fabrication

- [ ] ERC passes; STM32G0B1 alternate functions verified: PB8/PB9 FDCAN1, PB12/PB13 FDCAN2, PA6 TIM3_CH1, PA2/PA3 USART2, PA11/PA12 USB
- [ ] Two separate CAN buses: no shared termination, FDCAN1 only on ISO1050 #1, FDCAN2 only on SN65HVD230
- [ ] Isolation: only the ISO1050 #1 bus pins and the B0505S output cross the slot; J_CAN_HEAT and J_DOCK 3–6 on the island
- [ ] Q_STIR and Q_DOCK default OFF (gate pull-ups), driven by PB4 / PB5 through 2N7002; F_STIR 5 A, F_DOCK 2 A
- [ ] DOCK_SENSE_B 10k pull-up + 100 nF on PB3; DOCK_SENSE_A tied to GND at J_DOCK 7; 5V_ISO to J_DOCK 6 through F_CAN
- [ ] All MOSFET gates: 100R series + 10k pull-down (or pull-up for P-MOSFETs)
- [ ] 24 V protection order: polyfuse → Schottky → TVS; B0505S fed before D_OR1
- [ ] USB: 22R, USBLC6, 5.1k CC, VBUS divider on PA0, SW_BOOT on BOOT0/PA14 with 10k pull-down, nBOOT_SEL option documented
- [ ] All 10 connectors have correct pinouts and silkscreen; J_SU harness drawing matches §10.4

### 15.2 PCB Layout

- [ ] DRC passes; GND plane continuous except the CAN slot
- [ ] Creepage ≥ 6.4 mm across the isolation slot and across J_DOCK (2 ↔ 3, 6 ↔ 7), ≥ 2.5 mm around the relay contacts
- [ ] USB pair 90 Ω, < 30 mm
- [ ] 24V_STIR and DOCK_24V trace widths and pours per §11.3
- [ ] Buck loops < 10 mm; thermal vias and pours present
- [ ] J_CAN_HEAT and J_DOCK the only connectors touching the isolated island; J_CAN_STIR ≥ 20 mm from it

### 15.3 Post-Assembly

- [ ] 3.3 V, 5 V, 12 V rails within ±3 %; 5V_ISO within ±5 % with both ISO1050 bus sides loaded (bench pogo block to a Unified PCB)
- [ ] STM32 responds on SWD; DFU over USB-C with 24 V disconnected; console on J_DBG
- [ ] FDCAN1 at 500 kbps: HEAT_QUERY to the heating unit returns HEAT_STATUS (surface_temp, element_temp, pot_detected); frames reach the DU's `can0` through the bench dock
- [ ] FDCAN2 at 1 Mbps: motor enumerates through J_SU, velocity command spins the arm, feedback frames arrive at 50 Hz
- [ ] Q_STIR off at reset: 0 V on J_STIR_PWR until PB4 is set; Q_DOCK off at reset and with DOCK_SENSE_B open: 0 V on J_DOCK 1
- [ ] Seat the DU (DOCK_SENSE_B low): Q_DOCK closes after the debounce, DOCK_24V present, TELEMETRY `docked` set; lift it: hold within 10 ms, DOCK_24V off, STATUS DU_UNDOCKED
- [ ] CAN bootloader: an image streamed over 0x7E0/0x7E8 from the RV1106 verifies and boots; an interrupted update leaves the bootloader recoverable
- [ ] E-stop: relay opens, HEAT_ON_OFF=0 and E_STOP on FDCAN1, motor disable on FDCAN2 and Q_STIR off, all within 100 ms
- [ ] Fans: both PWM channels control speed; SET_EXHAUST over CAN changes duty
- [ ] All actuators, both 24 V switches and the relay off with the STM32 held in reset

---

## 16. Related Documentation

- [[01-Unified-PCB-Design|Unified PCB Design]] — the DU board (v4.0): RV1106, battery, dispensers, the dock's other half
- [[../02-Hardware/01-Epicura-Architecture|Epicura Architecture]] — three units, system wiring, CAN bus, dock, power
- [[../02-Hardware/02-Technical-Specifications|Technical Specifications]] — power specs
- [[../05-Subsystems/01-Electric-Heating|Electric Heating]] — electric heating unit (rice-cooker class) with internal temperature control, isolated CAN port, safety relay
- [[../05-Subsystems/02-Robotic-Arm|Robotic Arm]] — CAN servo stirring motor (SU)
- [[../05-Subsystems/05-Exhaust-Fume-Management|Exhaust & Fume Management]] — fans, filter cartridge, stage commands
- [[../03-Software/05-MCU-Functional-Specification|MCU Functional Specification]] — STM32 firmware requirements
- [[../03-Software/02-Controller-Software-Architecture|Controller Software Architecture]] — CAN ID map (§7.2)
- [[03-PCB-Design-Rules|PCB Design Rules]] — DRC rules and net classes

---

#epicura #pcb #csc #cooking-surface #stm32 #can #dock #power-electronics #hardware-design

---

## 17. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-11-04 | Manas Pradhan | Initial document for the three-unit architecture: the STM32G0B1RE and the Cooking Surface side of the Unified PCB v3.x (isolated CAN port with ISO1050DUB #1 + B0505S and J_CAN_HEAT, regular CAN port with SN65HVD230 + Q_STIR + J_CAN_STIR / J_STIR_PWR into the J_SU harness, safety relay, E-stop, pot reed, exhaust fans, 24 V input chain, USB-C DFU, SWD) move to this ~100 x 70 mm board in the CS base; new dock interface (J_DOCK 8-pin: Q_DOCK + F_DOCK switched 24 V on DOCK_SENSE_B, isolated CAN + 5V_ISO on pins 3–6, DOCK_SENSE_A/B), Q_STIR enable on PB4, DOCK_SENSE_B on PB3, Q_DOCK on PB5, USART2 as J_DBG console, STM32 firmware update over CAN (0x7E0/0x7E8); PCF8574, INA219, 24 V comparator, battery ADC and all dispensing / sensing / LED / buzzer functions dropped (on the DU board); 10 connectors, 26 free pins |
| 1.1 | 2026-11-04 | Manas Pradhan | Heater on J_CAN_HEAT is now the commercial electric heating unit (rice-cooker class: resistive plate, centre sensor, own PID, CAN port) instead of the induction surface; wording in the overview, block diagrams, scope table, J_CAN_HEAT, relay, pot-reed, task table and bring-up checklist; HEAT_STATUS `coil_temp` → `element_temp` (same byte). Same CAN contract, no schematic or BOM change |
