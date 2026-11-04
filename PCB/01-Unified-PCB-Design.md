---
created: 2026-03-06
modified: 2026-11-04
version: 4.1
status: Draft
---

# Unified PCB Design

## 1. Overview

Epicura is physically three units: the **Cooking Surface (CS)** with the electric heating unit, PSU, safety relay, E-stop and fans; the **Stirring Unit (SU)** on the gantry between the CS rear pillars; and the **Dispensing Unit (DU)** that sits on top, lifts straight off with no cable, and carries the brain of the machine. This document covers the **Unified PCB v4.0, the DU board**. The CS has its own board, the **Cooking Surface Controller (CSC, STM32G0B1RE)**, documented in [[02-Cooking-Surface-Controller|Cooking Surface Controller]]. The only cable between units is the CS → SU harness; the DU meets the CS through an 8-pad magnetic pogo dock (J_DOCK).

One 160 x 90 mm, 4-layer board carries everything electrical in the DU:

- **Application-processor slot (J_AP)** for the Luckfox Pico Zero (RV1106G3): 5 V power, SPI0/I2C3 break-out to the display, I2C3 to every on-board peripheral, four PWM and three GPIO, and the RV1106's own CAN node (MCP2518FD + ISO1050DUB #2) on the isolated bus that reaches the CS through the dock
- **CID + FID drivers**: the Dispenser Servo Block (PCA9685 on I2C3, switched 6.0 V rail, J_DSB1-4)
- **SLD drivers**: TB6612FNG for two 6 V peristaltic pumps, two 6 V solenoid MOSFETs, two NAU7802 24-bit load-cell ADCs behind a TCA9543A I2C mux (J_SLD)
- **Dock (J_DOCK)**: 8 flat gold pads on the underside — DOCK_24V charge input, isolated CAN_H/CAN_L with GND_ISO and 5V_ISO, two dock-sense lines
- **2-cell Li-ion pack as the primary supply** with on-board 1 A charger (TP5100 from DOCK_24V) and a passive VSYS diode-OR, so the DU runs docked from the CS and lifted (or through a mains outage) from its pack
- Power conversion (VSYS → 5 V, 3.3 V and 6.0 V; DOCK_24V → 12 V for the charger only), white LED ring, buzzer, battery/dock monitoring (INA219 x2)

There is no MCU on this board any more: the STM32, the safety relay, E-stop, pot reed, exhaust fans, the regular (stirring-motor) CAN port, the isolated DC-DC and the USB-C/SWD programming ports all live on the CSC in the CS base. There is no IR thermometer on the board or in the system: the cooking surface runs its own temperature loop and reports its measured temperature on the isolated CAN bus (see [[../05-Subsystems/01-Electric-Heating|Electric Heating]]).

### 1.1 Board Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Unified PCB v4.0 — Dispensing Unit (160 x 90 mm, 4-layer)               │
│                                                                          │
│   ┌──────────────────────────┐   ┌───────────────┐   ┌────────────────┐  │
│   │ J_AP: Luckfox Pico Zero  │   │ 2S Li-ion     │   │ J_DOCK (8 pads)│  │
│   │ slot (2x20 socket +      │   │ TP5100 1 A    │   │ DOCK_24V, GND  │  │
│   │ 4 standoffs)             │   │ J_BAT, INA219 │   │ CAN_H/L, GND_ISO│ │
│   └──┬────────┬──────────────┘   └───────┬───────┘   │ 5V_ISO, SENSE  │  │
│      │SPI0 CS1│I2C3 / GPIO / PWM         │BAT+       └───┬────────┬───┘  │
│      ▼        │                          ▼               │CAN     │24 V  │
│   ┌──────────────────────────────┐   VSYS = DOCK_24V ⊕ BAT+ ◄──────┘      │
│   │ MCP2518FD ──► ISO1050 #2 ────┼──► isolated island ◄──┘ (5V_ISO)       │
│   │ (RV1106 CAN node)            │                                        │
│   └──────────────────────────────┘                                        │
│                                                                          │
│   DSB: PCA9685 (I2C3) + 6.0 V rail ──► J_DSB1-4 (FID x3, CID x1 servos)   │
│   SLD: TB6612FNG + 2 MOSFET + TCA9543A/2x NAU7802 ──► J_SLD               │
│   White LED ring (Q_LED, PWM), buzzer (PWM), J_LCD break-out              │
└──────────────────────────────────────────────────────────────────────────┘
          ▲ dock pads (CS left pillar)                ▲ battery pack (2S1P)
```

### 1.2 Scope

| Item | Status | Notes |
|------|--------|-------|
| **Unified PCB v4.0** (this document) | Custom design required | DU board: AP slot + battery-first power + dispenser drivers + RV1106 CAN node + dock pads; 160 x 90 mm, 4-layer |
| **Cooking Surface Controller (CSC)** | Custom design required — see [[02-Cooking-Surface-Controller\|Cooking Surface Controller]] | STM32G0B1RE, both CAN transceivers for the CS side, relay, E-stop, fans, dock power switch; in the CS base |
| **Luckfox Pico Zero** | Off-the-shelf module | Plugs into J_AP; camera on its own CSI FPC; display routed through the PCB to J_LCD |
| **Dock pogo block** | Off-the-shelf 8-contact magnetic pogo pair (2.54 mm pitch) | Spring pins in the CS left pillar, flat pads on this board (J_DOCK) |
| **Electric heating unit** | Commercial rice-cooker-class heating unit (resistive plate, own PID, CAN port; 1,300 W assumed) (see [[../05-Subsystems/01-Electric-Heating\|Electric Heating]]) | On the same isolated bus, reached through the dock and the CSC's J_CAN_HEAT |
| **Stirring motor** | 24 V integrated BLDC servo with CAN interface (see [[../05-Subsystems/02-Robotic-Arm\|Robotic Arm]]) | Not connected to this board: regular CAN port and 24 V on the CSC (J_SU harness) |
| **Battery pack** | 2S1P Li-ion 18650 pack with protection module (PCM) | Primary supply of the DU; charged and monitored on the PCB; connects to J_BAT |

### 1.3 Changes from v3.x

| Area | v3.x (single board) | v4.0 (DU board) |
|------|---------------------|-----------------|
| Physical architecture | One board for the whole machine, 24 V cable from the PSU | Three units; this board lives in the lift-off DU and meets the CS through an 8-pad pogo dock (J_DOCK). The STM32 moves to the new CSC in the CS base |
| MCU | STM32G0B1RE on this board (dispense, safety, both CAN ports) | None. Dispensing, load cells, LED ring and buzzer are driven by the RV1106 directly over I2C3 / GPIO / PWM (`epicurad` `dispense` thread) |
| CAN | Two ports on this board: isolated (ISO1050 x2 + B0505S, J_CAN_HEAT) and regular (SN65HVD230, J_CAN_STIR) | One node: MCP2518FD + ISO1050DUB #2, bus side powered from the dock's 5V_ISO; CAN_H/CAN_L on dock pads 3–4. No termination, no isolated DC-DC, no regular port on this board |
| Power input | J_24V_IN (XT30) from the LRS-75-24; 24 V feeds 12 V, 6.5 V and VSYS | DOCK_24V on dock pad 1 (switched on the CSC, 2 A); feeds only the charger (via 12 V) and VSYS. The battery is the primary supply: VSYS = DOCK_24V ⊕ BAT+ |
| Battery | Backup for the logic domain, 0.5 A charge, voltage on an STM32 ADC divider | Primary supply for everything on the board, 1.0 A charge (R_CS 0.1 Ω), pack voltage + current on INA219 #2 (0x44) |
| Dispenser rail | 6.5 V from 24 V; D_DSB4 drop for the MG90S | 6.0 V (6V_DSP) from VSYS; all four servos, the pumps and the solenoids run on it; D_DSB4 removed |
| SLD | 12 V pumps and solenoids; 2x HX711 bit-banged by the STM32 | 6 V pumps (TB6612 VM = 6V_DSP) and 6 V NC solenoids; 2x NAU7802 on I2C3 behind a TCA9543A mux |
| Load/current monitor | INA219 on the 24 V input | INA219 #1 on 6V_DSP (0x40, DSB stall window), INA219 #2 on BAT+ (0x44) |
| LED ring | WS2812B, STM32 TIM15 + DMA, 74AHCT1G125, 3-pin J_LED | White ring, Q_LED low-side on an RV1106 PWM, 2-pin J_LED |
| Buzzer | STM32 TIM8 | RV1106 PWM via 2N7002 |
| Power-fail detect | 24 V divider on STM32 COMP1 | DOCK_PWR_OK (divider + 2N7002) and DOCK_SENSE_A on RV1106 GPIO |
| Moved to the CSC | — | Safety relay (Q_RLY), E-stop, pot reed (J_SAFE), exhaust fans (J_FAN), Q_STIR / J_STIR_PWR, J_CAN_STIR, J_CAN_HEAT, ISO1050 #1, B0505S, SN65HVD230, J_SWD, J_USB, service UART |
| Connectors | 19 | 11 (J_AP, J_LCD, J_DOCK, J_BAT, J_DSB1–4, J_SLD, J_LED, J_BUZZER) |

#### Changes from v2.x (v3.0, history)

| Area | v2.x (CM5 era) | v3.0 |
|------|----------------|------|
| Application processor | Raspberry Pi CM5 IO Board stacked under the PCB (J_CM5), SPI2 link | Luckfox Pico Zero in the J_AP slot on the PCB; isolated CAN bus is the host link (MCP2518FD on SPI0 CS1), UART2 service link |
| Dispensers | P-ASD pneumatics (pump, 6 solenoids, ADS1015, PCF8574 #1), CID stepper + linear actuator (DRV8876) | Dispenser Servo Block: PCA9685 + 4 servos on a switched 6.5 V rail (J_DSB1-4) |
| Stirring motor | 24 V BLDC with PWM + EN + DIR on PA8/PA4/PA5 | 24 V CAN servo on FDCAN2 (regular CAN port) + switched 24 V (J_STIR_PWR) |
| CAN | One isolated port, STM32 only | Two ports: isolated (heater, STM32 + RV1106 nodes) and regular (stirring motor) |
| Temperature sensing | MLX90614 IR thermometer on I2C1 (J_IR_LED) | None on the PCB: the cooking surface reports its own temperature |
| Backup power | External 12 V UPS input (J_12V_UPS) + single-cell TP4056/MT3608 micro-UPS for the STM32 | 2S Li-ion pack (J_BAT) + TP5100 charger; diode-OR into the 5 V buck keeps the whole logic domain (RV1106 + LCD + STM32) alive |
| 5 V buck input | 12 V UPS rail | VSYS = diode-OR of the 24 V rail and the battery |

### 1.4 Area Analysis

Roughly 70 ICs/discretes and 11 connectors occupy about 2,700 mm² of the ~12,000 mm² usable placement area (~23 % utilisation). The Pico Zero footprint (65 x 30 mm) sits over the digital zone on standoffs; the area under it is used for passives on the bottom side. The outline stays 160 x 90 mm so the DU enclosure, the camera position (centred over the pot) and the carousel chutes around the board are unchanged; the free area goes to the dock-pad edge and a wider isolation island.

---

## 2. Board-Level Block Diagram

```
J_DOCK 1 DOCK_24V ─── SMBJ24A ─── 10 µF ──┬──────────────────────────────┬──► R_DIV 100k/10k ──► 2N7002 ──► DOCK_PWR_OK (RV1106 GPIO, active-low)
(switched on the CSC, F_DOCK 2 A)         │                              │
                                          ▼                              ▼
                                  ┌──────────────┐               ┌──────────────────┐
                                  │ MP1584EN #1  │               │ D_SYS1 (SS54)    │
                                  │ 24V → 12V    │               │ DOCK_24V ──► VSYS│
                                  └──────┬───────┘               └────────┬─────────┘
                                         │ 12V (charger only)             │
                                         ▼                        VSYS ◄──┴── D_SYS2 (SS54) ◄── BAT+ ◄── R_SH2 (10 mΩ, INA219 #2 0x44) ◄── J_BAT (2S1P, 6.0–8.4 V, PCM)
                                  ┌──────────────┐                 │                                ▲
                                  │ TP5100 1.0 A │ ──► BAT+ ───────┼────────────────────────────────┘
                                  │ (R_CS 0.1 Ω) │                 │
                                  └──────────────┘                 ├──────────────────────────┐
                                                                   ▼                          ▼
                                                           ┌──────────────┐           ┌──────────────┐
                                                           │ TPS54531     │           │ MP1584EN #2  │
                                                           │ VSYS → 5V 5A │           │ VSYS → 6.0 V │
                                                           └──────┬───────┘           └──────┬───────┘
                                                                  │                          │ Q_DSB (PCF8574 #2 P4) ── 3 A PTC ── 1000 µF
                                   5V_MAIN ◄──────────────────────┘                          ▼
                                      ├──► J_AP pins 2/4 (Pico Zero)                   6V_DSP (switched) ──► R_SH1 (10 mΩ, INA219 #1 0x40)
                                      ├──► J_LCD VCC                                     ├──► PCA9685 CH0-3 ──► J_DSB1-4 (DS3218, 2x MG996R, MG90S)
                                      ├──► J_LED pin 1 (white ring, Q_LED low-side)      ├──► TB6612 VM ──► J_SLD pumps (6 V peristaltic x2)
                                      ├──► J_BUZZER pin 1                                └──► J_SLD solenoids (6 V NC x2, IRLML6344 low-side)
                                      └──► AMS1117-3.3 ──► 3.3V: PCA9685, PCF8574 #2, INA219 x2, TCA9543A, NAU7802 x2,
                                                                 MCP2518FD, ISO1050 #2 VCC1, TB6612 VCC

J_DOCK 6 5V_ISO / 5 GND_ISO ──► ISO1050DUB #2 VCC2 (bus side; generated on the CSC, F_CAN 100 mA)
J_DOCK 3 CAN_H / 4 CAN_L ──────► ISO1050DUB #2 bus pins (PESD1CAN) ◄── MCP2518FD ◄── J_AP SPI0 CS1 + nINT
J_DOCK 7 DOCK_SENSE_A ─── 10k pull-up to 3.3 V ──► RV1106 GPIO (low = docked)
J_DOCK 8 DOCK_SENSE_B ─── tied to GND on this board (tells the CSC the DU is present)

                     ┌────────────────────────────────────────────────────────────────┐
                     │  RV1106 (Luckfox Pico Zero in J_AP)                            │
                     │  SPI0 CS0 ──► J_LCD (ST7796S); SPI0 CS1 ──► MCP2518FD          │
                     │  I2C3 ──► FT6336 (0x38), PCF8574 #2 (0x21), PCA9685 (0x41),    │
                     │           INA219 #1 (0x40), INA219 #2 (0x44),                  │
                     │           TCA9543A (0x71) ──► NAU7802 oil / water (0x2A each)   │
                     │  GPIO ──► PCA9685 OE; GPIO ◄── DOCK_SENSE_A, DOCK_PWR_OK       │
                     │  PWM x4 ──► TB6612 PWMA/PWMB, Q_LED (ring), buzzer             │
                     │  PCF8574 #2 ──► solenoids, status LED, TB6612 AIN1/BIN1,       │
                     │                 Q_DSB; ◄── TP5100 CHRG/STDBY                   │
                     └────────────────────────────────────────────────────────────────┘
```

---

## 3. RV1106 Interface

With the STM32 gone, the Luckfox Pico Zero in J_AP is the only controller on the board. Everything it drives is reached through four header resources: SPI0 (LCD + CAN controller, unchanged), **I2C3** (the single I2C bus of the DU), **three GPIO** and **four PWM**. Pin numbers for the new GPIO and PWM lines are fixed at layout (§5.2); the header pins freed by the STM32's departure (8, 10, 32, 38) are the first candidates.

### 3.1 I2C3 — Single Peripheral Bus (J_AP 16/18, 400 kHz)

| Address | Device | Function | Linux driver |
|---------|--------|----------|--------------|
| 0x21 | PCF8574 #2 | 8-bit GPIO expander (§3.4) | `gpio-pcf857x` |
| 0x2A | NAU7802 (oil) | 24-bit load-cell ADC, TCA9543A channel 0 | `nau7802` (IIO) |
| 0x2A | NAU7802 (water) | 24-bit load-cell ADC, TCA9543A channel 1 | `nau7802` (IIO) |
| 0x38 | FT6336 | Capacitive touch controller on the 4" display, via J_LCD (unchanged) | `edt-ft5x06` |
| 0x40 | INA219 #1 | 6V_DSP rail voltage / current (DSB stall window) | `ina2xx` (hwmon) |
| 0x41 | PCA9685 | 16-channel servo PWM (DSB) | `pwm-pca9685` |
| 0x44 | INA219 #2 | BAT+ voltage / current (state of charge) | `ina2xx` (hwmon) |
| 0x71 | TCA9543A | 2-channel I2C mux in front of the two NAU7802 (same fixed address) | `i2c-mux-pca954x` |

4.7k pull-ups to 3.3 V on the PCB (the touch pull-ups of v3.x). I2C1 stays spare on the header if layout prefers to split the load cells or the touch controller onto a second bus.

> [!note]
> The PCA9685 answers its All-Call address 0x70 (MODE1.ALLCALL = 1 after reset) as well as 0x41, so the TCA9543A is strapped to **0x71** (A0 = 1, A1 = 0) and never shares an address with it. `epicurad` additionally clears ALLCALL at init so 0x70 stays free.

### 3.2 GPIO (3, fixed at layout)

| Signal | Direction (RV1106 view) | Circuit | Purpose |
|--------|-------------------------|---------|---------|
| PCA9685_OE | Out | PCA9685 OE, 10k pull-up to 3.3 V | Active-low output enable; servos limp at boot and whenever the RV1106 is in reset |
| DOCK_SENSE_A | In | J_DOCK pad 7, 10k pull-up to 3.3 V | Low = docked (pad grounded on the CS); GPIO edge interrupt + 1 Hz poll |
| DOCK_PWR_OK | In | DOCK_24V → 100k/10k divider → 2N7002 gate; drain 10k pull-up to 3.3 V | Active-low: low while DOCK_24V > ~15 V. High while docked = CS mains lost (AC Power Failure pause) |

### 3.3 PWM (4 channels, fixed at layout)

| Signal | Destination | Frequency | Purpose |
|--------|-------------|-----------|---------|
| PWM_PUMP_OIL | TB6612 PWMA via 100R | 20 kHz | Oil pump speed |
| PWM_PUMP_WATER | TB6612 PWMB via 100R | 20 kHz | Water pump speed |
| PWM_LED | Q_LED gate (IRLML6344) via 100R | 1 kHz | White ring: OFF 0 %, CAPTURE 100 %, NIGHT 30 % |
| PWM_BUZZER | Q_BUZZ gate (2N7002) via 100R | 2–4 kHz tone | Piezo buzzer |

All four PWM lines have 10k pull-downs so pumps, ring and buzzer are off while the RV1106 boots. `pwm-rockchip` exposes them in `/sys/class/pwm`.

### 3.4 PCF8574 #2 (I2C 0x21) Allocation

| Pin | Direction | Signal | Notes |
|-----|-----------|--------|-------|
| P0 | Out | SLD solenoid 1 (oil) | IRLML6344 gate via 100R, 10k pull-down |
| P1 | Out | SLD solenoid 2 (water) | IRLML6344 gate via 100R, 10k pull-down |
| P2 | Out | Status LED (green) | Active-low, 330R |
| P3 | Out | TB6612 AIN1 (oil pump direction) | AIN2 tied to GND |
| P4 | Out | 6V_DSP rail enable | Q_DSB P-MOSFET via 2N7002 level shifter; on only during a dispense |
| P5 | Out | TB6612 BIN1 (water pump direction) | BIN2 tied to GND |
| P6 | In | TP5100 CHRG (charging, active-low) | Open-drain from charger, 10k pull-up |
| P7 | In | TP5100 STDBY (charge complete, active-low) | Open-drain from charger, 10k pull-up |

### 3.5 Header Pin Changes vs v3.x

| Header resource | v3.x | v4.0 |
|-----------------|------|------|
| Pins 8/10 (UART2) | Service link to STM32 USART2 | Freed (the STM32 is in a different unit); STM32 firmware goes over CAN (§7) |
| GPIO 32 / GPIO 38 | STM32 BOOT0 / NRST | Freed |
| I2C3 (16/18) | FT6336 touch only | Every I2C device on the board (§3.1) |
| GPIO | MCP2518FD nINT, TP_RST | + PCA9685 OE, DOCK_SENSE_A, DOCK_PWR_OK |
| PWM | PWM10 (LCD backlight) | + 4 channels (§3.3) |
| SPI0 CS0/CS1, UART0 console, CSI camera | — | Unchanged |

---

## 4. Power Supply

### 4.1 Dock 24 V Input (DOCK_24V via J_DOCK)

```
J_DOCK pad 1 (DOCK_24V) ──┬── D2 (SMBJ24A TVS) ──── C_IN (10 µF, 50 V) ──── 24V Internal ──► MP1584EN #1, D_SYS1, DOCK_PWR_OK divider
J_DOCK pad 2 (GND)   ─────┴── GND
```

| Component | Part | Function |
|-----------|------|----------|
| D2 | SMBJ24A (24 V TVS, 600 W) | Transient suppression at the pad (hot-plug on the pogo contact) |
| C_IN | 10 µF 50 V MLCC + 100 nF | Pad-side decoupling |

There is no polyfuse and no reverse diode on this side: DOCK_24V is already fused (F_DOCK 2 A) and switched (Q_DOCK) on the CSC, and the keyed recess makes reverse connection impossible. Pads are dead until the CSC sees DOCK_SENSE_B low, so the pogo contacts never make or break under load. Input budget: ≤ 2 A at 24 V (pin and fuse rating), see §4.11.

### 4.2 24V → 12V Rail (MP1584EN #1) — Charger Only

| Parameter | Value |
|-----------|-------|
| Output | 12 V, 3 A max (loaded ≤ 1 A) |
| Load | TP5100 battery charger only (~0.8 A input at 1.0 A charge current) |
| Inductor | 33 µH, CDRH104R, Isat > 4 A |
| Output caps | 2x 22 µF MLCC (25 V) + 100 µF electrolytic |
| Feedback | 140k / 10k → 12.0 V |
| Protection | 1.5 A polyfuse + SMBJ12A on output |

No fans, pumps or solenoids remain on 12 V (fans are on the CSC; pumps and solenoids moved to 6V_DSP), so the rail exists only because the TP5100's input range does not reach 24 V. It is up only while docked with CS mains present.

### 4.3 VSYS → 6.0 V Dispenser Rail (MP1584EN #2, 6V_DSP)

| Parameter | Value |
|-----------|-------|
| Input | VSYS (23.5 V docked, 6.0–8.4 V on the pack) |
| Output | 6.0 V, 3 A max |
| Load | Four hobby servos, one moving at a time: DS3218 (C1, 2.2 A stall), 2x MG996R (F1, F2, ~2.5 A stall), MG90S (F3, ~0.7 A stall) — all rated for 6.0 V, no series drop needed; TB6612 VM (two 6 V peristaltic pumps, ~0.6 A each); two 6 V NC solenoids (~0.5 A each) |
| Inductor | 22 µH, CDRH104R, Isat > 4 A |
| Output caps | 2x 22 µF MLCC + 470 µF electrolytic (10 V, low ESR) |
| Feedback | 64.9k / 10k → 5.99 V |
| Rail switch | Q_DSB: AO3401-class P-MOSFET high-side switch driven by a 2N7002 level shifter from PCF8574 #2 P4; 1000 µF bulk and 3 A PTC on the switched side; INA219 #1 shunt (§8) on the switched side |
| Protection | 3 A PTC + SMBJ6.0A TVS |

The rail is on only while a dispense is in progress (no idle servo buzz or heat), so the pack sees no actuator load between dispenses. On an empty pack (6.2 V) the buck drops out to ≈ 6.0 V, which is acceptable; the dispense thread refuses new moves below the 6.2 V halt threshold anyway.

### 4.4 VSYS Diode-OR and Dock/Battery → 5V Rail (TPS54531)

The 5 V domain (Pico Zero, LCD, LED ring, buzzer, 3.3 V logic) is fed by a TPS54531 buck whose input, **VSYS**, is a Schottky-OR of the dock 24 V and the 2S battery — the same circuit as v3.x with new sources:

```
24V Internal (DOCK_24V) ── D_SYS1 (SS54) ──┐
                                            ├── VSYS ──── C_SYS (100 µF, 35 V) ──► TPS54531 VIN, MP1584EN #2 VIN
BAT+ (6.0–8.4 V) ───────── D_SYS2 (SS54) ──┘
```

| Parameter | Value |
|-----------|-------|
| IC | TPS54531DDAR (SOIC-8), VIN 3.5–28 V |
| Input | VSYS: ~23.5 V docked with CS mains, 5.5–7.9 V on the pack (lifted, or docked during an outage) |
| Output | 5V_MAIN, 5 A max |
| Load | Pico Zero (0.2 A typ, 0.5 A peak), LCD + backlight (0.15 A), AMS1117 (0.3 A), white LED ring (≤ 1 A, PWM), buzzer |
| Inductor | 10 µH, CDRH127, Isat > 6 A |
| Output caps | 2x 22 µF MLCC (10 V) + 220 µF electrolytic |
| Feedback | 52.3k / 10k → 4.98 V |
| Switching | 570 kHz; integrated OCP, thermal shutdown, UVLO |

Switch-over is passive: while DOCK_24V is present D_SYS2 is reverse-biased and the battery does not discharge; when the DU is lifted or the CS loses mains, VSYS falls to the battery voltage within the hold-up of C_SYS and the buck keeps regulating. No controller, no gap. VSYS also feeds MP1584EN #2 (§4.3), so dispensing works on either source.

### 4.5 5V_MAIN → 3.3V Rail (AMS1117-3.3)

```
5V_MAIN (TPS54531, dock or battery) ──► AMS1117-3.3 ──► 3.3V
```

The USB Schottky-OR of v3.x is gone (there is no USB-C on this board; the Pico Zero's own USB-C port is the DU's only one). 3.3 V loads: PCA9685, PCF8574 #2, INA219 x2, TCA9543A, NAU7802 x2 (including their bridge excitation, ~10 mA per cell), MCP2518FD, ISO1050 #2 VCC1, TB6612 VCC — ~0.15 A typical, 0.3 A max.

### 4.6 2-Cell Li-ion Pack — Primary Supply

#### 4.6.1 Pack

| Parameter | Value |
|-----------|-------|
| Configuration | 2S1P Li-ion, 2x 18650 (2,500–3,000 mAh) in a holder or shrink-wrapped pack |
| Voltage | 7.4 V nominal, 6.0 V (empty) to 8.4 V (full) |
| Energy | ~19 Wh (2,600 mAh) |
| Protection | Pack-mounted 2S PCM (per-cell over-voltage 4.25 V, under-voltage 2.8 V, over-current 3–5 A, short-circuit); balance is the PCM's job if fitted |
| Temperature sense | 10 kΩ NTC (B = 3435) glued to the cells, wired to J_BAT pin 3 |
| Connector | J_BAT, JST-XH 2.5 mm 3-pin: B+, B−, NTC |
| Cells | IEC 62133-2 / IS 16046 certified cells and pack (required for BIS) |
| Placement | In the DU, which is the cool zone by definition (the heat, PSU and fans are in the CS); keep the pack away from the LED ring and the camera window |
| Run time | Lifted-unit load ≈ 3 W (RV1106 + LCD + logic) → > 5 h on a 2,600 mAh pack; a full session docked draws from the dock, not the pack |

#### 4.6.2 Charger (TP5100)

```
12V Rail ──── C_CHG_IN (10 µF) ──► TP5100 VIN
                                     │
                                     ├── SW ── L_CHG (22 µH, Isat > 2 A) ──┬── R_CS (sense) ──► BAT+ (J_BAT pin 1)
                                     │        D_CHG (SS34) to GND         │
                                     ├── VS / BAT ◄──────────────────────┘   (charge current = 0.1 V / R_CS; 0.1 Ω → 1.0 A)
                                     ├── TEMP ◄── NTC (J_BAT pin 3) + 10k bias
                                     ├── CS (cell select) ── pulled for 2S (8.4 V)
                                     ├── CHRG ──► PCF8574 #2 P6 (10k pull-up to 3.3 V) + red LED
                                     └── STDBY ──► PCF8574 #2 P7 (10k pull-up to 3.3 V) + blue LED
```

| Parameter | Value |
|-----------|-------|
| IC | TP5100 (QFN-16, LCSC C379389), switching CC/CV charger, 1S/2S selectable |
| Input | 12 V rail (within the TP5100's 5–18 V range; 24 V exceeds it) |
| Charge voltage | 8.4 V (2S mode) |
| Charge current | **1.0 A** (R_CS = 0.1 Ω, 1 %, 0.5 W) — ≈ 3 h from empty while docked; the pack is now the primary supply, so it must refill within a normal cooking session |
| Pre-charge / termination | Trickle below 2S 6.0 V; termination at C/10; auto-recharge when the pack sags |
| Temperature window | NTC on TEMP pin: charging inhibited below 0 °C or above 45 °C |
| Status | CHRG and STDBY open-drain outputs to PCF8574 #2 P6/P7 (polled at 1 Hz by `epicurad`) and two 0603 LEDs |
| Charge source | 12 V rail only: the pack charges whenever the DU is docked and the CS has mains (DOCK_PWR_OK low) |

> [!note]
> Confirm the R_CS current-programming relation and the CS pin strapping against the TP5100 datasheet before layout; retailer summaries disagree on the input range (5–12 V vs 5–18 V), which is why the charger runs from the 12 V rail and not from 24 V. At 1.0 A check the TP5100's thermal pad pour (§12).

#### 4.6.3 Discharge Path and Monitoring

- BAT+ → R_SH2 (10 mΩ) → D_SYS2 (SS54) → VSYS → TPS54531 and MP1584EN #2 (§4.4, §4.3). Lifted-unit discharge ~0.45 A at 7.4 V; up to ~3 A for a servo move on the pack; the pack PCM over-current limit (3–5 A) is the ceiling, and the dispense thread holds one servo at a time.
- Pack monitoring: **INA219 #2 (I2C3 0x44)** across R_SH2 reports pack voltage (bus input on BAT+) and signed current (charge in, discharge out). It replaces the v3.x 100k/22k divider on an STM32 ADC. `epicurad` derives the state of charge from voltage + coulomb count and shows a battery indicator that is always visible on the DU display.
- Self-managed thresholds: warning banner at 6.4 V; checkpoint + halt at 6.2 V (boot recovery on the next power-up); the pack PCM cuts off at 2x 2.8 V = 5.6 V as a last resort.

#### 4.6.4 What the Battery Powers

Everything on the board. The pack is the primary supply of the DU; the dock only adds charge current and lifts VSYS above the pack voltage so the pack rests while docked.

| Domain | On battery | Notes |
|--------|------------|-------|
| 5V_MAIN: Pico Zero, LCD, 3.3 V logic, MCP2518FD, ISO1050 #2 logic side | Yes | ~3 W → > 5 h lifted |
| White LED ring (5V_MAIN, PWM) | Yes | `epicurad` keeps it OFF while lifted or on a power-fail pause |
| 6V_DSP (servos, pumps, solenoids) | Yes | Rail on only during a dispense; refused below 6.2 V |
| 12 V (charger) | No | DOCK_24V gone — no charging |
| 5V_ISO / CAN bus side | No | Comes from the dock; lifting the DU opens the bus by design |

The purpose: ride through a CS mains outage without losing the session (checkpoint, keep the UI and Wi-Fi alive, resume if mains returns within the food-safety window, see [[../03-Software/03-Main-Loop-State-Machine|Main Loop State Machine]] §5), and keep the DU alive while it is lifted off for loading and cleaning.

### 4.7 Dock Power Sense (DOCK_PWR_OK)

```
24V Internal (DOCK_24V) ─── R_DIV1 (100k) ──┬── R_DIV2 (10k) ─── GND
                                             ├── C_DIV (100nF) ─── GND
                                             └── Q_PWR (2N7002) gate ── drain ── 10k to 3.3V ──► RV1106 GPIO DOCK_PWR_OK (active-low)
```

| Parameter | Value |
|-----------|-------|
| Divider | 1/11 → 2.18 V at 24 V |
| Threshold | 2N7002 Vgs(th) 1.0–2.5 V → DOCK_PWR_OK goes high (inactive) when DOCK_24V < ~11–20 V; firmware confirms with a 300 ms TELEMETRY timeout |
| Recovery | DOCK_24V back for 2 s (firmware debounce) |
| Response | GPIO edge interrupt |

Together with DOCK_SENSE_A (§3.2) the RV1106 distinguishes the two pause causes: DOCK_SENSE_A low + DOCK_PWR_OK inactive = CS mains lost (AC Power Failure, auto-resume); DOCK_SENSE_A high = Unit Lifted (resume by user tap after re-docking). In both cases the heater and stir motor are already down on the CS side (keep-alive loss and the CSC's DOCK_SENSE_B hold). No 24 V comparator or STM32 is involved any more.

### 4.8 Stirring Motor 24 V Switch — removed

Q_STIR, F_STIR and J_STIR_PWR moved to the CSC (FDCAN2 side, enable from STM32 PB4); see [[02-Cooking-Surface-Controller|Cooking Surface Controller]] §6. No 24 V load remains on this board besides the charger buck and D_SYS1.

### 4.9 Per-Rail Protection

| Rail | Polyfuse | TVS | Notes |
|------|----------|-----|-------|
| DOCK_24V | 2 A on the CSC (F_DOCK) | SMBJ24A at the pad | Switched by Q_DOCK on the CSC; no reverse diode (keyed dock) |
| 12V | 1.5 A | SMBJ12A | Charger only |
| 6V_DSP | 3 A PTC | SMBJ6.0A | Switched rail; INA219 #1 shunt |
| 5V_MAIN | — | — | TPS54531 integrated OCP |
| 5V_ISO | 100 mA on the CSC (F_CAN) | — | Arrives over dock pad 6 |
| Battery | Pack PCM | — | D_SYS2 blocks back-charging from DOCK_24V; R_SH2 10 mΩ in the path |

### 4.10 Decoupling

- 10 µF at LDO input and output; 100 nF + 10 µF at each IC supply pin (PCA9685, PCF8574 #2, INA219 x2, TCA9543A, NAU7802 x2, MCP2518FD, ISO1050 #2 both sides, TB6612 VM and VCC)
- NAU7802: 0.1 µF + 10 µF on AVDD (internal LDO output used as bridge excitation), per datasheet, away from the bucks
- 10 µF + 100 nF at the dock pads on DOCK_24V and 5V_ISO

### 4.11 Power Budget Summary

| Rail | Source | Typical (A) | Peak (A) | Notes |
|------|--------|-------------|----------|-------|
| 3.3V (0.8 A max) | 5V_MAIN → AMS1117 | 0.15 | 0.30 | I2C ICs, NAU7802 x2, MCP2518FD, ISO1050 #2 logic side, TB6612 VCC |
| 5V_MAIN (5 A max) | VSYS → TPS54531 | 0.6 | 1.9 | Pico Zero + LCD + LDO + white ring (1 A peak) |
| 6V_DSP (3 A max) | VSYS → MP1584EN #2 | 0 | 2.5 | One servo at a time, or a pump + solenoid pair; rail off when idle |
| 12V (loaded ≤ 1 A) | DOCK_24V → MP1584EN #1 | 0.8 | 0.9 | TP5100 at 1.0 A charge; docked with mains only |
| 5V_ISO | Dock pad 6 (CSC B0505S) | 0.04 | 0.06 | ISO1050 #2 bus side only |
| **DOCK_24V input** | **CS via Q_DOCK, F_DOCK 2 A** | **~0.6** | **~1.6** | Charging + logic ≈ 0.6 A; + a servo move on the dock ≈ 1.6 A |
| Battery (lifted, or docked during an outage) | 2S pack → R_SH2 → D_SYS2 → VSYS | 0.45 @ 7.4 V | 3.0 | Whole board; the peak is a DS3218/MG996R stall on the pack |

> [!warning]
> The dock pin rating (2 A per contact) and F_DOCK (2 A) set the input ceiling. The 1.6 A peak assumes the charger is at full current while a servo indexes; `epicurad` must not raise the LED ring to 100 % at the same moment, or the charger input is the first thing to sag (harmless, but it lengthens the charge). Monitor INA219 #2 during integration.

### 4.12 Isolated 5V DC-DC — removed

The B0505S-1WR3 moved to the CSC, which generates the single 5V_ISO / GND_ISO domain for the whole isolated bus and extends it to this board over dock pads 5–6 (F_CAN 100 mA on the CSC). The ISO1050DUB #2 bus side is its only load here (~40 mA). No isolated DC-DC on this board.

---

## 5. Application-Processor Slot (J_AP) and Display Break-out

### 5.1 Mechanical

The Luckfox Pico Zero (65 x 30 mm) seats pin-down into **J_AP**, a 2x20 2.54 mm female socket (8.5 mm stacking height) on the top side of the PCB, and is held by four M2.5 standoffs matching the module's 3.0 mm mounting holes. Its USB-C port and microSD slot face the board edge for access; the CSI FPC connector faces inward with clearance for the camera cable, which runs to the IMX415-98 camera on the DU underside, centred over the pot behind the sealed window. The module's on-board Wi-Fi antenna must point away from the buck converters; the external-antenna connector is used when the enclosure is metal.

### 5.2 J_AP Pin Usage (RPi-style 40-pin numbering)

| Header pin | Signal | Direction (Pico Zero view) | Destination on PCB | Purpose |
|------------|--------|-----------------------------|--------------------|---------|
| 2, 4 | 5V | In | 5V_MAIN (battery / dock) | Module power, ~1 W (its own regulators make 3.3 V / core rails) |
| 1, 17 | 3.3V (module output) | Out | Not used | The board has its own 3.3 V |
| 6, 9, 14, 20, 25, 30, 34, 39 | GND | — | GND | Common ground |
| 19 | SPI0_MOSI | Out | J_LCD MOSI and MCP2518FD SI | Shared SPI0 |
| 21 | SPI0_MISO | In | MCP2518FD SO (and J_LCD MISO, unused) | Shared SPI0 |
| 23 | SPI0_CLK | Out | J_LCD SCK and MCP2518FD SCK | 50 MHz to LCD, ≤ 20 MHz when CS1 is active |
| 24 | SPI0_CS0 | Out | J_LCD CS | LCD chip select |
| SPI0_CS1 (GPIO 58) | SPI0_CS1 | Out | MCP2518FD nCS | RV1106 CAN node |
| spare GPIO (fixed at layout) | GPIO | In | MCP2518FD nINT | CAN RX/TX FIFO interrupt, active-low |
| 22 | GPIO | Out | J_LCD DC | LCD data/command |
| 26 | GPIO | Out | J_LCD RST | LCD reset |
| 12 | PWM10 | Out | J_LCD BL | Backlight PWM (via on-board 2N7002 if the module's LED pin needs > 20 mA) |
| 16 | I2C3_SDA | Bidir | J_LCD TP_SDA + all on-board I2C devices (§3.1) | 400 kHz, 4.7k pull-up on PCB |
| 18 | I2C3_SCL | Bidir | J_LCD TP_SCL + all on-board I2C devices (§3.1) | 400 kHz, 4.7k pull-up on PCB |
| 36 | GPIO | In | J_LCD TP_INT | Touch interrupt, active-low |
| spare GPIO (fixed at layout) | GPIO | Out | J_LCD TP_RST | Touch reset |
| spare GPIO (fixed at layout) | GPIO | Out | PCA9685 OE (10k pull-up) | Servo output enable, active-low |
| spare GPIO (fixed at layout) | GPIO | In | J_DOCK pad 7 DOCK_SENSE_A (10k pull-up) | Low = docked |
| spare GPIO (fixed at layout) | GPIO | In | Q_PWR drain (10k pull-up) DOCK_PWR_OK | Low = dock 24 V present |
| spare PWM (fixed at layout) x4 | PWM | Out | TB6612 PWMA, TB6612 PWMB, Q_LED gate, Q_BUZZ gate | Pumps, white ring, buzzer (§3.3) |
| 8, 10 | UART2 TX/RX | — | Not connected (freed) | Was the STM32 service link in v3.x |
| GPIO 32, GPIO 38 | — | — | Not connected (freed) | Were STM32 BOOT0 / NRST in v3.x |
| 35, 37 | UART0 TX/RX | — | J_AP test pads only | Luckfox debug console |
| all others | — | — | Not connected | |

> [!note]
> The RV1106 has no CAN peripheral. Its CAN node (MCP2518FD + ISO1050DUB #2, §7) lives on this board and is reached through SPI0 CS1 on J_AP; the bus itself arrives over the dock. The LCD and the CAN controller share SPI0 MOSI/MISO/CLK; the controller's 32-object RX FIFO absorbs frames while an LCD DMA chunk (≤ 10 ms) holds the bus. STM32 firmware updates no longer pass through this header: the RV1106 streams the image over CAN (0x7E0/0x7E8) to the CSC's in-application bootloader.

### 5.3 J_LCD — 4" ST7796S + FT6336 Display Break-out (1x14 pin header, 2.54 mm)

The display module's 14-pin header plugs into J_LCD with a short ribbon (< 20 cm) to the DU front face, where the panel is tilted 15–30°. Pin order follows the common 4.0" ST7796S capacitive SPI module; confirm against the chosen panel before layout.

| Pin | Signal | Source |
|-----|--------|--------|
| 1 | VCC (5 V) | 5V_MAIN |
| 2 | GND | GND |
| 3 | LCD_CS | J_AP 24 (SPI0_CS0) |
| 4 | LCD_RST | J_AP 26 |
| 5 | LCD_DC | J_AP 22 |
| 6 | LCD_MOSI | J_AP 19 |
| 7 | LCD_SCK | J_AP 23 |
| 8 | LCD_BL | J_AP 12 (PWM10) |
| 9 | LCD_MISO | J_AP 21 (unused by fbtft) |
| 10 | TP_SCL | J_AP 18 (I2C3_SCL) |
| 11 | TP_RST | J_AP spare GPIO |
| 12 | TP_SDA | J_AP 16 (I2C3_SDA) |
| 13 | TP_INT | J_AP 36 |
| 14 | GND | GND |

---

## 6. Dispenser and Liquid Drivers

### 6.1 Dispenser Servo Block — PCA9685 + Switched 6.0 V Rail (FID + CID)

```
I2C3 (J_AP 16/18) ──► PCA9685PW (addr 0x41, A0 = 1)
   RV1106 GPIO ──► OE (active-low; 10k pull-up → outputs off until epicurad drives it low)
   CH0 ──► 100R ──► J_DSB1 SIG    C1  CID carousel index servo   (DS3218 270°)
   CH1 ──► 100R ──► J_DSB2 SIG    F1  FID carousel index servo   (MG996R)
   CH2 ──► 100R ──► J_DSB3 SIG    F2  FID dose servo, ¼ tsp wheel (MG996R)
   CH3 ──► 100R ──► J_DSB4 SIG    F3  FID steam-gate servo       (MG90S)
   CH4–CH15 unused (test pads)

6.0 V (MP1584EN #2 from VSYS) ──► Q_DSB (AO3401, high-side) ──► 3 A PTC ──► R_SH1 (10 mΩ, INA219 #1) ──► 1000 µF ──┬── J_DSB1-4 V+ (6.0 V)
                                   gate ◄── 2N7002 ◄── PCF8574 #2 P4 (rail enable)                                ├── TB6612 VM
                                                                                                                  └── J_SLD 6V_SOL1 / 6V_SOL2
```

| Parameter | Value |
|-----------|-------|
| PWM driver | PCA9685PW (TSSOP-28), 16 ch, 12-bit, 50 Hz; EXTCLK not used; V+ pin left unconnected (servo power goes directly from the switched rail to J_DSB); Linux `pwm-pca9685` |
| Servo signal | 500–2,500 µs, 3.3 V logic (DS3218, MG996R and MG90S all accept 3.3 V) |
| Rail | 6.0 V / 3 A, switched on for the duration of a dispense; one servo moves at a time (`dispense` thread mutex); the MG90S no longer needs D_DSB4 |
| Connectors | J_DSB1–4: 3-pin JR/Futaba servo headers (SIG / V+ / GND), 2.54 mm, keyed, 300 mm extension leads |
| Fault detection | INA219 #1 current window on 6V_DSP during moves (> 2.5 A for 300 ms) + 3 s move timeout + PCA9685 I2C NACK → retry once (back off 10°) → DSB_STALL, detected by the RV1106 |
| Safe state | OE high and rail off at boot, on E_STOP, on UNIT_LIFTED and while the RV1106 is in reset (servos limp) |

### 6.2 SLD Peristaltic Pumps (2x) — TB6612FNG

```
6V_DSP ──── VM (TB6612FNG)
   PWMA ◄── RV1106 PWM_PUMP_OIL (100R)     AIN1 ◄── PCF8574 #2 P3   AIN2 ── GND   AO1/AO2 ──► J_SLD 1/2 (oil pump)
   PWMB ◄── RV1106 PWM_PUMP_WATER (100R)   BIN1 ◄── PCF8574 #2 P5   BIN2 ── GND   BO1/BO2 ──► J_SLD 3/4 (water pump)
   STBY ◄── 3.3V (10k)   VCC ── 3.3V            C_VM: 100 nF + 10 µF
```

| Parameter | Value |
|-----------|-------|
| IC | TB6612FNG (SSOP-24), VM = 6V_DSP (6.0 V), VCC 3.3 V |
| Pumps | Re-specified to 6 V DC peristaltic, ~0.6 A each |
| Current | 1.2 A continuous / 3.2 A peak per channel |
| Pull-downs | 10k on PWMA/PWMB (pumps off during boot); direction bits default low on the PCF8574 (forward) |

Pumps run only while 6V_DSP is enabled, so an idle or lifted DU cannot pump even if a PWM line is mis-driven.

### 6.3 SLD Solenoid Valves (2x)

Two 6 V normally-closed valves (drip prevention) on 6V_DSP, low-side switched by IRLML6344 with SS14 flyback, 100R gate resistor and 10k gate pull-down, gates driven from PCF8574 #2 P0 (oil) and P1 (water). Re-specified from 12 V to 6 V so that no 12 V load remains on the board.

### 6.4 SLD Reservoir Load Cells (2x) — NAU7802 via TCA9543A

```
I2C3 ──► TCA9543A (0x71) ──┬── ch0 ──► NAU7802 (0x2A) oil   ── AVDD/AVSS ──► J_SLD 9/10 (E+/E−); VIN+/VIN− ◄── J_SLD 11/12 (S+/S−)
                           └── ch1 ──► NAU7802 (0x2A) water ── AVDD/AVSS ──► J_SLD 13/14 (E+/E−); VIN+/VIN− ◄── J_SLD 15/16 (S+/S−)
```

| Parameter | Value |
|-----------|-------|
| ADC | NAU7802SGI (SOIC-16), 24-bit, PGA up to 128, internal LDO (3.0 V) used as bridge excitation, 10–320 SPS; Linux `nau7802` (IIO) |
| Mux | TCA9543A (TSSOP-14), 2 channels, because both NAU7802 share the fixed address 0x2A; Linux `i2c-mux-pca954x` |
| Load cells | 2x 2 kg bridge cells (oil and water reservoirs), 4-wire, excited and read on the PCB — the HX711 breakout boards of v3.x are gone |
| Sampling | 10 SPS per cell, ±1 g with averaging; the `dispense` thread closes the SLD loop at 10 Hz and stops at 95 % of target |
| Fault | I2C NACK or no data-ready → SLD_LOAD_CELL_DRIFT / SLD_DISPENSE_ERROR, RV1106-detected |

Bridge wiring on J_SLD is twisted and short (< 30 cm); keep the NAU7802 analog pins away from the 6V_DSP switch and the bucks.

### 6.5 Exhaust Fans — removed

The two 120 mm fans, their MOSFETs and J_FAN are on the CSC (STM32 PA6 / PB10, 12 V rail); the RV1106 sets the exhaust stage over CAN (SET_EXHAUST 0x260, Exhaust §3.4).

### 6.6 White LED Ring Driver

```
5V_MAIN ──► J_LED pin 1 (5V)  ──► ring (12–16 white LEDs, 5000 K, CRI ≥ 90, ≤ 1 A) ──► J_LED pin 2 (LED_RET) ──► Q_LED (IRLML6344, low-side) ──► GND
                                                                                       gate ◄── 100R ◄── RV1106 PWM_LED (1 kHz), 10k pull-down
```

The ring is a plain white ring with series resistors or a constant-current strip, mounted on the DU underside around the camera window; brightness is the RV1106 PWM duty (OFF 0 %, CAPTURE 100 %, NIGHT 30 %). No colour control, no WS2812B, no level shifter and no SET_LED CAN command any more.

### 6.7 Piezo Buzzer

5 V active piezo via 2N7002 low-side switch, gate from the RV1106 PWM_BUZZER through 100R with 10k pull-down; J_BUZZER 2-pin.

### 6.8 USB-C Programming Port — removed

There is no STM32 to program. The Pico Zero's own USB-C port serves the RV1106 (console, mass storage, `rkdeveloptool`); the CSC carries the USB-C DFU port for the STM32.

---

## 7. CAN Node

The board carries **one** CAN node, the RV1106's, on the isolated 500 kbps bus shared with the STM32 (on the CSC) and the heating unit. The bus and its isolated supply arrive over dock pads 3–6; nothing else on this board is on a CAN bus.

| | RV1106 node (this board) |
|---|---|
| Connector | J_DOCK pads 3 (CAN_H), 4 (CAN_L), 5 (GND_ISO), 6 (5V_ISO) |
| Controller | MCP2518FD on J_AP SPI0 CS1 (§7.2) |
| Transceiver | ISO1050DUB #2, 5 kV RMS; logic side on 3.3 V / GND, bus side on the dock's 5V_ISO / GND_ISO |
| Other nodes | STM32 (ISO1050DUB #1 on the CSC), heating unit MCU (its own transceiver) |
| Bit rate | 500 kbps, classic CAN 2.0B, 11-bit IDs |
| Termination | None on this board: 120 Ω on the CSC at J_CAN_HEAT and 120 Ω inside the heating unit. The dock → this board run is a stub ≤ 30 cm |
| Ground | GND_ISO only on the isolated island; the DU logic ground joins the CS ground only through dock pad 2 |
| Traffic | Host commands/telemetry (RV1106 ↔ STM32), heater setpoint and status, heartbeat, E_STOP, STM32 firmware update (0x7E0/0x7E8) — ID map in [[../03-Software/02-Controller-Software-Architecture#7.2 Message Types\|Controller Software Architecture §7.2]] |

Lifting the DU opens the bus and the 24 V at the same instant: the heating unit drops to zero ≤ 1 s after its last keep-alive, the STM32 enters safe hold on DOCK_SENSE_B, and this node simply sees bus-off until re-docked.

### 7.1 Isolated Transceiver — ISO1050DUB #2 on the Dock Supply

```
MCP2518FD TXCAN ──► TXD ┐  ISO1050DUB #2  ┌── CANH ─┬── PESD1CAN ── J_DOCK pad 3 (CAN_H)
MCP2518FD RXCAN ◄── RXD ┘  VCC1 3.3V / GND │  CANL ─┴────────────── J_DOCK pad 4 (CAN_L)
                            VCC2 5V_ISO / GND_ISO ◄── J_DOCK pads 6 / 5 (10 µF + 100 nF)
                  ─── 6 mm milled slot under the ISO1050 #2 and around dock pads 3–6 ───
```

- The transceiver sits on the system-ground side of the slot with its bus pins crossing into the isolated island; no other copper crosses the boundary (creepage ≥ 6.4 mm, IEC 60335 reinforced).
- Decoupling: 100 nF on VCC1, 100 nF + 10 µF on VCC2.
- The isolated island contains only ISO1050 #2's bus side, PESD1CAN and dock pads 3–6.

### 7.2 RV1106 CAN Node — MCP2518FD

| Parameter | Value |
|-----------|-------|
| IC | MCP2518FD-E/SL (SOIC-14), CAN FD controller used in classic mode |
| Clock | 40 MHz crystal (Y3) + 2x 15 pF |
| SPI | J_AP SPI0: MOSI 19, MISO 21, CLK 23, nCS = SPI0_CS1, ≤ 20 MHz |
| INT | nINT → J_AP spare GPIO (fixed at layout), 10k pull-up |
| Supply | 3.3 V, 100 nF + 10 µF |
| Driver | Linux `mcp251xfd` → SocketCAN `can0` at 500 kbps |

### 7.3 Regular Port — moved

The SN65HVD230, J_CAN_STIR, Q_STIR and J_STIR_PWR are on the CSC (STM32 FDCAN2, 1 Mbps) and reach the stirring motor through the J_SU harness inside the right pillar; see [[02-Cooking-Surface-Controller|Cooking Surface Controller]] §6.

---

## 8. Current Monitoring — INA219 x2

| Ref | I2C3 address | Shunt | Measures | Use |
|-----|--------------|-------|----------|-----|
| INA219 #1 | 0x40 | 10 mΩ 2512 in the switched 6V_DSP rail (after the PTC) | Rail voltage and servo / pump / solenoid current | DSB stall window (> 2.5 A for 300 ms) during carousel moves; pump no-flow plausibility; dispenser power telemetry. Moved from the 24 V input of v3.x |
| INA219 #2 | 0x44 | 10 mΩ 2512 in the BAT+ path between J_BAT and D_SYS2 | Pack voltage (bus input on BAT+), signed pack current | State of charge, charging / discharging, 6.4 V warning and 6.2 V halt; replaces the STM32 ADC divider |

Both are read by `epicurad` through the Linux `ina2xx` hwmon driver (1 Hz for the pack, 50 Hz while a dispense is active).

---

## 9. Protection Circuits

### 9.1 ESD Protection

- 33R series resistors on J_LCD and the four PWM / three GPIO lines leaving the Pico Zero to the drivers
- PRTR5V0U2X on the J_SLD bridge sense lines (the load cells are off-board, in the reservoir bays)
- The relay, E-stop and pot inputs (J_SAFE) and the USB port are on the CSC; their ESD parts go with them

### 9.2 Dock Pad ESD and Hot-Plug

The dock pads are the only contacts a user touches with the DU in hand (cleaning, loading):

- **PESD1CAN** across CAN_H / CAN_L on the isolated island (pads 3–4)
- **SMBJ24A** on DOCK_24V (pad 1) plus 10 µF 50 V, against the hot-plug transient when the CSC closes Q_DOCK
- 10 µF + 100 nF on 5V_ISO (pad 6)
- DOCK_SENSE_A (pad 7) and DOCK_PWR_OK: 100R series + 10k pull-up into the RV1106 GPIO; DOCK_SENSE_B (pad 8) is tied to GND through a 0R link
- Pads 1–2 and 3–6 are ≥ 6.4 mm apart on the pad pattern so the isolation holds across the connector

### 9.3 MOSFET Gate Safety

Every MOSFET gate and driver enable has a 100R series resistor and a 10k pull-down (or pull-up for the P-MOSFET Q_DSB and the PCA9685 OE) so that all actuators, the 6V_DSP rail, the LED ring and the buzzer are off while the RV1106 is booting, in reset or unpowered. The PCF8574 #2 outputs are high-impedance (pulled low by the gate resistors) until `epicurad` writes them.

Relay driver, E-stop input and pot-detection input: on the CSC (see [[02-Cooking-Surface-Controller|Cooking Surface Controller]] §8).

---

## 10. Connector Definitions (11 total)

| Ref | Type | Function |
|-----|------|----------|
| J_AP | 2x20 socket, 2.54 mm, 8.5 mm stacking | Luckfox Pico Zero slot |
| J_LCD | 1x14 header, 2.54 mm | 4" ST7796S + FT6336 display |
| J_DOCK | 8 flat gold pads, 2.54 mm pitch, bottom side at the board edge | Dock to the CS left pillar: charge 24 V, isolated CAN, 5V_ISO, dock sense |
| J_BAT | JST-XH 2.5 mm 3-pin | 2S Li-ion pack: B+, B−, NTC |
| J_DSB1–J_DSB4 | 3-pin servo header, 2.54 mm | C1, F1, F2, F3 servos |
| J_SLD | Hirose DF11-16DP-2DS (2x8, 2.0 mm) | SLD pumps, solenoids, two load-cell bridges |
| J_LED | JST-XH 2.5 mm 2-pin | White ring: 5V, LED_RET |
| J_BUZZER | JST-PH 2.0 mm 2-pin | Piezo buzzer |

Removed since v3.x (all now on the CSC): J_24V_IN, J_CAN_HEAT, J_CAN_STIR, J_STIR_PWR, J_FAN, J_SAFE, J_SWD, J_USB. Removed since v2: J_CM5 (replaced by J_AP), J_12V_UPS, J_BLDC, J_PASD, J_PRES, J_CID, J_IR_LED (replaced by J_LED), single-cell J_BAT (replaced by the 3-pin 2S J_BAT).

### 10.1 J_AP — see §5.2

### 10.2 J_LCD — see §5.3

### 10.3 J_DOCK — Dock Pads to the Cooking Surface (8 pads, 2.54 mm pitch, DU view)

| Pad | Signal | Direction | Notes |
|-----|--------|-----------|-------|
| 1 | DOCK_24V | In (CS → DU) | Charge input; switched on the CSC by Q_DOCK only while DOCK_SENSE_B is low, F_DOCK 2 A. SMBJ24A + 10 µF at the pad. Dead when the unit is lifted |
| 2 | GND | — | Charge return; ties the DU logic ground to the CS ground while docked |
| 3 | CAN_H | bus | Isolated 500 kbps bus (same bus as the heating unit); PESD1CAN |
| 4 | CAN_L | bus | |
| 5 | GND_ISO | — | Isolated bus ground; ISO1050 #2 GND2 |
| 6 | 5V_ISO | In (CS → DU) | From the B0505S-1WR3 on the CSC (F_CAN 100 mA); ISO1050 #2 VCC2 |
| 7 | DOCK_SENSE_A | In (CS → DU) | Tied to GND on the CS; 10k pull-up here → RV1106 GPIO; low = docked |
| 8 | DOCK_SENSE_B | Out (DU → CS) | Tied to GND here; 10k pull-up on the CSC → STM32 PB3; low = DU present (enables Q_DOCK) |

The mating part is an 8-contact magnetic pogo connector: spring pins (2 A each) in a keyed recess on top of the CS left pillar, flat pads on the DU underside, so the DU can only seat one way. The pad pattern is on L4 (bottom) at the board edge over the pillar; pads 3–6 sit on the isolated island (§11.2).

### 10.4 J_BAT — 2S Li-ion Pack (JST-XH 2.5 mm, 3-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | B+ | Pack positive (6.0–8.4 V) through the pack PCM; to TP5100 BAT, R_SH2 (INA219 #2) and D_SYS2 |
| 2 | B− | Pack negative = GND |
| 3 | NTC | 10 kΩ NTC on the cells → TP5100 TEMP |

> [!warning]
> Use only a pack with an integrated 2S protection module and IEC 62133-2 / IS 16046 certified cells. The pack lives in the DU, away from the heating unit and the PSU; keep it clear of the LED ring and the camera window.

### 10.5 J_CAN_HEAT — moved to the CSC

### 10.6 J_CAN_STIR — moved to the CSC

### 10.7 J_STIR_PWR — moved to the CSC

### 10.8 J_DSB1–J_DSB4 — Dispenser Servos (3-pin servo header, 2.54 mm)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | SIG | PCA9685 CHn via 100R |
| 2 | V+ | Switched 6.0 V rail (6V_DSP), all four headers; no series diode |
| 3 | GND | Ground |

J_DSB1 = C1 (CID index, DS3218 270°), J_DSB2 = F1 (FID index, MG996R), J_DSB3 = F2 (FID dose, MG996R), J_DSB4 = F3 (FID steam gate, MG90S).

### 10.9 J_SLD — SLD Pumps, Solenoids and Load Cells (Hirose DF11-16DP-2DS)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | PUMP1_AO1 | TB6612 A1 (oil pump +, 6 V) |
| 2 | PUMP1_AO2 | TB6612 A2 (oil pump −) |
| 3 | PUMP2_BO1 | TB6612 B1 (water pump +, 6 V) |
| 4 | PUMP2_BO2 | TB6612 B2 (water pump −) |
| 5 | 6V_SOL1 | Oil solenoid supply (6V_DSP) |
| 6 | SOL1_OUT | Oil solenoid drain (IRLML6344, PCF8574 #2 P0) |
| 7 | 6V_SOL2 | Water solenoid supply (6V_DSP) |
| 8 | SOL2_OUT | Water solenoid drain (IRLML6344, PCF8574 #2 P1) |
| 9 | LC_OIL_E+ | Oil cell excitation + (NAU7802 oil AVDD, 3.0 V) |
| 10 | LC_OIL_E− | Oil cell excitation − (NAU7802 oil AVSS) |
| 11 | LC_OIL_S+ | Oil cell signal + (NAU7802 oil VIN1P) |
| 12 | LC_OIL_S− | Oil cell signal − (NAU7802 oil VIN1N) |
| 13 | LC_WATER_E+ | Water cell excitation + (NAU7802 water AVDD) |
| 14 | LC_WATER_E− | Water cell excitation − (NAU7802 water AVSS) |
| 15 | LC_WATER_S+ | Water cell signal + (NAU7802 water VIN1P) |
| 16 | LC_WATER_S− | Water cell signal − (NAU7802 water VIN1N) |

Pins 9–16 are the two bridge cables (twisted pairs, shield to GND at the PCB end); the HX711 digital lines of v3.x are gone.

### 10.10 J_FAN — moved to the CSC

### 10.11 J_LED — White Ring (JST-XH 2-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 5V | 5V_MAIN (unswitched) |
| 2 | LED_RET | Ring return → Q_LED drain (IRLML6344, RV1106 PWM_LED) |

### 10.12 J_BUZZER — Piezo Buzzer (JST-PH 2-pin)

| Pin | Signal |
|-----|--------|
| 1 | 5V |
| 2 | BUZZ_OUT (2N7002 drain, RV1106 PWM_BUZZER) |

### 10.13 J_SAFE — moved to the CSC

### 10.14 J_SWD — moved to the CSC

### 10.15 J_USB — moved to the CSC

---

## 11. 4-Layer Stackup and Zone Layout

### 11.1 Layer Stack

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (only break: CAN isolation slot at the dock edge) | 1 oz |
| L3 | Split power plane: 5V/3.3V, 6V_DSP, VSYS/BAT+ (12 V is a local trace) | 1 oz |
| L4 (Bottom) | Signal + power traces, passives under the Pico Zero, J_DOCK pad pattern | 2 oz |

FR-4 Tg ≥ 150 °C, 1.6 mm, ENIG (hard gold on the eight dock pads if the fab offers it).

### 11.2 Board Layout Zones

```
┌──────────────────────────────── 160 mm ────────────────────────────────────────┐
│ ZONE C: DOCK + CAN NODE     │ ZONE A: AP SLOT + DIGITAL              │ ZONE B: BATTERY  │
│ ┌─ ISOLATED ISLAND ──────┐  │ ┌─────────────────────────────┐        │ J_BAT  R_SH2     │
│ │ pads 3-6 │ ISO1050 #2  │  │ │ J_AP 2x20 + 4 standoffs     │        │ INA219 #2        │
│ │ PESD1CAN │ (slot)      │  │ │ (Luckfox Pico Zero 65x30)   │        │ TP5100  L_CHG    │
│ └────────────────────────┘  │ └─────────────────────────────┘        │ CHG LEDs         │ 90 mm
│ pads 1-2, 7-8  SMBJ24A      │ MCP2518FD + Y3   AMS1117   PCF8574 #2  │ D_SYS1/2 → VSYS  │
│ Q_PWR (DOCK_PWR_OK)         │ PCA9685  TCA9543A  2x NAU7802  INA219 #1│ C_SYS 100 µF     │
├─────────────────────────────┼────────────────────────────────────────┼──────────────────┤
│ J_LCD   J_LED   J_BUZZER    │ ZONE D: ACTUATOR DRIVERS               │ ZONE F: BUCKS    │
│ Q_LED   Q_BUZZ              │ TB6612FNG, 2x IRLML6344 + flybacks     │ MP1584EN #1 (12V)│
│                             │ Q_DSB, PTC, R_SH1, 1000 µF             │ MP1584EN #2 (6V) │
│                             │ J_DSB1-4      J_SLD                    │ TPS54531 + L1-L3 │
└─────────────────────────────┴────────────────────────────────────────┴──────────────────┘
  ▲ board edge over the CS left pillar (J_DOCK pads on L4)
```

Placement rationale:
- **Zone C** sits at the board edge that lies over the CS left pillar when the DU is seated: the eight J_DOCK pads are on L4 at that edge, with pads 3–6 inside the isolation island next to ISO1050 #2 and pads 1–2 / 7–8 outside it. The isolation slot is only around ISO1050 #2 and dock pads 3–6; it no longer runs along a whole board edge.
- **Zone A**: the Pico Zero sits on standoffs over the I2C ICs; its SPI0 lines to the MCP2518FD (Zone C edge) and J_LCD are short (< 40 mm). Keep the module's antenna end at the board edge, away from Zone F. NAU7802 analog pins away from Q_DSB and the bucks.
- **Zone B**: J_BAT, R_SH2, INA219 #2, the TP5100 and the VSYS diodes together; the battery cable leaves the board edge towards the pack bay in the DU.
- **Zone F**: three bucks ≥ 25 mm from the Pico Zero and the NAU7802s, thermal pours ≥ 400 mm² each; the TP5100 pour ≥ 300 mm² for 1 A charge.
- **J_DSB1–4 and J_SLD** on the lower edge, nearest the carousels and reservoir bays.

### 11.3 Critical Layout Rules

| Rule | Specification | Rationale |
|------|---------------|-----------|
| CAN isolation | Creepage ≥ 6.4 mm GND ↔ GND_ISO; 6 mm milled slot around ISO1050 #2 and dock pads 3–6; no copper crossing | IEC 60335 reinforced insulation, carried across the dock |
| CAN stub | ISO1050 #2 bus pins to pads 3–4 < 30 mm; no termination on this board | The dock → board run is a stub ≤ 30 cm off the CSC-terminated bus |
| Dock pads | 8 pads, 2.54 mm pitch, ≥ 2.0 mm diameter, on L4 at the edge; ≥ 1.5 mm track to pad 1/2; mechanical keying by the enclosure recess | 2 A per contact, repeatable seating |
| SPI0 to MCP2518FD / J_LCD | Matched within 5 mm, 50 Ω, ground return under the trace | 50 MHz LCD clock |
| Pico Zero clearance | 3 mm keep-out around the module outline on L1; passives only under it on L4 | Mechanical fit, Wi-Fi antenna |
| Battery traces | ≥ 2 mm, B+ fused by the pack PCM; R_SH2 Kelvin-sensed; NTC trace away from the bucks | Up to 3 A discharge, clean current reading |
| VSYS node | ≥ 2 mm, D_SYS1/D_SYS2 within 10 mm of the TPS54531 input cap | Hold-up during switch-over |
| 6V_DSP | ≥ 1.5 mm from Q_DSB to J_DSB1-4 / TB6612 / J_SLD; R_SH1 Kelvin-sensed | 2.5 A servo stall, stall-window accuracy |
| Load-cell inputs | Bridge pairs routed as differential pairs over solid ground, ≥ 5 mm from any switching node | 24-bit ADC, ±1 g |
| Buck loops | Input cap → IC → inductor → output cap loop < 10 mm | EMI |
| Thermal vias | ≥ 6 under each buck IC and the TP5100 | Heat to L2 |

---

## 12. Thermal Analysis

| Component | Condition | Power (W) | Notes |
|-----------|-----------|-----------|-------|
| MP1584EN #1 (12 V) | 0.8 A while charging | ~1.0 | Docked with mains only |
| MP1584EN #2 (6.0 V) | 2.5 A, duty < 10 % | ~0.4 avg | Rail off when idle |
| TPS54531 (5 V from 24 V or the pack) | 0.6 A continuous | ~0.5 | ~87 % efficient at 24 V in; better on the pack |
| TP5100 charger | 1.0 A at 8.4 V from 12 V | ~1.0 | Charging only; pour per §11.3 |
| TB6612FNG | 1.2 A, intermittent | ~0.7 | |
| AMS1117-3.3 | 0.15 A | ~0.25 | |
| ISO1050 #2, MCP2518FD, NAU7802 x2, INA219 x2 | — | ~0.3 | |
| SS54 x2 (D_SYS1/2), IRLML6344 x3, Q_DSB | — | ~1.0 | Diode drops dominate |
| **Total** | | **~5 W typical** | At 40 °C ambient every IC stays below 110 °C junction with the pours in §11.3; the DU is the cool unit |

The Pico Zero (~1 W) dissipates through its own board; leave 10 mm of free air above it.

---

## 13. Unified BOM (changes highlighted)

| Ref | Part | Package | Qty | Unit (USD) | Notes |
|-----|------|---------|-----|-----------|-------|
| **AP slot + RV1106 CAN node** | | | | | |
| J_AP | 2x20 socket, 2.54 mm, 8.5 mm | TH | 1 | $0.80 | Pico Zero slot |
| STANDOFF x4 | M2.5 x 8 mm + screws | — | 4 | $0.10 | |
| U_CAN_AP | MCP2518FD-E/SL | SOIC-14 | 1 | $2.50 | RV1106 CAN node |
| Y3 | 40 MHz crystal + 2x 15 pF | SMD | 1 | $0.50 | |
| U_CAN2 | ISO1050DUB | SOIC-8 | 1 | $3.50 | **Bus side from dock 5V_ISO** |
| PESD1CAN, C_CAN x4 | — | — | 1 lot | $0.30 | |
| **Dock** | | | | | |
| J_DOCK | **8 gold pads on L4 (no part)** | PCB | — | — | **Mating pogo block in the system BOM** |
| D_TVS1, C_IN | SMBJ24A, 10 µF 50 V | SMB, 1206 | 2 | $0.35 | **DOCK_24V hot-plug** |
| Q_PWR, R_DIV1-2, R_PU x2 | 2N7002, 100k, 10k | SOT-23, 0402 | 1 lot | $0.08 | **DOCK_PWR_OK, DOCK_SENSE_A** |
| **Power conversion** | | | | | |
| U_BUCK1-2 | MP1584EN | SOT-23-8 | 2 | $0.80 | 12 V (charger), **6.0 V from VSYS** |
| U_BUCK3 | TPS54531DDAR | SOIC-8 | 1 | $1.50 | VSYS → 5 V |
| U_LDO | AMS1117-3.3 | SOT-223 | 1 | $0.30 | |
| L1, L2, L3 | 33 µH, 22 µH (CDRH104R), 10 µH (CDRH127) | — | 3 | $0.55 | |
| D_SYS1, D_SYS2 | SS54 | SMA | 2 | $0.20 | VSYS diode-OR (**DOCK_24V** / battery) |
| C_SYS | 100 µF, 35 V electrolytic | 8x10 | 1 | $0.25 | VSYS hold-up |
| **Battery (primary supply)** | | | | | |
| U_CHG | TP5100 | QFN-16 | 1 | $0.40 | 2S switching charger, **1.0 A** |
| L_CHG | 22 µH, Isat > 2 A, shielded | 6x6 | 1 | $0.30 | |
| D_CHG | SS34 | SMA | 1 | $0.15 | Charger catch diode |
| R_CS | **0.1 Ω 1 % 0.5 W** | 2010 | 1 | $0.10 | Charge current sense |
| U_MON2, R_SH2 | **INA219BIDR, 10 mΩ 2512** | — | 2 | $0.90 | **Pack voltage + current, 0x44** |
| R_NTC_BIAS, R_CHRG_PU, R_STBY_PU | 10k | 0402 | 3 | $0.01 | |
| D_CHG_LED, D_STB_LED + 1k | Red / blue 0603 | 0603 | 2 | $0.03 | |
| C_CHG_IN, C_CHG_OUT | 10 µF 25 V, 22 µF 16 V | 0805 | 2 | $0.12 | |
| J_BAT | JST-XH 2.5 mm 3-pin | TH | 1 | $0.10 | B+, B−, NTC |
| (pack) | 2S1P 18650 pack with PCM, 2,600 mAh, NTC | — | 1 | $12.00 | Not on the PCB; system BOM |
| **Dispenser Servo Block** | | | | | |
| U_PWM | PCA9685PW | TSSOP-28 | 1 | $1.50 | I2C3 0x41 |
| Q_DSB, Q_DSB_N | AO3401 + 2N7002 | SOT-23 | 2 | $0.15 | 6.0 V rail switch |
| F_DSB, C_DSB, D_TVS2 | 3 A PTC, 1000 µF 10 V, SMBJ6.0A | — | 3 | $0.60 | |
| U_MON1, R_SH1 | INA219BIDR, 10 mΩ 2512 | — | 2 | $0.90 | **Moved to the 6V_DSP rail, 0x40** |
| J_DSB1-4 | 3-pin servo header | TH | 4 | $0.10 | **D_DSB4 removed** |
| **SLD** | | | | | |
| U_TB | TB6612FNG | SSOP-24 | 1 | $1.50 | VM = 6V_DSP |
| Q_SOL1-2, D_FB1-2 | IRLML6344, SS14 | SOT-23, SMA | 4 | $0.15 | 6 V solenoids |
| U_MUX | **TCA9543A** | TSSOP-14 | 1 | $1.00 | **I2C mux 0x71** |
| U_ADC1-2 | **NAU7802SGI** | SOIC-16 | 2 | $2.00 | **24-bit load-cell ADCs, 0x2A** |
| C_NAU x6, R_NAU x4 | 10 µF, 100 nF, 1k | 0603/0402 | 1 lot | $0.15 | |
| D_ESD1-2 | PRTR5V0U2X | SOT-143B | 2 | $0.15 | Bridge sense lines |
| J_SLD | Hirose DF11-16DP-2DS | TH | 1 | $0.80 | Re-pinned |
| **LED, buzzer** | | | | | |
| Q_LED | **IRLML6344** | SOT-23 | 1 | $0.06 | **White ring low-side, RV1106 PWM** |
| Q_BUZZ | 2N7002 | SOT-23 | 1 | $0.05 | RV1106 PWM |
| J_LED, J_BUZZER | **JST-XH 2**, JST-PH 2 | TH | 2 | $0.08 | |
| **I2C expander** | | | | | |
| U_EXP2 | PCF8574 | SOIC-16 | 1 | $0.50 | 0x21, now on I2C3 |
| **Protection** | | | | | |
| F_12V, D_TVS3 | 1.5 A polyfuse, SMBJ12A | — | 2 | $0.45 | 12 V rail |
| **Display break-out** | | | | | |
| J_LCD | 1x14 header 2.54 mm | TH | 1 | $0.15 | |
| R_I2C3 x2, R_SER x12 | 4.7k, 33R | 0402 | 14 | $0.01 | |
| **Passives** | decoupling, pull-ups/downs, gate resistors, feedback dividers, crystal caps | — | ~90 | — | ~$2.00 |
| **PCB** | 4-layer 160 x 90 mm, ENIG, 2 oz outer, dock pads | — | 1 | $5.50 | JLCPCB, 5 pcs |

Removed from this board in v4.0 (to the CSC or deleted): STM32G0B1RET6 + crystals, ISO1050DUB #1, B0505S-1WR3, SN65HVD230, Q_STIR/F_STIR/C_STIR, Q_RLY + flyback, fan MOSFETs, 74AHCT1G125, Q_LED_P/Q_LED_N, 2x HX711 breakouts, D_DSB4, R_BAT1/R_BAT2, USB-C lot, J_24V_IN, J_CAN_HEAT, J_CAN_STIR, J_STIR_PWR, J_FAN, J_SAFE, J_SWD, J_USB, SW_BOOT/SW_RST.

### 13.1 Cost Summary

| Category | Prototype (1x) | Volume (100x) |
|----------|----------------|---------------|
| AP slot + RV1106 CAN node (socket, standoffs, MCP2518FD, crystal, ISO1050 #2) | $7.70 | $5.50 |
| Dock (TVS, sense) | $0.45 | $0.30 |
| Power conversion (3 bucks, LDO, VSYS OR, passives) | $7.20 | $4.60 |
| Battery charger + INA219 #2 | $2.30 | $1.60 |
| Dispenser Servo Block + INA219 #1 | $3.70 | $2.60 |
| SLD drivers (TB6612, solenoids, TCA9543A, 2x NAU7802) | $5.90 | $4.00 |
| LED / buzzer / expander / protection | $1.20 | $0.80 |
| Display break-out, passives | $2.30 | $1.40 |
| Connectors (11) | $2.60 | $1.70 |
| PCB | $5.50 | $2.50 |
| **Board total** | **~$39** | **~$25** |
| Battery pack (system BOM, off-board) | $12.00 | $8.00 |
| 8-pin magnetic pogo pair (system BOM, with the CS pillar) | $6.00 | $4.00 |

Compared with v3.1 (~$62): the MCU, the second ISO1050, B0505S, SN65HVD230, motor switch, relay, fans, USB and eight connectors (≈ −$27) leave for the CSC or disappear; the NAU7802 pair, TCA9543A and INA219 #2 (≈ +$4) join. The CSC (≈ $25 PCB + parts) and the pogo pair are added to the system BOM; the net system delta is ≈ +$32 (see the project BOM).

---

## 14. Manufacturing Specifications

| Parameter | Value |
|-----------|-------|
| Layers / thickness | 4 / 1.6 mm |
| Copper | 2 oz outer, 1 oz inner |
| Min trace / space | 0.15 mm signal; power per §11.3 |
| Min via | 0.3 mm drill, 0.6 mm pad |
| Finish | ENIG; hard gold on the J_DOCK pads if offered |
| Material | FR-4, Tg ≥ 150 °C |
| Dimensions | 160 x 90 mm |
| Mounting | 4x M3 corner holes; 4x M2.5 holes for the Pico Zero standoffs (match the module's 3.0 mm hole pattern, 65 x 30 mm outline) |
| Milling | 6 mm CAN isolation slot around ISO1050 #2 and dock pads 3–6 |
| Impedance | 50 Ω SPI0 |

### 14.1 Assembly Notes

- SMT both sides (passives under the Pico Zero on L4), reflow; then hand solder TH: J_AP, J_LCD, J_SLD, JST-XH/PH, J_DSB x4, electrolytics
- Fit the Pico Zero standoffs before populating J_AP so the socket height is set by the standoffs
- Keep solder mask and silkscreen off the eight J_DOCK pads; label the pad row "DOCK — pads 3-6 ISOLATED"
- Pin 1 silkscreen triangles on all connectors

---

## 15. Firmware Integration Notes (RV1106 / `epicurad`)

### 15.1 Thread Allocation Touching This Board

| Thread | Rate | Functions |
|--------|------|-----------|
| `dispense` (new, SCHED_OTHER) | 50 Hz while active | PCA9685 ramped moves (≤ 3°/tick) with OE and 6V_DSP enable, one servo at a time; stall = INA219 #1 window + 3 s timeout + I2C NACK → retry once → DSB_STALL; SLD closed loop on the NAU7802 at 10 Hz, stop at 95 % of target, solenoid + pump sequencing; tare, preflight, home, status. Internal queue from the recipe engine with the former CAN command names (DISPENSE_FID/CID/SLD, QUERY_WEIGHT, PREFLIGHT, TARE, DISPENSER_HOME, DISPENSER_STATUS) |
| sensor path: power/dock monitor | 1 Hz + GPIO edges | INA219 #2 (batt_mV, batt_mA, state of charge), PCF8574 #2 P6/P7 (charging / complete), DOCK_SENSE_A (docked), DOCK_PWR_OK (dock_power); battery indicator always visible in the UI |
| `can` | event | MCP2518FD SocketCAN `can0`; bus-off while lifted is expected |
| UI / LED / buzzer | event | PWM_LED duty OFF / CAPTURE / NIGHT; buzzer tones on PWM_BUZZER |
| swupdate handler | on demand | STM32 image streamed over CAN (0x7E0 FW_UPDATE_REQ / 0x7E8 FW_UPDATE_RSP) to the CSC's in-application bootloader; replaces UART2 + BOOT0/NRST |

### 15.2 RV1106 Peripheral Allocation

| Peripheral | Header | Function |
|------------|--------|----------|
| SPI0 CS0 / CS1 | 19/21/23/24 + GPIO 58 | ST7796S LCD; MCP2518FD (500 kbps `can0`) |
| I2C3 | 16/18 | PCF8574 #2 0x21, PCA9685 0x41, INA219 0x40 / 0x44, TCA9543A 0x71 → NAU7802 0x2A x2, FT6336 0x38 |
| GPIO x3 | fixed at layout | PCA9685 OE (out), DOCK_SENSE_A (in, edge), DOCK_PWR_OK (in, edge) |
| PWM x4 | fixed at layout | TB6612 PWMA / PWMB, Q_LED, buzzer |
| PWM10 | 12 | LCD backlight |
| GPIO | 22, 26, 36, spare | LCD DC / RST, touch INT / RST, MCP2518FD nINT |

### 15.3 Power-Fail and Lift Sequences

1. **CS mains lost while docked:** DOCK_PWR_OK goes inactive with DOCK_SENSE_A still low; confirmed by a 300 ms TELEMETRY timeout → PAUSED (AC Power Failure). VSYS is already on the pack (passive OR, no reboot). The heater and stir motor are down on the CS side. Auto-resume to RESUMING when TELEMETRY returns within 5 min (the CSC reboots in < 1 s), prompt if < 30 min, ERROR beyond.
2. **Unit lifted:** DOCK_SENSE_A goes high → PAUSED (UNIT_LIFTED); LED ring OFF, dispensing refused; `can0` goes bus-off. Resume only by user tap after re-docking (DOCK_SENSE_A low, TELEMETRY back); same 30 min food-safety ceiling.
3. **Battery low:** warning banner at 6.4 V; at 6.2 V checkpoint + halt (RECOVERING on the next boot). The pack PCM at 5.6 V is the last resort.

### 15.4 Battery Charger Behaviour

- Charging runs autonomously whenever the 12 V rail is up (docked, CS mains present); `epicurad` only reads CHRG/STDBY and INA219 #2
- Fault indication: CHRG and STDBY both inactive with the pack below 8.0 V for > 10 min while DOCK_PWR_OK is active → battery fault banner (pack missing, NTC out of range, or charger fault); cooking is still allowed
- Self-test at boot: INA219 #2 bus voltage between 5.5 V and 8.5 V, else the same banner

---

## 16. Design Verification Checklist

### 16.1 Pre-Fabrication

- [ ] ERC passes; no STM32 symbol or net left on the board
- [ ] J_AP pinout matches the Luckfox Pico Zero 40-pin header; SPI0_CS1, the five spare GPIO (MCP2518FD INT, TP_RST, PCA9685 OE, DOCK_SENSE_A, DOCK_PWR_OK) and the four PWM confirmed against the Luckfox pinout; pins 8/10/32/38 unconnected or reused
- [ ] Pico Zero outline, 3.0 mm hole pattern and standoff height checked against the module drawing
- [ ] J_LCD pin order matches the chosen 4" ST7796S/FT6336 module
- [ ] Isolation: only the ISO1050 #2 bus pins cross the slot; dock pads 3–6 inside the island, pads 1–2 / 7–8 outside; no termination resistor on this board
- [ ] J_DOCK pad pattern matches the pogo block (2.54 mm pitch, pad 1 position vs the keyed recess) and the DU underside drawing
- [ ] TP5100 strapped for 2S (8.4 V), R_CS = 0.1 Ω, NTC bias per datasheet, CHRG/STDBY pulled up to 3.3 V
- [ ] VSYS diode-OR: D_SYS1 from DOCK_24V, D_SYS2 from B+ via R_SH2; TPS54531 feedback 4.98 V; MP1584EN #2 feedback 5.99 V from VSYS; C_SYS rated 35 V
- [ ] INA219 #1 A0/A1 → 0x40 on 6V_DSP; INA219 #2 → 0x44 on BAT+; both shunts Kelvin-connected
- [ ] TCA9543A address vs PCA9685 All-Call (§3.1 note) resolved; NAU7802 AVDD decoupling per datasheet
- [ ] Q_DSB default OFF (gate pull-up), driven by PCF8574 #2 P4 through 2N7002; PCA9685 OE 10k pull-up, A0 = 1 → 0x41; PCF8574 #2 A0 = 1 → 0x21
- [ ] All MOSFET gates: 100R series + 10k pull-down (or pull-up for P-MOSFETs); all four PWM lines pulled down
- [ ] DOCK_24V: SMBJ24A + 10 µF at the pad; DOCK_PWR_OK divider 100k/10k; DOCK_SENSE_A 10k pull-up; DOCK_SENSE_B to GND
- [ ] All 11 connectors have correct pinouts and silkscreen; J_SLD pins 9–16 per §10.9

### 16.2 PCB Layout

- [ ] DRC passes; GND plane continuous except the CAN slot around ISO1050 #2 and dock pads 3–6
- [ ] Creepage ≥ 6.4 mm across the isolation slot and across the dock pad row (pads 2 ↔ 3, 6 ↔ 7)
- [ ] Pico Zero keep-out respected; standoff holes placed; antenna end at the board edge away from the bucks
- [ ] SPI0 traces to MCP2518FD and J_LCD matched within 5 mm
- [ ] Battery, VSYS and 6V_DSP trace widths per §11.3; shunt Kelvin traces
- [ ] Load-cell bridge pairs routed differentially over solid ground
- [ ] Buck loops < 10 mm; thermal vias and pours present (incl. TP5100)
- [ ] J_DOCK pads on L4 at the pillar edge, mask-free, matching the mechanical drawing

### 16.3 Post-Assembly

- [ ] 3.3 V, 5 V rails within ±3 % on the pack alone (no dock); 12 V and 6.0 V within ±3 % with DOCK_24V applied
- [ ] Pico Zero boots in J_AP from the pack; LCD renders through J_LCD; touch events on I2C3
- [ ] I2C3 scan finds 0x21, 0x38, 0x40, 0x41, 0x44, 0x71; 0x2A appears behind each TCA9543A channel
- [ ] `can0` (MCP2518FD) exchanges frames with the CSC's FDCAN1 at 500 kbps through a bench pogo block; HEAT_QUERY to the heating unit returns HEAT_STATUS
- [ ] DSB: each servo reaches the commanded angle on 6.0 V; rail switches off after a move; OE high leaves servos limp; INA219 #1 shows the move current
- [ ] SLD: pumps run both directions on 6 V; solenoids click; both NAU7802 channels stable ±1 g
- [ ] White ring: PWM 0 / 30 / 100 % give OFF / NIGHT / CAPTURE; ≤ 1 A at 100 %
- [ ] Battery: pack charges to 8.4 V at ~1.0 A from the 12 V rail; CHRG then STDBY read correctly on P6/P7; INA219 #2 voltage within 2 % and current sign correct (charge vs discharge)
- [ ] Pull DOCK_24V during a cook simulation: 5V_MAIN stays within ±5 % with no reboot of the Pico Zero; DOCK_PWR_OK goes inactive; UI shows "Power lost — running on battery"
- [ ] Lift the DU from the bench block: DOCK_SENSE_A goes high, `can0` bus-off, UI shows UNIT_LIFTED; re-dock → bus recovers, resume on tap
- [ ] Battery run-time ≥ 5 h at the lifted-unit load; halt at 6.2 V, recovery on the next boot
- [ ] NTC disconnected → charger stops (CHRG/STDBY both inactive) → battery fault banner
- [ ] All actuators, the 6V_DSP rail, the ring and the buzzer off with the Pico Zero held in reset or removed

---

## 17. Related Documentation

- [[02-Cooking-Surface-Controller|Cooking Surface Controller]] — the CS board: STM32G0B1RE, both CAN transceivers, relay, E-stop, fans, dock power switch
- [[../02-Hardware/01-Epicura-Architecture|Epicura Architecture]] — three units, system wiring, CAN bus, dock, power
- [[../02-Hardware/02-Technical-Specifications|Technical Specifications]] — power specs incl. the DU battery
- [[../05-Subsystems/01-Electric-Heating|Electric Heating]] — module with internal temperature control, isolated CAN port
- [[../05-Subsystems/02-Robotic-Arm|Robotic Arm]] — CAN servo stirring motor (SU)
- [[../05-Subsystems/03-Ingredient-Dispensing|Ingredient Dispensing]] — DSB and SLD
- [[../03-Software/05-MCU-Functional-Specification|MCU Functional Specification]] — STM32 firmware requirements (CSC)
- [[../03-Software/02-Controller-Software-Architecture|Controller Software Architecture]] — CAN ID map (§7.2), `epicurad` threads
- [[03-PCB-Design-Rules|PCB Design Rules]] — DRC rules and net classes
- Archived: [[_archive/01-Controller-PCB-Design|Controller PCB]] and [[_archive/02-Driver-PCB-Design|Driver PCB]] (2-board era)

---

#epicura #pcb #unified #dispensing-unit #rv1106 #can #battery #dock #power-electronics #hardware-design

---

## 18. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 4.1 | 2026-11-04 | Manas Pradhan | Heater node on the isolated bus is now the commercial electric heating unit (rice-cooker class, own PID, CAN port) instead of the induction surface: §1 overview, scope table, CAN node table, dock lift-off and battery placement notes, bring-up checklist reworded; HEAT_STATUS `coil_temp` → `element_temp`. Same CAN contract, no schematic or BOM change |
| 4.0 | 2026-11-04 | Manas Pradhan | Three-unit architecture: this board becomes the Dispensing Unit board; the STM32G0B1RE and everything only it used (crystals, J_SWD, J_USB, USART2 service link, BOOT0/NRST from the Pico Zero), ISO1050DUB #1, B0505S, SN65HVD230, J_CAN_HEAT, J_CAN_STIR, J_STIR_PWR, Q_STIR, J_FAN + fan MOSFETs, Q_RLY, J_SAFE, the 24 V comparator divider, 2x HX711, 74AHCT1G125, WS2812B ring, the battery ADC divider and D_DSB4 leave the board (most to the new Cooking Surface Controller, [[02-Cooking-Surface-Controller\|02-Cooking-Surface-Controller]]). Added: J_DOCK (8 pogo pads: DOCK_24V, GND, isolated CAN_H/CAN_L, GND_ISO, 5V_ISO, DOCK_SENSE_A/B), INA219 #2 (0x44) in the BAT+ path, TCA9543A (0x71) + 2x NAU7802 (0x2A) load-cell ADCs, white LED ring on Q_LED (RV1106 PWM, 2-pin J_LED), DOCK_PWR_OK and DOCK_SENSE_A sense to RV1106 GPIO. The battery is the primary supply: VSYS = DOCK_24V ⊕ BAT+ (same diode-OR), TP5100 charge current 1.0 A (R_CS 0.1 Ω) from a 12 V rail that now feeds the charger only; 6.0 V switched dispenser rail (6V_DSP) from VSYS for all four servos, 6 V pumps and 6 V solenoids; INA219 #1 (0x40) moved to 6V_DSP; 5V_ISO from the dock, no isolated DC-DC. RV1106 I2C3 is the single peripheral bus (FT6336, PCF8574 #2, PCA9685, INA219 x2, TCA9543A/NAU7802); 3 new GPIO (PCA9685 OE, DOCK_SENSE_A, DOCK_PWR_OK) and 4 PWM (pumps, ring, buzzer); header pins 8/10/32/38 freed; STM32 OTA over CAN 0x7E0/0x7E8. §3 rewritten as the RV1106 interface with the PCF8574 #2 map (P3/P5 = TB6612 AIN1/BIN1); J_SLD re-pinned for bridge excitation/sense; 11 connectors; zones with the dock edge and a local isolation slot; BOM, thermal, firmware notes and checklists rewritten |
| 3.1 | 2026-11-04 | Manas Pradhan | DSB servos sized per axis (DS3218 270° on C1, MG996R on F1/F2, MG90S on F3; spec dispensing v7.3 / architecture v8.2): D_DSB4 (M7) added in series on the J_DSB4 V+ pin to drop the 6.5 V rail to ≈ 5.8 V for the 6 V-rated MG90S; §4.3 load, §6.1, §10.8 and BOM updated; no other board change |
| 3.0 | 2026-11-04 | Manas Pradhan | Single-board redesign around the Luckfox Pico Zero: J_AP slot (2x20 socket + standoffs) with SPI0/I2C3 break-out to J_LCD, UART2 service link, BOOT0/NRST control and the RV1106 CAN node (MCP2518FD + ISO1050DUB #2); CID + FID drivers are the Dispenser Servo Block (PCA9685, switched 6.5 V rail, J_DSB1-4) and SLD keeps TB6612/solenoids/HX711 on J_SLD; two CAN ports — isolated J_CAN_HEAT (FDCAN1, 2x ISO1050DUB, B0505S) for the cooking surface and regular J_CAN_STIR (FDCAN2 on PB12/PB13, SN65HVD230, 1 Mbps) plus switched 24 V (Q_STIR, J_STIR_PWR) for a CAN servo stirring motor, replacing PWM/EN/DIR on PA8/PA4/PA5; MLX90614 IR thermometer and J_IR_LED removed (the surface runs its own temperature loop; J_LED carries the ring only); 12 V UPS input and single-cell TP4056/MT3608 micro-UPS replaced by a 2S Li-ion pack (J_BAT 3-pin with NTC) charged by a TP5100 from the 12 V rail, with a VSYS diode-OR (24 V / battery) feeding the TPS54531 so the entire 5 V logic domain rides through outages; battery voltage on PC1, charger status on PCF8574 #2 P6/P7, motor power enable on P5; P-ASD, CID stepper/linear actuator, PCF8574 #1, ADS1015, DRV8876 and their connectors removed; 19 connectors; BOM, thermal, zones and checklists rewritten |
| 2.0 | 2026-03-18 | Manas Pradhan | Added Mornsun B0505S-1WR3 isolated DC-DC for the ISO1050 bus side; J_CAN pin 4 = 5V_ISO |
| 1.9 | 2026-03-11 | Manas Pradhan | Added PCF8574 #2 (0x21) for GPIO headroom; SLD solenoids, status LED and LED ring enable moved to it |
| 1.8 | 2026-03-11 | Manas Pradhan | CID-1 DC linear actuator replaced by NEMA 23 stepper + external driver; J_CID 14-pin |
| 1.7 | 2026-03-10 | Manas Pradhan | Six connectors converted to dual-row Hirose DF11 |
| 1.6 | 2026-03-10 | Manas Pradhan | CID limit switches added |
| 1.5 | 2026-03-10 | Manas Pradhan | J_PRES pressure-sensor connector added |
| 1.4 | 2026-03-10 | Manas Pradhan | Datasheet fixes: PA1 COMP1, buzzer on PC7/TIM8_CH2, status LED PC8 |
| 1.3 | 2026-03-10 | Manas Pradhan | I2C1_SCL PB6 → PA15 |
| 1.2 | 2026-03-09 | Manas Pradhan | USB-C programming port and single-cell Li-Ion micro-UPS added; 3-way 5 V Schottky-OR |
| 1.1 | 2026-03-07 | Manas Pradhan | Stacking order swapped: Unified PCB above CM5IO |
| 1.0 | 2026-03-06 | Manas Pradhan | Initial unified document — Controller PCB (v11.0) and Driver PCB (v8.0) merged |
