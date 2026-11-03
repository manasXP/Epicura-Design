---
created: 2026-03-06
modified: 2026-11-04
version: 3.0
status: Draft
---

# Unified PCB Design

## 1. Overview

This document covers the single custom PCB for Epicura. One 160 x 90 mm, 4-layer board carries everything electrical except the induction surface, the stirring motor and the AC power supply:

- **STM32G474RE** real-time controller (dispensers, safety, both CAN ports, LED ring, sensors)
- **Application-processor slot (J_AP)** for the Luckfox Pico Zero (RV1106G3): 5 V power, SPI0/I2C3 break-out to the display, UART service link, BOOT0/NRST control, and the RV1106's own CAN node (MCP2518FD) on the isolated bus
- **CID + FID drivers**: the Dispenser Servo Block (PCA9685, switched 6.5 V rail, J_DSB1-4)
- **SLD drivers**: TB6612FNG peristaltic pump driver, two solenoid MOSFETs, two HX711 load-cell channels (J_SLD)
- **Two CAN ports**: an **isolated** port to the cooking surface (J_CAN_HEAT, ISO1050DUB x2 in one isolated 5 V domain) and a **regular** non-isolated port to the stirring motor (J_CAN_STIR, SN65HVD230 on FDCAN2)
- **2-cell Li-ion backup** with on-board charger (TP5100) and automatic switch-over of the 5 V logic domain
- Power conversion (24 V in; 12 V, 6.5 V, 5 V, 3.3 V rails), exhaust fans, buzzer, LED ring, safety relay, E-stop, USB-C programming port

There is no IR thermometer on the board or in the system: the cooking surface runs its own temperature loop and reports its measured temperature on the isolated CAN bus (see [[../05-Subsystems/01-Induction-Heating|Induction Heating]]).

### 1.1 Single-Board Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Unified PCB (160 x 90 mm, 4-layer)                                       │
│                                                                          │
│   ┌──────────────────────────┐   ┌───────────────┐   ┌────────────────┐  │
│   │ J_AP: Luckfox Pico Zero  │   │ STM32G474RE   │   │ 2S Li-ion      │  │
│   │ slot (2x20 socket +      │   │ FreeRTOS      │   │ TP5100 charger │  │
│   │ 4 standoffs)             │   │               │   │ J_BAT          │  │
│   └──────┬───────────────────┘   └──┬──────┬─────┘   └────────────────┘  │
│          │ SPI0 CS1 (MCP2518FD)     │FDCAN1│FDCAN2                        │
│          ▼                          ▼      ▼                             │
│   ┌──────────────────────────────┐  ┌──────────────┐                     │
│   │ Isolated CAN domain          │  │ SN65HVD230   │──► J_CAN_STIR       │
│   │ ISO1050 #1 (STM32)           │  │ (regular CAN)│    stirring motor   │
│   │ ISO1050 #2 (RV1106 MCP2518FD)│  └──────────────┘                     │
│   │ B0505S 5V_ISO ──► J_CAN_HEAT │──► cooking surface                    │
│   └──────────────────────────────┘                                       │
│                                                                          │
│   DSB: PCA9685 + 6.5 V rail ──► J_DSB1-4 (FID x3, CID x1 servos)         │
│   SLD: TB6612FNG + 2 MOSFET + 2 HX711 ──► J_SLD                           │
│   Fans, buzzer, LED ring, relay, E-stop, USB-C, SWD, J_LCD break-out      │
└──────────────────────────────────────────────────────────────────────────┘
          ▲ 24 V DC (Mean Well LRS-75-24)            ▲ battery pack (2S1P)
```

### 1.2 Scope

| Item | Status | Notes |
|------|--------|-------|
| **Unified PCB** (this document) | Custom design required | STM32G474RE + AP slot + power + dispenser drivers + two CAN ports + battery backup; 160 x 90 mm, 4-layer |
| **Luckfox Pico Zero** | Off-the-shelf module | Plugs into J_AP; camera on its own CSI FPC; display routed through the PCB to J_LCD |
| **Microwave induction surface** | Commercial module with CAN port and internal temperature control (see [[../05-Subsystems/01-Induction-Heating\|Induction Heating]]) | Isolated CAN port J_CAN_HEAT |
| **Stirring motor** | 24 V integrated BLDC servo with CAN interface (see [[../05-Subsystems/02-Robotic-Arm\|Robotic Arm]]) | Regular CAN port J_CAN_STIR + 24 V on J_STIR_PWR |
| **Battery pack** | 2S1P Li-ion 18650 pack with protection module (PCM) | Charged and monitored on the PCB; connects to J_BAT |

### 1.3 Changes from v2.x

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

Roughly 95 ICs/discretes and 19 connectors occupy about 3,600 mm² of the ~12,000 mm² usable placement area (~30 % utilisation). The Pico Zero footprint (65 x 30 mm) sits over the digital zone on standoffs; the area under it is used for passives on the bottom side. No board size increase is needed.

---

## 2. Board-Level Block Diagram

```
24V DC Input ─── F1 (5A) ─── SS54 ─── SMBJ24A ─── 24V Internal Rail ──┬──► INA219 shunt ──► 24V_MON
                                                                      │
          ┌───────────────────────┬──────────────────────┬────────────┼─────────────────────┐
          ▼                       ▼                      ▼            ▼                     ▼
  ┌──────────────┐        ┌──────────────┐      ┌─────────────┐  Voltage div.   ┌──────────────────┐
  │ MP1584EN #1  │        │ MP1584EN #2  │      │ Q_STIR      │  100k/10k       │ D_SYS1 (SS54)    │
  │ 24V → 12V 3A │        │ 24V → 6.5V 3A│      │ P-MOSFET    │     │           │ 24V ──► VSYS     │
  └──────┬───────┘        └──────┬───────┘      │ 24V_STIR    │     ▼           └────────┬─────────┘
         │                       │              └──────┬──────┘  PA1 (COMP1)              │
     12V Rail             6.5V DSB rail                │        Power Fail        VSYS ◄──┴── D_SYS2 (SS54) ◄── BAT+
         │                (switched by               J_STIR_PWR                     │
   ┌─────┼────────┐        PCF8574 #2 P4)                                           ▼
   ▼     ▼        ▼              │                                           ┌──────────────┐
 Fans  SLD pumps  TP5100         ▼                                           │ TPS54531     │
 2x    TB6612 +   2S charger   PCA9685 ──► J_DSB1-4                          │ VSYS → 5V 5A │
       2 solenoid  │                                                         └──────┬───────┘
                   ▼                                                                │
               J_BAT (2S1P pack, 6.0–8.4 V, PCM) ──────────────────────────► BAT+   5V_MAIN ──┬──► J_AP pins 2/4 (Pico Zero)
                                                                                              ├──► J_LCD VCC, LED ring (switched)
USB-C VBUS ─── F_USB (500mA) ─── 5V_USB ── D_OR2 ──┐                                          ├──► B0505S ──► 5V_ISO (CAN domain)
                                                    ├── 5V_MERGED ──► AMS1117-3.3 ──► 3.3V    │
                                   5V_MAIN ── D_OR1 ┘                                          └──► Buzzer

                     ┌────────────────────────────────────────────────────────────────┐
                     │  STM32G474RE (LQFP-64)                                         │
                     │  FDCAN1 (PB8/PB9) ──► ISO1050DUB #1 ─┐ isolated domain         │
   J_AP SPI0 CS1 ───►│  MCP2518FD (RV1106 node) ──► ISO1050DUB #2 ─┴─► J_CAN_HEAT     │
                     │  FDCAN2 (PB12/PB13) ──► SN65HVD230 ──► J_CAN_STIR               │
                     │  USART2 (PA2/PA3) ◄──► J_AP UART2 (service link)                │
                     │  I2C1 (PA15/PB7) ──► PCA9685 (0x41), PCF8574 #2 (0x21), INA219 │
                     │  TIM15 (PB14) ──► 74AHCT1G125 ──► J_LED (WS2812B)               │
                     │  TIM3/TIM2 ──► fans; TIM8 ──► buzzer; GPIO ──► TB6612, relay    │
                     │  ADC: PC0 VBUS detect, PC1 battery voltage                     │
                     │  USB (PA11/PA12) ──► J_USB                                      │
                     └────────────────────────────────────────────────────────────────┘
```

---

## 3. STM32G474RE Pin Allocation

### 3.1 Pin Map (LQFP-64)

```
STM32G474RE (LQFP-64) — Unified PCB v3 Pin Assignment
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌─── FDCAN1: Isolated bus to the cooking surface ─────────┐   │
│  │  PB8  (FDCAN1_RX, AF9) ◄── ISO1050DUB #1 RXD               │
│  │  PB9  (FDCAN1_TX, AF9) ──► ISO1050DUB #1 TXD               │
│  │  500 kbps classic CAN; nodes: STM32, RV1106 (MCP2518FD),  │
│  │  induction surface MCU                                    │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── FDCAN2: Regular bus to the stirring motor ────────────┐  │
│  │  PB12 (FDCAN2_RX, AF9) ◄── SN65HVD230 R                   │
│  │  PB13 (FDCAN2_TX, AF9) ──► SN65HVD230 D                   │
│  │  1 Mbps classic CAN (motor protocol), non-isolated        │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── USART2: Service link to the RV1106 ───────────────────┐  │
│  │  PA2  (USART2_TX) ──► J_AP pin 10 (UART2_RX)              │
│  │  PA3  (USART2_RX) ◄── J_AP pin 8  (UART2_TX)              │
│  │  BOOT0 ◄── J_AP GPIO 32 (via 1k) ; NRST ◄── J_AP GPIO 38  │
│  │  ST system bootloader + debug console only                │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── I2C1: On-board peripherals ───────────────────────────┐  │
│  │  PA15 (I2C1_SCL, AF4) ──► PCA9685 (0x41) DSB servo PWM    │
│  │  PB7  (I2C1_SDA, AF4) ◄─► PCF8574 #2 (0x21) GPIO expander │
│  │                        ◄─► INA219 (0x40) 24 V monitor     │
│  │  100 kHz, 4.7k pull-ups to 3.3V, 3 devices                │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── DSB: Dispenser Servo Block (FID + CID) ───────────────┐  │
│  │  PB4  (GPIO)      ──► PCA9685 OE (active-low; high = off) │
│  │  PCF8574 #2 P4    ──► 6.5 V DSB rail enable (Q_DSB)       │
│  │  PCA9685 CH0..CH3 ──► J_DSB1..4 (C1, F1, F2, F3)           │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── SLD: Standard Liquid Dispenser ───────────────────────┐  │
│  │  PC3  (GPIO)      ──► SLD-OIL Pump PWM (TB6612 PWMA)      │
│  │  PC4  (GPIO)      ──► SLD-OIL Pump DIR (TB6612 AIN1)      │
│  │  PC5  (GPIO)      ──► SLD-WATER Pump PWM (TB6612 PWMB)    │
│  │  PC6  (GPIO)      ──► SLD-WATER Pump DIR (TB6612 BIN1)    │
│  │  PCF8574 #2 P0    ──► SLD-OIL Solenoid (IRLML6344)        │
│  │  PCF8574 #2 P1    ──► SLD-WATER Solenoid (IRLML6344)      │
│  │  PC11 / PC12      ──► / ◄── HX711 SCK / DOUT (oil)        │
│  │  PC9  / PC10      ──► / ◄── HX711 SCK / DOUT (water)      │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Exhaust Fans (2x 120mm) ──────────────────────────────┐  │
│  │  PA6  (TIM3_CH1)  ──► Fan 1 PWM (25 kHz)                  │
│  │  PB10 (TIM2_CH3)  ──► Fan 2 PWM (25 kHz)                  │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── LED Ring (WS2812B) ───────────────────────────────────┐  │
│  │  PB14 (TIM15_CH1 + DMA) ──► 74AHCT1G125 ──► J_LED DATA    │
│  │  PCF8574 #2 P3    ──► ring 5 V switch (Q_LED_N/Q_LED_P)   │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Audio ────────────────────────────────────────────────┐  │
│  │  PC7  (TIM8_CH2)  ──► Piezo Buzzer PWM (via 2N7002)       │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── USB: Native USB 2.0 Full-Speed ───────────────────────┐  │
│  │  PA11 (USB_DM) ◄─► 22R ── USBLC6-2SC6 ── J_USB D-         │
│  │  PA12 (USB_DP) ◄─► 22R ── USBLC6-2SC6 ── J_USB D+         │
│  │  PC0  (ADC)    ◄── VBUS detect (100k/100k divider)        │
│  │  BOOT0 ── SW_BOOT (tactile to 3.3V) + 10k pull-down       │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Power Monitoring ─────────────────────────────────────┐  │
│  │  PA1  (COMP1_INP) ◄── 24V divider 100k/10k (power fail)   │
│  │  PC1  (ADC1_IN7)  ◄── BAT+ divider 100k/22k (pack volts)  │
│  │  PCF8574 #2 P6/P7 ◄── TP5100 CHRG / STDBY (open-drain)    │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── Stirring motor power ─────────────────────────────────┐  │
│  │  PCF8574 #2 P5    ──► Q_STIR (24 V_STIR high-side switch) │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  ┌─── GPIO: Safety & Control ───────────────────────────────┐  │
│  │  PB0  (GPIO)      ──► Safety Relay (via Q_RLY 2N7002)     │
│  │  PB1  (GPIO)      ◄── Pot Detection (reed switch, backup) │
│  │  PB2  (GPIO/EXTI) ◄── E-Stop Button (NC, active-low)      │
│  │  PCF8574 #2 P2    ──► Status LED                          │
│  └───────────────────────────────────────────────────────────┘
│                                                                │
│  Debug: SWD via PA13 (SWDIO) / PA14 (SWCLK), J_SWD             │
│  Clock: 8 MHz HSE (PF0, PF1); 32.768 kHz LSE (PC14, PC15)      │
│  Reset: NRST with 100nF + SW_RST; also driven by J_AP GPIO 38  │
│  Free: PA0, PA4, PA5, PA7, PA8, PA9, PA10, PB3, PB5, PB6,      │
│        PB11, PB15, PC2, PC8, PC13, PD2 (16 pins)               │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 Pin Summary Table

| Pin | Function | Peripheral | Direction | Destination | Subsystem |
|-----|----------|------------|-----------|-------------|-----------|
| PA0 | — | Available | — | — | — |
| PA1 | COMP1_INP | 24V Power Fail Detection | Input | 100k/10k divider | Power |
| PA2 | USART2_TX | Service link | Output | J_AP pin 10 (RV1106 UART2_RX) | Comms |
| PA3 | USART2_RX | Service link | Input | J_AP pin 8 (RV1106 UART2_TX) | Comms |
| PA4 | — | Available (was BLDC EN) | — | — | — |
| PA5 | — | Available (was BLDC DIR) | — | — | — |
| PA6 | TIM3_CH1 | Exhaust Fan 1 PWM | Output | IRLML6344 gate → J_FAN | Exhaust |
| PA7 | — | Available | — | — | — |
| PA8 | — | Available (was BLDC PWM) | — | — | — |
| PA9 | — | Available | — | — | — |
| PA10 | — | Available | — | — | — |
| PA11 | USB_DM | USB D- | Bidir | USBLC6-2SC6 → 22R → J_USB | USB |
| PA12 | USB_DP | USB D+ | Bidir | USBLC6-2SC6 → 22R → J_USB | USB |
| PA13 | SWDIO | SWD Debug | Bidir | J_SWD | Debug |
| PA14 | SWCLK | SWD Debug | Input | J_SWD | Debug |
| PA15 | I2C1_SCL | PCA9685 / PCF8574 #2 / INA219 | Output | On-board ICs | I2C |
| PB0 | GPIO | Safety Relay | Output | Q_RLY (2N7002) → J_SAFE | Safety |
| PB1 | GPIO | Pot Detection (backup) | Input | Reed switch (10k pull-up) → J_SAFE | Safety |
| PB2 | GPIO (EXTI) | E-Stop | Input | NC button (10k pull-up, RC debounce) → J_SAFE | Safety |
| PB3 | — | Available | — | — | — |
| PB4 | GPIO | PCA9685 OE | Output | PCA9685 OE (10k pull-up = off) | DSB |
| PB5 | — | Available | — | — | — |
| PB6 | — | Available | — | — | — |
| PB7 | I2C1_SDA | PCA9685 / PCF8574 #2 / INA219 | Bidir | On-board ICs | I2C |
| PB8 | FDCAN1_RX | Isolated CAN RX | Input | ISO1050DUB #1 RXD | CAN (heater) |
| PB9 | FDCAN1_TX | Isolated CAN TX | Output | ISO1050DUB #1 TXD | CAN (heater) |
| PB10 | TIM2_CH3 | Exhaust Fan 2 PWM | Output | IRLML6344 gate → J_FAN | Exhaust |
| PB11 | — | Available | — | — | — |
| PB12 | FDCAN2_RX | Regular CAN RX | Input | SN65HVD230 R | CAN (stir) |
| PB13 | FDCAN2_TX | Regular CAN TX | Output | SN65HVD230 D | CAN (stir) |
| PB14 | TIM15_CH1 | WS2812B data (PWM + DMA) | Output | 74AHCT1G125 → J_LED | LED ring |
| PB15 | — | Available | — | — | — |
| PC0 | ADC1_IN6 | USB VBUS Detect | Input | 100k/100k divider from 5V_USB | USB |
| PC1 | ADC1_IN7 | Battery pack voltage | Input | 100k/22k divider from BAT+ | Battery |
| PC2 | — | Available | — | — | — |
| PC3 | GPIO | SLD Pump 1 PWM | Output | TB6612 PWMA | SLD |
| PC4 | GPIO | SLD Pump 1 DIR | Output | TB6612 AIN1 | SLD |
| PC5 | GPIO | SLD Pump 2 PWM | Output | TB6612 PWMB | SLD |
| PC6 | GPIO | SLD Pump 2 DIR | Output | TB6612 BIN1 | SLD |
| PC7 | TIM8_CH2 | Buzzer PWM | Output | 2N7002 gate → J_BUZZER | Audio |
| PC8 | — | Available | — | — | — |
| PC9 | GPIO | HX711 SCK (SLD water) | Output | J_SLD pin 15 | SLD |
| PC10 | GPIO | HX711 DOUT (SLD water) | Input | J_SLD pin 16 | SLD |
| PC11 | GPIO | HX711 SCK (SLD oil) | Output | J_SLD pin 13 | SLD |
| PC12 | GPIO | HX711 DOUT (SLD oil) | Input | J_SLD pin 14 | SLD |
| PC13 | — | Available | — | — | — |
| PD2 | — | Available | — | — | — |

### 3.3 PCF8574 #2 (I2C 0x21) Allocation

| Pin | Direction | Signal | Notes |
|-----|-----------|--------|-------|
| P0 | Out | SLD solenoid 1 (oil) | IRLML6344 gate via 100R, 10k pull-down |
| P1 | Out | SLD solenoid 2 (water) | IRLML6344 gate via 100R, 10k pull-down |
| P2 | Out | Status LED (green) | Active-low, 330R |
| P3 | Out | LED ring 5 V enable | Q_LED_N → Q_LED_P high-side switch |
| P4 | Out | DSB 6.5 V rail enable | Q_DSB P-MOSFET via level shifter; on only during a dispense |
| P5 | Out | Stirring motor 24 V enable | Q_STIR P-MOSFET via level shifter; off in boot-safe state and on E-stop |
| P6 | In | TP5100 CHRG (charging, active-low) | Open-drain from charger, 10k pull-up |
| P7 | In | TP5100 STDBY (charge complete, active-low) | Open-drain from charger, 10k pull-up |

### 3.4 Pin Usage Summary

| Port | Used | Available |
|------|------|-----------|
| PA | PA1, PA2, PA3, PA6, PA11–PA15 (9) | PA0, PA4, PA5, PA7, PA8, PA9, PA10 (7) |
| PB | PB0, PB1, PB2, PB4, PB7–PB10, PB12–PB14 (11) | PB3, PB5, PB6, PB11, PB15 (5) |
| PC | PC0, PC1, PC3–PC7, PC9–PC12, PC14, PC15 (13) | PC2, PC8, PC13 (3) |
| PD | — | PD2 (1) |

The arm's PWM/EN/DIR pins and the stepper/pneumatic lines of earlier revisions are gone; the board has 16 spare GPIO.

---

## 4. Power Supply

### 4.1 24V Input Protection Chain

```
24V from PSU ──── F1 (5A Polyfuse) ──── D1 (SS54 Schottky) ────┬── D2 (SMBJ24A TVS) ──── 24V Internal
                                                               │
                                                              GND
```

| Component | Part | Function |
|-----------|------|----------|
| F1 | 5A resettable polyfuse (1812) | Overcurrent protection |
| D1 | SS54 (5A, 40V Schottky) | Reverse polarity protection (0.5 V drop) |
| D2 | SMBJ24A (24V TVS, 600W) | Transient suppression |

### 4.2 24V → 12V Rail (MP1584EN #1)

| Parameter | Value |
|-----------|-------|
| Output | 12 V, 3 A max |
| Load | Exhaust fans (2x 0.5 A), SLD peristaltic pumps (2x 0.6 A, during dispense), SLD solenoids (2x 0.5 A, during dispense), TP5100 battery charger (~0.4 A while charging) |
| Inductor | 33 µH, CDRH104R, Isat > 4 A |
| Output caps | 2x 22 µF MLCC (25 V) + 100 µF electrolytic |
| Feedback | 140k / 10k → 12.0 V |
| Protection | 3 A polyfuse + SMBJ12A on output |

The 5 V buck no longer hangs off this rail (see §4.4), so the 12 V rail carries only actuators and the charger: ~1.4 A typical, 3 A worst case with both pumps and both solenoids energised at once (the firmware never does this).

### 4.3 24V → 6.5V DSB Rail (MP1584EN #2)

| Parameter | Value |
|-----------|-------|
| Output | 6.5 V, 3 A max |
| Load | Four DS3218-class servos (one moving at a time, ~2.5 A stall) |
| Inductor | 22 µH, CDRH104R, Isat > 4 A |
| Output caps | 2x 22 µF MLCC + 470 µF electrolytic (10 V, low ESR) |
| Feedback | 71.5k / 10k → 6.52 V |
| Rail switch | Q_DSB: AO3401-class P-MOSFET high-side switch driven by a 2N7002 level shifter from PCF8574 #2 P4; 1000 µF bulk and 3 A PTC on the switched side |
| Protection | 3 A polyfuse + SMBJ6.5A TVS |

The rail is on only while a dispense is in progress (no idle servo buzz or heat).

### 4.4 VSYS Diode-OR and 24V/Battery → 5V Rail (TPS54531)

The 5 V logic domain (Pico Zero, LCD, STM32, CAN domain, LED ring) is fed by a TPS54531 buck whose input, **VSYS**, is a Schottky-OR of the 24 V internal rail and the 2S battery:

```
24V Internal ──── D_SYS1 (SS54) ──┐
                                   ├── VSYS ──── C_SYS (100 µF, 35 V) ──► TPS54531 VIN
BAT+ (6.0–8.4 V) ─ D_SYS2 (SS54) ──┘
```

| Parameter | Value |
|-----------|-------|
| IC | TPS54531DDAR (SOIC-8), VIN 3.5–28 V |
| Input | VSYS: ~23.5 V on mains, 5.5–7.9 V on battery |
| Output | 5V_MAIN, 5 A max |
| Load | Pico Zero (0.2 A typ, 0.5 A peak), LCD + backlight (0.15 A), AMS1117 (0.2 A), B0505S (0.05 A), LED ring (1 A peak, switched), buzzer |
| Inductor | 10 µH, CDRH127, Isat > 6 A |
| Output caps | 2x 22 µF MLCC (10 V) + 220 µF electrolytic |
| Feedback | 52.3k / 10k → 4.98 V |
| Switching | 570 kHz; integrated OCP, thermal shutdown, UVLO |

Switch-over is passive: while 24 V is present D_SYS2 is reverse-biased and the battery does not discharge; when the PSU drops out VSYS falls to the battery voltage within the hold-up of C_SYS and the buck keeps regulating. No controller, no gap.

### 4.5 Two-Source 5V Schottky-OR → 3.3V Rail (AMS1117-3.3)

```
5V_MAIN (TPS54531, 24 V or battery) ── D_OR1 (SS14) ──┐
                                                        ├── 5V_MERGED ──► AMS1117-3.3 ──► 3.3V
5V_USB  (USB-C VBUS, 500 mA polyfuse) ── D_OR2 (SS14) ──┘
```

The USB path powers only the STM32 (3.3 V) for bench programming without a PSU or battery; it does not feed the Pico Zero or the 5V_MAIN loads (D_OR1 blocks back-feed). 3.3 V loads: STM32 (~0.15 A), PCA9685, PCF8574 #2, INA219, SN65HVD230, ISO1050 VCC1 x2, MCP2518FD, 74AHCT1G125, HX711 supply via J_SLD.

### 4.6 2-Cell Li-ion Backup

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
| Placement | In the cool zone of the enclosure (< 45 °C), away from the induction module and the PSU |

#### 4.6.2 Charger (TP5100)

```
12V Rail ──── C_CHG_IN (10 µF) ──► TP5100 VIN
                                     │
                                     ├── SW ── L_CHG (22 µH, Isat > 2 A) ──┬── R_CS (sense) ──► BAT+ (J_BAT pin 1)
                                     │        D_CHG (SS34) to GND         │
                                     ├── VS / BAT ◄──────────────────────┘   (charge current = 0.1 V / R_CS; 0.2 Ω → 0.5 A)
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
| Charge current | 0.5 A (R_CS = 0.2 Ω, 1 %, 0.5 W) — ~5–6 h full charge; keeps the charger within the 12 V rail budget |
| Pre-charge / termination | Trickle below 2S 6.0 V; termination at C/10; auto-recharge when the pack sags |
| Temperature window | NTC on TEMP pin: charging inhibited below 0 °C or above 45 °C |
| Status | CHRG and STDBY open-drain outputs to PCF8574 #2 P6/P7 (polled at 1 Hz) and two 0603 LEDs |
| Charge source | 12 V rail only: the pack charges whenever the PSU is on |

> [!note]
> Confirm the R_CS current-programming relation and the CS pin strapping against the TP5100 datasheet before layout; retailer summaries disagree on the input range (5–12 V vs 5–18 V), which is why the charger runs from the 12 V rail and not from 24 V.

#### 4.6.3 Discharge Path and Monitoring

- BAT+ → D_SYS2 (SS54) → VSYS → TPS54531 (§4.4). Maximum discharge ~1.2 A at 6 V for the full 5 V load; the pack PCM over-current limit (3–5 A) is well above this.
- Pack voltage → R_BAT1 (100k) / R_BAT2 (22k) divider → PC1 (ADC1_IN7): 8.4 V reads 1.51 V, 6.0 V reads 1.08 V. The STM32 reports `batt_mV` and `on_battery` in STATUS (0x312) and TELEMETRY flags; the RV1106 shows the state of charge and warns below 6.4 V.
- Under-voltage: at 6.2 V the STM32 asks the RV1106 to checkpoint and shut down (the pack PCM cuts off at 2x 2.8 V = 5.6 V as a last resort).

#### 4.6.4 What the Battery Powers

| Domain | On battery | Notes |
|--------|------------|-------|
| 5V_MAIN: Pico Zero, LCD, STM32 (3.3 V), CAN transceivers, MCP2518FD | Yes | ~3 W total → > 4 h from a 2,600 mAh pack |
| LED ring (5 V, switched) | Off | Firmware turns it off on POWER_FAIL |
| 12 V rail (fans, pumps, solenoids, charger) | No | 24 V gone |
| 6.5 V DSB rail, 24 V_STIR, safety relay | No | Actuators and the heater are down; the induction surface has its own AC inlet and is off anyway |

The purpose is the same as the earlier UPS: ride through short outages, checkpoint the cooking session, keep the UI and Wi-Fi alive to tell the user, and resume if mains returns within the food-safety window (see [[../03-Software/03-Main-Loop-State-Machine|Main Loop State Machine]] §5).

### 4.7 24V Power Failure Detection

```
24V Internal ─── R_DIV1 (100k) ──┬── R_DIV2 (10k) ─── GND
                                  ├── C_DIV (100nF) ─── GND
                                  └── PA1 (COMP1_INP)
```

| Parameter | Value |
|-----------|-------|
| Divider | 1/11 → 2.18 V at 24 V |
| Threshold | ~1.5 V (COMP1 internal reference) → trips at 24 V < 16.5 V |
| Recovery | > 20 V sustained 2 s (firmware debounce) |
| Response | < 100 µs (comparator interrupt) |

On trip the STM32 sets `on_battery`, drops all 12 V / 6.5 V / 24 V_STIR loads (they are already unpowered), turns the LED ring off and reports POWER_FAIL in STATUS (0x312).

### 4.8 Stirring Motor 24 V Switch (Q_STIR)

```
24V Internal ──► F_STIR (5A polyfuse) ──► Q_STIR: AO4407A (P-MOSFET, SO-8, -30 V, -12 A) ──► 24V_STIR ──► J_STIR_PWR pin 1
                                             Gate ── 10k to 24V (default OFF)
                                             Gate ── Q_STIR_N: 2N7002 drain; 2N7002 gate ── 100R ── PCF8574 #2 P5, 10k pull-down
                                          C_STIR: 100 µF (35 V) + 10 µF MLCC on 24V_STIR
```

The motor's own firmware stops it when CAN commands cease, but the switch gives the STM32 a command-independent way to de-energise the arm in the boot-safe state and on E-stop, and lets the INA219 attribute 24 V current to the arm when the DSB rail is off.

### 4.9 Per-Rail Protection

| Rail | Polyfuse | TVS | Notes |
|------|----------|-----|-------|
| 24V input | 5 A (F1) | SMBJ24A | Reverse-polarity SS54 |
| 24V_STIR | 5 A (F_STIR) | — | Behind Q_STIR |
| 12V | 3 A | SMBJ12A | Fans, pumps, solenoids, charger |
| 6.5V DSB | 3 A PTC | SMBJ6.5A | Switched rail |
| 5V_MAIN | — | — | TPS54531 integrated OCP |
| 5V_USB | 500 mA (F_USB) | USBLC6 on data | |
| Battery | Pack PCM | — | D_SYS2 blocks back-charging from 24 V |

### 4.10 Decoupling

- 100 nF MLCC within 5 mm of every STM32 VDD/VSS pair; 1 µF + ferrite bead on VDDA
- 10 µF at LDO input and output; 100 nF + 10 µF at each IC supply pin (PCA9685, PCF8574, INA219, MCP2518FD, SN65HVD230, ISO1050 both sides, 74AHCT1G125, TB6612 VM and VCC)

### 4.11 Power Budget Summary

| Rail | Source | Typical (A) | Peak (A) | Notes |
|------|--------|-------------|----------|-------|
| 3.3V (0.8 A max) | 5V_MERGED → AMS1117 | 0.20 | 0.35 | STM32, I2C ICs, CAN logic sides, HX711 |
| 5V_MAIN (5 A max) | VSYS → TPS54531 | 0.7 | 2.2 | Pico Zero + LCD + LDO + B0505S + LED ring (1 A peak) |
| 12V (3 A max) | 24V → MP1584EN #1 | 1.4 | 3.0 | Fans continuous; pumps/solenoids and charger intermittent |
| 6.5V DSB (3 A max) | 24V → MP1584EN #2 | 0 | 2.5 | One servo at a time, rail off when idle |
| 5V_ISO (200 mA max) | 5V_MAIN → B0505S | 0.08 | 0.15 | Two ISO1050 bus sides (~40 mA each) |
| 24V_STIR | 24V via Q_STIR | 1.0 | 3.0 | CAN servo motor |
| **24V input** | **Mean Well LRS-75-24 (3.2 A)** | **~2.1** | **~3.0** | Firmware sequences DSB moves and dispenses so the peak is never simultaneous with full stir torque |
| Battery (on outage) | 2S pack → D_SYS2 → VSYS | 0.45 @ 7.4 V | 0.8 | Logic domain only |

> [!warning]
> The 24 V budget is tight at peak. The firmware mutex that already serialises DSB servo moves must also hold the stir motor at reduced torque (velocity limit) while a carousel indexes. Monitor the INA219 during integration and move to the LRS-100-24 (4.2 A) if the measured peak exceeds 3 A.

### 4.12 Isolated 5V DC-DC (B0505S-1WR3)

The two ISO1050DUB transceivers on the heater bus share one isolated supply domain: 5V_ISO / GND_ISO from a Mornsun B0505S-1WR3 fed from 5V_MAIN (before the Schottky-OR, for margin).

| Parameter | Value |
|-----------|-------|
| IC | B0505S-1WR3 (SIP-4), 1 W, 3 kVDC isolation |
| Input | 5V_MAIN; C_ISO_IN 10 µF |
| Output | 5V_ISO, 200 mA max; C_ISO_OUT 10 µF; loads: ISO1050 #1 + #2 VCC2 (~80 mA), optional J_CAN_HEAT pin 4 |

---

## 5. Application-Processor Slot (J_AP) and Display Break-out

### 5.1 Mechanical

The Luckfox Pico Zero (65 x 30 mm) seats pin-down into **J_AP**, a 2x20 2.54 mm female socket (8.5 mm stacking height) on the top side of the PCB, and is held by four M2.5 standoffs matching the module's 3.0 mm mounting holes. Its USB-C port and microSD slot face the board edge for access; the CSI FPC connector faces inward with clearance for the camera cable. The module's on-board Wi-Fi antenna must point away from the buck converters; the external-antenna connector is used when the enclosure is metal.

### 5.2 J_AP Pin Usage (RPi-style 40-pin numbering)

| Header pin | Signal | Direction (Pico Zero view) | Destination on PCB | Purpose |
|------------|--------|-----------------------------|--------------------|---------|
| 2, 4 | 5V | In | 5V_MAIN (battery-backed) | Module power, ~1 W (its own regulators make 3.3 V / core rails) |
| 1, 17 | 3.3V (module output) | Out | Not used | STM32 side has its own 3.3 V |
| 6, 9, 14, 20, 25, 30, 34, 39 | GND | — | GND | Common ground |
| 8 | UART2_TX | Out | STM32 PA3 (USART2_RX) | Service link (ST bootloader, debug) |
| 10 | UART2_RX | In | STM32 PA2 (USART2_TX) | Service link |
| 19 | SPI0_MOSI | Out | J_LCD MOSI and MCP2518FD SI | Shared SPI0 |
| 21 | SPI0_MISO | In | MCP2518FD SO (and J_LCD MISO, unused) | Shared SPI0 |
| 23 | SPI0_CLK | Out | J_LCD SCK and MCP2518FD SCK | 50 MHz to LCD, ≤ 20 MHz when CS1 is active |
| 24 | SPI0_CS0 | Out | J_LCD CS | LCD chip select |
| SPI0_CS1 (GPIO 58) | SPI0_CS1 | Out | MCP2518FD nCS | RV1106 CAN node |
| spare GPIO (fixed at layout) | GPIO | In | MCP2518FD nINT | CAN RX/TX FIFO interrupt, active-low |
| 22 | GPIO | Out | J_LCD DC | LCD data/command |
| 26 | GPIO | Out | J_LCD RST | LCD reset |
| 12 | PWM10 | Out | J_LCD BL | Backlight PWM (via on-board 2N7002 if the module's LED pin needs > 20 mA) |
| 16 | I2C3_SDA | Bidir | J_LCD TP_SDA | FT6336 touch, 4.7k pull-up on PCB |
| 18 | I2C3_SCL | Bidir | J_LCD TP_SCL | FT6336 touch, 4.7k pull-up on PCB |
| 36 | GPIO | In | J_LCD TP_INT | Touch interrupt, active-low |
| spare GPIO (fixed at layout) | GPIO | Out | J_LCD TP_RST | Touch reset |
| GPIO 32 | GPIO | Out | STM32 BOOT0 via R_AP_BOOT (1k) | Enter the ST system bootloader for STM32 firmware update |
| GPIO 38 | GPIO | Out | STM32 NRST via R_AP_RST (1k), open-drain | Reset pulse during firmware update; SW_RST in parallel |
| 35, 37 | UART0 TX/RX | — | J_AP test pads only | Luckfox debug console |
| all others | — | — | Not connected | |

> [!note]
> The RV1106 has no CAN peripheral. Its CAN node (MCP2518FD + ISO1050DUB #2, §7.2) lives on this board and is reached through SPI0 CS1 on J_AP. The LCD and the CAN controller share SPI0 MOSI/MISO/CLK; the controller's 32-object RX FIFO absorbs frames while an LCD DMA chunk (≤ 10 ms) holds the bus.

### 5.3 J_LCD — 4" ST7796S + FT6336 Display Break-out (1x14 pin header, 2.54 mm)

The display module's 14-pin header plugs into J_LCD with a short ribbon (< 20 cm). Pin order follows the common 4.0" ST7796S capacitive SPI module; confirm against the chosen panel before layout.

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

### 5.4 Service UART and Firmware Update

USART2 (PA2/PA3) ↔ UART2 (J_AP 8/10) at 3.3 V carries only the ST system-bootloader protocol (STM32 firmware updates pushed by the RV1106, which drives BOOT0 and NRST through J_AP GPIO 32/38) and an optional debug console at 1 Mbps. All runtime commands and telemetry travel on the isolated CAN bus.

---

## 6. Dispenser and Liquid Drivers

### 6.1 Dispenser Servo Block — PCA9685 + Switched 6.5 V Rail (FID + CID)

```
I2C1 (PA15/PB7) ──► PCA9685PW (addr 0x41, A0 = 1)
   PB4 ──► OE (active-low; 10k pull-up → outputs off until the STM32 drives it low)
   CH0 ──► 100R ──► J_DSB1 SIG    C1  CID carousel index servo
   CH1 ──► 100R ──► J_DSB2 SIG    F1  FID carousel index servo
   CH2 ──► 100R ──► J_DSB3 SIG    F2  FID dose servo (¼ tsp measure wheel)
   CH3 ──► 100R ──► J_DSB4 SIG    F3  FID steam-gate servo
   CH4–CH15 unused (test pads)

6.5 V (MP1584EN #2) ──► Q_DSB (AO3401, high-side) ──► 3 A PTC ──► 1000 µF ──► J_DSB1-4 V+
                          gate ◄── 2N7002 ◄── PCF8574 #2 P4 (rail enable)
```

| Parameter | Value |
|-----------|-------|
| PWM driver | PCA9685PW (TSSOP-28), 16 ch, 12-bit, 50 Hz; EXTCLK not used; V+ pin left unconnected (servo power goes directly from the switched rail to J_DSB) |
| Servo signal | 500–2,500 µs, 3.3 V logic (DS3218-class servos accept 3.3 V) |
| Rail | 6.5 V / 3 A, switched on for the duration of a dispense; one servo moves at a time (firmware mutex) |
| Connectors | J_DSB1–4: 3-pin JR/Futaba servo headers (SIG / V+ / GND), 2.54 mm, keyed, 300 mm extension leads |
| Fault detection | INA219 24 V current window during moves + 3 s move timeout + PCA9685 I2C NACK → DSB_STALL |
| Safe state | OE high and rail off at boot, on E-stop and on heartbeat loss (servos limp) |

### 6.2 SLD Peristaltic Pumps (2x) — TB6612FNG

```
12V Rail ──── VM (TB6612FNG)
   PWMA ◄── PC3 (100R)   AIN1 ◄── PC4 (100R)   AIN2 ── GND     AO1/AO2 ──► J_SLD 1/2 (oil pump)
   PWMB ◄── PC5 (100R)   BIN1 ◄── PC6 (100R)   BIN2 ── GND     BO1/BO2 ──► J_SLD 3/4 (water pump)
   STBY ◄── 3.3V (10k)   VCC ── 3.3V            C_VM: 100 nF + 10 µF
```

| Parameter | Value |
|-----------|-------|
| IC | TB6612FNG (SSOP-24), VM 12 V, VCC 3.3 V |
| Current | 1.2 A continuous / 3.2 A peak per channel; pumps draw ~0.6 A |
| Pull-downs | 10k on PWMA/PWMB (pumps off during boot) |

### 6.3 SLD Solenoid Valves (2x)

Two 12 V normally-closed valves (drip prevention), low-side switched by IRLML6344 with SS14 flyback, 100R gate resistor and 10k gate pull-down, gates driven from PCF8574 #2 P0 (oil) and P1 (water).

### 6.4 SLD Reservoir Load Cells (2x) — HX711

Two 2 kg load cells with external HX711 breakout boards, powered at 3.3 V from J_SLD and bit-banged by the STM32: oil on PC11 (SCK) / PC12 (DOUT), water on PC9 / PC10. 100R series resistors on all four lines; 10 Hz sampling (RATE low); ±1 g with averaging.

### 6.5 Exhaust Fans (2x)

Two 120 mm 12 V fans, low-side PWM at 25 kHz via IRLML6344 + SS14 flyback: fan 1 on PA6 (TIM3_CH1), fan 2 on PB10 (TIM2_CH3). Both independently controllable.

### 6.6 LED Ring (WS2812B) Driver

```
PB14 (TIM15_CH1, PWM + DMA, 800 kHz bit stream) ──► 74AHCT1G125 (3.3 → 5 V, OE tied low) ──► 33R ──► J_LED pin 2 (DATA)
5V_MAIN ──► Q_LED_P (SI2301, high-side) ──► J_LED pin 1 (5V_SW)   gate ◄── Q_LED_N (2N7002) ◄── PCF8574 #2 P3
```

The ring (12–16 LEDs, ~1 A full white) is driven by the STM32, not the RV1106; the RV1106 requests OFF / CAPTURE / NIGHT modes over CAN (SET_LED 0x220). Default OFF (10k pull-up on the P-MOSFET gate).

### 6.7 Piezo Buzzer

5 V active piezo via 2N7002 low-side switch, gate from PC7 (TIM8_CH2) through 100R with 10k pull-down; J_BUZZER 2-pin.

### 6.8 USB-C Programming Port

Unchanged from v2: 16-pin mid-mount USB-C, 5.1k CC pull-downs, USBLC6-2SC6 ESD, 22R series on D+/D−, 500 mA VBUS polyfuse → 5V_USB → D_OR2, VBUS detect divider → PC0, SW_BOOT on BOOT0 for DFU. Powers the STM32 alone for bench work.

---

## 7. CAN Ports

The board has two electrically separate CAN buses on two STM32 FDCAN controllers. They never share a transceiver, a termination or a connector.

| | Isolated port — cooking surface | Regular port — stirring motor |
|---|---|---|
| Connector | J_CAN_HEAT (JST-XH 4-pin) | J_CAN_STIR (JST-XH 3-pin) + J_STIR_PWR (XT30) |
| STM32 controller | FDCAN1, PB8/PB9 | FDCAN2, PB12/PB13 |
| Transceivers | ISO1050DUB #1 (STM32) and ISO1050DUB #2 (RV1106 via MCP2518FD), 5 kV RMS, in one isolated 5V_ISO domain | SN65HVD230 (3.3 V, non-isolated) |
| Nodes | STM32, RV1106, induction surface MCU | STM32, stirring motor controller |
| Bit rate | 500 kbps, classic CAN 2.0B, 11-bit IDs | 1 Mbps, classic CAN 2.0B, 29-bit IDs as the motor protocol requires |
| Termination | 120 Ω on the isolated side at J_CAN_HEAT (both on-board nodes are a short stub); 120 Ω inside the module | 120 Ω at J_CAN_STIR; 120 Ω in or at the motor (verify the motor's built-in termination) |
| Ground | GND_ISO only; shield to GND_ISO at the PCB end | System GND (motor shares the 24 V PSU) |
| Traffic | Host commands/telemetry (RV1106 ↔ STM32), heater setpoint and status, heartbeat, E_STOP — ID map in [[../03-Software/02-Controller-Software-Architecture#7.2 Message Types\|Controller Software Architecture §7.2]] | Motor enable/disable, velocity (and position for homing) commands at 50 Hz; feedback: velocity, position, current, temperature, fault flags |

### 7.1 Isolated Port — ISO1050DUB x2 + B0505S

```
PB9 (FDCAN1_TX) ──► TXD ┐  ISO1050DUB #1  ┌── CANH ─┐
PB8 (FDCAN1_RX) ◄── RXD ┘  VCC1 3.3V / GND │  CANL ─┤
                            VCC2 5V_ISO / GND_ISO              ├── R_TERM1 (120R) ── J_CAN_HEAT 1 (CANH), 2 (CANL)
MCP2518FD TXCAN ──► TXD ┐  ISO1050DUB #2  ┌── CANH ─┤          J_CAN_HEAT 3 GND_ISO, 4 5V_ISO (F_CAN 100 mA)
MCP2518FD RXCAN ◄── RXD ┘  VCC1 3.3V / GND │  CANL ─┘
                            VCC2 5V_ISO / GND_ISO
                  ─── 6 mm milled slot under both ISO1050 and the B0505S output ───
```

- Both transceivers sit on the system-ground side of the slot with their bus pins crossing into the isolated island; no other copper crosses the boundary (creepage ≥ 6.4 mm, IEC 60335 reinforced).
- Decoupling per transceiver: 100 nF on VCC1, 100 nF + 10 µF on VCC2.
- Shielded twisted pair ≤ 1 m to the module.

### 7.2 RV1106 CAN Node — MCP2518FD

| Parameter | Value |
|-----------|-------|
| IC | MCP2518FD-E/SL (SOIC-14), CAN FD controller used in classic mode |
| Clock | 40 MHz crystal (Y3) + 2x 15 pF |
| SPI | J_AP SPI0: MOSI 19, MISO 21, CLK 23, nCS = SPI0_CS1, ≤ 20 MHz |
| INT | nINT → J_AP spare GPIO (fixed at layout), 10k pull-up |
| Supply | 3.3 V, 100 nF + 10 µF |
| Driver | Linux `mcp251xfd` → SocketCAN `can0` at 500 kbps |

### 7.3 Regular Port — SN65HVD230

```
PB13 (FDCAN2_TX) ──► D  ┐ SN65HVD230 (SOIC-8, 3.3 V)  ┌── CANH ──┬── R_TERM2 (120R) ──┬── J_CAN_STIR 1 (CANH)
PB12 (FDCAN2_RX) ◄── R  ┘ Rs ── 10k to GND (slope ctl) └── CANL ──┘                    └── J_CAN_STIR 2 (CANL)
                                                                                           J_CAN_STIR 3 (GND)
PESD1CAN (ESD) across CANH/CANL; 100 nF on VCC
```

| Parameter | Value |
|-----------|-------|
| Transceiver | SN65HVD230DR, 3.3 V, up to 1 Mbps |
| Bit rate | 1 Mbps (motor protocol default); FDCAN2 nominal bit timing from the 170 MHz kernel clock |
| Cable | Twisted pair ≤ 1 m to the motor on the gantry, routed with the 24 V_STIR pair |
| Fail-safe | Motor firmware command timeout (≤ 500 ms) must be configured so a silent bus stops the arm; Q_STIR (§4.8) removes 24 V independently |

---

## 8. Current Monitoring — INA219

INA219 on the 24 V input rail (10 mΩ 2512 shunt, I2C 0x40): power telemetry, DSB stall window during carousel moves, and plausibility check of the stir motor's own current report. Unchanged from v2.

---

## 9. Protection Circuits

### 9.1 ESD Protection

- PRTR5V0U2X on J_SAFE inputs and J_LED data; PESD1CAN on J_CAN_STIR (J_CAN_HEAT is protected by the transceiver's isolation rating plus a PESD1CAN on the isolated side)
- USBLC6-2SC6 on USB D+/D−
- 33R series resistors on J_LCD and J_LED logic lines

### 9.2 Relay Driver (Q_RLY)

PB0 → 330R → 2N7002 → Omron G5V-2 coil (12 V) with 1N4148 flyback; relay cuts AC to the induction surface. NC wiring: relay de-energised = heater disconnected.

### 9.3 E-Stop Input

NC button on J_SAFE pin 3 → PB2 (EXTI) with 10k pull-up and 100 nF; pressed or wire-broken = LOW = stop.

### 9.4 Pot Detection Input (backup)

Reed switch on J_SAFE pin 2 → PB1 with 10k pull-up. Primary pot detection is the module's own, reported in HEAT_STATUS; the reed switch is an optional cross-check.

### 9.5 MOSFET Gate Safety

Every MOSFET gate and driver enable has a 100R series resistor and a 10k pull-down (or pull-up for P-MOSFET switches) so that all actuators, both power switches (Q_DSB, Q_STIR), the LED ring and the relay are off while the STM32 is in reset or unpowered.

---

## 10. Connector Definitions (19 total)

| Ref | Type | Function |
|-----|------|----------|
| J_AP | 2x20 socket, 2.54 mm, 8.5 mm stacking | Luckfox Pico Zero slot |
| J_LCD | 1x14 header, 2.54 mm | 4" ST7796S + FT6336 display |
| J_24V_IN | XT30 2-pin | 24 V DC input |
| J_BAT | JST-XH 2.5 mm 3-pin | 2S Li-ion pack: B+, B−, NTC |
| J_CAN_HEAT | JST-XH 2.5 mm 4-pin | Isolated CAN to the cooking surface |
| J_CAN_STIR | JST-XH 2.5 mm 3-pin | Regular CAN to the stirring motor |
| J_STIR_PWR | XT30 2-pin | Switched 24 V to the stirring motor |
| J_DSB1–J_DSB4 | 3-pin servo header, 2.54 mm | C1, F1, F2, F3 servos |
| J_SLD | Hirose DF11-16DP-2DS (2x8, 2.0 mm) | SLD pumps, solenoids, two HX711 |
| J_FAN | JST-XH 2.5 mm 4-pin | Two exhaust fans |
| J_LED | JST-XH 2.5 mm 3-pin | WS2812B ring: 5V_SW, DATA, GND |
| J_BUZZER | JST-PH 2.0 mm 2-pin | Piezo buzzer |
| J_SAFE | Hirose DF11-4DP-2DS (2x2) | Relay drive, pot detect, E-stop, GND |
| J_SWD | 2x5 1.27 mm | Cortex debug |
| J_USB | USB-C 16-pin mid-mount | STM32 DFU / CDC |

Removed since v2: J_CM5 (replaced by J_AP), J_12V_UPS, J_BLDC, J_PASD, J_PRES, J_CID, J_IR_LED (replaced by J_LED), single-cell J_BAT (replaced by the 3-pin 2S J_BAT).

### 10.1 J_AP — see §5.2

### 10.2 J_LCD — see §5.3

### 10.3 J_24V_IN — 24 V DC Power Input (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | +24V | From Mean Well LRS-75-24 |
| 2 | GND | Power ground |

### 10.4 J_BAT — 2S Li-ion Pack (JST-XH 2.5 mm, 3-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | B+ | Pack positive (6.0–8.4 V) through the pack PCM; to TP5100 BAT, D_SYS2 and the PC1 divider |
| 2 | B− | Pack negative = GND |
| 3 | NTC | 10 kΩ NTC on the cells → TP5100 TEMP |

> [!warning]
> Use only a pack with an integrated 2S protection module and IEC 62133-2 / IS 16046 certified cells. Mount it in the enclosure's cool zone (< 45 °C), never above or beside the induction module or the PSU.

### 10.5 J_CAN_HEAT — Isolated CAN to the Cooking Surface (JST-XH 2.5 mm, 4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | CAN_H | Isolated bus |
| 2 | CAN_L | Isolated bus |
| 3 | GND_ISO | Isolated ground (NOT system GND); cable shield lands here |
| 4 | 5V_ISO | Optional isolated 5 V for an external node, F_CAN 100 mA polyfuse |

### 10.6 J_CAN_STIR — Regular CAN to the Stirring Motor (JST-XH 2.5 mm, 3-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | CAN_H | Non-isolated, 1 Mbps |
| 2 | CAN_L | |
| 3 | GND | System ground reference for the motor's CAN transceiver |

### 10.7 J_STIR_PWR — Stirring Motor Power (XT30)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 24V_STIR | Switched by Q_STIR, 5 A polyfuse |
| 2 | GND | Power ground |

### 10.8 J_DSB1–J_DSB4 — Dispenser Servos (3-pin servo header, 2.54 mm)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | SIG | PCA9685 CHn via 100R |
| 2 | V+ | Switched 6.5 V DSB rail |
| 3 | GND | Ground |

J_DSB1 = C1 (CID index), J_DSB2 = F1 (FID index), J_DSB3 = F2 (FID dose), J_DSB4 = F3 (FID steam gate).

### 10.9 J_SLD — SLD Pumps, Solenoids and Load Cells (Hirose DF11-16DP-2DS)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | PUMP1_AO1 | TB6612 A1 (oil pump +) |
| 2 | PUMP1_AO2 | TB6612 A2 (oil pump −) |
| 3 | PUMP2_BO1 | TB6612 B1 (water pump +) |
| 4 | PUMP2_BO2 | TB6612 B2 (water pump −) |
| 5 | 12V_SOL1 | Oil solenoid supply |
| 6 | SOL1_OUT | Oil solenoid drain (IRLML6344, PCF8574 #2 P0) |
| 7 | 12V_SOL2 | Water solenoid supply |
| 8 | SOL2_OUT | Water solenoid drain (IRLML6344, PCF8574 #2 P1) |
| 9, 10 | Reserved | |
| 11 | 3.3V | HX711 supply (both boards) |
| 12 | GND | HX711 ground |
| 13 | HX711_OIL_SCK | PC11 |
| 14 | HX711_OIL_DOUT | PC12 |
| 15 | HX711_WATER_SCK | PC9 |
| 16 | HX711_WATER_DOUT | PC10 |

### 10.10 J_FAN — Exhaust Fans (JST-XH 4-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 12V_FAN1 | Fan 1 supply |
| 2 | FAN1_OUT | Fan 1 drain (PA6 PWM) |
| 3 | 12V_FAN2 | Fan 2 supply |
| 4 | FAN2_OUT | Fan 2 drain (PB10 PWM) |

### 10.11 J_LED — WS2812B Ring (JST-XH 3-pin)

| Pin | Signal | Notes |
|-----|--------|-------|
| 1 | 5V_SW | Switched 5 V (Q_LED_P, PCF8574 #2 P3) |
| 2 | DATA | 74AHCT1G125 output via 33R, 800 kHz |
| 3 | GND | |

### 10.12 J_BUZZER — Piezo Buzzer (JST-PH 2-pin)

| Pin | Signal |
|-----|--------|
| 1 | 5V |
| 2 | BUZZ_OUT (2N7002 drain, PC7 PWM) |

### 10.13 J_SAFE — Safety I/O (Hirose DF11-4DP-2DS)

| Pin | Signal | STM32 Pin | Notes |
|-----|--------|-----------|-------|
| 1 | RELAY_DRV | PB0 via Q_RLY | Safety relay coil (12 V) |
| 2 | POT_DET | PB1 | Reed switch, 10k pull-up (backup) |
| 3 | E_STOP | PB2 | NC button, 10k pull-up, RC debounce |
| 4 | GND | GND | |

### 10.14 J_SWD — SWD Debug (2x5, 1.27 mm)

Standard Cortex debug pinout: VCC 3.3 V, SWDIO (PA13), SWCLK (PA14), SWO (PB3, optional), NRST, GND.

### 10.15 J_USB — USB-C 2.0 (16-pin mid-mount)

VBUS → F_USB → 5V_USB; CC1/CC2 5.1k to GND; D+/D− → USBLC6 → 22R → PA12/PA11. SuperSpeed and SBU pins unconnected.

---

## 11. 4-Layer Stackup and Zone Layout

### 11.1 Layer Stack

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (only break: CAN isolation slot) | 1 oz |
| L3 | Split power plane: 5V/3.3V, 12V, 24V/VSYS | 1 oz |
| L4 (Bottom) | Signal + power traces, passives under the Pico Zero | 2 oz |

FR-4 Tg ≥ 150 °C, 1.6 mm, ENIG.

### 11.2 Board Layout Zones

```
┌──────────────────────────────── 160 mm ────────────────────────────────┐
│ ZONE A: AP SLOT + DIGITAL           │ ZONE B: COMMS      │ ZONE C: POWER IN │
│ ┌─────────────────────────────┐     │ MCP2518FD + Y3     │ J_24V_IN  XT30   │
│ │ J_AP 2x20 + 4 standoffs     │     │ SN65HVD230         │ F1, SS54, TVS    │
│ │ (Luckfox Pico Zero 65x30)   │     │ J_CAN_STIR  J_LCD  │ INA219 + shunt   │
│ └─────────────────────────────┘     │ J_SWD  J_USB       │ D_SYS1/2 → VSYS  │ 90 mm
│ STM32G474RE, crystals, AMS1117,     │ SW_BOOT  SW_RST    │ J_BAT  TP5100    │
│ PCF8574 #2, PCA9685, Q_LED, D_OR1-2 │ 74AHCT1G125  J_LED │ L_CHG  CHG LEDs  │
├─────────────────────────────────────┼────────────────────┼──────────────────┤
│ ZONE D: ACTUATOR DRIVERS            │ ZONE E: SWITCHES   │ ZONE F: BUCKS    │
│ TB6612FNG, 4x IRLML6344 + flybacks  │ Q_DSB, Q_STIR      │ MP1584EN #1, #2  │
│ Q_RLY, Q_BUZZ, HX711 series Rs      │ PTC, 1000 µF       │ TPS54531 + L1-L3 │
│ ┌── CAN ISOLATION (6 mm slot) ──────────────────┐        │ C_SYS 100 µF     │
│ │ B0505S │ ISO1050 #1 │ ISO1050 #2 │ SLOT │ J_CAN_HEAT   │                  │
│ └───────────────────────────────────────────────┘        └──────────────────┘
│ J_DSB1-4   J_SLD   J_FAN   J_BUZZER   J_SAFE   J_STIR_PWR                      │
└────────────────────────────────────────────────────────────────────────────────┘
```

Placement rationale:
- **Zone A**: the Pico Zero sits on standoffs over the STM32 and I2C ICs; its SPI0 lines to the MCP2518FD (Zone B) and J_LCD are short (< 40 mm). Keep the module's antenna end at the board edge, away from Zone F.
- **Zone C**: J_BAT and the TP5100 next to the 12 V input of the charger and the VSYS diodes; the battery cable leaves the board edge towards the cool compartment.
- **Zone F**: three bucks ≥ 25 mm from the STM32 and the Pico Zero, thermal pours ≥ 400 mm² each.
- **CAN isolation slot** along the bottom edge; J_CAN_HEAT is the only connector on the isolated island.
- **J_CAN_STIR and J_STIR_PWR** adjacent so the motor harness is one bundle.

### 11.3 Critical Layout Rules

| Rule | Specification | Rationale |
|------|---------------|-----------|
| CAN isolation | Creepage ≥ 6.4 mm GND ↔ GND_ISO; 6 mm milled slot; no copper crossing | IEC 60335 reinforced insulation |
| Two CAN buses | Stub lengths < 30 mm, 120 Ω at each connector, never route the two buses adjacent | Signal integrity, no cross-talk between isolated and regular bus |
| SPI0 to MCP2518FD / J_LCD | Matched within 5 mm, 50 Ω, ground return under the trace | 50 MHz LCD clock |
| USB D+/D− | 90 Ω differential, matched ± 0.5 mm, < 30 mm | USB 2.0 FS |
| Pico Zero clearance | 3 mm keep-out around the module outline on L1; passives only under it on L4 | Mechanical fit, Wi-Fi antenna |
| Battery traces | ≥ 1.5 mm, B+ fused by the pack PCM; divider and NTC traces away from the bucks | 1.2 A discharge, clean ADC reading |
| VSYS node | ≥ 2 mm, D_SYS1/D_SYS2 within 10 mm of the TPS54531 input cap | Hold-up during switch-over |
| 24V_STIR | ≥ 2 mm, Q_STIR with ≥ 200 mm² pour | 3 A motor peak |
| Buck loops | Input cap → IC → inductor → output cap loop < 10 mm | EMI |
| Thermal vias | ≥ 9 under the STM32 pad, ≥ 6 under each buck IC | Heat to L2 |
| Mains relay | ≥ 2.5 mm clearance around relay contacts | IEC 60335-1 |

---

## 12. Thermal Analysis

| Component | Condition | Power (W) | Notes |
|-----------|-----------|-----------|-------|
| MP1584EN #1 (12 V) | 1.4 A continuous | ~1.9 | Lighter than v2 (5 V buck moved off this rail) |
| MP1584EN #2 (6.5 V) | 2.5 A, duty < 10 % | ~0.4 avg | Rail off when idle |
| TPS54531 (5 V from 24 V) | 0.7 A continuous | ~0.6 | ~87 % efficient at 24 V in |
| TP5100 charger | 0.5 A at 8.4 V from 12 V | ~0.7 | Charging only |
| TB6612FNG | 1.2 A, intermittent | ~0.7 | |
| AMS1117-3.3 | 0.2 A | ~0.3 | |
| ISO1050 x2, SN65HVD230, MCP2518FD | — | ~0.4 | |
| SS54 x3 (input, D_SYS1), IRLML6344 x4, Q_STIR, Q_DSB | — | ~1.8 | Diode drops dominate |
| **Total** | | **~6.5 W typical** | At 40 °C ambient every IC stays below 110 °C junction with the pours in §11.3 |

The Pico Zero (~1 W) dissipates through its own board; leave 10 mm of free air above it.

---

## 13. Unified BOM (changes highlighted)

| Ref | Part | Package | Qty | Unit (USD) | Notes |
|-----|------|---------|-----|-----------|-------|
| **Microcontroller** | | | | | |
| U_MCU | STM32G474RET6 | LQFP-64 | 1 | $8.00 | |
| Y1, Y2 | 8 MHz, 32.768 kHz crystals | SMD | 2 | $0.50 | |
| **AP slot** | | | | | |
| J_AP | 2x20 socket, 2.54 mm, 8.5 mm | TH | 1 | $0.80 | Pico Zero slot |
| STANDOFF x4 | M2.5 x 8 mm + screws | — | 4 | $0.10 | |
| U_CAN_AP | MCP2518FD-E/SL | SOIC-14 | 1 | $2.50 | RV1106 CAN node |
| Y3 | 40 MHz crystal + 2x 15 pF | SMD | 1 | $0.50 | |
| **Power conversion** | | | | | |
| U_BUCK1-2 | MP1584EN | SOT-23-8 | 2 | $0.80 | 12 V, 6.5 V |
| U_BUCK3 | TPS54531DDAR | SOIC-8 | 1 | $1.50 | VSYS → 5 V |
| U_LDO | AMS1117-3.3 | SOT-223 | 1 | $0.30 | |
| L1, L2, L3 | 33 µH, 22 µH (CDRH104R), 10 µH (CDRH127) | — | 3 | $0.55 | |
| D_SYS1, D_SYS2 | **SS54** | SMA | 2 | $0.20 | **VSYS diode-OR (24 V / battery)** |
| C_SYS | **100 µF, 35 V electrolytic** | 8x10 | 1 | $0.25 | **VSYS hold-up** |
| D_OR1, D_OR2 | SS14 | SMA | 2 | $0.10 | 5V_MAIN / 5V_USB OR |
| **Battery backup** | | | | | |
| U_CHG | **TP5100** | QFN-16 | 1 | $0.40 | **2S switching charger, 0.5 A** |
| L_CHG | **22 µH, Isat > 2 A, shielded** | 6x6 | 1 | $0.30 | |
| D_CHG | **SS34** | SMA | 1 | $0.15 | Charger catch diode |
| R_CS | **0.2 Ω 1 % 0.5 W** | 2010 | 1 | $0.10 | Charge current sense |
| R_BAT1, R_BAT2 | **100k, 22k 1 %** | 0402 | 2 | $0.01 | Pack voltage divider → PC1 |
| R_NTC_BIAS, R_CHRG_PU, R_STBY_PU | 10k | 0402 | 3 | $0.01 | |
| D_CHG_LED, D_STB_LED + 1k | Red / blue 0603 | 0603 | 2 | $0.03 | |
| C_CHG_IN, C_CHG_OUT | 10 µF 25 V, 22 µF 16 V | 0805 | 2 | $0.12 | |
| J_BAT | **JST-XH 2.5 mm 3-pin** | TH | 1 | $0.10 | B+, B−, NTC |
| (pack) | **2S1P 18650 pack with PCM, 2,600 mAh, NTC** | — | 1 | $12.00 | **Not on the PCB; system BOM** |
| **Isolated CAN (heater)** | | | | | |
| U_CAN1, U_CAN2 | ISO1050DUB | SOIC-8 | 2 | $3.50 | STM32 and RV1106 nodes |
| U_ISO_DC | B0505S-1WR3 | SIP-4 | 1 | $2.50 | 5V_ISO |
| R_TERM1, F_CAN, C_CAN x5, PESD1CAN | — | — | 1 lot | $0.40 | |
| J_CAN_HEAT | JST-XH 4-pin | TH | 1 | $0.15 | |
| **Regular CAN (stirring motor)** | | | | | |
| U_CAN3 | **SN65HVD230DR** | SOIC-8 | 1 | $0.90 | **FDCAN2 transceiver** |
| R_TERM2, R_RS, PESD1CAN, C | 120R, 10k, ESD, 100 nF | — | 1 lot | $0.20 | |
| J_CAN_STIR | **JST-XH 3-pin** | TH | 1 | $0.12 | |
| Q_STIR, Q_STIR_N | **AO4407A + 2N7002** | SO-8, SOT-23 | 2 | $0.45 | **24 V_STIR high-side switch** |
| F_STIR | 5 A polyfuse | 1812 | 1 | $0.40 | |
| C_STIR | 100 µF 35 V + 10 µF | — | 2 | $0.35 | |
| J_STIR_PWR | **XT30 2-pin** | TH | 1 | $0.40 | |
| **Dispenser Servo Block** | | | | | |
| U_PWM | PCA9685PW | TSSOP-28 | 1 | $1.50 | I2C 0x41 |
| Q_DSB, Q_DSB_N | AO3401 + 2N7002 | SOT-23 | 2 | $0.15 | 6.5 V rail switch |
| F_DSB, C_DSB | 3 A PTC, 1000 µF 10 V | — | 2 | $0.50 | |
| J_DSB1-4 | 3-pin servo header | TH | 4 | $0.10 | |
| **SLD** | | | | | |
| U_TB | TB6612FNG | SSOP-24 | 1 | $1.50 | |
| Q_SOL1-2, D_FB1-2 | IRLML6344, SS14 | SOT-23, SMA | 4 | $0.15 | |
| J_SLD | Hirose DF11-16DP-2DS | TH | 1 | $0.80 | |
| **Fans, LED, buzzer** | | | | | |
| Q_FAN1-2, D_FB3-4 | IRLML6344, SS14 | — | 4 | $0.15 | |
| U_LS | 74AHCT1G125 | SOT-23-5 | 1 | $0.30 | WS2812B level shift |
| Q_LED_N, Q_LED_P | 2N7002, SI2301 | SOT-23 | 2 | $0.08 | |
| Q_BUZZ | 2N7002 | SOT-23 | 1 | $0.05 | |
| J_FAN, J_LED, J_BUZZER | JST-XH 4, JST-XH 3, JST-PH 2 | TH | 3 | $0.12 | |
| **I2C expander + monitor** | | | | | |
| U_EXP2 | PCF8574 | SOIC-16 | 1 | $0.50 | 0x21 |
| U_MON, R_SHUNT | INA219BIDR, 10 mΩ 2512 | — | 2 | $0.90 | |
| **Protection** | | | | | |
| F1, D_REV, D_TVS1-3, F2-F3 | 5 A polyfuse, SS54, SMBJ24A/12A/6.5A, 3 A polyfuses | — | 1 lot | $1.60 | |
| Q_RLY, D_FLY | 2N7002, 1N4148WS | — | 2 | $0.08 | |
| D_ESD1-2 | PRTR5V0U2X | SOT-143B | 2 | $0.15 | |
| **USB-C** | | | | | |
| J_USB, U_ESD, R_DP/DM, R_CC1-2, R_VBUS1-2, F_USB, SW_BOOT | — | — | 1 lot | $0.80 | |
| **Display break-out** | | | | | |
| J_LCD | 1x14 header 2.54 mm | TH | 1 | $0.15 | |
| R_I2C3 x2, R_SER x10 | 4.7k, 33R | 0402 | 12 | $0.01 | |
| **Passives** | decoupling, pull-ups/downs, gate resistors, feedback dividers, crystal caps | — | ~120 | — | ~$2.50 |
| **Mechanical** | SW_RST, SJ1, J_SWD, J_SAFE, J_24V_IN | — | 5 | — | $1.40 |
| **PCB** | 4-layer 160 x 90 mm, ENIG, 2 oz outer | — | 1 | $5.50 | JLCPCB, 5 pcs |

### 13.1 Cost Summary

| Category | Prototype (1x) | Volume (100x) |
|----------|----------------|---------------|
| MCU + crystals | $8.50 | $6.00 |
| AP slot + RV1106 CAN node (socket, standoffs, MCP2518FD, crystal) | $4.20 | $3.00 |
| Power conversion (3 bucks, LDO, VSYS OR, passives) | $9.80 | $6.20 |
| Battery charger + monitoring (on-board) | $1.50 | $1.00 |
| Isolated CAN (2x ISO1050, B0505S, passives) | $10.00 | $7.00 |
| Regular CAN + motor power switch | $2.80 | $1.80 |
| Dispenser Servo Block | $2.55 | $1.80 |
| SLD drivers | $2.60 | $1.90 |
| Fans / LED / buzzer / relay / ESD | $1.80 | $1.10 |
| I2C expander + INA219 | $1.40 | $1.00 |
| USB-C, display break-out, passives, mechanical | $6.00 | $3.50 |
| Connectors (19) | $5.40 | $3.30 |
| PCB | $5.50 | $2.50 |
| **Board total** | **~$62** | **~$40** |
| Battery pack (system BOM, off-board) | $12.00 | $8.00 |

Compared with v2.0 (~$58): +$2.50 for the second ISO1050 and MCP2518FD, +$1.80 for the regular CAN port and motor switch, −$1.90 for the removed 12 V UPS input and single-cell micro-UPS, −$0.60 for the removed IR connector, and the P-ASD/CID driver parts (−$5) replaced by the DSB (+$2.55). The off-the-shelf 12 V UPS ($15–30) and the MLX90614 ($12) leave the system BOM; the 2S pack ($12) joins it.

---

## 14. Manufacturing Specifications

| Parameter | Value |
|-----------|-------|
| Layers / thickness | 4 / 1.6 mm |
| Copper | 2 oz outer, 1 oz inner |
| Min trace / space | 0.15 mm signal; power per §11.3 |
| Min via | 0.3 mm drill, 0.6 mm pad |
| Finish | ENIG |
| Material | FR-4, Tg ≥ 150 °C |
| Dimensions | 160 x 90 mm |
| Mounting | 4x M3 corner holes; 4x M2.5 holes for the Pico Zero standoffs (match the module's 3.0 mm hole pattern, 65 x 30 mm outline) |
| Milling | 6 mm CAN isolation slot |
| Impedance | 50 Ω SPI0; 90 Ω differential USB |

### 14.1 Assembly Notes

- SMT both sides (passives under the Pico Zero on L4), reflow; then hand/wave solder TH: J_AP, J_LCD, J_SLD, J_SAFE, JST-XH/PH, XT30 x2, J_DSB x4, B0505S, electrolytics
- Fit the Pico Zero standoffs before populating J_AP so the socket height is set by the standoffs
- Pin 1 silkscreen triangles on all connectors; label J_CAN_HEAT "ISOLATED — heater only" and J_CAN_STIR "MOTOR" to prevent swapping

---

## 15. Firmware Integration Notes

### 15.1 FreeRTOS Task Allocation

| Task | Rate | Priority | Functions |
|------|------|----------|-----------|
| Motor Control | 50 Hz | Highest | FDCAN2: velocity/position commands to the stir motor, feedback parsing (rpm, current, temperature), stir patterns, ramps, stall detection from motor current |
| Safety Monitor | 10 Hz | Highest | HEAT_STATUS limits (surface/coil over-temperature), E-stop, heartbeat, Q_STIR/Q_DSB/relay authority, IWDG kick |
| Dispense | 50 Hz (while active) | High | PCA9685 moves (one servo at a time), DSB rail, TB6612 pumps, solenoids, HX711 closed loop |
| CAN Comms (FDCAN1) | 20 Hz | High | Host commands, ACK/NAK, TELEMETRY/STATUS/LOADCELLS, SET_LED, HEAT_STATUS capture |
| Sensor Poll | 10 Hz | Medium | HX711, INA219, battery ADC (PC1), charger status (PCF8574 #2 P6/P7), 24 V comparator state |

### 15.2 STM32 Peripheral Allocation

| Peripheral | Pins | Function |
|------------|------|----------|
| FDCAN1 | PB8/PB9 | Isolated heater bus, 500 kbps, 11-bit IDs |
| FDCAN2 | PB12/PB13 | Stir motor bus, 1 Mbps, 29-bit IDs |
| USART2 | PA2/PA3 | Service link (bootloader/debug) |
| I2C1 | PA15/PB7 | PCA9685 0x41, PCF8574 #2 0x21, INA219 0x40 |
| TIM15_CH1 + DMA | PB14 | WS2812B |
| TIM3_CH1, TIM2_CH3 | PA6, PB10 | Fans, 25 kHz |
| TIM8_CH2 | PC7 | Buzzer |
| ADC1 | PC0, PC1 | VBUS detect, battery voltage |
| COMP1 | PA1 | 24 V power-fail |
| USB FS | PA11/PA12 | DFU / CDC |

### 15.3 Power Failure Sequence

1. COMP1 trips (24 V < 16.5 V) → `on_battery = true`; actuators are already unpowered; firmware sets PCF8574 #2 P3/P4/P5 low and PCA9685 OE high, relay off
2. STATUS (0x312) with POWER_FAIL and `batt_mV` to the RV1106 within 50 ms; RV1106 checkpoints the session, dims the LCD, shows "Power lost — running on battery"
3. Battery voltage polled at 1 Hz; warning to the RV1106 at 6.4 V, shutdown request at 6.2 V (RV1106 syncs and halts; STM32 enters STOP mode)
4. On 24 V recovery (> 20 V for 2 s) → POWER_FAIL cleared; RV1106 decides whether to resume (see state machine §5)

### 15.4 Battery Charger Behaviour

- Charging runs autonomously whenever the 12 V rail is up; the STM32 only reads CHRG/STDBY and the pack voltage
- Fault indication: CHRG and STDBY both inactive with the pack below 8.0 V for > 10 min → BATTERY_FAULT in STATUS (pack missing, NTC out of range, or charger fault)
- Self-test at boot: pack voltage between 5.5 V and 8.5 V, else BATTERY_FAULT (cooking is still allowed; the warning is informational)

---

## 16. Design Verification Checklist

### 16.1 Pre-Fabrication

- [ ] ERC passes; STM32 alternate functions verified: PB8/PB9 FDCAN1 (AF9), PB12/PB13 FDCAN2 (AF9), PA15/PB7 I2C1 (AF4), PB14 TIM15_CH1, PA2/PA3 USART2
- [ ] J_AP pinout matches the Luckfox Pico Zero 40-pin header; SPI0_CS1 and the two spare GPIO (MCP2518FD INT, TP_RST) confirmed against the Luckfox pinout; GPIO 32/38 to BOOT0/NRST
- [ ] Pico Zero outline, 3.0 mm hole pattern and standoff height checked against the module drawing
- [ ] J_LCD pin order matches the chosen 4" ST7796S/FT6336 module
- [ ] Two separate CAN buses: no shared termination, FDCAN1 only on ISO1050 #1, FDCAN2 only on SN65HVD230
- [ ] Isolation: only the two ISO1050 bus pins and the B0505S output cross the slot
- [ ] TP5100 strapped for 2S (8.4 V), R_CS = 0.2 Ω, NTC bias per datasheet, CHRG/STDBY pulled up to 3.3 V
- [ ] VSYS diode-OR: D_SYS1 from 24 V internal, D_SYS2 from B+; TPS54531 feedback 4.98 V; C_SYS rated 35 V
- [ ] Battery divider 100k/22k on PC1 reads ≤ 1.6 V at 8.4 V
- [ ] Q_STIR and Q_DSB default OFF (gate pull-ups), driven by PCF8574 #2 P5 / P4 through 2N7002
- [ ] PCA9685 OE 10k pull-up (off at boot), A0 = 1 → 0x41; PCF8574 #2 A0 = 1 → 0x21; INA219 0x40
- [ ] All MOSFET gates: 100R series + 10k pull-down (or pull-up for P-MOSFETs)
- [ ] 24 V protection order: polyfuse → Schottky → TVS
- [ ] USB: 22R, USBLC6, 5.1k CC, VBUS divider on PC0, SW_BOOT on BOOT0 with 10k pull-down
- [ ] All 19 connectors have correct pinouts and silkscreen

### 16.2 PCB Layout

- [ ] DRC passes; GND plane continuous except the CAN slot
- [ ] Creepage ≥ 6.4 mm across the isolation slot, ≥ 2.5 mm around the relay
- [ ] Pico Zero keep-out respected; standoff holes placed; antenna end at the board edge away from the bucks
- [ ] SPI0 traces to MCP2518FD and J_LCD matched within 5 mm; USB pair 90 Ω
- [ ] Battery, VSYS and 24V_STIR trace widths per §11.3
- [ ] Buck loops < 10 mm; thermal vias and pours present
- [ ] J_CAN_HEAT the only connector on the isolated island; J_CAN_STIR ≥ 20 mm from it

### 16.3 Post-Assembly

- [ ] 3.3 V, 5 V, 12 V, 6.5 V rails within ±3 %
- [ ] STM32 responds on SWD; DFU over USB-C with 24 V and battery disconnected
- [ ] Pico Zero boots in J_AP; LCD renders through J_LCD; touch events on I2C3
- [ ] `can0` on the RV1106 (MCP2518FD) and FDCAN1 on the STM32 exchange frames at 500 kbps; HEAT_QUERY to the module returns HEAT_STATUS
- [ ] FDCAN2 at 1 Mbps: motor enumerates, velocity command spins the arm, feedback frames arrive at 50 Hz
- [ ] Q_STIR off at reset: 0 V on J_STIR_PWR until PCF8574 #2 P5 is set
- [ ] I2C scan finds 0x21, 0x40, 0x41 (three devices)
- [ ] DSB: each servo reaches the commanded angle; rail switches off after a move; OE high leaves servos limp
- [ ] SLD: pumps run both directions; solenoids click; both HX711 channels stable ±1 g
- [ ] Battery: pack charges to 8.4 V at ~0.5 A from the 12 V rail; CHRG then STDBY read correctly on P6/P7; PC1 reads pack voltage within 2 %
- [ ] Pull the 24 V input during a cook simulation: 5V_MAIN stays within ±5 % with no reboot of the Pico Zero; COMP1 trips; STATUS carries POWER_FAIL and `batt_mV`; LED ring turns off
- [ ] Battery run-time ≥ 4 h at the logic-domain load; shutdown request at 6.2 V
- [ ] NTC disconnected → charger stops (CHRG/STDBY both inactive) → BATTERY_FAULT reported
- [ ] E-stop: relay opens, HEAT_ON_OFF=0 and E_STOP on FDCAN1, motor disable on FDCAN2 and Q_STIR off, all within 100 ms
- [ ] All actuators and both power switches off with the STM32 held in reset

---

## 17. Related Documentation

- [[../02-Hardware/01-Epicura-Architecture|Epicura Architecture]] — system wiring, CAN buses, power
- [[../02-Hardware/02-Technical-Specifications|Technical Specifications]] — power specs incl. battery backup
- [[../05-Subsystems/01-Induction-Heating|Induction Heating]] — module with internal temperature control, isolated CAN port
- [[../05-Subsystems/02-Robotic-Arm|Robotic Arm]] — CAN servo stirring motor
- [[../05-Subsystems/03-Ingredient-Dispensing|Ingredient Dispensing]] — DSB and SLD
- [[../03-Software/05-MCU-Functional-Specification|MCU Functional Specification]] — STM32 firmware requirements
- [[../03-Software/02-Controller-Software-Architecture|Controller Software Architecture]] — CAN ID map (§7.2)
- [[03-PCB-Design-Rules|PCB Design Rules]] — DRC rules and net classes
- Archived: [[_archive/01-Controller-PCB-Design|Controller PCB]] and [[_archive/02-Driver-PCB-Design|Driver PCB]] (2-board era)

---

#epicura #pcb #unified #stm32 #rv1106 #can #battery #power-electronics #hardware-design

---

## 18. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
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
