---
created: 2026-11-04
modified: 2026-11-04
version: 1.6
status: Draft
---

# Cooking Surface Controller (CSC) PCB Design

## 1. Overview

The **Cooking Surface Controller (CSC)** is the second custom board of Epicura and the only one in the **Cooking Surface (CS)**, the mains-powered base unit. It sits in the CS rear beside the three Mean Well PSUs (LRS-75-24 #1 for the Dispensing Unit's dock charge feed, LRS-75-24 #2 floating for the Stirring Unit, LRS-35-12 for this board, the relays and the fans) and the two exhaust fans, and carries the **STM32G0B1RET6** that was on the Unified PCB up to v3.x together with everything that belongs physically to the base: the electric heating unit's safety relay, the E-stop button, the pot reed switch, the exhaust fans, the second isolated CAN port and the pass-through of the separate 24 V supply of the **Stirring Unit (SU)**, and the power + CAN side of the dock that the **Dispensing Unit (DU)** seats on.

One ~100 x 70 mm, 4-layer board carries:

- **STM32G0B1RET6** (Cortex-M0+ @ 64 MHz, 2x FDCAN), FreeRTOS, MISRA C, X-CUBE-CLASSB: safety monitor, heater off-switch, stir motor control, fans, dock interlock
- **Isolated CAN port #1** (FDCAN1 → ISO1050DUB #1, B0505S #1): the 500 kbps bus shared by the STM32, the heating unit (J_CAN_HEAT) and the DU STM32's non-isolated SN65HVD230 node in the DU (dock pins 3–5; pin 6 reserved)
- **Isolated CAN port #2** (FDCAN2 → ISO1050DUB #3, B0505S #2, J_CAN_STIR) for the CAN servo stirring motor, plus the pass-through of the SU's own floating 24 V rail (J_SU_IN → F_SU → J_STIR_PWR) and the pilot relay K_SU that cuts the SU PSU's AC input; CAN and 24 V are bundled into the SU harness ending at J_SU on the right pillar. The SU ground (GND_SU) is never bonded to the CS ground
- **Dock interface (J_DOCK)**: switched, fused 24 V charge feed (Q_DOCK, F_DOCK) to the DU, the isolated bus (pins 3–5) and a reserved 5V_ISO pin over the pogo block, DOCK_SENSE_B interlock
- **Safety I/O**: two Omron G5V-2 pilot relays cutting AC to the heating unit (K_SAFE, on its inlet loom) and to the SU PSU (K_SU), E-stop NC button, pot reed switch (J_SAFE), two 12 V exhaust fans (J_FAN)
- **Power**: the board, both relay coils and the fans run from **12V_IN**, a dedicated Mean Well LRS-35-12 (J_12V_IN); one buck makes 5 V, an LDO 3.3 V. The LRS-75-24 #1 24 V (J_DU_IN) only passes through Q_DOCK to the Dispensing Unit; neither 24 V PSU powers this board. USB-C DFU + SWD + debug UART for the bench

The CSC has no I2C devices, no dispensers, no load cells, no battery and no display: those are on the Unified PCB v5.0 in the DU ([[01-Unified-PCB-Design|Unified PCB Design]]), driven by the DU's own STM32G0B1RE. It has no temperature sensor either: the heating unit runs its own loop on its centre (pot-bottom) sensor and reports over CAN (see [[../05-Subsystems/01-Electric-Heating|Electric Heating]]).

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
│   │                   │    ┌── SU domain (5V_ISO2 / GND_SU, floating) ──┐ │
│   │                   │──FDCAN2──► B0505S #2 ► ISO1050 #3 ► J_CAN_STIR ─┐│ │
│   │                   │    │ J_SU_IN (LRS-75-24 #2) ─ F_SU ─► J_STIR_PWR ┴┼─► J_SU
│   │                   │    └───────────────────────────────────────────┘ │
│   │                   │──PB4────► Q_KSU ──► K_SU (pilot) ──► SU PSU AC    │
│   │                   │──PB5────► Q_DOCK (24 V, F_DOCK 2 A) ──► J_DOCK 1 │
│   │                   │◄─PB3───── DOCK_SENSE_B ◄──────────────── J_DOCK 8│
│   │                   │──PB0────► Q_RLY ──► G5V-2 relay ──► J_SAFE 1     │
│   │                   │◄─PB1/PB2─ pot reed, E-stop ◄──────── J_SAFE 2/3  │
│   │                   │──PA6/PB10► fan MOSFETs ──► J_FAN (12 V)          │
│   └───────────────────┘                                                  │
│   J_USB (DFU), J_SWD, J_DBG, SW_BOOT, SW_RST, status LED                  │
└──────────────────────────────────────────────────────────────────────────┘
   ▲ J_12V_IN (LRS-35-12, this board)    ▲ J_DU_IN (LRS-75-24 #1, DU only)   ▲ J_SU_IN (LRS-75-24 #2, SU, floating)
```

### 1.2 Scope

| Item | Status | Notes |
|------|--------|-------|
| **Cooking Surface Controller** (this document) | Custom design required | STM32G0B1RE + two isolated CAN ports + SU supply pass-through + dock power/CAN + safety I/O + fans, on its own 12 V PSU; ~100 x 70 mm, 4-layer |
| **Unified PCB v5.0** | Custom design — see [[01-Unified-PCB-Design\|Unified PCB Design]] | DU board: RV1106 + DU STM32G0B1RE (UART host link), battery, dispensers, DU STM32 CAN node (FDCAN1 + non-isolated SN65HVD230, gateway for the RV1106; the DU ties GND_ISO to its ground); meets this board through the dock |
| **Electric heating unit** | Commercial rice-cooker-class heating unit (resistive plate, own PID, CAN port; 1,300 W assumed) (see [[../05-Subsystems/01-Electric-Heating\|Electric Heating]]) | Isolated CAN port J_CAN_HEAT; AC inlet cut by the safety relay K_SAFE (J_SAFE). Its internal wiring and its own control-board supply are not touched |
| **Stirring motor** | 24 V integrated BLDC servo with CAN interface (see [[../05-Subsystems/02-Robotic-Arm\|Robotic Arm]]) | Isolated CAN port #2 J_CAN_STIR + the SU's own 24 V on J_STIR_PWR (from J_SU_IN), harness inside the right pillar to J_SU |
| **Dock pogo block** | Off-the-shelf 8-contact magnetic pogo pair (2.54 mm pitch), spring-pin half in the CS left pillar | Harness to J_DOCK (8-pin JST-XH) |
| **Mean Well LRS-75-24 #1 (DU)** | Off-the-shelf PSU in the CS rear | 24 V, 3.2 A to J_DU_IN: the Dispensing Unit's supply only (DOCK_24V charge feed over the dock through Q_DOCK); nothing on the CSC runs from it. Its −V is bonded to the CSC ground at J_DU_IN, which the DU shares while docked (dock pin 2) |
| **Mean Well LRS-75-24 #2 (SU)** | Off-the-shelf PSU in the CS rear, AC input through K_SU | Floating 24 V, 3.2 A to J_SU_IN; its output (24V_SU / GND_SU) is never bonded to the CS ground |
| **Mean Well LRS-35-12 (CSC)** | Off-the-shelf PSU in the CS rear | 12 V, 3 A to J_12V_IN: this board, both relay coils and the two fans (~1.3 A typical); its −V is the CSC ground |

### 1.3 What Moved Here from the Unified PCB v3.x

| Function | Unified PCB v3.x | CSC v1.0 |
|----------|------------------|----------|
| MCU | STM32G0B1RE, 8 MHz HSE, J_SWD, J_USB, SW_BOOT/SW_RST | Same parts, same clock; PCF8574 dropped (all outputs are direct GPIO) |
| Isolated CAN | ISO1050DUB #1 (STM32) + #2 (RV1106) + B0505S on one board, J_CAN_HEAT | ISO1050DUB #1 + B0505S #1 here; the bus continues over J_DOCK pins 3–5 to a non-isolated SN65HVD230 on the DU board (DU STM32 FDCAN1), which ties GND_ISO to the DU ground; pin 6 (5V_ISO) is exported but reserved |
| Stir CAN + motor power | SN65HVD230 on FDCAN2 (non-isolated), Q_STIR from PCF8574 #2 P5, J_CAN_STIR, J_STIR_PWR | Isolated: ISO1050DUB #3 + B0505S #2 on FDCAN2; the motor runs from its own LRS-75-24 #2 (J_SU_IN → F_SU → J_STIR_PWR, floating SU domain); Q_STIR replaced by the AC pilot relay K_SU on PB4; both connectors feed the SU harness (J_SU) |
| Safety | Q_RLY on PB0, pot reed PB1, E-stop PB2, J_SAFE | Unchanged |
| Fans | PA6 (TIM3_CH1), PB10, J_FAN, 12 V | Unchanged (PB10 as software PWM on the G0) |
| Board power | 24 V from the LRS-75-24 via J_24V_IN, F1 5 A, SS54, SMBJ24A, MP1584EN #1 → 12 V | 12 V from a dedicated LRS-35-12 (J_12V_IN, F1 2 A, SS34, SMBJ12A, 100 µF); no 24 → 12 V buck. The LRS-75-24 #1 24 V enters on J_DU_IN only for Q_DOCK |
| 5 V / 3.3 V | TPS54531 from VSYS (battery-backed) | MP1584EN 12 → 5 V (no battery on this board), AMS1117-3.3, 5V_USB OR for the bench |
| Dock | — | New: Q_DOCK + F_DOCK → DOCK_24V, DOCK_SENSE_B interlock on PB3, J_DOCK 8-pin |
| Dropped from the STM32 side | PCF8574, INA219, PA1 24 V comparator, PA4 battery ADC, PCA9685 / TB6612 / HX711 / solenoids / buzzer / LED ring, USART2 service link to the RV1106 | On the DU board (driven by the DU STM32 since Unified PCB v5.0) or gone; USART2 is now a debug console on J_DBG |

---

## 2. Board-Level Block Diagram

```
J_12V_IN ─── F1 (2A) ─── SS34 ─── SMBJ12A ─── C_IN 100 µF ─── 12V Internal ──┬──────────────────┬──────────────────┐
(Mean Well LRS-35-12, dedicated to this board;                               │                  │                  │
 nothing from the two 24 V PSUs)                                              ▼                  ▼                  ▼
                                                                            J_FAN          K_SAFE G5V-2       K_SU G5V-2
                                                                          (2x fans)       coil (Q_RLY, PB0)  coil (Q_KSU, PB4)
                                                                                          → heating unit     → SU PSU AC loom
                                                                                            AC inlet loom

J_DU_IN (LRS-75-24 #1, DU only) ─── F_DU (3A) ─── SS54 ─── SMBJ24A ─── 24V_DU ──► Q_DOCK (AO4407A, PB5 → 2N7002) ──► F_DOCK (2 A) ──► DOCK_24V ──► J_DOCK pin 1
         −V bonded to GND here (the DU shares it on dock pin 2)                                                                                 (to the DU charger)

12V Internal ──► MP1584EN (12 V → 5 V, 3A) ──► 5V_MAIN ──┬── D_OR1 (SS14) ──┐
                                                          │                  ├── 5V_MERGED ──► AMS1117-3.3 ──► 3.3V (STM32, ISO1050 #1 / #3 VCC1)
USB-C VBUS ─── F_USB (500 mA) ─── 5V_USB ── D_OR2 (SS14) ─┼──────────────────┘
                                                          ├──► B0505S #1 ──► 5V_ISO / GND_ISO ──┬──► ISO1050 #1 VCC2
                                                          │                                      ├──► J_CAN_HEAT pin 4 (F_CAN 100 mA)
                                                          │                                      └──► J_DOCK pin 6 (shared F_CAN)
                                                          └──► B0505S #2 ──► 5V_ISO2 / GND_SU ──────► ISO1050 #3 VCC2 (SU domain)

J_SU_IN (LRS-75-24 #2, floating) ─── F_SU (5A) ─── SMBJ24A (to GND_SU) ─── 24V_SU ──► J_STIR_PWR ──┐
         AC input switched by K_SU          SU island: GND_SU = B0505S #2 output ground, never GND      ├──► J_SU (SU harness, right pillar)
                                                                                      J_CAN_STIR ──┘

                     ┌────────────────────────────────────────────────────────────────┐
                     │  STM32G0B1RET6 (LQFP-64), 8 MHz HSE                            │
                     │  FDCAN1 (PB8/PB9) ──► ISO1050DUB #1 ──► CAN_H/L ─┬─ R_TERM1 ─ J_CAN_HEAT (heating unit)
                     │                                                   └─────────── J_DOCK 3/4 (DU STM32 node)
                     │  FDCAN2 (PB12/PB13) ──► ISO1050DUB #3 ──► J_CAN_STIR (1 Mbps, SU domain)
                     │  PB0 ──► Q_RLY ──► K_SAFE; PB1 ◄── pot reed; PB2 ◄── E-stop    │
                     │  PB3 ◄── DOCK_SENSE_B; PB4 ──► Q_KSU ──► K_SU; PB5 ──► Q_DOCK    │
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
│  ┌─── FDCAN2: Isolated bus to the stirring motor ───────────┐  │
│  │  PB12 (FDCAN2_RX, AF3) ◄── ISO1050DUB #3 RXD               │
│  │  PB13 (FDCAN2_TX, AF3) ──► ISO1050DUB #3 TXD               │
│  │  1 Mbps classic CAN (motor protocol); bus side in the     │
│  │  SU domain (5V_ISO2 / GND_SU), isolated from CS ground    │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Power switches ───────────────────────────────────────┐  │
│  │  PB4  (GPIO)      ──► K_SU enable (Q_KSU 2N7002 → G5V-2   │
│  │                       coil; pilot relay on the SU PSU AC) │
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
│  │  PB0  (GPIO)      ──► Heater relay K_SAFE (via Q_RLY)     │
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
| PB0 | GPIO | Heater safety relay | Output | Q_RLY (2N7002) → K_SAFE G5V-2 coil → J_SAFE 1 | Safety |
| PB1 | GPIO | Pot Detection (backup) | Input | Reed switch (10k pull-up) → J_SAFE | Safety |
| PB2 | GPIO (EXTI) | E-Stop | Input | NC button (10k pull-up, 100 nF) → J_SAFE | Safety |
| PB3 | GPIO (EXTI) | DOCK_SENSE_B | Input | J_DOCK pin 8 (10k pull-up; low = DU present) | Dock |
| PB4 | GPIO | K_SU enable | Output | Q_KSU (2N7002) → K_SU G5V-2 coil → J_SAFE 5 (SU PSU AC) | Stir |
| PB5 | GPIO | Q_DOCK enable | Output | 2N7002 → AO4407A (DOCK_24V) | Dock |
| PB6 | — | Available (I2C1_SCL pad, unpopulated) | — | — | — |
| PB7 | — | Available (I2C1_SDA pad, unpopulated) | — | — | — |
| PB8 | FDCAN1_RX | Isolated CAN RX | Input | ISO1050DUB #1 RXD | CAN (heater / DU) |
| PB9 | FDCAN1_TX | Isolated CAN TX | Output | ISO1050DUB #1 TXD | CAN (heater / DU) |
| PB10 | GPIO (SW PWM) | Exhaust Fan 2 PWM | Output | IRLML6344 gate → J_FAN | Exhaust |
| PB11 | — | Available | — | — | — |
| PB12 | FDCAN2_RX | Isolated CAN #2 RX | Input | ISO1050DUB #3 RXD | CAN (stir) |
| PB13 | FDCAN2_TX | Isolated CAN #2 TX | Output | ISO1050DUB #3 TXD | CAN (stir) |
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

Everything the STM32 used to drive over I2C (PCF8574, INA219, PCA9685) is gone from this unit (it is on the DU STM32 now), so every output is a direct GPIO and 26 pins stay free for the CS-side extras that may come later (fan tachometers, a second pot sensor, module interlocks).

---

## 4. Power Supply

### 4.1 12 V Input from the LRS-35-12 (J_12V_IN)

The CSC, both relay coils and the two exhaust fans run from **12V_IN**, a dedicated Mean Well LRS-35-12 (12 V, 3 A) in the CS rear. Neither 24 V PSU powers this board, and nothing is tapped from the heating unit: the OEM unit's internal wiring and its control-board supply stay untouched, so the pre-certified unit is used exactly as shipped apart from its CAN port and the K_SAFE relay on its inlet loom.

```
12V from LRS-35-12 (J_12V_IN) ──── F1 (2A Polyfuse) ──── D1 (SS34 Schottky) ────┬── D2 (SMBJ12A TVS) ── C_IN 100 µF ──── 12V Internal
                                                                                  │
                                                                                 GND (LRS-35-12 −V = board ground)
```

| Component | Part | Function |
|-----------|------|----------|
| F1 | 2A resettable polyfuse (1812) | Overcurrent protection |
| D1 | SS34 (3A, 40V Schottky) | Reverse polarity protection (0.4 V drop) |
| D2 | SMBJ12A (12V TVS, 600W) | Transient suppression |
| C_IN | 100 µF electrolytic (25 V) + 10 µF MLCC | Bulk for fan start-up and relay pull-in |

| Parameter | Value |
|-----------|-------|
| Supply | Mean Well LRS-35-12: 12 V, 3 A, 36 W, 99 × 82 × 30 mm; AC input on the CS inlet loom, unswitched (live whenever the appliance is on, so the CSC boots independently of K_SAFE and K_SU) |
| Load | Fans 1.0 A, two G5V-2 coils 0.08 A, the 5 V buck ≈ 0.25 A input: ~1.3 A typical, ~1.4 A peak (≈ 17 W); > 50 % headroom |
| Why a third PSU | A 12 V tap off the heating unit's control board (CSC v1.5) would have made the board depend on an uncharacterised rail (isolation, voltage, a few watts of spare capacity against 12 W of fans) and forced K_SAFE onto the element loom inside the certified unit. One LRS-35-12 removes all of that for about $12 |

12 V Internal feeds the fans, both relay coils and the 5 V buck. The CSC has no battery: when mains drops, all three PSUs, the board, the fans, the relays and the dock 24 V go down together, and the DU carries the session on its own pack.

### 4.2 24 V Dispensing-Unit Rail (J_DU_IN)

```
24V from LRS-75-24 #1 (J_DU_IN) ──── F_DU (3A Polyfuse) ──── D_DU (SS54 Schottky) ────┬── SMBJ24A ──── 24V_DU ──► Q_DOCK (§4.7)
                                                                                        │
                                                                                       GND (LRS #1 −V bonded here)
```

The LRS-75-24 #1 is the Dispensing Unit's supply and nothing else: 24V_DU goes only to Q_DOCK → F_DOCK → DOCK_24V. Its −V is bonded to the CSC ground at J_DU_IN because the dock's DOCK_SENSE_A/B lines and the charge return (pin 2) need the DU and CS grounds common while docked; it is the one supply in the base whose output is not floating. With the stir motor on LRS #2 and the CSC on its own LRS-35-12, LRS #1 carries ≈ 0.6 A typical / 1.6 A peak, so an LRS-50-24 (2.2 A) would do; LRS-75-24 is kept for a single part number.

### 4.3 12V → 5V Rail (MP1584EN)

| Parameter | Value |
|-----------|-------|
| Output | 5V_MAIN, 3 A max (loaded < 0.5 A); input 12V Internal |
| Load | AMS1117-3.3 (~0.2 A), B0505S #1 and #2 (~0.1 A in each), status LED |
| Inductor | 22 µH, CDRH104R |
| Output caps | 2x 22 µF MLCC + 100 µF electrolytic |
| Feedback | 52.3k / 10k → 4.98 V |
| Protection | 1 A polyfuse + SMBJ5.0A on output |

The TPS54531 and the VSYS diode-OR of the Unified PCB are not needed here: there is no battery and no Pico Zero on this board, so one MP1584EN is enough (the 24 → 12 V MP1584EN #1 of v1.0–1.4 is gone: the board takes 12 V directly).

### 4.4 Two-Source 5V Schottky-OR → 3.3V Rail (AMS1117-3.3)

```
5V_MAIN (MP1584EN, 12V_IN) ──────────────────── D_OR1 (SS14) ──┐
                                                                ├── 5V_MERGED ──► AMS1117-3.3 ──► 3.3V
5V_USB  (USB-C VBUS, 500 mA polyfuse) ───────── D_OR2 (SS14) ──┘
```

The USB path powers the STM32 (3.3 V) for bench programming without the LRS-35-12; it does not back-feed 5V_MAIN or the B0505S (D_OR1 blocks it), so both isolated domains, the relays and the fans stay dead on USB alone. 3.3 V loads: STM32 (~0.15 A), ISO1050 #1 and #3 VCC1 — ~0.2 A.

### 4.5 Isolated 5V DC-DC #1 (B0505S-1WR3 #1)

The isolated CAN domain of bus #1 (5V_ISO / GND_ISO) is generated once, here, for ISO1050 #1's bus side and, optionally, the heating unit's node (if it takes bus power on J_CAN_HEAT pin 4). Since Unified PCB v5.0 the DU node is a non-isolated SN65HVD230 that draws nothing from it: J_DOCK pin 6 stays exported through F_CAN but is reserved, and the DU ties GND_ISO (pin 5) to its own logic ground, which is the single ground reference of bus #1.

> [!note] Why the barrier is on this board only
> The DU ground meets the CS ground on J_DOCK pin 2 (24 V return), so port #1's ISO1050 is a **signal-reference / ground-loop barrier**, not a safety barrier: the bus has exactly one ground path (through the DU, pin 5) and none through the CSC logic ground. Mains safety isolation is provided by the LRS-75-24 and, for the heating unit, by its own control board — **selection requirement: the OEM board's CAN port must be SELV** (galvanically isolated inside the unit from its mains-side electronics).

| Parameter | Value |
|-----------|-------|
| IC | B0505S-1WR3 #1 (SIP-4), 1 W, 3 kVDC isolation |
| Input | 5V_MAIN (before D_OR1, so it is never fed from USB); C_ISO_IN 10 µF |
| Output | 5V_ISO, 200 mA max; C_ISO_OUT 10 µF; loads: ISO1050 #1 VCC2 (~40 mA), optional J_CAN_HEAT pin 4; J_DOCK pin 6 exported but unused by the DU since Unified PCB v5.0 |
| Fuse | F_CAN 100 mA polyfuse on the exported 5V_ISO (J_CAN_HEAT pin 4 and J_DOCK pin 6 share it) |

### 4.6 Stirring Unit Supply Domain (J_SU_IN, F_SU, K_SU, B0505S #2)

The Stirring Unit has its own PSU, a second Mean Well LRS-75-24 in the CS rear, and its 24 V output is a floating domain (24V_SU / GND_SU) that is never bonded to the CS ground. The rail enters the CSC on J_SU_IN, crosses the board on the **SU island** (its own slot, like the CAN island of §5) and leaves on J_STIR_PWR into the SU harness; the island exists only so the harness picks up fuse, TVS, the CAN pair and the 24 V pair as one bundle.

```
LRS-75-24 #2 (AC in through K_SU) ──► J_SU_IN 1 ──► F_SU (5 A polyfuse) ──┬── SMBJ24A (to GND_SU) ──► 24V_SU ──► J_STIR_PWR 1
                                     J_SU_IN 2 ──► GND_SU ───────────────┴── C_SU 100 µF (35 V) + 10 µF ────► J_STIR_PWR 2

5V_MAIN (before D_OR1) ──► B0505S-1WR3 #2 ──► 5V_ISO2 / GND_ISO2 ──► ISO1050DUB #3 VCC2 / GND2;  GND_ISO2 = GND_SU (0R link on the island)

12V ──► K_SU: Omron G5V-2 #2 coil ◄── Q_KSU: 2N7002 drain; gate ── 100R ── PB4, 10k pull-down; 1N4148 flyback
        K_SU contacts ──► J_SAFE 5 ──► pilot for the SU PSU's AC-side relay on the inlet loom (NC wiring: de-energised = SU PSU off)
```

| Parameter | Value |
|-----------|-------|
| Supply | Mean Well LRS-75-24 #2, 24 V, 3.2 A; output floating (the LRS-75 chassis is earthed, its −V is left unbonded) |
| Input | J_SU_IN (XT30): 24V_SU, GND_SU; F_SU 5 A polyfuse; SMBJ24A to GND_SU; C_SU 100 µF + 10 µF |
| Domain | 24V_SU, GND_SU, 5V_ISO2 on the SU island; no copper to GND, 24 V Internal or GND_ISO |
| Transceiver supply | B0505S #2 (SIP-4, 1 W, 3 kVDC), input 5V_MAIN before D_OR1, output 5V_ISO2 referenced to GND_SU; load ISO1050 #3 VCC2 (~40 mA) only. The CSC side of bus #2 therefore stays alive with the SU PSU off, so a silent bus means the motor is down, not this transceiver |
| Power cut | K_SU (G5V-2 #2, 12 V coil, Q_KSU from PB4) on the SU PSU's AC input. Default OFF (gate pull-down): the SU is unpowered while the STM32 is in reset or unpowered, at boot until the Safety Monitor enables it, and on E-stop, fault and DU_UNDOCKED. Relay drop-out plus the LRS-75 hold-up (≈ 16 ms at full load) de-energises the motor within ~50 ms |
| Start-up | After K_SU closes the LRS-75 needs ≈ 300 ms to come up; the Motor Control task waits for the motor's boot frame before commanding |

K_SU replaces the 24 V high-side switch Q_STIR of the single-board design: a switch in the floating rail would need an isolated gate drive, while a relay on the PSU's AC input needs nothing new and keeps the command-independent stop the Safety Monitor requires (motor firmware timeout → stir disable frame → K_SU, in that order).

### 4.7 Dock 24 V Switch (Q_DOCK)

```
24V_DU (J_DU_IN, §4.2) ──► Q_DOCK: AO4407A (P-MOSFET, SO-8) ──► F_DOCK (2 A polyfuse) ──► DOCK_24V ──► J_DOCK pin 1
                    Gate ── 10k to 24V_DU (default OFF)
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
| 12V_IN input | 2 A (F1) | SMBJ12A | Reverse-polarity SS34; 100 µF bulk; from the LRS-35-12 |
| 24V_DU input | 3 A (F_DU) | SMBJ24A | Reverse-polarity SS54; LRS-75-24 #1, feeds Q_DOCK only |
| 24V_SU | 5 A (F_SU) | SMBJ24A (to GND_SU) | Floating SU domain from J_SU_IN; AC side cut by K_SU |
| DOCK_24V | 2 A (F_DOCK) | SMBJ24A at J_DOCK | Behind Q_DOCK; only on while DOCK_SENSE_B is low |
| 5V_MAIN | 1 A | SMBJ5.0A | LDO, B0505S #1 and #2 |
| 5V_USB | 500 mA (F_USB) | USBLC6 on data | Bench only |
| 5V_ISO | 100 mA (F_CAN) | — | Exported on J_CAN_HEAT 4 and J_DOCK 6 (reserved; the DU draws nothing since Unified PCB v5.0) |
| 5V_ISO2 | — | — | On-board only (ISO1050 #3 bus side), referenced to GND_SU |

### 4.9 Decoupling

- 100 nF MLCC within 5 mm of every STM32 VDD/VSS pair; 1 µF + ferrite bead on VDDA
- 10 µF at LDO input and output; 100 nF + 10 µF at each IC supply pin (ISO1050 #1 and #3 both sides, B0505S #1 and #2 input and output)
- 10 µF + 100 nF on DOCK_24V and 5V_ISO at J_DOCK; 100 µF + 10 µF on 24V_SU at J_STIR_PWR

### 4.10 Power Budget Summary

| Rail | Source | Typical (A) | Peak (A) | Notes |
|------|--------|-------------|----------|-------|
| 3.3V (0.8 A max) | 5V_MERGED → AMS1117 | 0.18 | 0.25 | STM32, ISO1050 #1 and #3 logic sides |
| 5V_MAIN (3 A max) | 12V_IN → MP1584EN | 0.4 | 0.5 | LDO + B0505S #1 + B0505S #2 |
| 5V_ISO (200 mA max) | 5V_MAIN → B0505S #1 | 0.04 | 0.12 | ISO1050 #1 bus side, optional module node (the DU node is non-isolated and draws nothing) |
| 5V_ISO2 (200 mA max) | 5V_MAIN → B0505S #2 | 0.04 | 0.05 | ISO1050 #3 bus side (SU domain) |
| **12V_IN input** | **Mean Well LRS-35-12 (3 A)** | **~1.3** | **~1.4** | Two fans (1.0 A) + two relay coils (0.08 A) + the 5 V buck (≈ 0.25 A in); ≈ 17 W of 36 W |
| 24V_SU (SU domain) | LRS-75-24 #2 via J_SU_IN | 1.0 | 3.0 | CAN servo motor; separate PSU, not part of the DU PSU budget |
| DOCK_24V | 24V_DU via Q_DOCK | 0.6 | 1.6 | DU charger (1 A at 8.4 V) + DU logic, + a DU servo move |
| **24V_DU input (DU PSU)** | **Mean Well LRS-75-24 #1 (3.2 A)** | **~0.6** | **~1.6** | DU charge + DU dispense on the dock only; nothing on the CSC draws from it |

> [!note]
> Each supply in the base has one customer: LRS-35-12 → the CSC, fans and relays; LRS-75-24 #1 → the DU through the dock; LRS-75-24 #2 → the SU. LRS #1 could drop to an LRS-50-24 (2.2 A against a 1.6 A peak) but stays LRS-75-24 for a single part number. The stir velocity limit during a DSB move remains a torque/noise rule on the RV1106 side, not a power rule; the CSC's Safety Monitor still caps stir torque when the motor's reported current exceeds its limit.

---

## 5. Isolated CAN Port #1 (FDCAN1)

The isolated 500 kbps bus is the system's host link and heater link in one: the STM32 (here), the heating unit (J_CAN_HEAT) and the DU STM32 (through the dock, gateway for the RV1106) are its three nodes. Only this end of the bus is isolated: the DU node is a plain SN65HVD230 and the DU ties GND_ISO to its ground at the dock, so the isolated domain is referenced at the far node and never here (§4.5).

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
| Transceiver | ISO1050DUB #1, 5 kV RMS; logic side 3.3 V / GND, bus side 5V_ISO / GND_ISO from the on-board B0505S #1 |
| Nodes | STM32 (src_node 2, "STIR"), DU STM32 ("AP", FDCAN1 + non-isolated SN65HVD230 on the Unified PCB v5.0, transparent gateway for the RV1106 over the UART host link), heating unit MCU ("HEAT") |
| Bit rate | 500 kbps, classic CAN 2.0B, 11-bit IDs |
| Termination | R_TERM1 120 Ω here at J_CAN_HEAT and 120 Ω inside the heating unit. The dock → Unified PCB run is a stub ≤ 30 cm; the DU board has no termination |
| Ground | GND_ISO only on the island here; the cable shield to the heating unit lands on GND_ISO at the PCB end. The domain's single reference to a real ground is on the DU board (J_DOCK pin 5 → DU ground); with the DU lifted it floats, which is harmless for the two remaining nodes (ISO1050 #1 and the heater's SELV port) |
| Traffic | Host commands/telemetry (RV1106 ↔ STM32), heater setpoint and status, heartbeat, E_STOP, STM32 firmware update (0x7E0/0x7E8) — ID map in [[../03-Software/02-Controller-Software-Architecture#7.2 Message Types\|Controller Software Architecture §7.2]] |

- The transceiver sits on the system-ground side of the slot with its bus pins crossing into the isolated island; the B0505S output crosses it too; no other copper crosses the boundary (creepage ≥ 6.4 mm, IEC 60335 reinforced).
- Decoupling: 100 nF on VCC1, 100 nF + 10 µF on VCC2.
- Shielded twisted pair ≤ 1 m to the heating unit; the dock harness to the pillar block ≤ 30 cm.
- Lifting the DU removes the DU STM32 (AP) node from the bus: the STM32 keeps the heater off on keep-alive loss (≤ 1 s) and holds safe on DOCK_SENSE_B (§7.2); the bus itself stays terminated and keeps working between the STM32 and the heating unit.

---

## 6. Isolated CAN Port #2 (FDCAN2) and Stirring-Unit Harness

The stir bus is the second isolated port of the board. Its bus side lives in the SU domain (5V_ISO2 / GND_SU, §4.6), so the only galvanic path between the Cooking Surface and the Stirring Unit is the two ISO1050 barriers and the two B0505S transformers: the SU's own PSU, the motor and the CAN pair share GND_SU, and GND_SU never meets the CS ground.

```
PB13 (FDCAN2_TX) ──► TXD ┐  ISO1050DUB #3  ┌── CANH ──┬── R_TERM2 (120R) ──┬── J_CAN_STIR 1 (CANH)
PB12 (FDCAN2_RX) ◄── RXD ┘  VCC1 3.3V / GND │  CANL ───┤   PESD1CAN          │   J_CAN_STIR 2 (CANL)
                            VCC2 5V_ISO2 / GND_SU     │                     └── J_CAN_STIR 3 (GND_SU)
24V_SU (J_SU_IN → F_SU, §4.6) ──► J_STIR_PWR 1; GND_SU ──► J_STIR_PWR 2
                  ─── 6 mm milled slot under ISO1050 #3 and the B0505S #2 output; J_SU_IN, J_CAN_STIR and J_STIR_PWR on the SU island ───
```

| Parameter | Value |
|-----------|-------|
| Transceiver | ISO1050DUB #3, 5 kV RMS, up to 1 Mbps; logic side 3.3 V / GND, bus side 5V_ISO2 / GND_SU from the on-board B0505S #2 |
| Bit rate | 1 Mbps (motor protocol default), 29-bit IDs as the motor requires; FDCAN2 bit timing from the 64 MHz kernel clock |
| Termination | 120 Ω at J_CAN_STIR; 120 Ω in or at the motor (verify the motor's built-in termination) |
| Ground | GND_SU only on the SU island; the SU PSU's −V, the motor's CAN transceiver and the harness shield all reference it |
| Harness | J_CAN_STIR and J_STIR_PWR are bundled into one 5-way harness that runs inside the right rear pillar to **J_SU** (Molex Micro-Fit 3.0, 5-way: 24V_SU, GND_SU, CAN_H, CAN_L, SHIELD) at the pillar top, where the Stirring Unit plugs in. ≤ 1 m, twisted CAN pair with the 24 V pair, shield to GND_SU at the CSC end |
| Fail-safe | Motor firmware command timeout (≤ 500 ms) must be configured so a silent bus stops the arm; the stir disable frame is the second layer; K_SU removes the SU PSU's AC independently of both |

The two CAN buses never share a transceiver, a termination, a DC-DC or a connector; the two islands sit on opposite board edges and the two buses are never routed adjacent. ISO1050 #3 is specified with the same part as #1 so one footprint, one DC-DC and one slot rule cover both ports.

---

## 7. Dock Interface (J_DOCK)

### 7.1 Pinout (CS view)

J_DOCK is an 8-pin JST-XH header; a short harness carries it to the spring-pin half of the magnetic pogo block in a keyed recess on top of the CS left pillar. The DU's flat gold pads (Unified PCB J_DOCK) meet it when the DU is seated.

| Pin | Signal | Direction | On the CSC |
|-----|--------|-----------|------------|
| 1 | DOCK_24V | CS → DU | 24V_DU (LRS-75-24 #1 via J_DU_IN) → Q_DOCK (AO4407A, PB5 via 2N7002) → F_DOCK 2 A; SMBJ24A + 10 µF at the header. Live only while DOCK_SENSE_B is low |
| 2 | GND | — | System GND; charge return. LRS #1's −V is bonded to the CSC ground at J_DU_IN, so this pin ties the DU logic ground to the CS ground while docked |
| 3 | CAN_H | bus | Isolated bus, from the island (ISO1050 #1 bus side) |
| 4 | CAN_L | bus | |
| 5 | GND_ISO | — | Isolated bus ground; the DU ties it to its own logic ground (the bus's single ground reference) |
| 6 | 5V_ISO | CS → DU | B0505S #1 output through F_CAN 100 mA (shared with J_CAN_HEAT pin 4); reserved — unused since Unified PCB v5.0 (the DU node is non-isolated) |
| 7 | DOCK_SENSE_A | CS → DU | Tied to GND here (0R link); the DU pulls it up and reads low = docked |
| 8 | DOCK_SENSE_B | DU → CS | 10k pull-up to 3.3 V here → STM32 PB3 (EXTI), 100 nF; the DU ties it to GND; low = DU present |

Pins 1–2 and 7–8 are on the system-ground side; pins 3–6 are on the isolated island. The header is placed on the island boundary so the harness carries both domains without copper crossing the slot on the PCB; the harness itself keeps the two pairs apart (two 4-way sub-looms).

### 7.2 DOCK_SENSE_B Interlock

| Event | STM32 response |
|-------|----------------|
| DOCK_SENSE_B low (DU seated), HB_AP present | NORMAL: Q_DOCK closed (DOCK_24V live), bus traffic with the RV1106 |
| DOCK_SENSE_B goes high (DU lifted) | Immediate safe hold (EXTI, ≤ 10 ms): HEAT_ON_OFF = 0 to the heating unit, stir disable frame on FDCAN2, K_SU off (SU PSU de-energised), Q_DOCK off, STATUS (0x312) with fault code 0x17 DU_UNDOCKED (Info). Not an E_STOP: the relay stays closed, fans run per the thermal task |
| DOCK_SENSE_B low again and HB_AP returns | Back to NORMAL; Q_DOCK closes after a 100 ms debounce so the pogo contacts are seated before 24 V appears |
| DOCK_SENSE_B low but no HB_AP for 5 s | Existing heartbeat rule: safe state, Q_DOCK stays closed (the DU may be booting or charging) |

The heater also stops by itself ≤ 1 s after the RV1106's last keep-alive once the DU is off the bus, so the DU_UNDOCKED hold is a belt-and-braces measure, not the only one. Q_DOCK default is OFF (gate pull-up), so the pads are dead while the STM32 is in reset.

---

## 8. Safety and Auxiliary I/O

### 8.1 Relay Drivers (Q_RLY, Q_KSU)

**K_SAFE (heater):** PB0 → 330R → 2N7002 (Q_RLY) → Omron G5V-2 coil (12 V) with 1N4148 flyback; the contacts pilot the power relay on the heating unit's AC inlet loom through J_SAFE pin 1 (the AC wiring itself stays off the PCB and outside the OEM unit, which is not opened). The CSC runs from its own LRS-35-12 (§4.1), so cutting the unit's inlet never affects the controller. NC wiring: relay de-energised = heating unit disconnected.

**K_SU (SU PSU):** PB4 → 100R → 2N7002 (Q_KSU) → Omron G5V-2 #2 coil (12 V) with 1N4148 flyback; the contacts pilot the SU PSU's AC-side relay through J_SAFE pin 5, same scheme and same loom as the heater's. NC wiring: relay de-energised = LRS-75-24 #2 off = Stirring Unit unpowered (§4.6).

### 8.2 E-Stop Input

The hard E-stop button is on the CS front panel, wired to J_SAFE pin 3 → PB2 (EXTI) with 10k pull-up and 100 nF; pressed or wire-broken = LOW = stop: K_SAFE off, HEAT_ON_OFF = 0 and E_STOP (0x001) on FDCAN1, motor disable on FDCAN2, K_SU off, all within 100 ms. The DU's UI has a soft stop that broadcasts E_STOP over CAN while docked; the STM32 treats both the same.

### 8.3 Pot Detection Input (backup)

Reed switch on J_SAFE pin 2 → PB1 with 10k pull-up. Primary pot detection is the heating unit's own (spring-loaded centre sensor pressed by the pot), reported as `pot_detected` in HEAT_STATUS; the reed switch is an optional cross-check read by the Thermal/Fan task and reported in the TELEMETRY `pot_detected` flag.

### 8.4 Exhaust Fans (2x)

Two 120 mm 12 V fans at the CS rear behind the filter cartridge, low-side PWM via IRLML6344 + SS14 flyback: fan 1 on PA6 (TIM3_CH1, 25 kHz), fan 2 on PB10 (software PWM from the Thermal/Fan task at 1 Hz duty updates). Both independently controllable; stage and duty come from the RV1106 over CAN (SET_EXHAUST 0x260, GET_EXHAUST 0x261, EXHAUST_AUTO 0x262). Fan tachometers are optional (free pins PB14/PB15 reserved).

### 8.5 Status LED

PC7 → 330R → green LED, active-low: boot, NORMAL heartbeat blink, safe-hold pattern.

### 8.6 MOSFET Gate Safety

Every MOSFET gate and driver enable has a 100R series resistor and a 10k pull-down (or pull-up for the P-MOSFET switch Q_DOCK) so that both fans, Q_DOCK and both relays (K_SAFE, K_SU) are off while the STM32 is in reset or unpowered.

---

## 9. Programming and Debug

| Path | Interface | Use |
|------|-----------|-----|
| **CAN bootloader** (production) | FDCAN1, IDs 0x7E0 FW_UPDATE_REQ (AP → STIR: seq, op ERASE/WRITE/VERIFY/BOOT, 5 data bytes) and 0x7E8 FW_UPDATE_RSP (STIR → AP: seq, op, result, 4 data bytes) | The RV1106's swupdate handler streams the STM32 image over the isolated bus to an in-application, OpenBLT-style bootloader in the first flash sector. Update mode only, lowest priority; the application is only entered after VERIFY passes. There is no UART or BOOT0/NRST line from the RV1106 any more |
| **USB-C DFU / CDC** (bench) | J_USB, PA11/PA12, SW_BOOT on BOOT0 (PA14 on the G0; nBOOT_SEL cleared so the pin is sampled), SW_RST | ST system bootloader DFU with J_12V_IN disconnected (USB powers the 3.3 V domain only, §4.4); CDC console once the application runs |
| **SWD** (factory, debug) | J_SWD 2x5 1.27 mm: VCC 3.3 V, SWDIO (PA13), SWCLK (PA14), SWO (n/a on the M0+, pin left for the standard pinout), NRST, GND | Initial bootloader flash, debugger |
| **Debug console** | J_DBG 3-pin (GND, TX PA2, RX PA3), 3.3 V, 115200 8N1 | FreeRTOS trace, Class B self-test log; no host link |

The USB-C port is 16-pin mid-mount with 5.1k CC pull-downs, USBLC6-2SC6 ESD, 22R series on D+/D−, 500 mA VBUS polyfuse → 5V_USB → D_OR2, VBUS detect divider → PA0.

---

## 10. Connector Definitions (12 total)

| Ref | Type | Function |
|-----|------|----------|
| J_12V_IN | XT30 2-pin | 12 V DC from the LRS-35-12: powers the CSC, fans and relay coils |
| J_DU_IN | XT30 2-pin | 24 V DC from the LRS-75-24 #1 (DU PSU), to Q_DOCK only |
| J_SU_IN | XT30 2-pin | Floating 24 V DC input from the LRS-75-24 #2 (SU domain) |
| J_CAN_HEAT | JST-XH 2.5 mm 4-pin | Isolated CAN to the heating unit |
| J_CAN_STIR | JST-XH 2.5 mm 3-pin | Isolated CAN #2 to the stirring motor (SU harness) |
| J_STIR_PWR | XT30 2-pin | SU 24 V pass-through to the stirring motor (SU harness) |
| J_DOCK | JST-XH 2.5 mm 8-pin | Harness to the pillar pogo block: DOCK_24V, GND, isolated CAN, 5V_ISO, sense |
| J_FAN | JST-XH 2.5 mm 4-pin | Two exhaust fans |
| J_SAFE | Hirose DF11-6DP-2DS (2x3) | Relay drives (heater, SU PSU), pot detect, E-stop, GND |
| J_DBG | 1x3 header, 2.54 mm | Debug console (USART2) |
| J_SWD | 2x5 1.27 mm | Cortex debug |
| J_USB | USB-C 16-pin mid-mount | STM32 DFU / CDC (bench) |

### 10.1 J_12V_IN — 12 V Board Supply (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 12V_IN | From Mean Well LRS-35-12 (§4.1); F1 2 A, SS34, SMBJ12A, 100 µF |
| 2 | GND | LRS-35-12 −V = board ground |

### 10.1a J_DU_IN — 24 V DC for the Dispensing Unit (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 24V_DU | From Mean Well LRS-75-24 #1; F_DU 3 A, SS54, SMBJ24A; feeds Q_DOCK only |
| 2 | GND | LRS #1 −V, bonded to the board ground here |

### 10.2 J_CAN_HEAT — Isolated CAN to the Heating Unit (JST-XH 2.5 mm, 4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | CAN_H | Isolated bus, R_TERM1 120 Ω here |
| 2 | CAN_L | |
| 3 | GND_ISO | Isolated ground (NOT system GND); cable shield lands here |
| 4 | 5V_ISO | Optional isolated 5 V for the heating unit's node, F_CAN 100 mA polyfuse (shared with J_DOCK pin 6) |

### 10.3 J_CAN_STIR — Isolated CAN #2 to the Stirring Motor (JST-XH 2.5 mm, 3-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | CAN_H | Isolated (SU domain), 1 Mbps, R_TERM2 120 Ω |
| 2 | CAN_L | |
| 3 | GND_SU | SU ground reference for the motor's CAN transceiver (NOT system GND) |

### 10.4 J_SU_IN — Stirring Unit 24 V Input (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 24V_SU | From Mean Well LRS-75-24 #2 (AC input through K_SU); F_SU 5 A, SMBJ24A on the island |
| 2 | GND_SU | SU ground; floating with respect to the CS GND |

### 10.5 J_STIR_PWR — Stirring Motor Power (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 24V_SU | Pass-through from J_SU_IN behind F_SU |
| 2 | GND_SU | SU ground |

J_CAN_STIR + J_STIR_PWR → one harness inside the right pillar → **J_SU** (Molex Micro-Fit 3.0, 5-way) at the pillar top:

| J_SU pin | Signal | From |
|----------|--------|------|
| 1 | 24V_SU | J_STIR_PWR 1 |
| 2 | GND_SU | J_STIR_PWR 2 |
| 3 | CAN_H | J_CAN_STIR 1 |
| 4 | CAN_L | J_CAN_STIR 2 |
| 5 | SHIELD | J_CAN_STIR 3 (GND_SU at the CSC end only) |

Nothing on this harness references the CS ground; the SU's chassis, if metallic, is bonded to protective earth through the pillar, not to GND_SU.

### 10.6 J_DOCK — see §7.1

### 10.7 J_FAN — Exhaust Fans (JST-XH 4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 12V_FAN1 | Fan 1 supply |
| 2 | FAN1_OUT | Fan 1 drain (PA6 PWM) |
| 3 | 12V_FAN2 | Fan 2 supply |
| 4 | FAN2_OUT | Fan 2 drain (PB10 PWM) |

### 10.8 J_SAFE — Safety I/O (Hirose DF11-6DP-2DS, 2x3)

| Pin | Signal | STM32 Pin | Notes |
|-----|--------|-----------|-------|
| 1 | RELAY_DRV | PB0 via Q_RLY | Heater safety relay K_SAFE (12 V coil) |
| 2 | POT_DET | PB1 | Reed switch, 10k pull-up (backup) |
| 3 | E_STOP | PB2 | NC button on the CS front, 10k pull-up, RC debounce |
| 4 | GND | GND | |
| 5 | SU_RELAY_DRV | PB4 via Q_KSU | SU PSU pilot relay K_SU (12 V coil) |
| 6 | GND | GND | Return for pin 5 |

### 10.9 J_DBG — Debug Console (1x3, 2.54 mm)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | GND | |
| 2 | TX | PA2 (USART2_TX), 3.3 V |
| 3 | RX | PA3 (USART2_RX), 3.3 V, 10k pull-up |

### 10.10 J_SWD — SWD Debug (2x5, 1.27 mm)

Standard Cortex debug pinout: VCC 3.3 V, SWDIO (PA13), SWCLK (PA14), SWO (unused), NRST, GND.

### 10.11 J_USB — USB-C 2.0 (16-pin mid-mount)

VBUS → F_USB → 5V_USB; CC1/CC2 5.1k to GND; D+/D− → USBLC6 → 22R → PA12/PA11. SuperSpeed and SBU pins unconnected.

---

## 11. 4-Layer Stackup and Zone Layout

### 11.1 Layer Stack

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (only breaks: the two CAN isolation slots; GND_ISO and GND_SU pours on their islands) | 1 oz |
| L3 | Split power plane: 3.3 V / 5 V, 12 V, 24 V | 1 oz |
| L4 (Bottom) | Signal + power traces | 2 oz |

FR-4 Tg ≥ 150 °C, 1.6 mm, ENIG. Same stack and net classes as the Unified PCB ([[03-PCB-Design-Rules|PCB Design Rules]]).

### 11.2 Board Layout Zones

```
┌─────────────────────────── ~100 mm ───────────────────────────┐
│ ZONE A: MCU + DEBUG            │ ZONE C: POWER IN + SWITCHES   │
│ STM32G0B1RET6, Y1 (8 MHz)      │ J_12V_IN F1 SS34 TVS C_IN     │
│ AMS1117, D_OR1-2               │ J_DU_IN F_DU SS54 TVS; MP1584EN│
│ J_SWD  J_USB  J_DBG            │ Q_DOCK + F_DOCK                │ ~70 mm
│ SW_BOOT  SW_RST  status LED    │ Q_RLY + K_SAFE, Q_KSU + K_SU   │
├────────────────────────────────┼───────────────────────────────┤
│ ZONE B: ISOLATED CAN #2 (SU)   │ ZONE D: ISOLATED CAN #1        │
│ ┌── 6 mm slot ───────────────┐ │ ┌── 6 mm slot ───────────────┐│
│ │B0505S #2│ISO1050 #3│SU isl.│ │ │ B0505S #1│ISO1050 #1│island ││
│ └── J_SU_IN J_CAN_STIR J_STIR_PWR ┘ └────────── J_CAN_HEAT  J_DOCK ┘│
│ (SU harness exit, right)       │ (dock harness exit, left)      │
│ fan MOSFETs  J_FAN  J_SAFE     │                                │
└────────────────────────────────┴───────────────────────────────┘
```

Placement rationale:
- **Zone D** at the left edge, nearest the left pillar: J_DOCK straddles the island boundary (pins 1–2, 7–8 outside, 3–6 inside); J_CAN_HEAT is on the island; the slot runs under ISO1050 #1 and the B0505S #1 output.
- **Zone B** at the right edge, nearest the right pillar: a second slot under ISO1050 #3 and the B0505S #2 output; J_SU_IN, J_CAN_STIR and J_STIR_PWR all on the SU island so the SU harness leaves the board as one bundle and the 24V_SU / GND_SU copper never touches the GND side. The two islands are on opposite edges, ≥ 40 mm apart, never routed parallel.
- **Zone C**: both input chains (12V_IN, 24V_DU), the 5 V buck, Q_DOCK and both relays together; each relay with its 12 V coil drive here, its contact side wired off-board on the heating unit's AC loom (≥ 2.5 mm clearance around the contact pins, IEC 60335-1).
- **Zone A**: the STM32 ≥ 25 mm from the bucks; USB pair < 30 mm to J_USB.

### 11.3 Critical Layout Rules

| Rule | Specification | Rationale |
|------|---------------|-----------|
| CAN isolation #1 | Creepage ≥ 6.4 mm GND ↔ GND_ISO; 6 mm milled slot under ISO1050 #1 and the B0505S #1 output; no copper crossing; J_DOCK pins 2 ↔ 3 and 6 ↔ 7 spaced for ≥ 6.4 mm (skip a position or use two 4-pin headers if the pitch cannot) | IEC 60335 reinforced insulation, carried across the dock |
| CAN isolation #2 (SU) | Same rule: creepage ≥ 6.4 mm GND ↔ GND_SU, 6 mm slot under ISO1050 #3 and the B0505S #2 output, no copper crossing; J_SU_IN, J_CAN_STIR and J_STIR_PWR entirely on the SU island | Both sides are SELV so functional separation would do; one DRC rule for both islands keeps the SU ground genuinely floating |
| Two CAN buses | Stub lengths < 30 mm, 120 Ω at each connector, never route the two buses adjacent; islands on opposite edges | Signal integrity, no cross-talk |
| USB D+/D− | 90 Ω differential, matched ± 0.5 mm, < 30 mm | USB 2.0 FS |
| 24V_SU | ≥ 2 mm, F_SU pads with ≥ 200 mm² pour, on the SU island only | 3 A motor peak |
| DOCK_24V | ≥ 1.5 mm, Q_DOCK with ≥ 150 mm² pour | 2 A fused |
| Buck loops | Input cap → IC → inductor → output cap loop < 10 mm | EMI |
| Thermal vias | ≥ 9 under the STM32 pad, ≥ 6 under the buck IC | Heat to L2 |
| Mains relay | ≥ 2.5 mm clearance around relay contact pins; contact side on L1 only | IEC 60335-1 |

---

## 12. Protection Circuits

- **12V_IN input**: polyfuse → Schottky → TVS → 100 µF (§4.1); **24V_DU input**: polyfuse → Schottky → TVS (§4.2)
- **ESD**: PRTR5V0U2X on the J_SAFE inputs and DOCK_SENSE_B; PESD1CAN on the isolated sides of J_CAN_STIR and of J_CAN_HEAT / J_DOCK 3–4; USBLC6-2SC6 on USB D+/D−
- **Dock**: SMBJ24A + 10 µF on DOCK_24V at J_DOCK; F_DOCK 2 A; Q_DOCK default OFF and interlocked on DOCK_SENSE_B so the pogo contacts never make or break under load
- **Isolated domains**: F_CAN 100 mA on the exported 5V_ISO; both B0505S inputs taken before D_OR1 so USB never feeds an island
- **SU domain**: F_SU 5 A and SMBJ24A (to GND_SU) on 24V_SU; K_SU default OFF so the SU PSU is dead while the STM32 is in reset
- **MOSFET gates**: 100R series + 10k pull-down / pull-up on every gate (§8.6)
- **Relays**: 1N4148 flyback on both; NC wiring so loss of coil power disconnects the heating unit (K_SAFE, on its inlet loom) and the SU PSU (K_SU)
- **Watchdog**: IWDG kicked by the Safety Monitor; X-CUBE-CLASSB self-tests at boot and periodically (CPU, flash CRC, RAM march, clock)

---

## 13. Thermal Analysis

| Component | Condition | Power (W) | Notes |
|-----------|-----------|-----------|-------|
| MP1584EN (12 → 5 V) | 0.4 A | ~0.25 | |
| AMS1117-3.3 | 0.2 A | ~0.35 | |
| B0505S #1 and #2, ISO1050 #1 and #3 | — | ~0.6 | |
| SS34 (D1, 1.4 A), SS54 (D_DU, 1.6 A), Q_DOCK, Q_RLY, Q_KSU, fan MOSFETs | 1.4 A / 1.6 A / 1.6 A / coils / 0.5 A each | ~1.3 | Diode drops dominate |
| G5V-2 coils (K_SAFE, K_SU) | 12 V, 40 mA each | ~1.0 | Energised while the heater / the SU are allowed |
| **Total** | | **~3.5 W typical** | In the CS rear airflow of the exhaust fans; every IC below 110 °C junction with the pours in §11.3 |

---

## 14. Firmware Integration Notes

### 14.1 FreeRTOS Task Allocation

| Task | Rate | Priority | Functions |
|------|------|----------|-----------|
| Motor Control | 50 Hz | Highest | FDCAN2: velocity/position commands to the stir motor, feedback parsing (rpm, current, temperature), stir patterns, ramps, stall detection from motor current |
| Safety Monitor | 10 Hz | Highest | HEAT_STATUS limits (surface/element over-temperature), E-stop, HB_AP heartbeat (safe state after 5 s), DOCK_SENSE_B hold, K_SAFE / K_SU / Q_DOCK authority, IWDG kick |
| CAN Comms (FDCAN1) | 20 Hz | High | Host commands, ACK/NAK, TELEMETRY / STATUS, HEAT_STATUS capture, heating unit 1 s keep-alive, FW_UPDATE handling into the bootloader |
| Thermal / Fan | 1 Hz | Medium | Fan duty (PA6 / PB10), optional tachometers, pot reed, exhaust stage (0x260–0x262) |

The Dispense and Sensor Poll tasks of the single-board design are gone: dispensing, load cells, battery and charger monitoring run on the RV1106.

### 14.2 STM32 Peripheral Allocation

| Peripheral | Pins | Function |
|------------|------|----------|
| FDCAN1 | PB8/PB9 | Isolated bus (heating unit + DU), 500 kbps, 11-bit IDs |
| FDCAN2 | PB12/PB13 | Isolated stir motor bus (SU domain), 1 Mbps, 29-bit IDs |
| USART2 | PA2/PA3 | Debug console (J_DBG) |
| TIM3_CH1 | PA6 | Fan 1, 25 kHz |
| GPIO | PB10 | Fan 2, software PWM |
| GPIO | PB0, PB1, PB2 | Relay, pot reed, E-stop (EXTI) |
| GPIO | PB3, PB4, PB5 | DOCK_SENSE_B (EXTI), K_SU, Q_DOCK |
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
- [ ] Two separate CAN buses: no shared termination or DC-DC, FDCAN1 only on ISO1050 #1, FDCAN2 only on ISO1050 #3
- [ ] Isolation #1: only the ISO1050 #1 bus pins and the B0505S #1 output cross the slot; J_CAN_HEAT and J_DOCK 3–6 on the island
- [ ] Isolation #2: only the ISO1050 #3 bus pins and the B0505S #2 output cross the SU slot; J_SU_IN, J_CAN_STIR, J_STIR_PWR on the SU island; GND_ISO2 linked to GND_SU on the island only; no net joins GND_SU to GND
- [ ] Q_DOCK default OFF (gate pull-up) from PB5; K_SAFE and K_SU default OFF (2N7002 gate pull-downs) from PB0 / PB4; F_SU 5 A, F_DOCK 2 A
- [ ] DOCK_SENSE_B 10k pull-up + 100 nF on PB3; DOCK_SENSE_A tied to GND at J_DOCK 7; 5V_ISO to J_DOCK 6 through F_CAN
- [ ] All MOSFET gates: 100R series + 10k pull-down (or pull-up for P-MOSFETs)
- [ ] Both input chains: polyfuse → Schottky → TVS (12V_IN with 100 µF bulk; 24V_DU to Q_DOCK only); B0505S fed before D_OR1; no 24 → 12 V buck on the board
- [ ] K_SAFE pilots the heating unit's AC inlet loom; nothing is tapped from inside the OEM unit
- [ ] USB: 22R, USBLC6, 5.1k CC, VBUS divider on PA0, SW_BOOT on BOOT0/PA14 with 10k pull-down, nBOOT_SEL option documented
- [ ] All 12 connectors have correct pinouts and silkscreen; J_SU harness drawing matches §10.5; J_SAFE is the 2x3 DF11

### 15.2 PCB Layout

- [ ] DRC passes; GND plane continuous except the two CAN slots
- [ ] Creepage ≥ 6.4 mm across both isolation slots and across J_DOCK (2 ↔ 3, 6 ↔ 7), ≥ 2.5 mm around both relays' contact pins
- [ ] USB pair 90 Ω, < 30 mm
- [ ] 24V_SU and DOCK_24V trace widths and pours per §11.3; 12V_IN ≥ 1 mm to the fan MOSFETs
- [ ] Buck loops < 10 mm; thermal vias and pours present
- [ ] J_CAN_HEAT and J_DOCK the only connectors touching island #1; J_SU_IN, J_CAN_STIR and J_STIR_PWR the only ones touching the SU island; the islands ≥ 40 mm apart

### 15.3 Post-Assembly

- [ ] 3.3 V, 5 V rails within ±3 % and 12V Internal ≥ 10.8 V with both fans at full duty and both relays energised; 5V_ISO within ±5 % with ISO1050 #1's bus side and the heater node loaded (the DU draws nothing from it); 5V_ISO2 within ±5 %; > 10 MΩ between GND and GND_SU with both PSUs disconnected
- [ ] STM32 responds on SWD; DFU over USB-C with J_12V_IN disconnected; console on J_DBG
- [ ] FDCAN1 at 500 kbps: HEAT_QUERY to the heating unit returns HEAT_STATUS (surface_temp, element_temp, pot_detected); frames reach the DU STM32 (and the RV1106 behind its host link) through the bench dock
- [ ] FDCAN2 at 1 Mbps through ISO1050 #3: motor enumerates through J_SU, velocity command spins the arm, feedback frames arrive at 50 Hz
- [ ] K_SU off at reset: LRS-75-24 #2 unpowered and 0 V on J_STIR_PWR until PB4 is set; motor boot frame seen ≈ 300 ms after K_SU closes; Q_DOCK off at reset and with DOCK_SENSE_B open: 0 V on J_DOCK 1
- [ ] Seat the DU (DOCK_SENSE_B low): Q_DOCK closes after the debounce, DOCK_24V present, TELEMETRY `docked` set; lift it: hold within 10 ms, DOCK_24V off, STATUS DU_UNDOCKED
- [ ] CAN bootloader: an image streamed over 0x7E0/0x7E8 from the RV1106 verifies and boots; an interrupted update leaves the bootloader recoverable
- [ ] E-stop: K_SAFE opens (heating unit inlet off), HEAT_ON_OFF=0 and E_STOP on FDCAN1, motor disable on FDCAN2 and K_SU off, all within 100 ms; motor de-energised within ~50 ms of K_SU opening
- [ ] Fans: both PWM channels control speed; SET_EXHAUST over CAN changes duty
- [ ] All actuators, Q_DOCK and both relays off with the STM32 held in reset

---

## 16. Related Documentation

- [[01-Unified-PCB-Design|Unified PCB Design]] — the DU board (v4.0): RV1106, battery, dispensers, the dock's other half
- [[../02-Hardware/01-Epicura-Architecture|Epicura Architecture]] — three units, system wiring, CAN bus, dock, power
- [[../02-Hardware/02-Technical-Specifications|Technical Specifications]] — power specs
- [[../05-Subsystems/01-Electric-Heating|Electric Heating]] — electric heating unit (rice-cooker class) with internal temperature control, isolated CAN port, safety relay
- [[../05-Subsystems/02-Robotic-Arm|Robotic Arm]] — CAN servo stirring motor (SU), its own PSU and the isolated stir bus
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
| 1.2 | 2026-11-04 | Manas Pradhan | The other Epicura node on the isolated bus is now the DU STM32 (FDCAN1 + ISO1050DUB #2 on the Unified PCB v5.0, gateway for the RV1106 over a UART host link) instead of the RV1106's MCP2518FD; wording in the overview, scope and comparison tables, block diagram, §5 node table, lift note and bring-up checklist. No schematic or BOM change on this board |
| 1.6 | 2026-11-05 | Manas Pradhan | Board power from a dedicated Mean Well LRS-35-12 instead of the heating-unit tap of v1.5: J_HU_PWR / 12V_HU → J_12V_IN / 12V_IN (XT30; F1 2 A, SS34, SMBJ12A, 100 µF), the donor-rail requirements, warning and IRM-20-12 fallback are withdrawn, K_SAFE returns to the heating unit's AC inlet loom (the OEM unit is not opened), three PSUs each with one customer (LRS-35-12 → CSC + fans + relays, LRS-75-24 #1 → DU, LRS-75-24 #2 → SU); §4.1, scope table, diagrams, budget, protection, connectors and checklist updated; still 12 connectors |
| 1.5 | 2026-11-05 | Manas Pradhan | Board power moves to the heating unit: the CSC, both G5V-2 coils and the fans run from 12V_HU, a 12 V SELV tap off the OneChef control-board supply (new J_HU_PWR, F1 2 A, SS34, SMBJ12A, 470 µF; requirements and IRM-20-12 fallback in §4.1); the 24 → 12 V MP1584EN #1 is dropped and the remaining MP1584EN makes 5 V from 12 V; J_24V_IN becomes J_DU_IN (F_DU 3 A, SS54, SMBJ24A) carrying LRS-75-24 #1 to Q_DOCK only, its −V bonded to the board ground (§4.2); K_SAFE re-specified onto the element loom so the OEM board stays live (§8.1); budgets (12V_HU ~1.3/1.4 A, 24V_DU ~0.6/1.6 A), protection table, zone C, thermal (~3.5 W) and checklist updated; 12 connectors |
| 1.4 | 2026-11-04 | Manas Pradhan | DU side of bus #1 is no longer isolated (Unified PCB v5.2: the DU STM32 drives a non-isolated SN65HVD230; ISO1050 #2 and the DU island are gone): B0505S #1 now feeds only ISO1050 #1's bus side (+ optional heater pin 4), J_DOCK pin 6 (5V_ISO) stays exported but reserved, the DU ties GND_ISO (pin 5) to its ground as the bus's single reference. §4.5 gains the "why the barrier is on this board only" note (port-1 isolation is a ground-loop barrier since DU and CS grounds meet on pin 2; the heating unit's OEM CAN port must be SELV). Overview bullets, scope/comparison rows, §4.5 loads, §4.9 / §4.11 rows, §5 intro and node/ground rows, §7.1 pins 5–6 and the post-assembly check updated; SU port (#2) unchanged |
| 1.3 | 2026-11-04 | Manas Pradhan | Both CAN ports are now isolated and each unit has its own PSU (LRS-75-24 #1 = Dispensing Unit via the dock, also powering the CSC; #2 = Stirring Unit): FDCAN2 moves from the SN65HVD230 to ISO1050DUB #3 with B0505S #2 (5V_ISO2 referenced to GND_SU); a second Mean Well LRS-75-24 feeds the SU through a new floating SU island (J_SU_IN → F_SU → J_STIR_PWR, 24V_SU / GND_SU never bonded to the CS ground); Q_STIR and F_STIR are dropped and PB4 now drives the pilot relay K_SU that cuts the SU PSU's AC input (J_SAFE grows to the 2x3 DF11, pins 5–6); the heater relay is named K_SAFE; J_SU and J_CAN_STIR pinouts re-referenced to GND_SU; second 6 mm slot in Zone B; power budget, thermal, layout rules, protection and bring-up checklist updated; 11 connectors |
