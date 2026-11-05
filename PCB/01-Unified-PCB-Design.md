---
created: 2026-03-06
modified: 2026-11-05
version: 5.3
status: Draft
---

# Unified PCB Design

## 1. Overview

Epicura is physically three units: the **Cooking Surface (CS)** with the electric heating unit, PSU, safety relay, E-stop and fans; the **Stirring Unit (SU)** on the gantry between the CS rear pillars; and the **Dispensing Unit (DU)** that sits on top, lifts straight off with no cable, and carries the brain of the machine. This document covers the **Unified PCB v5.0, the DU board**. The CS has its own board, the **Cooking Surface Controller (CSC, STM32G0B1RE)**, documented in [[02-Cooking-Surface-Controller|Cooking Surface Controller]]. The only cable between units is the CS → SU harness; the DU meets the CS through an 8-pad magnetic pogo dock (J_DOCK).

One 4-layer board carries everything electrical in the DU. Since v5.3 (2026-11-05) the DU is a Ø350 mm disc and the board is an **annular sector** in its outer service ring (preliminary R 135–172 mm × ≈ 150°, ≈ 16,000 mm²; outline to be fixed with the DU CAD) instead of the 160 x 90 mm rectangle:

- **Application-processor slot (J_AP)** for the Luckfox Pico Zero (RV1106G3): 5 V power, SPI0/I2C3 break-out to the display (LCD + touch), the camera on its own CSI FPC, and a UART host link (UART2, with BOOT0/NRST control) to the DU STM32. Nothing else on the board hangs off the Pico Zero header
- **DU STM32 (STM32G0B1RET6)**, the same part as on the CSC, added back in v5.0: it is the DU's CAN node (FDCAN1 + a non-isolated SN65HVD230, since v5.2) on the bus that reaches the CS through the dock and is isolated there, at the CSC's port, a transparent gateway between that bus and the RV1106's host link, and the owner of every dispenser driver (I2C1 + timers), the LED ring, the buzzer and the battery/dock monitors
- **CID + FID drivers**: the Dispenser Servo Block (PCA9685 on the STM32's I2C1, switched 6.0 V rail, J_DSB1-4)
- **SLD drivers**: TB6612FNG for the two 6 V pumps (peristaltic for oil, diaphragm for water) and six Hall-switch inputs (pump tachs, tank present, tank low) on a 12-pin J_SLD — the robot-mop liquid system of v5.3; no solenoid valves and no load cells
- **Dock (J_DOCK)**: 8 flat gold pads on the underside — DOCK_24V charge input, CAN_H/CAN_L with GND_ISO (tied to the DU ground here as the bus reference) and a reserved 5V_ISO pad, two dock-sense lines
- **2-cell Li-ion pack as the primary supply** with on-board 1 A charger (TP5100 from DOCK_24V) and a passive VSYS diode-OR, so the DU runs docked from the CS and lifted (or through a mains outage) from its pack
- Power conversion (VSYS → 5 V, 3.3 V and 6.0 V; DOCK_24V → 12 V for the charger only), white LED ring, buzzer, battery/dock monitoring (INA219 x2), J_SWD for the DU STM32

The MCU is back on this board since v5.0 (2026-11-04): the Pico Zero header does not have the I/O to drive the camera, the LCD and three dispensers cleanly, so the DU STM32 takes the dispensers, the ring, the buzzer, the battery/dock monitors and the CAN node, and the RV1106 keeps camera, display, Wi-Fi and one UART. The DU STM32 has no safety authority: it forwards the RV1106's frames and never originates HEAT_ON_OFF or E_STOP. The safety relay, E-stop, pot reed, exhaust fans, the regular (stirring-motor) CAN port, the isolated DC-DC and the USB-C DFU port stay on the CSC in the CS base. There is no IR thermometer on the board or in the system: the cooking surface runs its own temperature loop and reports its measured temperature on the isolated CAN bus (see [[../05-Subsystems/01-Electric-Heating|Electric Heating]]).

### 1.1 Board Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Unified PCB v5.3 — Dispensing Unit (annular sector, 4-layer)            │
│                                                                          │
│   ┌──────────────────────────┐   ┌───────────────┐   ┌────────────────┐  │
│   │ J_AP: Luckfox Pico Zero  │   │ 2S Li-ion     │   │ J_DOCK (8 pads)│  │
│   │ slot (2x20 socket +      │   │ TP5100 1 A    │   │ DOCK_24V, GND  │  │
│   │ 4 standoffs)             │   │ J_BAT, INA219 │   │ CAN_H/L, GND_ISO│ │
│   └──┬───────────────────────┘   └───────┬───────┘   │ 5V_ISO, SENSE  │  │
│      │UART2 + BOOT0/NRST (host link)     │BAT+       └───┬────────┬───┘  │
│      ▼                                   ▼               │CAN     │24 V  │
│   ┌──────────────────────────────┐   VSYS = DOCK_24V ⊕ BAT+ ◄──────┘      │
│   │ DU STM32 (G0B1RET6)          │                      │                 │
│   │ USART1 ◄─► host link         │                      │                 │
│   │ FDCAN1 ──► SN65HVD230 ───────┼──► dock pads 3/4 ◄───┘ (no isolation)  │
│   │ I2C1 ──► DSB / SLD / INA219  │   (DU CAN node + gateway)              │
│   │ TIM3/TIM1/TIM14 ──► pumps,   │                                        │
│   │   LED ring, buzzer; J_SWD    │                                        │
│   └──────────────────────────────┘                                        │
│                                                                          │
│   DSB: PCA9685 (I2C1) + 6.0 V rail ──► J_DSB1-4 (FID x3, CID x1 servos)   │
│   SLD: TB6612FNG pumps x2 + 6 Hall inputs (tach/present/low) ──► J_SLD    │
│   White LED ring (Q_LED, PWM), buzzer (PWM), J_LCD break-out (SPI0/I2C3)  │
└──────────────────────────────────────────────────────────────────────────┘
          ▲ dock pads (CS left pillar)                ▲ battery pack (2S2P)
```

### 1.2 Scope

| Item | Status | Notes |
|------|--------|-------|
| **Unified PCB v5.3** (this document) | Custom design required | DU board: AP slot + DU STM32 (CAN node, gateway, dispenser drivers, SLD Hall inputs) + battery-first power + dock pads; annular sector in the DU service ring, 4-layer |
| **Cooking Surface Controller (CSC)** | Custom design required — see [[02-Cooking-Surface-Controller\|Cooking Surface Controller]] | STM32G0B1RE, both CAN transceivers for the CS side, relay, E-stop, fans, dock power switch; in the CS base |
| **Luckfox Pico Zero** | Off-the-shelf module | Plugs into J_AP; camera on its own CSI FPC; display routed through the PCB to J_LCD |
| **Dock pogo block** | Off-the-shelf 8-contact magnetic pogo pair (2.54 mm pitch) | Spring pins in the CS left pillar, flat pads on this board (J_DOCK) |
| **Electric heating unit** | Commercial rice-cooker-class heating unit (resistive plate, own PID, CAN port; 1,300 W assumed) (see [[../05-Subsystems/01-Electric-Heating\|Electric Heating]]) | On the same isolated bus, reached through the dock and the CSC's J_CAN_HEAT |
| **Stirring motor** | 24 V integrated BLDC servo with CAN interface (see [[../05-Subsystems/02-Robotic-Arm\|Robotic Arm]]) | Not connected to this board: the CSC's second isolated CAN port (ISO1050DUB #3) and the Stirring Unit's own floating 24 V (LRS-75-24 #2, cut by K_SU) on the J_SU harness |
| **Battery pack** | 2S2P Li-ion 18650 pack, 5,200 mAh, with protection module (PCM) | Primary supply of the DU; charged and monitored on the PCB; connects to J_BAT |

### 1.3 Changes from v4.x

| Area | v4.x (DU board, no MCU) | v5.0 (DU board with the DU STM32) |
|------|-------------------------|-----------------------------------|
| MCU | None: the RV1106 drove every peripheral directly | **STM32G0B1RET6 back on the board** (same part as the CSC): dispensers, SLD sensing, LED ring, buzzer, battery/dock monitors and the CAN node; no safety authority |
| CAN node | MCP2518FD on J_AP SPI0 CS1 + nINT, 40 MHz crystal (Y3), Linux `mcp251xfd` / SocketCAN `can0` | **DU STM32 FDCAN1 (PB8/PB9) → ISO1050DUB #2** (→ SN65HVD230 in v5.2, next row); MCP2518FD, Y3, SPI0 CS1 and nINT removed. Same bus, same node name (AP), same ID map; the STM32 is a transparent gateway to the RV1106 |
| CAN transceiver (v5.2) | ISO1050DUB #2 on the dock's 5V_ISO / GND_ISO, isolated island + 6 mm slot (v4.x–v5.1) | **SN65HVD230 on 3.3 V, referenced to the DU ground**; isolation is on the CSC's two ports only, so this board has no island, slot or isolated domain. Dock pad 5 (GND_ISO) is tied to GND next to the pads (bus reference), pad 6 (5V_ISO) is reserved / not connected |
| RV1106 ↔ MCU link | — (no UART; the STM32 was in another unit) | **Host link**: J_AP UART2 (pins 8/10) ↔ STM32 USART1 (PA9/PA10), 921,600 baud, COBS + CRC-16 frames carrying CAN-shaped messages; GPIO 32 → BOOT0, GPIO 38 → NRST restored for flashing through the ST system bootloader |
| Programming | Nothing to program on the board | J_SWD (2x5 1.27 mm) restored for the bench; field update by the RV1106 over the host link; still no USB-C on the board |
| RV1106 I2C3 | Every I2C device on the board (touch, PCF8574 #2, PCA9685, INA219 x2, TCA9543A/NAU7802) | **FT6336 touch only** |
| Peripheral I2C | — | **STM32 I2C1 (PB6/PB7)** carries PCF8574 #2, PCA9685, INA219 x2 at the same addresses (the TCA9543A → NAU7802 x2 of v5.0–v5.2 are gone in v5.3) |
| SLD (v5.3) | 6 V peristaltic pumps x2, 6 V NC solenoids x2 (IRLML6344 on PCF8574 #2 P0/P1), 2x NAU7802 behind a TCA9543A, 16-pin J_SLD | **Robot-mop liquid system**: drawer tanks with a bottom self-sealing coupling, pumps in the body (6 V peristaltic oil, 6 V diaphragm water) each with a Hall tach, passive check valve in each nozzle cartridge, float + magnet low-level and fixed-magnet presence Hall switches per tank. Open-loop metering by tach pulses × cal_ul in the DU STM32; six GPIO (PC0–PC5), 12-pin J_SLD; solenoids, load cells, NAU7802 and TCA9543A removed |
| Outline (v5.3) | 160 x 90 mm rectangle | Annular sector in the Ø350 DU's service ring (R 135–172 mm × ≈ 150°, TBC); same schematic |
| PWM / GPIO | RV1106 PWM x4 (pumps, ring, buzzer) and GPIO x3 (PCA9685 OE, DOCK_SENSE_A, DOCK_PWR_OK), pins fixed at layout | STM32 TIM3 (pumps), TIM1 (ring), TIM14 (buzzer); PB0 OE, PB3 DOCK_SENSE_A, PB5 DOCK_PWR_OK; status LED on PC7 (PCF8574 #2 P2 freed) |
| Dock / battery sense to the RV1106 | GPIO edges + `ina2xx` hwmon | DU_STATUS (0x311) over the host link at 1 Hz and on any flag change |
| Connectors | 11 | 12 (+ J_SWD) |
| Area | ~2,700 mm² of parts | ~3,100 mm² (+ STM32, crystal, J_SWD) in v5.0; ~3,000 mm² in v5.3 (NAU7802 x2, TCA9543A, solenoid MOSFETs gone) |

#### Changes from v3.x (v4.0, history)

| Area | v3.x (single board) | v4.0 (DU board) |
|------|---------------------|-----------------|
| Physical architecture | One board for the whole machine, 24 V cable from the PSU | Three units; this board lives in the lift-off DU and meets the CS through an 8-pad pogo dock (J_DOCK). The STM32 moves to the new CSC in the CS base |
| MCU | STM32G0B1RE on this board (dispense, safety, both CAN ports) | None. Dispensing, load cells, LED ring and buzzer are driven by the RV1106 directly over I2C3 / GPIO / PWM (`epicurad` `dispense` thread) |
| CAN | Two ports on this board: isolated (ISO1050 x2 + B0505S, J_CAN_HEAT) and regular (SN65HVD230, J_CAN_STIR) | One node: MCP2518FD + ISO1050DUB #2, bus side powered from the dock's 5V_ISO; CAN_H/CAN_L on dock pads 3–4. No termination, no isolated DC-DC, no stir port on this board (that is the CSC's second, isolated port) |
| Power input | J_24V_IN (XT30) from the LRS-75-24; 24 V feeds 12 V, 6.5 V and VSYS | DOCK_24V on dock pad 1 (switched on the CSC, 2 A); feeds only the charger (via 12 V) and VSYS. The battery is the primary supply: VSYS = DOCK_24V ⊕ BAT+ |
| Battery | Backup for the logic domain, 0.5 A charge, voltage on an STM32 ADC divider | Primary supply for everything on the board, 1.0 A charge (R_CS 0.1 Ω), pack voltage + current on INA219 #2 (0x44) |
| Dispenser rail | 6.5 V from 24 V; D_DSB4 drop for the MG90S | 6.0 V (6V_DSP) from VSYS; all four servos, the pumps and the solenoids run on it; D_DSB4 removed |
| SLD | 12 V pumps and solenoids; 2x HX711 bit-banged by the STM32 | 6 V pumps (TB6612 VM = 6V_DSP) and 6 V NC solenoids; 2x NAU7802 on I2C3 behind a TCA9543A mux |
| Load/current monitor | INA219 on the 24 V input | INA219 #1 on 6V_DSP (0x40, DSB stall window), INA219 #2 on BAT+ (0x44) |
| LED ring | WS2812B, STM32 TIM15 + DMA, 74AHCT1G125, 3-pin J_LED | White ring, Q_LED low-side on an RV1106 PWM, 2-pin J_LED |
| Buzzer | STM32 TIM8 | RV1106 PWM via 2N7002 |
| Power-fail detect | 24 V divider on STM32 COMP1 | DOCK_PWR_OK (divider + 2N7002) and DOCK_SENSE_A on RV1106 GPIO |
| Moved to the CSC | — | Safety relay (Q_RLY), E-stop, pot reed (J_SAFE), exhaust fans (J_FAN), Q_STIR / J_STIR_PWR, J_CAN_STIR, J_CAN_HEAT, ISO1050 #1, B0505S, SN65HVD230, J_SWD, J_USB, service UART (on the CSC v1.3, Q_STIR and the SN65HVD230 have since given way to the relay K_SU and ISO1050DUB #3 + B0505S #2) |
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

Roughly 70 ICs/discretes and 12 connectors occupy about 2,900 mm² (the DU STM32, its crystal and J_SWD add ~400 mm² over v4.x, the MCP2518FD and Y3 give ~100 mm² back, the v5.2 move to an SN65HVD230 removes the isolation island and slot, ~150 mm², and v5.3 drops the NAU7802 pair, TCA9543A and solenoid MOSFETs, ~200 mm²). The Pico Zero footprint (65 x 30 mm) lies along the arc over the digital zone on standoffs; the area under it is used for passives on the bottom side. Since v5.3 the outline is an annular sector in the Ø350 mm DU's outer service ring (preliminary R 135–172 mm × ≈ 150°, ≈ 16,000 mm² gross, ~13,000 mm² usable, ~22 % utilisation), with the dock pads at the left-rear end over the pillar; the carousels sit inside the ring and the camera stays in the hub, off the board.

---

## 2. Board-Level Block Diagram

```
J_DOCK 1 DOCK_24V ─── SMBJ24A ─── 10 µF ──┬──────────────────────────────┬──► R_DIV 100k/10k ──► 2N7002 ──► DOCK_PWR_OK (STM32 PB5, active-low)
(switched on the CSC, F_DOCK 2 A)         │                              │
                                          ▼                              ▼
                                  ┌──────────────┐               ┌──────────────────┐
                                  │ MP1584EN #1  │               │ D_SYS1 (SS54)    │
                                  │ 24V → 12V    │               │ DOCK_24V ──► VSYS│
                                  └──────┬───────┘               └────────┬─────────┘
                                         │ 12V (charger only)             │
                                         ▼                        VSYS ◄──┴── D_SYS2 (SS54) ◄── BAT+ ◄── R_SH2 (10 mΩ, INA219 #2 0x44) ◄── J_BAT (2S2P, 6.0–8.4 V, PCM)
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
                                      ├──► J_LED pin 1 (white ring, Q_LED low-side)      ├──► TB6612 VM ──► J_SLD pumps (6 V: peristaltic oil, diaphragm water)
                                      ├──► J_BUZZER pin 1                                (no solenoids since v5.3: the nozzle check valves are passive)
                                      └──► AMS1117-3.3 ──► 3.3V: DU STM32, PCA9685, PCF8574 #2, INA219 x2, SLD Hall switches x6,
                                                                 SN65HVD230, TB6612 VCC

J_DOCK 5 GND_ISO ──────────────► tied to GND here (0R, bus reference); J_DOCK 6 5V_ISO ──► reserved, not connected
J_DOCK 3 CAN_H / 4 CAN_L ──────► SN65HVD230 bus pins (PESD1CAN, 3.3 V, DU ground) ◄── DU STM32 FDCAN1 (PB8/PB9)
J_DOCK 7 DOCK_SENSE_A ─── 10k pull-up to 3.3 V ──► DU STM32 PB3 (low = docked)
J_DOCK 8 DOCK_SENSE_B ─── tied to GND on this board (tells the CSC the DU is present)

                     ┌────────────────────────────────────────────────────────────────┐
                     │  RV1106 (Luckfox Pico Zero in J_AP)                            │
                     │  SPI0 CS0 ──► J_LCD (ST7796S); I2C3 ──► FT6336 (0x38) touch    │
                     │  UART2 (pins 8/10) ◄──► DU STM32 USART1 (host link, 921,600)   │
                     │  GPIO 32 ──► BOOT0; GPIO 38 ──► NRST (system-bootloader flash) │
                     │  CSI FPC ──► IMX415-98 camera                                  │
                     └───────────────────────────┬────────────────────────────────────┘
                                                 │ host link (COBS frames, CAN-shaped)
                     ┌───────────────────────────▼────────────────────────────────────┐
                     │  DU STM32 (STM32G0B1RET6, FreeRTOS)                            │
                     │  FDCAN1 ──► SN65HVD230 ──► dock bus (DU CAN node, gateway)     │
                     │  I2C1 ──► PCF8574 #2 (0x21), PCA9685 (0x41), INA219 #1 (0x40), │
                     │           INA219 #2 (0x44); PC0–PC5 ◄── J_SLD Hall x6 (tach/lvl) │
                     │  PB0 ──► PCA9685 OE; PB3 ◄── DOCK_SENSE_A; PB5 ◄── DOCK_PWR_OK │
                     │  TIM3 ──► TB6612 PWMA/PWMB; TIM1 ──► Q_LED (ring); TIM14 ──► buzzer│
                     │  PC7 ──► status LED; SWD ──► J_SWD                             │
                     │  PCF8574 #2 ──► TB6612 AIN1/BIN1, Q_DSB (P0/P1 spare);         │
                     │                 ◄── TP5100 CHRG/STDBY                          │
                     └────────────────────────────────────────────────────────────────┘
```

---

## 3. RV1106 and DU STM32 Interfaces

Two controllers share the board. The Luckfox Pico Zero in J_AP keeps only what needs Linux: the display (SPI0 CS0 + I2C3 touch), the camera (CSI), Wi-Fi, and a UART **host link** to the DU STM32 with BOOT0/NRST control. The **DU STM32** (§6.9) owns every other peripheral on the board through its I2C1, three timers and a few GPIO, and is the board's CAN node (§7). Nothing on the board is driven from a Pico Zero GPIO or PWM any more except the LCD lines.

### 3.1 RV1106 I2C3 — Touch Only (J_AP 16/18, 400 kHz)

| Address | Device | Function | Linux driver |
|---------|--------|----------|--------------|
| 0x38 | FT6336 | Capacitive touch controller on the 4" display, via J_LCD | `edt-ft5x06` |

4.7k pull-ups to 3.3 V on the PCB (the touch pull-ups of v3.x). The dispenser and power peripherals that sat on this bus in v4.x are on the STM32's I2C1 (§3.2); I2C1 on the Pico Zero header stays unused.

### 3.2 DU STM32 I2C1 — Peripheral Bus (PB6 SCL / PB7 SDA, 400 kHz)

| Address | Device | Function |
|---------|--------|----------|
| 0x21 | PCF8574 #2 | 8-bit GPIO expander (§3.5) |
| 0x40 | INA219 #1 | 6V_DSP rail voltage / current (DSB stall window) |
| 0x41 | PCA9685 | 16-channel servo PWM (DSB) |
| 0x44 | INA219 #2 | BAT+ voltage / current (state of charge) |

4.7k pull-ups to 3.3 V on the PCB; same addresses as v4.x, now driven by the STM32 HAL I2C driver in the `du` firmware (no Linux drivers). The NAU7802 x2 (0x2A) and the TCA9543A (0x71) of v5.0–v5.2 are gone since v5.3 (SLD metering is by pump tach, §6.4); I2C2/I2C3 of the STM32 stay free.

> [!note]
> The PCA9685 answers its All-Call address 0x70 (MODE1.ALLCALL = 1 after reset) as well as 0x41, so nothing else may sit at 0x70 (the TCA9543A of v5.0–v5.2 was strapped to 0x71 for this reason). The DU firmware clears ALLCALL at init so 0x70 stays free.

### 3.3 DU STM32 GPIO

| Signal | STM32 pin | Direction | Circuit | Purpose |
|--------|-----------|-----------|---------|---------|
| PCA9685_OE | PB0 | Out | PCA9685 OE, 10k pull-up to 3.3 V | Active-low output enable; servos limp at boot and whenever the STM32 is in reset |
| DOCK_SENSE_A | PB3 (EXTI) | In | J_DOCK pad 7, 10k pull-up to 3.3 V | Low = docked (pad grounded on the CS); edge interrupt + 1 Hz poll; reported in DU_STATUS |
| DOCK_PWR_OK | PB5 (EXTI) | In | DOCK_24V → 100k/10k divider → 2N7002 gate; drain 10k pull-up to 3.3 V | Active-low: low while DOCK_24V > ~15 V. High while docked = CS mains lost (AC Power Failure pause); reported in DU_STATUS |
| STATUS_LED | PC7 | Out | Green LED, 330R | Firmware heartbeat (was PCF8574 #2 P2) |
| BOOT0 | PA14 | In | ← J_AP GPIO 32, 10k pull-down | System-bootloader entry for flashing over the host link |
| NRST | NRST | In | ← J_AP GPIO 38 (open-drain), 10k pull-up, 100 nF | Reset from the RV1106 |

### 3.4 DU STM32 PWM (timers)

| Signal | STM32 pin / timer | Destination | Frequency | Purpose |
|--------|-------------------|-------------|-----------|---------|
| PWM_PUMP_OIL | PA6, TIM3_CH1 | TB6612 PWMA via 100R | 20 kHz | Oil pump speed |
| PWM_PUMP_WATER | PA7, TIM3_CH2 | TB6612 PWMB via 100R | 20 kHz | Water pump speed |
| PWM_LED | PA8, TIM1_CH1 | Q_LED gate (IRLML6344) via 100R | 1 kHz | White ring: OFF 0 %, CAPTURE 100 %, NIGHT 30 % (SET_LED 0x220 over the host link) |
| PWM_BUZZER | PA4, TIM14_CH1 | Q_BUZZ gate (2N7002) via 100R | 2–4 kHz tone | Piezo buzzer (SET_BUZZER 0x226 over the host link) |

All four PWM lines have 10k pull-downs so pumps, ring and buzzer are off while the STM32 boots or is held in reset. Six inputs on PC0–PC5 read the SLD Hall switches on J_SLD (§6.4): PC0 / PC1 pump tachs (EXTI0 / EXTI1), PC2 / PC4 tank present (EXTI2 / EXTI4; a rising edge arms the prime routine), PC3 / PC5 tank low (polled at 1 Hz); all open-drain with 10k pull-ups to 3.3 V and 100 R series on the board. EXTI lines 3 and 5 stay with DOCK_SENSE_A (PB3) and DOCK_PWR_OK (PB5).

### 3.5 PCF8574 #2 (I2C1 0x21) Allocation

| Pin | Direction | Signal | Notes |
|-----|-----------|--------|-------|
| P0 | — | Spare | SLD oil solenoid until v5.2 (removed with the solenoids) |
| P1 | — | Spare | SLD water solenoid until v5.2 (removed with the solenoids) |
| P2 | — | Spare | Status LED moved to STM32 PC7 in v5.0 |
| P3 | Out | TB6612 AIN1 (oil pump direction) | AIN2 tied to GND |
| P4 | Out | 6V_DSP rail enable | Q_DSB P-MOSFET via 2N7002 level shifter; on only during a dispense |
| P5 | Out | TB6612 BIN1 (water pump direction) | BIN2 tied to GND |
| P6 | In | TP5100 CHRG (charging, active-low) | Open-drain from charger, 10k pull-up |
| P7 | In | TP5100 STDBY (charge complete, active-low) | Open-drain from charger, 10k pull-up |

### 3.6 Host Link (RV1106 UART2 ↔ DU STM32 USART1)

```
J_AP pin 8  (UART2_TX) ──► 33R ──► PA10 (USART1_RX)        3.3 V logic both sides, no level shifter
J_AP pin 10 (UART2_RX) ◄── 33R ◄── PA9  (USART1_TX)        921,600 baud 8N1, STM32 side on DMA
J_AP GPIO 32 ──► 10k pull-down ──► PA14 / BOOT0
J_AP GPIO 38 ──► NRST (10k pull-up, 100 nF)
```

| Parameter | Value |
|-----------|-------|
| Framing | CAN-shaped frames: id (uint16 LE, 11-bit), dlc, data[0..7], CRC-16/CCITT-FALSE; COBS-encoded, 0x00 terminated; ≤ 14 bytes on the wire |
| Routing | DU-local IDs (0x220 SET_LED, 0x226 SET_BUZZER, 0x230–0x239 dispensing, 0x311 DU_STATUS, 0x3FE DU_ACK) end on the STM32; every other ID is forwarded unchanged to/from FDCAN1 (the RV1106 is still CAN node "AP", src_node 3). ID map in [[../03-Software/02-Controller-Software-Architecture#7.2 Message Types\|Controller Software Architecture §7.2]] |
| Host loss | No HB_AP (0x010) for 5 s → 6V_DSP off, PCA9685 OE high, pumps off, ring off; DU_STATUS keeps going. The bus loses HB_AP at the same moment, so the CSC goes safe by its own 5 s rule |
| Flashing | RV1106 raises BOOT0, pulses NRST, drives the ST system bootloader on USART1, verifies, drops BOOT0, resets. No in-application bootloader on the DU STM32 |

### 3.7 Header Pin Changes vs v4.x

| Header resource | v3.x | v4.0 | v5.0 |
|-----------------|------|------|------|
| Pins 8/10 (UART2) | Service link to STM32 USART2 | Freed | **Host link to the DU STM32 USART1** (all RV1106 ↔ STM32 and RV1106 ↔ CAN traffic) |
| GPIO 32 / GPIO 38 | STM32 BOOT0 / NRST | Freed | **DU STM32 BOOT0 / NRST** |
| I2C3 (16/18) | FT6336 touch only | Every I2C device on the board | **FT6336 touch only** (§3.1) |
| SPI0 CS1 (GPIO 58) + nINT | MCP2518FD | MCP2518FD | **Freed** (no SPI CAN controller) |
| GPIO | MCP2518FD nINT, TP_RST | + PCA9685 OE, DOCK_SENSE_A, DOCK_PWR_OK | TP_RST only; the three dispenser/dock GPIO moved to the STM32 (§3.3) |
| PWM | PWM10 (LCD backlight) | + 4 channels | **PWM10 only**; pumps, ring and buzzer on STM32 timers (§3.4) |
| SPI0 CS0, UART0 console, CSI camera | — | Unchanged | Unchanged |

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

No fans or pumps remain on 12 V (fans are on the CSC; pumps moved to 6V_DSP), so the rail exists only because the TP5100's input range does not reach 24 V. It is up only while docked with CS mains present.

### 4.3 VSYS → 6.0 V Dispenser Rail (MP1584EN #2, 6V_DSP)

| Parameter | Value |
|-----------|-------|
| Input | VSYS (23.5 V docked, 6.0–8.4 V on the pack) |
| Output | 6.0 V, 3 A max |
| Load | Four hobby servos, one moving at a time: DS3218 (C1, 2.2 A stall), 2x MG996R (F1, F2, ~2.5 A stall), MG90S (F3, ~0.7 A stall) — all rated for 6.0 V, no series drop needed; TB6612 VM (6 V peristaltic oil pump ~0.3 A, 6 V diaphragm water pump ~0.6 A, one at a time); no solenoids since v5.3 |
| Inductor | 22 µH, CDRH104R, Isat > 4 A |
| Output caps | 2x 22 µF MLCC + 470 µF electrolytic (10 V, low ESR) |
| Feedback | 64.9k / 10k → 5.99 V |
| Rail switch | Q_DSB: AO3401-class P-MOSFET high-side switch driven by a 2N7002 level shifter from PCF8574 #2 P4; 1000 µF bulk and 3 A PTC on the switched side; INA219 #1 shunt (§8) on the switched side |
| Protection | 3 A PTC + SMBJ6.0A TVS |

The rail is on only while a dispense is in progress (no idle servo buzz or heat), so the pack sees no actuator load between dispenses. On an empty pack (6.2 V) the buck drops out to ≈ 6.0 V, which is acceptable; the DU firmware refuses new moves below the 6.2 V halt threshold anyway.

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

The USB Schottky-OR of v3.x is gone (there is no USB-C on this board; the Pico Zero's own USB-C port is the DU's only one). 3.3 V loads: DU STM32 (~30 mA), PCA9685, PCF8574 #2, INA219 x2, six off-board Hall switches (~20 mA), SN65HVD230 (~15 mA), TB6612 VCC — ~0.19 A typical, 0.32 A max.

### 4.6 2-Cell Li-ion Pack — Primary Supply

#### 4.6.1 Pack

| Parameter | Value |
|-----------|-------|
| Configuration | 2S2P Li-ion, 4x 18650 (2,600 mAh each) in a holder or shrink-wrapped pack (changed from 2S1P 2026-11-04) |
| Voltage | 7.4 V nominal, 6.0 V (empty) to 8.4 V (full) |
| Energy | ~38 Wh (5,200 mAh) |
| Protection | Pack-mounted 2S PCM (per-cell over-voltage 4.25 V, under-voltage 2.8 V, over-current 3–5 A, short-circuit); balance is the PCM's job if fitted |
| Temperature sense | 10 kΩ NTC (B = 3435) glued to the cells, wired to J_BAT pin 3 |
| Connector | J_BAT, JST-XH 2.5 mm 3-pin: B+, B−, NTC |
| Cells | IEC 62133-2 / IS 16046 certified cells and pack (required for BIS) |
| Placement | In the DU, which is the cool zone by definition (the heat, PSU and fans are in the CS); keep the pack away from the LED ring and the camera window |
| Run time | Lifted-unit load ≈ 3 W (RV1106 + LCD + logic) → > 12 h on the 5,200 mAh pack; a full session docked draws from the dock, not the pack |

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
| Charge current | **1.0 A** (R_CS = 0.1 Ω, 1 %, 0.5 W) — ≈ 5.5 h from empty (0.2 C into 5,200 mAh) while docked; the pack is the primary supply, so a single session tops it up only partially, but > 12 h of lifted run time makes that acceptable |
| Pre-charge / termination | Trickle below 2S 6.0 V; termination at C/10; auto-recharge when the pack sags |
| Temperature window | NTC on TEMP pin: charging inhibited below 0 °C or above 45 °C |
| Status | CHRG and STDBY open-drain outputs to PCF8574 #2 P6/P7 (polled at 1 Hz by the DU STM32, reported in DU_STATUS) and two 0603 LEDs |
| Charge source | 12 V rail only: the pack charges whenever the DU is docked and the CS has mains (DOCK_PWR_OK low) |

> [!note]
> Confirm the R_CS current-programming relation and the CS pin strapping against the TP5100 datasheet before layout; retailer summaries disagree on the input range (5–12 V vs 5–18 V), which is why the charger runs from the 12 V rail and not from 24 V. At 1.0 A check the TP5100's thermal pad pour (§12).

#### 4.6.3 Discharge Path and Monitoring

- BAT+ → R_SH2 (10 mΩ) → D_SYS2 (SS54) → VSYS → TPS54531 and MP1584EN #2 (§4.4, §4.3). Lifted-unit discharge ~0.45 A at 7.4 V; up to ~3 A for a servo move on the pack; the pack PCM over-current limit (3–5 A) is the ceiling, and the DU firmware holds one servo at a time.
- Pack monitoring: **INA219 #2 (STM32 I2C1 0x44)** across R_SH2 reports pack voltage (bus input on BAT+) and signed current (charge in, discharge out). It replaces the v3.x 100k/22k divider on an STM32 ADC. The DU STM32 derives the state of charge from voltage + coulomb count and sends it in DU_STATUS (0x311); `epicurad` shows a battery indicator that is always visible on the DU display.
- Self-managed thresholds: warning banner at 6.4 V; checkpoint + halt at 6.2 V (boot recovery on the next power-up); the pack PCM cuts off at 2x 2.8 V = 5.6 V as a last resort.

#### 4.6.4 What the Battery Powers

Everything on the board. The pack is the primary supply of the DU; the dock only adds charge current and lifts VSYS above the pack voltage so the pack rests while docked.

| Domain | On battery | Notes |
|--------|------------|-------|
| 5V_MAIN: Pico Zero, LCD, 3.3 V logic (DU STM32, I2C ICs, SN65HVD230) | Yes | ~3 W → > 12 h lifted |
| White LED ring (5V_MAIN, PWM) | Yes | `epicurad` keeps it OFF while lifted or on a power-fail pause |
| 6V_DSP (servos, pumps) | Yes | Rail on only during a dispense; refused below 6.2 V |
| 12 V (charger) | No | DOCK_24V gone — no charging |
| CAN transceiver (SN65HVD230 on 3.3 V) | Yes | Stays powered; lifting the DU opens the bus by design (bus-off until re-docked) |

The purpose: ride through a CS mains outage without losing the session (checkpoint, keep the UI and Wi-Fi alive, resume if mains returns within the food-safety window, see [[../03-Software/03-Main-Loop-State-Machine|Main Loop State Machine]] §5), and keep the DU alive while it is lifted off for loading and cleaning.

### 4.7 Dock Power Sense (DOCK_PWR_OK)

```
24V Internal (DOCK_24V) ─── R_DIV1 (100k) ──┬── R_DIV2 (10k) ─── GND
                                             ├── C_DIV (100nF) ─── GND
                                             └── Q_PWR (2N7002) gate ── drain ── 10k to 3.3V ──► DU STM32 PB5 DOCK_PWR_OK (active-low)
```

| Parameter | Value |
|-----------|-------|
| Divider | 1/11 → 2.18 V at 24 V |
| Threshold | 2N7002 Vgs(th) 1.0–2.5 V → DOCK_PWR_OK goes high (inactive) when DOCK_24V < ~11–20 V; firmware confirms with a 300 ms TELEMETRY timeout |
| Recovery | DOCK_24V back for 2 s (firmware debounce) |
| Response | EXTI edge interrupt on the DU STM32 → DU_STATUS flag change within 20 ms |

Together with DOCK_SENSE_A (§3.3) the DU STM32 reports both lines in DU_STATUS (0x311), and the RV1106 distinguishes the two pause causes from them: docked + dock_power lost = CS mains lost (AC Power Failure, auto-resume); docked cleared = Unit Lifted (resume by user tap after re-docking). In both cases the heater and stir motor are already down on the CS side (keep-alive loss and the CSC's DOCK_SENSE_B hold). No 24 V comparator is involved any more.

### 4.8 Stirring Motor 24 V Switch — removed

Q_STIR, F_STIR and J_STIR_PWR moved to the CSC in v4.0; on the CSC v1.3 the motor runs from the Stirring Unit's own LRS-75-24 #2 (floating 24V_SU / GND_SU, J_SU_IN → F_SU → J_STIR_PWR) and Q_STIR / F_STIR are replaced by the pilot relay K_SU (STM32 PB4) on that PSU's AC input; see [[02-Cooking-Surface-Controller|Cooking Surface Controller]] §4.6 and §6. No 24 V load remains on this board besides the charger buck and D_SYS1.

### 4.9 Per-Rail Protection

| Rail | Polyfuse | TVS | Notes |
|------|----------|-----|-------|
| DOCK_24V | 2 A on the CSC (F_DOCK) | SMBJ24A at the pad | Switched by Q_DOCK on the CSC; no reverse diode (keyed dock) |
| 12V | 1.5 A | SMBJ12A | Charger only |
| 6V_DSP | 3 A PTC | SMBJ6.0A | Switched rail; INA219 #1 shunt |
| 5V_MAIN | — | — | TPS54531 integrated OCP |
| 5V_ISO (pad 6) | 100 mA on the CSC (F_CAN) | — | Reserved; not connected on this board since v5.2 |
| Battery | Pack PCM | — | D_SYS2 blocks back-charging from DOCK_24V; R_SH2 10 mΩ in the path |

### 4.10 Decoupling

- 10 µF at LDO input and output; 100 nF + 10 µF at each IC supply pin (PCA9685, PCF8574 #2, INA219 x2, SN65HVD230, TB6612 VM and VCC); the DU STM32 gets 100 nF per VDD pin + 4.7 µF bulk and 1 µF on VDDA (§6.9)
- 100 nF on J_SLD pin 5 (3V3_SNS, supply of the six off-board Hall switches)
- 10 µF + 100 nF at the dock pad on DOCK_24V

### 4.11 Power Budget Summary

| Rail | Source | Typical (A) | Peak (A) | Notes |
|------|--------|-------------|----------|-------|
| 3.3V (0.8 A max) | 5V_MAIN → AMS1117 | 0.19 | 0.32 | DU STM32, I2C ICs, SLD Hall switches x6 (~20 mA), SN65HVD230 (~15 mA), TB6612 VCC |
| 5V_MAIN (5 A max) | VSYS → TPS54531 | 0.6 | 1.9 | Pico Zero + LCD + LDO + white ring (1 A peak) |
| 6V_DSP (3 A max) | VSYS → MP1584EN #2 | 0 | 2.5 | One servo at a time, or one pump (water diaphragm ≈ 0.6 A); rail off when idle |
| 12V (loaded ≤ 1 A) | DOCK_24V → MP1584EN #1 | 0.8 | 0.9 | TP5100 at 1.0 A charge; docked with mains only |
| 5V_ISO (pad 6) | CSC B0505S #1 via F_CAN | 0 | 0 | Reserved; nothing on this board uses it since v5.2 |
| **DOCK_24V input** | **CS via Q_DOCK, F_DOCK 2 A** | **~0.6** | **~1.6** | Charging + logic ≈ 0.6 A; + a servo move on the dock ≈ 1.6 A |
| Battery (lifted, or docked during an outage) | 2S pack → R_SH2 → D_SYS2 → VSYS | 0.45 @ 7.4 V | 3.0 | Whole board; the peak is a DS3218/MG996R stall on the pack |

> [!warning]
> The dock pin rating (2 A per contact) and F_DOCK (2 A) set the input ceiling. The 1.6 A peak assumes the charger is at full current while a servo indexes; the DU firmware must not raise the LED ring to 100 % at the same moment, or the charger input is the first thing to sag (harmless, but it lengthens the charge). Monitor INA219 #2 during integration.

### 4.12 Isolated 5V DC-DC — removed

The B0505S-1WR3 #1 moved to the CSC, which generates the 5V_ISO / GND_ISO domain for its isolated port #1. Since v5.2 this board draws nothing from it: the DU transceiver is a non-isolated SN65HVD230 on 3.3 V, dock pad 5 (GND_ISO) is tied to the DU ground next to the pads to give the bus its single reference, and pad 6 (5V_ISO) is reserved. No isolated DC-DC and no isolated domain on this board; the isolators and their islands are on the CSC.

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
| 19 | SPI0_MOSI | Out | J_LCD MOSI | LCD only (SPI0 is no longer shared) |
| 21 | SPI0_MISO | In | J_LCD MISO (unused by fbtft) | |
| 23 | SPI0_CLK | Out | J_LCD SCK | 50 MHz to the LCD |
| 24 | SPI0_CS0 | Out | J_LCD CS | LCD chip select |
| SPI0_CS1 (GPIO 58) | — | — | Not connected (freed in v5.0) | Was the MCP2518FD nCS in v3.x/v4.x |
| 22 | GPIO | Out | J_LCD DC | LCD data/command |
| 26 | GPIO | Out | J_LCD RST | LCD reset |
| 12 | PWM10 | Out | J_LCD BL | Backlight PWM (via on-board 2N7002 if the module's LED pin needs > 20 mA) |
| 16 | I2C3_SDA | Bidir | J_LCD TP_SDA (FT6336 only, §3.1) | 400 kHz, 4.7k pull-up on PCB |
| 18 | I2C3_SCL | Bidir | J_LCD TP_SCL (FT6336 only, §3.1) | 400 kHz, 4.7k pull-up on PCB |
| 36 | GPIO | In | J_LCD TP_INT | Touch interrupt, active-low |
| spare GPIO (fixed at layout) | GPIO | Out | J_LCD TP_RST | Touch reset |
| 8 | UART2_TX | Out | 33R → DU STM32 PA10 (USART1_RX) | Host link, 921,600 baud (§3.6) |
| 10 | UART2_RX | In | 33R ← DU STM32 PA9 (USART1_TX) | Host link |
| GPIO 32 | GPIO | Out | DU STM32 BOOT0 (PA14), 10k pull-down | System-bootloader entry for flashing |
| GPIO 38 | GPIO (open-drain) | Out | DU STM32 NRST, 10k pull-up + 100 nF | Reset the DU STM32 |
| 35, 37 | UART0 TX/RX | — | J_AP test pads only | Luckfox debug console |
| all others | — | — | Not connected | |

> [!note]
> The RV1106 has no CAN peripheral and, since v5.0, no SPI CAN controller either: the DU's CAN node is the DU STM32's FDCAN1 (§7), and the RV1106 reaches the bus through the host link on UART2, where the STM32 forwards every non-local frame unchanged. SPI0 therefore carries the LCD alone. Firmware updates: the DU STM32 is flashed by the RV1106 through BOOT0/NRST and the ST system bootloader on this same UART; the CSC's image is still streamed over CAN (0x7E0/0x7E8) to the CSC's in-application bootloader, through the gateway.

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
STM32 I2C1 (PB6/PB7) ──► PCA9685PW (addr 0x41, A0 = 1)
   STM32 PB0 ──► OE (active-low; 10k pull-up → outputs off until the DU firmware drives it low)
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
| PWM driver | PCA9685PW (TSSOP-28), 16 ch, 12-bit, 50 Hz; EXTCLK not used; V+ pin left unconnected (servo power goes directly from the switched rail to J_DSB); driven by the DU STM32 over I2C1 |
| Servo signal | 500–2,500 µs, 3.3 V logic (DS3218, MG996R and MG90S all accept 3.3 V) |
| Rail | 6.0 V / 3 A, switched on for the duration of a dispense; one servo moves at a time (DU firmware Dispense task mutex); the MG90S no longer needs D_DSB4 |
| Connectors | J_DSB1–4: 3-pin JR/Futaba servo headers (SIG / V+ / GND), 2.54 mm, keyed, 300 mm extension leads |
| Fault detection | INA219 #1 current window on 6V_DSP during moves (> 2.5 A for 300 ms) + 3 s move timeout + PCA9685 I2C NACK → retry once (back off 10°) → DSB_STALL, detected by the DU STM32 and returned to the RV1106 as a NAK result / DU_STATUS error_code |
| Safe state | OE high and rail off at boot, on E_STOP, on UNIT_LIFTED, on host-link loss (no HB_AP for 5 s) and while the STM32 is in reset (servos limp) |

### 6.2 SLD Peristaltic Pumps (2x) — TB6612FNG

```
6V_DSP ──── VM (TB6612FNG)
   PWMA ◄── STM32 PA6 TIM3_CH1 PWM_PUMP_OIL (100R)     AIN1 ◄── PCF8574 #2 P3   AIN2 ── GND   AO1/AO2 ──► J_SLD 1/2 (oil pump)
   PWMB ◄── STM32 PA7 TIM3_CH2 PWM_PUMP_WATER (100R)   BIN1 ◄── PCF8574 #2 P5   BIN2 ── GND   BO1/BO2 ──► J_SLD 3/4 (water pump)
   STBY ◄── 3.3V (10k)   VCC ── 3.3V            C_VM: 100 nF + 10 µF
```

| Parameter | Value |
|-----------|-------|
| IC | TB6612FNG (SSOP-24), VM = 6V_DSP (6.0 V), VCC 3.3 V |
| Pumps | Oil: 6 V DC peristaltic, ≈ 0.4 mL/rev, ≈ 0.3 A, two magnets on the roller head → Hall tach (≈ 0.2 mL/pulse). Water: 6 V DC diaphragm, ≈ 0.25 mL/stroke, ≈ 1 L/min, ≈ 0.6 A, one magnet on the motor shaft → Hall tach (≈ 0.25 mL/pulse). Both in the DU body under the tank bays (v5.3 robot-mop liquid system) |
| Current | 1.2 A continuous / 3.2 A peak per channel |
| Pull-downs | 10k on PWMA/PWMB (pumps off during boot); direction bits default low on the PCF8574 (forward) |

Pumps run only while 6V_DSP is enabled, so an idle or lifted DU cannot pump even if a PWM line is mis-driven.

### 6.3 SLD Solenoid Valves — removed (v5.3)

The two 6 V NC valves, their IRLML6344 + SS14 low-side drivers and PCF8574 #2 P0/P1 are gone. A spring check valve (≈ 5 kPa cracking, food-grade PP/silicone) inside each snap-out nozzle cartridge blocks drip, siphoning from the tank above and steam up the line — passive, as on robot mops. P0/P1 are spare.

### 6.4 SLD Tank and Pump Sensing (6x Hall switch) — replaces the NAU7802 load cells (v5.3)

```
3.3V ──► J_SLD 5 (3V3_SNS, 100 nF) ──► six AH3572-class Hall switches off-board (tank bays, pump brackets)
                                        open-drain outputs, 10k pull-ups to 3.3 V + 100 R series on the PCB:
   J_SLD 7  OIL_TACH    ──► PC0 (EXTI0)   2 pulses per roller-head revolution ≈ 0.2 mL each
   J_SLD 8  H2O_TACH    ──► PC1 (EXTI1)   1 pulse per motor revolution ≈ 0.25 mL
   J_SLD 9  OIL_PRESENT ──► PC2 (EXTI2)   fixed magnet in the oil tank wall; rising edge arms the prime
   J_SLD 10 OIL_LOW     ──► PC3 (polled)  float magnet, trips at ≈ 20 mL left
   J_SLD 11 H2O_PRESENT ──► PC4 (EXTI4)   fixed magnet in the water tank wall
   J_SLD 12 H2O_LOW     ──► PC5 (polled)  float magnet, trips at ≈ 50 mL left
   J_SLD 6  GND_SNS
```

| Parameter | Value |
|-----------|-------|
| Switches | 6x digital unipolar Hall switch, 3.3 V, open-drain (AH3572 or equivalent), off-board in the tank bays and pump brackets; 3 mm neodymium magnets, encapsulated, never in contact with food |
| Metering | Open loop in the DU STM32 Dispense task: `volume = pulses × cal_ul` (µL per pulse, per channel, set by SLD_CAL 0x237 at boot and after calibration; defaults 200 µL oil / 250 µL water); the pump stops at target_pulses; ±5 % after calibration |
| Prime | `prime_pending` set at boot and on a tank-present rising edge; a fixed pulse count (oil 15 / water 32, ≈ 1.5x the line volume) is pumped before the next dose or on SLD_PRIME (0x233); refused while the DU is lifted |
| Faults | No tach pulse within 1 s of pump start, pulse rate < 25 % of nominal for 2 s, or INA219 #1 pump current outside 50 mA–1.5 A → SLD_DISPENSE_ERROR; tank-present low → SLD_TANK_ABSENT (NAK); tank-low → SLD_LOW_LEVEL (NAK before a dose, DU_STATUS bit7 during) |
| Removed | 2x NAU7802SGI, TCA9543A, their decoupling, the PRTR5V0U2X pair and the two bridge cables; nothing analog is left on J_SLD |

Hall lines are short (< 30 cm) and digital; route them away from the 6V_DSP switch node and the pump outputs.

### 6.5 Exhaust Fans — removed

The two 120 mm fans, their MOSFETs and J_FAN are on the CSC (STM32 PA6 / PB10, 12 V rail); the RV1106 sets the exhaust stage over CAN (SET_EXHAUST 0x260, Exhaust §3.4).

### 6.6 White LED Ring Driver

```
5V_MAIN ──► J_LED pin 1 (5V)  ──► ring (12–16 white LEDs, 5000 K, CRI ≥ 90, ≤ 1 A) ──► J_LED pin 2 (LED_RET) ──► Q_LED (IRLML6344, low-side) ──► GND
                                                                                       gate ◄── 100R ◄── STM32 PA8 TIM1_CH1 PWM_LED (1 kHz), 10k pull-down
```

The ring is a plain white ring with series resistors or a constant-current strip, mounted on the DU underside around the camera window; brightness is the STM32 PWM duty (OFF 0 %, CAPTURE 100 %, NIGHT 30 %), commanded by the RV1106 with SET_LED (0x220) over the host link. No colour control, no WS2812B, no level shifter; SET_LED is a host-link message, never a CAN bus frame.

### 6.7 Piezo Buzzer

5 V active piezo via 2N7002 low-side switch, gate from the STM32 PA4 TIM14_CH1 PWM_BUZZER through 100R with 10k pull-down; J_BUZZER 2-pin. Tones are requested by the RV1106 with SET_BUZZER (0x226) over the host link.

### 6.8 USB-C Programming Port — removed

There is no USB-C on this board. The Pico Zero's own USB-C port serves the RV1106 (console, mass storage, `rkdeveloptool`); the DU STM32 is flashed by the RV1106 through BOOT0/NRST and the ST system bootloader on USART1 (the host link, §3.6), with J_SWD (§10.14) for the bench; the CSC carries the USB-C DFU port for its own STM32.

### 6.9 DU STM32 — Microcontroller (STM32G0B1RET6)

The same part as on the CSC, so both boards share one HAL, FreeRTOS, MISRA C code base and the X-CUBE-CLASSB start-up self-tests (`epicura-firmware` builds a `csc` and a `du` image from one tree). On this board it is the CAN node, the host-link gateway and the dispenser controller; it has no safety authority.

| Parameter | Value |
|-----------|-------|
| Part | STM32G0B1RET6, LQFP-64, Cortex-M0+ @ 64 MHz, 512 KB flash (dual bank), 144 KB SRAM |
| Clock | 8 MHz HSE (Y1 + 2x 15 pF) → PLL 64 MHz; LSE pads (PC14/PC15) unpopulated; LSI for the IWDG |
| Power | 3.3 V rail; 100 nF per VDD pin + 4.7 µF bulk, 1 µF on VDDA; ~30 mA |
| Reset / boot | NRST from J_AP GPIO 38 (open-drain, 10k pull-up, 100 nF); BOOT0 on PA14 from J_AP GPIO 32 (10k pull-down); nBOOT_SEL cleared in option bytes so the pin selects the system bootloader |
| Debug | SWD on PA13/PA14 → J_SWD (2x5 1.27 mm, §10.14); no USB |
| Used peripherals | USART1 (host link, DMA), FDCAN1 (dock bus), I2C1 (peripherals), TIM3 (pumps), TIM1 (ring), TIM14 (buzzer), 9 GPIO (OE, two dock inputs, six SLD Hall inputs) + status LED, IWDG; FDCAN2, USB, ADC unused |

Pin map:

| Pin | Function | Signal | Circuit |
|-----|----------|--------|---------|
| PA9 / PA10 | USART1 TX / RX | HOST_TX / HOST_RX | ↔ J_AP pin 10 (UART2 RX) / pin 8 (UART2 TX); 33R series; system-bootloader UART |
| PA14 (SWCLK / BOOT0) | BOOT0 | BOOT0 | ← J_AP GPIO 32, 10k pull-down |
| NRST | reset | NRST | ← J_AP GPIO 38 (open-drain), 10k pull-up, 100 nF |
| PA13 / PA14 | SWDIO / SWCLK | SWD | J_SWD 2x5 1.27 mm |
| PB8 / PB9 | FDCAN1 RX / TX | CAN | SN65HVD230 R / D (§7) |
| PB6 / PB7 | I2C1 SCL / SDA | I2C1 400 kHz | PCA9685 0x41, PCF8574 #2 0x21, INA219 #1 0x40, INA219 #2 0x44; 4.7k pull-ups (§3.2) |
| PA6 / PA7 | TIM3_CH1 / CH2 | PWM_PUMP_OIL / PWM_PUMP_WATER | TB6612 PWMA / PWMB via 100R, 20 kHz |
| PA8 | TIM1_CH1 | PWM_LED | Q_LED gate (IRLML6344) via 100R, 1 kHz |
| PA4 | TIM14_CH1 | PWM_BUZZER | Q_BUZZ gate (2N7002) via 100R, 2–4 kHz |
| PB0 | GPIO out | PCA9685_OE | 10k pull-up (servos off at boot / in reset) |
| PB3 | GPIO EXTI | DOCK_SENSE_A | J_DOCK pad 7, 10k pull-up, low = docked |
| PB5 | GPIO EXTI | DOCK_PWR_OK | Q_PWR drain, 10k pull-up, low = DOCK_24V present |
| PC0 / PC1 | GPIO EXTI0 / EXTI1 | SLD_OIL_TACH / SLD_H2O_TACH | J_SLD 7 / 8, Hall tach pulses, 10k pull-up + 100R (§6.4) |
| PC2 / PC4 | GPIO EXTI2 / EXTI4 | TANK_OIL_PRESENT / TANK_H2O_PRESENT | J_SLD 9 / 11, Hall, rising edge arms the prime routine |
| PC3 / PC5 | GPIO in (polled) | TANK_OIL_LOW / TANK_H2O_LOW | J_SLD 10 / 12, float Hall |
| PC7 | GPIO out | Status LED | green, 330R |
| PF0 / PF1 | HSE | 8 MHz | Y1 + 2x 15 pF |
| PC14 / PC15 | LSE pads | — | unpopulated |
| PB12 / PB13 | FDCAN2 | — | unused on the DU (no stir CAN port); test pads |
| PA11 / PA12 | USB | — | not used (no USB-C on the DU) |
| all others | — | Available | ~34 pins free for DU-side extras (canister presence switches, a lid sensor) |

The host-link circuit is in §3.6. Firmware behaviour (tasks, routing, host-loss rule) is in §15.1 and in [[../03-Software/05-MCU-Functional-Specification|MCU Functional Specification]].

---

## 7. CAN Node

The board carries **one** CAN node, the DU STM32's FDCAN1, on the 500 kbps bus shared with the CSC STM32 and the heating unit. The bus arrives over dock pads 3–4 with its ground reference on pad 5; nothing else on this board is on a CAN bus. Since v5.2 the node is **not isolated**: the barrier for this bus is the ISO1050DUB #1 at the CSC's port #1, the DU side is a plain SN65HVD230 referenced to the DU logic ground (which is the CS ground through dock pad 2 while docked). The CSC's port-1 isolator is therefore a signal-reference / ground-loop barrier, not a safety barrier; mains safety isolation is the LRS-75-24 and, for the heating unit, its own board, whose CAN port must be SELV. On the bus the node is still **AP (src_node 3)**: the DU STM32 forwards the RV1106's frames from the host link unchanged and passes bus frames back up, so the CSC and the heating unit see exactly what they saw in v4.x.

| | DU node (this board) |
|---|---|
| Connector | J_DOCK pads 3 (CAN_H), 4 (CAN_L), 5 (GND_ISO, tied to GND here), 6 (5V_ISO, reserved) |
| Controller | DU STM32 FDCAN1, PB8 (RX) / PB9 (TX), classic-CAN mode (§7.2) |
| Transceiver | SN65HVD230 (SOIC-8, 3.3 V), non-isolated, referenced to the DU ground; Rs 10k to GND (slope control); 100 nF on VCC |
| Other nodes | CSC STM32 (ISO1050DUB #1 on the CSC), heating unit MCU (its own transceiver) |
| Bit rate | 500 kbps, classic CAN 2.0B, 11-bit IDs |
| Termination | None on this board: 120 Ω on the CSC at J_CAN_HEAT and 120 Ω inside the heating unit. The dock → this board run is a stub ≤ 30 cm |
| Ground | Common DU ground. Dock pad 5 (GND_ISO) is linked to it with a 0R next to the pads so the bus has exactly one reference, defined here at the far node (never on the CSC); pad 2 carries the 24 V return |
| Traffic | Host commands/telemetry (RV1106 ↔ CSC STM32, through the gateway), heater setpoint and status, heartbeat, E_STOP, CSC firmware update (0x7E0/0x7E8) — ID map in [[../03-Software/02-Controller-Software-Architecture#7.2 Message Types\|Controller Software Architecture §7.2]]. Dispensing, LED and buzzer messages (0x220, 0x226, 0x230–0x239, 0x311, 0x3FE) stay on the host link and never reach the bus |

Lifting the DU opens the bus and the 24 V at the same instant: the heating unit drops to zero ≤ 1 s after its last keep-alive, the CSC STM32 enters safe hold on DOCK_SENSE_B, and this node sees bus-off until re-docked (reported as can_bus_ok = 0 in DU_STATUS; automatic bus-off recovery).

### 7.1 Transceiver — SN65HVD230 (non-isolated)

```
STM32 PB9 (FDCAN1_TX) ──► D  ┐  SN65HVD230 (3.3 V)  ┌── CANH ─┬── PESD1CAN ── J_DOCK pad 3 (CAN_H)
STM32 PB8 (FDCAN1_RX) ◄── R  ┘  Rs ── 10k to GND     └── CANL ─┴────────────── J_DOCK pad 4 (CAN_L)
                                VCC 3.3V / GND (100 nF)        J_DOCK pad 5 (GND_ISO) ── 0R ── GND
                                                                J_DOCK pad 6 (5V_ISO) ── n.c. (reserved)
```

- No isolation slot, island or isolated supply on this board (removed in v5.2): the transceiver and the dock pads sit on the common ground plane with ordinary SELV spacing.
- Pad 5 is the bus's single ground reference: the CSC's ISO1050 #1 bus side floats on the B0505S #1 output and is referenced to the DU ground only here (and to the heating unit's port on J_CAN_HEAT). No ground loop through the CSC logic ground.
- Decoupling: 100 nF on VCC; PESD1CAN across CAN_H / CAN_L at the pads.

### 7.2 DU STM32 CAN Node — FDCAN1

| Parameter | Value |
|-----------|-------|
| Controller | STM32G0B1 FDCAN1 (PB8 RX / PB9 TX, AF3), classic CAN 2.0B frames, 500 kbps from the 64 MHz kernel clock; same bit timing as the CSC's FDCAN1 |
| Filters | Accept 0x00x, 0x01x, 0x18x, 0x3xx, 0x7E8 (the AP filter set) into RX FIFO 0; everything accepted is forwarded to the host link |
| Gateway | Host-link frames whose ID is not DU-local are transmitted on FDCAN1 as received (ID, DLC, data); 32-frame TX queue; bus-off → automatic recovery, flagged in DU_STATUS |
| Removed in v5.0 | MCP2518FD-E/SL, its 40 MHz crystal (Y3), SPI0 CS1 and nINT, the Linux `mcp251xfd` driver and SocketCAN `can0` |
| Removed in v5.2 | ISO1050DUB #2, its bus-side decoupling, the isolation island and the 6 mm slot; 5V_ISO is no longer used on the DU (pad 6 reserved) |

### 7.3 Regular Port — moved

The stir port is on the CSC: STM32 FDCAN2 → ISO1050DUB #3 + B0505S #2 (isolated CAN port #2, 1 Mbps, J_CAN_STIR) with the Stirring Unit's own floating 24 V (LRS-75-24 #2 → J_SU_IN → F_SU → J_STIR_PWR, AC input cut by the relay K_SU) reaching the stirring motor through the J_SU harness inside the right pillar (24V_SU, GND_SU, CAN_H, CAN_L, SHIELD; nothing on it references the CS ground); see [[02-Cooking-Surface-Controller|Cooking Surface Controller]] §4.6 and §6.

---

## 8. Current Monitoring — INA219 x2

| Ref | I2C1 address (DU STM32) | Shunt | Measures | Use |
|-----|-------------------------|-------|----------|-----|
| INA219 #1 | 0x40 | 10 mΩ 2512 in the switched 6V_DSP rail (after the PTC) | Rail voltage and servo / pump current | DSB stall window (> 2.5 A for 300 ms) during carousel moves; SLD pump-current window (50 mA–1.5 A) during a dispense; dispenser power telemetry. Moved from the 24 V input of v3.x |
| INA219 #2 | 0x44 | 10 mΩ 2512 in the BAT+ path between J_BAT and D_SYS2 | Pack voltage (bus input on BAT+), signed pack current | State of charge, charging / discharging, 6.4 V warning and 6.2 V halt; replaces the STM32 ADC divider |

Both are read by the DU STM32 over I2C1 (1 Hz for the pack in the Monitor task, 50 Hz on the rail while a dispense is active); the pack figures reach the RV1106 in DU_STATUS (0x311).

---

## 9. Protection Circuits

### 9.1 ESD Protection

- 33R series resistors on J_LCD and on the host-link UART and BOOT0/NRST lines between the Pico Zero and the DU STM32
- 100 R series + 10k pull-ups on the six J_SLD Hall lines (the switches are off-board, in the tank bays and pump brackets); no analog lines leave the board since v5.3
- J_SWD: 33R series on SWDIO/SWCLK (bench-only connector)
- The relay, E-stop and pot inputs (J_SAFE) and the USB port are on the CSC; their ESD parts go with them

### 9.2 Dock Pad ESD and Hot-Plug

The dock pads are the only contacts a user touches with the DU in hand (cleaning, loading):

- **PESD1CAN** across CAN_H / CAN_L at pads 3–4 (referenced to the DU ground)
- **SMBJ24A** on DOCK_24V (pad 1) plus 10 µF 50 V, against the hot-plug transient when the CSC closes Q_DOCK
- Pad 5 (GND_ISO) tied to GND through a 0R link next to the pads; pad 6 (5V_ISO) not connected
- DOCK_SENSE_A (pad 7) and DOCK_PWR_OK: 100R series + 10k pull-up into the DU STM32 GPIO (PB3 / PB5); DOCK_SENSE_B (pad 8) is tied to GND through a 0R link
- No isolation boundary on this board: the pad row uses ordinary SELV spacing; the bus is isolated at the CSC's port, not here

### 9.3 MOSFET Gate Safety

Every MOSFET gate and driver enable has a 100R series resistor and a 10k pull-down (or pull-up for the P-MOSFET Q_DSB and the PCA9685 OE) so that all actuators, the 6V_DSP rail, the LED ring and the buzzer are off while the DU STM32 is booting, held in reset by the RV1106 or unpowered. The PCF8574 #2 outputs are high-impedance (pulled low by the gate resistors) until the DU firmware writes them, and the firmware itself returns everything to this state on host-link loss (§3.6).

Relay driver, E-stop input and pot-detection input: on the CSC (see [[02-Cooking-Surface-Controller|Cooking Surface Controller]] §8).

---

## 10. Connector Definitions (12 total)

| Ref | Type | Function |
|-----|------|----------|
| J_AP | 2x20 socket, 2.54 mm, 8.5 mm stacking | Luckfox Pico Zero slot |
| J_LCD | 1x14 header, 2.54 mm | 4" ST7796S + FT6336 display |
| J_DOCK | 8 flat gold pads, 2.54 mm pitch, bottom side at the board edge | Dock to the CS left pillar: charge 24 V, CAN (isolated at the CSC), GND_ISO reference, reserved 5V_ISO, dock sense |
| J_BAT | JST-XH 2.5 mm 3-pin | 2S Li-ion pack: B+, B−, NTC |
| J_DSB1–J_DSB4 | 3-pin servo header, 2.54 mm | C1, F1, F2, F3 servos |
| J_SLD | Hirose DF11-12DP-2DS (2x6, 2.0 mm) | SLD pumps x2, six Hall switches (tach x2, present x2, low x2) |
| J_LED | JST-XH 2.5 mm 2-pin | White ring: 5V, LED_RET |
| J_BUZZER | JST-PH 2.0 mm 2-pin | Piezo buzzer |
| J_SWD | 2x5 1.27 mm (Cortex debug) | DU STM32 SWD, bench only (back since v5.0) |

Removed since v3.x (all now on the CSC): J_24V_IN, J_CAN_HEAT, J_CAN_STIR, J_STIR_PWR, J_FAN, J_SAFE, J_USB (J_SWD returned in v5.0). Removed since v2: J_CM5 (replaced by J_AP), J_12V_UPS, J_BLDC, J_PASD, J_PRES, J_CID, J_IR_LED (replaced by J_LED), single-cell J_BAT (replaced by the 3-pin 2S J_BAT).

### 10.1 J_AP — see §5.2

### 10.2 J_LCD — see §5.3

### 10.3 J_DOCK — Dock Pads to the Cooking Surface (8 pads, 2.54 mm pitch, DU view)

| Pad | Signal | Direction | Notes |
|-----|--------|-----------|-------|
| 1 | DOCK_24V | In (CS → DU) | Charge input; switched on the CSC by Q_DOCK only while DOCK_SENSE_B is low, F_DOCK 2 A. SMBJ24A + 10 µF at the pad. Dead when the unit is lifted |
| 2 | GND | — | Charge return; ties the DU logic ground to the CS ground while docked |
| 3 | CAN_H | bus | 500 kbps bus (same bus as the heating unit), isolated at the CSC's port #1; SN65HVD230 + PESD1CAN here |
| 4 | CAN_L | bus | |
| 5 | GND_ISO | — | Bus ground reference; tied to the DU ground here (0R next to the pads). This is the only place the CSC's isolated domain #1 meets a ground |
| 6 | 5V_ISO | In (CS → DU) | Reserved: still driven from the B0505S #1 on the CSC through F_CAN 100 mA, not connected on this board since v5.2 |
| 7 | DOCK_SENSE_A | In (CS → DU) | Tied to GND on the CS; 10k pull-up here → DU STM32 PB3; low = docked (reported to the RV1106 in DU_STATUS) |
| 8 | DOCK_SENSE_B | Out (DU → CS) | Tied to GND here; 10k pull-up on the CSC → CSC STM32 PB3; low = DU present (enables Q_DOCK) |

The mating part is an 8-contact magnetic pogo connector: spring pins (2 A each) in a keyed recess on top of the CS left pillar, flat pads on the DU underside, so the DU can only seat one way. The pad pattern is on L4 (bottom) at the board edge over the pillar, all eight pads on the common ground side (no island since v5.2, §11.2).

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

### 10.9 J_SLD — SLD Pumps and Hall Switches (Hirose DF11-12DP-2DS)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | PUMP_OIL_AO1 | TB6612 A1 (oil peristaltic pump +, 6 V) |
| 2 | PUMP_OIL_AO2 | TB6612 A2 (oil pump −) |
| 3 | PUMP_H2O_BO1 | TB6612 B1 (water diaphragm pump +, 6 V) |
| 4 | PUMP_H2O_BO2 | TB6612 B2 (water pump −) |
| 5 | 3V3_SNS | 3.3 V for the six Hall switches (100 nF at the pin) |
| 6 | GND_SNS | Sense return |
| 7 | OIL_TACH | Oil pump Hall tach → PC0 (EXTI0) |
| 8 | H2O_TACH | Water pump Hall tach → PC1 (EXTI1) |
| 9 | OIL_PRESENT | Oil tank present Hall → PC2 (EXTI2) |
| 10 | OIL_LOW | Oil tank float Hall → PC3 |
| 11 | H2O_PRESENT | Water tank present Hall → PC4 (EXTI4) |
| 12 | H2O_LOW | Water tank float Hall → PC5 |

Pins 7–12 are open-drain Hall outputs with 10k pull-ups to 3.3 V and 100 R series on the PCB. The 16-pin connector, the solenoid pins and the two bridge cables of v4.0–v5.2 are gone.

### 10.10 J_FAN — moved to the CSC

### 10.11 J_LED — White Ring (JST-XH 2-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 5V | 5V_MAIN (unswitched) |
| 2 | LED_RET | Ring return → Q_LED drain (IRLML6344, STM32 PA8 PWM_LED) |

### 10.12 J_BUZZER — Piezo Buzzer (JST-PH 2-pin)

| Pin | Signal |
|-----|--------|
| 1 | 5V |
| 2 | BUZZ_OUT (2N7002 drain, STM32 PA4 PWM_BUZZER) |

### 10.13 J_SAFE — moved to the CSC

### 10.14 J_SWD — DU STM32 SWD Debug (2x5, 1.27 mm)

Standard Cortex debug pinout: VCC 3.3 V, SWDIO (PA13), SWCLK (PA14), SWO (unused on the M0+, pin left for the standard pinout), NRST, GND. Bench and factory use only; in the field the RV1106 flashes the DU STM32 over the host link (§3.6). Same connector as the CSC's J_SWD.

### 10.15 J_USB — moved to the CSC

---

## 11. 4-Layer Stackup and Zone Layout

### 11.1 Layer Stack

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (no isolation slot since v5.2) | 1 oz |
| L3 | Split power plane: 5V/3.3V, 6V_DSP, VSYS/BAT+ (12 V is a local trace) | 1 oz |
| L4 (Bottom) | Signal + power traces, passives under the Pico Zero, J_DOCK pad pattern | 2 oz |

FR-4 Tg ≥ 150 °C, 1.6 mm, ENIG (hard gold on the eight dock pads if the fab offers it).

### 11.2 Board Layout Zones

```
┌──────────────────────────────── 160 mm ────────────────────────────────────────┐
│ ZONE C: DOCK + CAN NODE     │ ZONE A: AP SLOT + DIGITAL              │ ZONE B: BATTERY  │
│ ┌─ CAN NODE ─────────────┐  │ ┌─────────────────────────────┐        │ J_BAT  R_SH2     │
│ │ pads 3-6 │ SN65HVD230  │  │ │ J_AP 2x20 + 4 standoffs     │        │ INA219 #2        │
│ │ PESD1CAN │ (no slot)   │  │ │ (Luckfox Pico Zero 65x30)   │        │ TP5100  L_CHG    │
│ └────────────────────────┘  │ └─────────────────────────────┘        │ CHG LEDs         │ 90 mm
│ pads 1-2, 7-8  SMBJ24A      │ DU STM32 + Y1  J_SWD  AMS1117  PCF8574 #2│ D_SYS1/2 → VSYS │
│ Q_PWR (DOCK_PWR_OK)         │ PCA9685  INA219 #1  (R_HALL, R_PU)      │ C_SYS 100 µF     │
├─────────────────────────────┼────────────────────────────────────────┼──────────────────┤
│ J_LCD   J_LED   J_BUZZER    │ ZONE D: ACTUATOR DRIVERS               │ ZONE F: BUCKS    │
│ Q_LED   Q_BUZZ              │ TB6612FNG (pumps); no solenoid MOSFETs │ MP1584EN #1 (12V)│
│                             │ Q_DSB, PTC, R_SH1, 1000 µF             │ MP1584EN #2 (6V) │
│                             │ J_DSB1-4      J_SLD                    │ TPS54531 + L1-L3 │
└─────────────────────────────┴────────────────────────────────────────┴──────────────────┘
  ▲ board edge over the CS left pillar (J_DOCK pads on L4)
```

The zone diagram is drawn for the rectangular v5.0–v5.2 outline. In v5.3 the same zones are laid out along the annular sector in the DU service ring — the dock edge (Zone C) at the left-rear end over the pillar, the battery (Zone B) at the other end next to the pack bay, the bucks between — and will be redrawn with the DU CAD.

Placement rationale:
- **Zone C** sits at the board edge that lies over the CS left pillar when the DU is seated: the eight J_DOCK pads are on L4 at that edge, with the SN65HVD230, PESD1CAN and the pad-5 ground link right behind pads 3–5. There is no isolation island or slot on this board since v5.2 (the bus is isolated at the CSC's port).
- **Zone A**: the Pico Zero sits on standoffs over the I2C ICs; its SPI0 lines to J_LCD are short (< 40 mm). The DU STM32 and Y1 sit beside the module, between J_AP (host link pins 8/10/32/38 within 30 mm) and the Zone C edge (FDCAN1 to the SN65HVD230 within 30 mm); J_SWD at the board edge next to it. Keep the module's antenna end at the board edge, away from Zone F and the crystal.
- **Zone B**: J_BAT, R_SH2, INA219 #2, the TP5100 and the VSYS diodes together; the battery cable leaves the board edge towards the pack bay in the DU.
- **Zone F**: three bucks ≥ 25 mm from the Pico Zero, thermal pours ≥ 400 mm² each; the TP5100 pour ≥ 300 mm² for 1 A charge.
- **J_DSB1–4 and J_SLD** on the inner edge of the sector, nearest the carousels and the tank bays.

### 11.3 Critical Layout Rules

| Rule | Specification | Rationale |
|------|---------------|-----------|
| CAN node | No isolation on this board: SN65HVD230, PESD1CAN and pads 3–6 on the common ground; pad 5 (GND_ISO) linked to GND with a 0R within 5 mm of the pad; ordinary SELV spacing across the pad row | The bus is isolated at the CSC's port; one ground reference per bus, defined at this far node |
| CAN stub | SN65HVD230 bus pins to pads 3–4 < 30 mm; no termination on this board | The dock → board run is a stub ≤ 30 cm off the CSC-terminated bus |
| Dock pads | 8 pads, 2.54 mm pitch, ≥ 2.0 mm diameter, on L4 at the edge; ≥ 1.5 mm track to pad 1/2; mechanical keying by the enclosure recess | 2 A per contact, repeatable seating |
| SPI0 to J_LCD | Matched within 5 mm, 50 Ω, ground return under the trace | 50 MHz LCD clock |
| Host link (UART2 ↔ USART1, BOOT0, NRST) | < 50 mm, routed over solid ground, away from the bucks and 6V_DSP switch; 33R at the driver end | 921,600 baud, clean bootloader entry |
| DU STM32 crystal | Y1 and its caps within 5 mm of PF0/PF1, ground guard, no traces under it; FDCAN1 traces to the SN65HVD230 < 30 mm | Clock integrity, CAN timing |
| Pico Zero clearance | 3 mm keep-out around the module outline on L1; passives only under it on L4 | Mechanical fit, Wi-Fi antenna |
| Battery traces | ≥ 2 mm, B+ fused by the pack PCM; R_SH2 Kelvin-sensed; NTC trace away from the bucks | Up to 3 A discharge, clean current reading |
| VSYS node | ≥ 2 mm, D_SYS1/D_SYS2 within 10 mm of the TPS54531 input cap | Hold-up during switch-over |
| 6V_DSP | ≥ 1.5 mm from Q_DSB to J_DSB1-4 / TB6612 / J_SLD; R_SH1 Kelvin-sensed | 2.5 A servo stall, stall-window accuracy |
| SLD Hall inputs | Digital lines with 100 R series at the connector, ≥ 5 mm from the 6V_DSP switch node and the pump outputs | Clean tach pulses, no false present / low edges |
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
| DU STM32, SN65HVD230, INA219 x2, Hall switches x6 | — | ~0.4 | STM32 ≈ 0.1 W at 64 MHz |
| SS54 x2 (D_SYS1/2), Q_LED, Q_DSB | — | ~1.0 | Diode drops dominate |
| **Total** | | **~5 W typical** | At 40 °C ambient every IC stays below 110 °C junction with the pours in §11.3; the DU is the cool unit |

The Pico Zero (~1 W) dissipates through its own board; leave 10 mm of free air above it.

---

## 13. Unified BOM (changes highlighted)

| Ref | Part | Package | Qty | Unit (USD) | Notes |
|-----|------|---------|-----|-----------|-------|
| **AP slot + DU STM32 + CAN node** | | | | | |
| J_AP | 2x20 socket, 2.54 mm, 8.5 mm | TH | 1 | $0.80 | Pico Zero slot |
| STANDOFF x4 | M2.5 x 8 mm + screws | — | 4 | $0.10 | |
| U_MCU | **STM32G0B1RET6** | LQFP-64 | 1 | $3.50 | **DU STM32: CAN node, host-link gateway, dispenser controller (back since v5.0)** |
| Y1 | **8 MHz crystal + 2x 15 pF** | SMD | 1 | $0.30 | **HSE** |
| R_BOOT0, R_NRST, C_NRST, R_HL x2 | 10k, 10k, 100 nF, 33R | 0402 | 1 lot | $0.03 | **BOOT0 pull-down, NRST pull-up, host-link series** |
| J_SWD | **2x5 1.27 mm header** | TH | 1 | $0.30 | **Cortex debug, bench only** |
| U_CAN | **SN65HVD230** | SOIC-8 | 1 | $0.80 | **Non-isolated, 3.3 V, DU ground; driven by STM32 FDCAN1; replaces ISO1050DUB #2 (v5.2)** |
| PESD1CAN, C_CAN, R_RS, R_GISO | —, 100 nF, 10k, 0R | — | 1 lot | $0.20 | Rs slope control; pad-5 ground link |
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
| (pack) | 2S2P 18650 pack with PCM, 5,200 mAh, NTC | — | 1 | $22.00 | Not on the PCB; system BOM |
| **Dispenser Servo Block** | | | | | |
| U_PWM | PCA9685PW | TSSOP-28 | 1 | $1.50 | STM32 I2C1 0x41 |
| Q_DSB, Q_DSB_N | AO3401 + 2N7002 | SOT-23 | 2 | $0.15 | 6.0 V rail switch |
| F_DSB, C_DSB, D_TVS2 | 3 A PTC, 1000 µF 10 V, SMBJ6.0A | — | 3 | $0.60 | |
| U_MON1, R_SH1 | INA219BIDR, 10 mΩ 2512 | — | 2 | $0.90 | **Moved to the 6V_DSP rail, 0x40** |
| J_DSB1-4 | 3-pin servo header | TH | 4 | $0.10 | **D_DSB4 removed** |
| **SLD** | | | | | |
| U_TB | TB6612FNG | SSOP-24 | 1 | $1.50 | VM = 6V_DSP |
| R_HALL x6, R_PU x6, C_SNS | 100R, 10k, 100 nF | 0402 / 0603 | 13 | $0.05 | **J_SLD Hall inputs PC0–PC5 (v5.3)** |
| J_SLD | Hirose DF11-12DP-2DS | TH | 1 | $0.60 | **12-pin since v5.3** |
| **LED, buzzer** | | | | | |
| Q_LED | IRLML6344 | SOT-23 | 1 | $0.06 | White ring low-side, **STM32 TIM1_CH1 PWM** |
| Q_BUZZ | 2N7002 | SOT-23 | 1 | $0.05 | **STM32 TIM14_CH1 PWM** |
| J_LED, J_BUZZER | JST-XH 2, JST-PH 2 | TH | 2 | $0.08 | |
| **I2C expander** | | | | | |
| U_EXP2 | PCF8574 | SOIC-16 | 1 | $0.50 | 0x21, on STM32 I2C1 (P2 spare) |
| **Protection** | | | | | |
| F_12V, D_TVS3 | 1.5 A polyfuse, SMBJ12A | — | 2 | $0.45 | 12 V rail |
| **Display break-out** | | | | | |
| J_LCD | 1x14 header 2.54 mm | TH | 1 | $0.15 | |
| R_I2C x4, R_SER x12 | 4.7k (I2C3 + I2C1 pull-ups), 33R | 0402 | 16 | $0.01 | |
| **Passives** | decoupling (incl. STM32 VDD/VDDA), pull-ups/downs, gate resistors, feedback dividers, crystal caps | — | ~95 | — | ~$2.10 |
| **PCB** | 4-layer annular sector (R 135–172 mm × ≈ 150°, ≈ 16,000 mm², TBC with the DU CAD), ENIG, 2 oz outer, dock pads | — | 1 | $8.00 | JLCPCB, 5 pcs; estimate for the arc outline |

Removed from this board in v5.3 (robot-mop liquid system): Q_SOL1-2 + D_FB1-2, U_MUX TCA9543A, U_ADC1-2 NAU7802SGI with C_NAU / R_NAU, D_ESD1-2, the 16-pin J_SLD. Removed from this board in v5.2: U_CAN2 ISO1050DUB #2 and its bus-side decoupling (replaced by the SN65HVD230; the board has no isolated domain). Removed from this board in v5.0: U_CAN_AP MCP2518FD-E/SL and Y3 (40 MHz crystal + caps). Added in v5.0: U_MCU STM32G0B1RET6, Y1 8 MHz crystal + caps, J_SWD, the BOOT0/NRST/host-link resistor lot and the STM32 decoupling.

Removed from this board in v4.0 (to the CSC or deleted): STM32G0B1RET6 + crystals (back in v5.0), ISO1050DUB #1, B0505S-1WR3, SN65HVD230, Q_STIR/F_STIR/C_STIR, Q_RLY + flyback, fan MOSFETs, 74AHCT1G125, Q_LED_P/Q_LED_N, 2x HX711 breakouts, D_DSB4, R_BAT1/R_BAT2, USB-C lot, J_24V_IN, J_CAN_HEAT, J_CAN_STIR, J_STIR_PWR, J_FAN, J_SAFE, J_SWD (back in v5.0), J_USB, SW_BOOT/SW_RST.

### 13.1 Cost Summary

| Category | Prototype (1x) | Volume (100x) |
|----------|----------------|---------------|
| AP slot + DU STM32 + CAN node (socket, standoffs, STM32G0B1RET6, 8 MHz crystal, J_SWD, SN65HVD230) | $6.00 | $4.50 |
| Dock (TVS, sense) | $0.45 | $0.30 |
| Power conversion (3 bucks, LDO, VSYS OR, passives) | $7.20 | $4.60 |
| Battery charger + INA219 #2 | $2.30 | $1.60 |
| Dispenser Servo Block + INA219 #1 | $3.70 | $2.60 |
| SLD drivers (TB6612, Hall pull-ups) | $1.80 | $1.20 |
| LED / buzzer / expander / protection | $1.20 | $0.80 |
| Display break-out, passives | $2.40 | $1.50 |
| Connectors (12) | $2.70 | $1.80 |
| PCB (annular sector, estimate) | $8.00 | $3.50 |
| **Board total** | **~$36** | **~$22** |
| Battery pack (system BOM, off-board) | $22.00 | $15.00 |
| 8-pin magnetic pogo pair (system BOM, with the CS pillar) | $6.00 | $4.00 |

Compared with v4.x (~$39): the STM32G0B1RET6, its crystal, J_SWD and a few resistors (≈ +$4) replace the MCP2518FD and its 40 MHz crystal (≈ −$3), and in v5.2 the SN65HVD230 replaces ISO1050DUB #2 (≈ −$2.70); net ≈ −$2 per board. In v5.3 the NAU7802 pair, TCA9543A, solenoid MOSFETs and ESD parts leave (≈ −$4) while the arc outline adds ≈ +$2.50 to the bare board. Compared with v3.1 (~$62): the second ISO1050, B0505S, SN65HVD230, motor switch, relay, fans, USB and seven connectors (≈ −$24) leave for the CSC or disappear; INA219 #2 joins (the NAU7802 pair and TCA9543A came and went again in v5.3); the MCU is back. The CSC (≈ $25 PCB + parts) and the pogo pair are added to the system BOM; the net system delta vs v3.1 is ≈ +$33 (see the project BOM).

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
| Dimensions | Annular sector R 135–172 mm × ≈ 150° (v5.3, TBC with the DU CAD; was 160 x 90 mm) |
| Mounting | 4x M3 holes along the arc (positions TBC); 4x M2.5 holes for the Pico Zero standoffs (match the module's 3.0 mm hole pattern, 65 x 30 mm outline) |
| Milling | None (no isolation slot since v5.2) |
| Impedance | 50 Ω SPI0 |

### 14.1 Assembly Notes

- SMT both sides (passives under the Pico Zero on L4), reflow; then hand solder TH: J_AP, J_LCD, J_SLD, JST-XH/PH, J_DSB x4, J_SWD, electrolytics
- Program the DU STM32 option bytes (nBOOT_SEL = 0 so BOOT0 is pin-controlled) through J_SWD at first power-up; afterwards the RV1106 flashes it over the host link
- Fit the Pico Zero standoffs before populating J_AP so the socket height is set by the standoffs
- Keep solder mask and silkscreen off the eight J_DOCK pads; label the pad row "DOCK — pads 3-6 ISOLATED"
- Pin 1 silkscreen triangles on all connectors

---

## 15. Firmware Integration Notes (DU STM32 + RV1106 `epicurad`)

### 15.1 DU STM32 Tasks (FreeRTOS, `du` image of `epicura-firmware`)

| Task | Rate | Functions |
|------|------|-----------|
| Link / Gateway | event-driven, ≤ 5 ms latency | USART1 DMA RX/TX, COBS + CRC-16 codec; routes DU-local IDs (0x220, 0x226, 0x230–0x239) to the Dispense / UI handlers and answers with DU_ACK (0x3FE); forwards every other host frame to FDCAN1 and every accepted bus frame (0x00x, 0x01x, 0x18x, 0x3xx, 0x7E8) back up; HB_AP (0x010) watchdog: 5 s without it → dispensers and ring off |
| Dispense | 50 Hz while a move is active | PCA9685 ramped moves (≤ 3°/tick) with OE and 6V_DSP enable, one servo at a time; stall = INA219 #1 window (> 2.5 A for 300 ms) + 3 s timeout + I2C NACK → retry once (10° back-off) → DSB_STALL; SLD open loop: tank present / low check, prime if pending, pump tach pulses (EXTI PC0 / PC1) counted to target_pulses, no-flow and current-window faults; SLD_PRIME, SLD_CAL, PREFLIGHT, DISPENSER_HOME, DISPENSER_STATUS, QUERY_SLD answered in the ACK data |
| Monitor | 1 Hz + EXTI edges | INA219 #2 (batt_mV, batt_mA → state of charge), PCF8574 #2 P6/P7 (charging / complete), DOCK_SENSE_A (docked), DOCK_PWR_OK (dock_power), FDCAN1 bus state → DU_STATUS (0x311) at 1 Hz and on any flag change; LED duty (SET_LED) and buzzer tones (SET_BUZZER); IWDG kick |

The DU STM32 has no safety task: it never originates HEAT_ON_OFF or E_STOP, and its only autonomous action is the host-loss shutdown of its own actuators.

### 15.2 RV1106 `epicurad` Threads Touching This Board

| Thread | Rate | Functions |
|--------|------|-----------|
| `link` (SCHED_FIFO; was `can`) | event | Owns `/dev/ttyS2` at 921,600 baud, COBS/CRC codec; delivers DU_ACK / DU_STATUS to the `dispense` thread and the power/dock monitor, bus frames (TELEMETRY, STATUS, HEAT_STATUS, ACK, FW_UPDATE_RSP) to `heat` and the engine exactly as the raw CAN socket did; can_bus_ok = 0 while lifted is expected |
| `heat` (SCHED_FIFO) | 2 Hz | Unchanged: HEAT_SET_TEMP / HEAT_SET_POWER keep-alive and HEAT_STATUS, now through the link |
| `dispense` (SCHED_OTHER) | per command | Thin client: same `dispense_submit()` / `dispense_wait()` API and command names, but it sends 0x230–0x239 over the link and waits for 0x3FE instead of driving I2C/PWM; DU_LINK_LOST (no DU_STATUS for 3 s) → pause |
| power/dock monitor | on DU_STATUS | batt_mV, batt_mA, soc_pct, docked, dock_power, charging flags from 0x311; battery indicator always visible in the UI |
| UI / LED / buzzer | event | SET_LED 0x220 (OFF / NIGHT / CAPTURE), SET_BUZZER 0x226 over the link |
| swupdate handler | on demand | DU STM32: BOOT0 (GPIO 32) high, NRST (GPIO 38) pulse, ST system bootloader on UART2, verify, BOOT0 low, reset. CSC STM32: image streamed over CAN (0x7E0 / 0x7E8) through the gateway to the CSC's in-application bootloader |

### 15.3 Peripheral Allocation

RV1106 header:

| Peripheral | Header | Function |
|------------|--------|----------|
| SPI0 CS0 | 19/21/23/24 | ST7796S LCD (SPI0 CS1 freed) |
| I2C3 | 16/18 | FT6336 0x38 only |
| UART2 | 8/10 | Host link to the DU STM32 USART1 |
| GPIO 32 / GPIO 38 | 32 / 38 | DU STM32 BOOT0 / NRST |
| PWM10 | 12 | LCD backlight |
| GPIO | 22, 26, 36, spare | LCD DC / RST, touch INT / RST |

DU STM32:

| Peripheral | Pins | Function |
|------------|------|----------|
| USART1 | PA9 / PA10 | Host link, 921,600 baud, DMA |
| FDCAN1 | PB8 / PB9 | SN65HVD230 → dock bus (AP node) |
| I2C1 | PB6 / PB7 | PCF8574 #2 0x21, PCA9685 0x41, INA219 0x40 / 0x44 |
| TIM3 CH1/CH2, TIM1 CH1, TIM14 CH1 | PA6 / PA7, PA8, PA4 | TB6612 PWMA / PWMB, Q_LED, buzzer |
| GPIO | PB0, PB3, PB5, PC0–PC5, PC7 | PCA9685 OE (out), DOCK_SENSE_A (in, EXTI), DOCK_PWR_OK (in, EXTI), SLD Hall inputs (tachs EXTI0/1, present EXTI2/4, low polled), status LED |
| SWD | PA13 / PA14 | J_SWD |

### 15.4 Power-Fail and Lift Sequences

1. **CS mains lost while docked:** DU_STATUS reports dock_power = 0 with docked still 1 (within 20 ms of the DOCK_PWR_OK edge); confirmed by a 300 ms TELEMETRY timeout → PAUSED (AC Power Failure). VSYS is already on the pack (passive OR, no reboot of either controller). The heater and stir motor are down on the CS side. Auto-resume to RESUMING when TELEMETRY returns within 5 min (the CSC reboots in < 1 s), prompt if < 30 min, ERROR beyond.
2. **Unit lifted:** DU_STATUS reports docked = 0 and can_bus_ok = 0 → PAUSED (UNIT_LIFTED); LED ring OFF, dispensing refused by both the engine and the DU firmware. Resume only by user tap after re-docking (docked = 1, bus recovered, TELEMETRY back); same 30 min food-safety ceiling.
3. **Battery low:** warning banner at 6.4 V; at 6.2 V checkpoint + halt (RECOVERING on the next boot); the DU firmware refuses dispenses below 6.2 V on its own. The pack PCM at 5.6 V is the last resort.
4. **Host link lost (RV1106 crash or reset):** the DU STM32 drops its actuators and the ring after 5 s without HB_AP; the bus loses HB_AP at the same time, so the CSC goes safe. When `epicurad` returns, the link resynchronises on the next frame.

### 15.5 Battery Charger Behaviour

- Charging runs autonomously whenever the 12 V rail is up (docked, CS mains present); the DU STM32 only reads CHRG/STDBY and INA219 #2 and reports them in DU_STATUS
- Fault indication (in `epicurad`): charging and charge_done both clear with the pack below 8.0 V for > 10 min while dock_power is set → battery fault banner (pack missing, NTC out of range, or charger fault); cooking is still allowed
- Self-test at boot: INA219 #2 bus voltage between 5.5 V and 8.5 V, else the same banner

---

## 16. Design Verification Checklist

### 16.1 Pre-Fabrication

- [ ] ERC passes; the DU STM32 (STM32G0B1RET6) pin map matches §6.9 (USART1 PA9/PA10, FDCAN1 PB8/PB9, I2C1 PB6/PB7, TIM3 PA6/PA7, TIM1 PA8, TIM14 PA4, PB0/PB3/PB5/PC7, PC0–PC5 SLD Hall inputs); no MCP2518FD symbol, SPI0_CS1 net, NAU7802, TCA9543A or solenoid MOSFET left on the board
- [ ] Board outline is the annular sector agreed with the DU CAD (R 135–172 mm × ≈ 150°, dock pads at the left-rear end over the pillar)
- [ ] J_AP pinout matches the Luckfox Pico Zero 40-pin header; UART2 on pins 8/10, GPIO 32 → BOOT0 and GPIO 38 → NRST confirmed against the Luckfox pinout; TP_RST is the only spare GPIO still used; I2C3 carries the touch controller only
- [ ] Host link: 33R series both directions, BOOT0 10k pull-down, NRST 10k pull-up + 100 nF, NRST driven open-drain from the RV1106; J_SWD pinout per §10.14; Y1 8 MHz with 2x 15 pF on PF0/PF1
- [ ] Pico Zero outline, 3.0 mm hole pattern and standoff height checked against the module drawing
- [ ] J_LCD pin order matches the chosen 4" ST7796S/FT6336 module
- [ ] CAN node: SN65HVD230 Rs 10k to GND; pad 5 (GND_ISO) linked to GND with a 0R next to the pads; pad 6 (5V_ISO) not connected; no ISO1050, isolated net or termination resistor left on this board
- [ ] J_DOCK pad pattern matches the pogo block (2.54 mm pitch, pad 1 position vs the keyed recess) and the DU underside drawing
- [ ] TP5100 strapped for 2S (8.4 V), R_CS = 0.1 Ω, NTC bias per datasheet, CHRG/STDBY pulled up to 3.3 V
- [ ] VSYS diode-OR: D_SYS1 from DOCK_24V, D_SYS2 from B+ via R_SH2; TPS54531 feedback 4.98 V; MP1584EN #2 feedback 5.99 V from VSYS; C_SYS rated 35 V
- [ ] INA219 #1 A0/A1 → 0x40 on 6V_DSP; INA219 #2 → 0x44 on BAT+; both shunts Kelvin-connected
- [ ] Nothing at 0x70 (PCA9685 All-Call, §3.2 note); J_SLD Hall lines: 10k pull-ups to 3.3 V, 100 R series, 100 nF on 3V3_SNS
- [ ] Q_DSB default OFF (gate pull-up), driven by PCF8574 #2 P4 through 2N7002; PCA9685 OE 10k pull-up on STM32 PB0, A0 = 1 → 0x41; PCF8574 #2 A0 = 1 → 0x21; all of them on the STM32 I2C1 with 4.7k pull-ups
- [ ] All MOSFET gates: 100R series + 10k pull-down (or pull-up for P-MOSFETs); all four STM32 PWM lines pulled down
- [ ] DOCK_24V: SMBJ24A + 10 µF at the pad; DOCK_PWR_OK divider 100k/10k into STM32 PB5; DOCK_SENSE_A 10k pull-up into STM32 PB3; DOCK_SENSE_B to GND
- [ ] All 12 connectors have correct pinouts and silkscreen; J_SLD 12 pins per §10.9

### 16.2 PCB Layout

- [ ] DRC passes; GND plane continuous (no isolation slot)
- [ ] Dock pad row at ordinary SELV spacing; pad 1 (DOCK_24V) clearance ≥ 1.0 mm to its neighbours
- [ ] Pico Zero keep-out respected; standoff holes placed; antenna end at the board edge away from the bucks
- [ ] SPI0 traces to J_LCD matched within 5 mm; host-link and BOOT0/NRST traces < 50 mm over solid ground; Y1 within 5 mm of PF0/PF1 with a ground guard; FDCAN1 to the SN65HVD230 < 30 mm
- [ ] Battery, VSYS and 6V_DSP trace widths per §11.3; shunt Kelvin traces
- [ ] J_SLD Hall lines routed away from the 6V_DSP switch node and the pump outputs
- [ ] Buck loops < 10 mm; thermal vias and pours present (incl. TP5100)
- [ ] J_DOCK pads on L4 at the pillar edge, mask-free, matching the mechanical drawing

### 16.3 Post-Assembly

- [ ] 3.3 V, 5 V rails within ±3 % on the pack alone (no dock); 12 V and 6.0 V within ±3 % with DOCK_24V applied
- [ ] Pico Zero boots in J_AP from the pack; LCD renders through J_LCD; touch events on I2C3 (I2C3 scan finds 0x38 only)
- [ ] DU STM32: option bytes set (nBOOT_SEL = 0) and a first image loaded through J_SWD; status LED blinks; the RV1106 then re-flashes it over the host link (BOOT0 high, NRST pulse, system bootloader on UART2) and the new image boots
- [ ] Host link loopback at 921,600 baud: COBS frames with CRC round-trip RV1106 → STM32 → RV1106 with no loss over 10 min; DU_STATUS (0x311) arrives at 1 Hz
- [ ] STM32 I2C1 scan finds 0x21, 0x40, 0x41, 0x44 and nothing else
- [ ] FDCAN1 exchanges frames with the CSC's FDCAN1 at 500 kbps through a bench pogo block; HEAT_QUERY sent from the RV1106 over the host link reaches the heating unit and HEAT_STATUS comes back up the link; TELEMETRY from the CSC arrives at 20 Hz
- [ ] Stop `epicurad` (or hold the Pico Zero in reset) with a servo move in progress: after 5 s without HB_AP the DU STM32 drops 6V_DSP, raises OE and turns the ring off; the CSC reports safe state
- [ ] DSB: each servo reaches the commanded angle on 6.0 V; rail switches off after a move; OE high leaves servos limp; INA219 #1 shows the move current
- [ ] SLD: both pumps run on 6 V (oil in both directions); tach pulses count on PC0 / PC1 at every PWM setting; tank present / low inputs follow a magnet; after SLD_CAL a 100-pulse dose lands within ±5 % of the calibration; the nozzle check valves hold with a full tank
- [ ] White ring: PWM 0 / 30 / 100 % give OFF / NIGHT / CAPTURE; ≤ 1 A at 100 %
- [ ] Battery: pack charges to 8.4 V at ~1.0 A from the 12 V rail; CHRG then STDBY read correctly on P6/P7 and show as charging / charge_done in DU_STATUS; INA219 #2 voltage within 2 % and current sign correct (charge vs discharge)
- [ ] Pull DOCK_24V during a cook simulation: 5V_MAIN stays within ±5 % with no reboot of the Pico Zero or the DU STM32; DU_STATUS dock_power clears within 20 ms; UI shows "Power lost — running on battery"
- [ ] Lift the DU from the bench block: DU_STATUS docked and can_bus_ok clear, UI shows UNIT_LIFTED; re-dock → FDCAN1 recovers from bus-off, resume on tap
- [ ] Battery run-time ≥ 12 h at the lifted-unit load; halt at 6.2 V, recovery on the next boot
- [ ] NTC disconnected → charger stops (CHRG/STDBY both inactive) → battery fault banner
- [ ] All actuators, the 6V_DSP rail, the ring and the buzzer off with the DU STM32 held in reset (NRST low from the RV1106 or J_SWD) or with the Pico Zero removed (host loss)

---

## 17. Related Documentation

- [[02-Cooking-Surface-Controller|Cooking Surface Controller]] — the CS board: STM32G0B1RE, both CAN transceivers, relay, E-stop, fans, dock power switch
- [[../02-Hardware/01-Epicura-Architecture|Epicura Architecture]] — three units, system wiring, CAN bus, dock, power
- [[../02-Hardware/02-Technical-Specifications|Technical Specifications]] — power specs incl. the DU battery
- [[../05-Subsystems/01-Electric-Heating|Electric Heating]] — module with internal temperature control, SELV CAN port on the CSC's isolated port #1
- [[../05-Subsystems/02-Robotic-Arm|Robotic Arm]] — CAN servo stirring motor (SU)
- [[../05-Subsystems/03-Ingredient-Dispensing|Ingredient Dispensing]] — DSB and SLD
- [[../03-Software/05-MCU-Functional-Specification|MCU Functional Specification]] — STM32 firmware requirements (CSC and DU instances)
- [[../03-Software/02-Controller-Software-Architecture|Controller Software Architecture]] — CAN ID map (§7.2), host-link protocol, `epicurad` threads
- [[03-PCB-Design-Rules|PCB Design Rules]] — DRC rules and net classes
- Archived: [[_archive/01-Controller-PCB-Design|Controller PCB]] and [[_archive/02-Driver-PCB-Design|Driver PCB]] (2-board era)

---

#epicura #pcb #unified #dispensing-unit #rv1106 #can #battery #dock #power-electronics #hardware-design

---

## 18. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 5.3 | 2026-11-05 | Manas Pradhan | Robot-mop liquid system and the Ø350 mm DU disc: the SLD becomes two drawer tanks (200 mL oil / 500 mL water, bottom self-sealing coupling, float + presence magnets) with the pumps in the DU body — 6 V peristaltic for oil, 6 V diaphragm for water, each with a Hall tach — and a passive check valve in each nozzle cartridge; the two solenoid valves (Q_SOL1-2, D_FB1-2, PCF8574 #2 P0/P1 now spare), both NAU7802SGI, the TCA9543A, their decoupling and the PRTR5V0U2X pair leave the board. Six Hall inputs on PC0–PC5 (tachs EXTI0/1, present EXTI2/4, low polled), J_SLD re-pinned to a 12-pin DF11, I2C1 down to four devices (0x21, 0x40, 0x41, 0x44). Open-loop metering by tach pulses × cal_ul in the Dispense task with a prime routine (SLD_PRIME 0x233) and SLD_CAL (0x237); QUERY_WEIGHT → QUERY_SLD. Board outline becomes an annular sector in the DU's outer service ring (R 135–172 mm × ≈ 150°, TBC with the DU CAD; zone diagram to be redrawn). §1, §1.3, §3.2–§3.5, §4.10–§4.11, §6.2–§6.4, §6.9, §8, §9.1, §10, §11, §12, §13 (~$36 / ~$22), §15 and §16 updated |
| 5.2 | 2026-11-04 | Manas Pradhan | DU CAN node no longer isolated: ISO1050DUB #2, its bus-side supply, the isolated island and the 6 mm slot leave the board; the DU STM32's FDCAN1 drives an SN65HVD230 (3.3 V, Rs 10k) referenced to the DU ground. Isolation stays on the CSC only (ISO1050 #1 + B0505S #1 for the DU/heater bus, ISO1050 #3 + B0505S #2 for the SU bus); port-1 isolation is a signal-reference / ground-loop barrier, not a safety barrier, since the DU ground meets the CS ground on dock pad 2 — mains safety is the LRS-75-24 and the heating unit's own SELV CAN port (new selection requirement). Dock pad 5 (GND_ISO) is tied to GND next to the pads as the bus's single reference, pad 6 (5V_ISO) is reserved / not connected. §1 bullets and diagram, §1.3 (new v5.2 row), §1.4 (~2,950 mm²), §2, §4.6.4 / §4.9–§4.12, §6.9 pin row, §7 (7.1 rewritten, 7.2 removed-in-v5.2 row), §9.2, §10 (J_DOCK), §11 (no slot, zone C), §12, §13 (−$2.70, ~$37 / ~$24), §14, §15, §16 and §17 updated |
| 5.1 | 2026-11-04 | Manas Pradhan | Aligned to CSC v1.3 (no change to this board): the CSC's stir port is described as its second isolated CAN port (ISO1050DUB #3 + B0505S #2) with the Stirring Unit on its own floating LRS-75-24 #2 cut by the relay K_SU instead of the SN65HVD230 + Q_STIR / F_STIR — §1 scope row, v3.x comparison notes, §4 24 V note, PB12/PB13 row and §10 J_CAN_STIR / J_STIR_PWR note. No schematic or BOM change on the DU board |
| 5.0 | 2026-11-04 | Manas Pradhan | MCU back in the Dispensing Unit: a second STM32G0B1RET6 (same part as the CSC) joins the board as the DU's CAN node (FDCAN1 PB8/PB9 → ISO1050DUB #2, still node "AP" on the bus, CAN ID map unchanged), a transparent gateway to the RV1106 and the owner of every dispenser driver. Removed: MCP2518FD, its 40 MHz crystal (Y3), SPI0 CS1 / nINT, `mcp251xfd` / SocketCAN. Added: host link J_AP UART2 (pins 8/10) ↔ STM32 USART1 (PA9/PA10) at 921,600 baud with COBS + CRC-16 CAN-shaped frames and the DU-local ID block (0x220 SET_LED, 0x226 SET_BUZZER, 0x230–0x239 dispensing, 0x311 DU_STATUS, 0x3FE DU_ACK); GPIO 32 → BOOT0 and GPIO 38 → NRST for flashing through the ST system bootloader; J_SWD (2x5 1.27 mm) restored, 12 connectors; 8 MHz HSE (Y1). RV1106 I2C3 keeps the FT6336 only; STM32 I2C1 (PB6/PB7) carries PCF8574 #2, PCA9685, INA219 x2, TCA9543A/NAU7802 at the same addresses; pumps on TIM3 (PA6/PA7), ring on TIM1 (PA8), buzzer on TIM14 (PA4); PCA9685 OE PB0, DOCK_SENSE_A PB3, DOCK_PWR_OK PB5, status LED PC7 (PCF8574 #2 P2 spare). Host-loss rule: 5 s without HB_AP → dispensers and ring off (no safety authority on the DU STM32). §1–§3 (new §3.2–§3.7), §5.2, §6.1–§6.9 (new §6.9 DU STM32), §7, §8, §9, §10.14, §11, §12, §13 (+$1, ~$40), §14.1, §15 (DU tasks, `link` thread, thin `dispense` client, DU_STATUS-based power-fail/lift sequences) and §16 rewritten; area ~3,100 mm² |
| 4.2 | 2026-11-04 | Manas Pradhan | Dispensing Unit battery doubled to 2S2P 18650, 5,200 mAh (~38 Wh; was 2S1P 2,600 mAh): block diagram, §4.6.1 pack table, charge time ≈ 5.5 h at the unchanged 1.0 A TP5100 setting (R_CS 0.1 Ω), rail table run time > 12 h, off-board pack $12 → $22 (production $8 → $15), checklist. No PCB change: J_BAT, charger, INA219 #2 and VSYS diode-OR are unchanged |
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
