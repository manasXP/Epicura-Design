# epi-Design/PCB (Unified PCB + Cooking Surface Controller)

KiCad 9 project and design documents for the two custom boards of the [Epicura](../../README.md) autonomous kitchen robot: the **Unified PCB** (v4.0, the Dispensing Unit board) and the **Cooking Surface Controller** (CSC, v1.0, in the Cooking Surface base). Part of the `epi-Design` workspace (GitHub `manasXP/Epicura-Design`), alongside [`../UX/`](../UX/README.md).

## Overview

Epicura is three units — Cooking Surface (CS, mains), Stirring Unit (SU, fed 24 V by the CS) and Dispensing Unit (DU, lifts off, battery-powered) — and each custom board belongs to one of them.

**Unified PCB v4.0** (DU, 4-layer 160 x 90 mm) carries:

- **J_AP slot** (2x20 socket + standoffs) for the Luckfox Pico Zero (RV1106G3) application processor, with the display broken out on J_LCD. The RV1106 is the only controller on the board: dispensers, load cells, LED ring and buzzer hang off its I2C3 / GPIO / PWM.
- **Dispenser Servo Block (DSB)**: PCA9685 on I2C3 driving four servos on a switched 6.0 V rail (J_DSB1-4: C1 CID index DS3218 270°, F1/F2 FID index and dose MG996R, F3 steam gate MG90S).
- **SLD liquid drivers**: TB6612FNG for two 6 V peristaltic pumps, two 6 V solenoid MOSFETs, two NAU7802 load-cell ADCs behind a TCA9543A mux, on J_SLD.
- **RV1106 CAN node** (MCP2518FD + ISO1050DUB #2) on the isolated 500 kbps bus, which reaches the CS over the **dock** (J_DOCK, 8 pogo pads: DOCK_24V, GND, CAN_H/L, GND_ISO, 5V_ISO, DOCK_SENSE_A/B).
- **Power**: the 2-cell Li-ion pack (J_BAT, TP5100 1 A charger, INA219 monitor) is the primary supply; VSYS = DOCK_24V ⊕ BAT+ feeds a TPS54531 (5 V), AMS1117-3.3 and an MP1584EN (6.0 V dispenser rail); a second MP1584EN makes 12 V for the charger only.

**Cooking Surface Controller v1.0** (CS, 4-layer ~100 x 70 mm) carries:

- **STM32G0B1RET6** MCU (Cortex-M0+ @ 64 MHz, 2x FDCAN) running FreeRTOS: heater off-switch, safety watchdog, stir motor control, fans, dock interlock. Replaced the STM32G474RE on 2026-11-04 since no control loop runs on the MCU.
- **Two CAN ports**: isolated J_CAN_HEAT to the electric heating unit (ISO1050DUB #1 + B0505S; the same bus continues to the DU over J_DOCK pins 3–6) and regular J_CAN_STIR to the 24 V CAN servo stirring motor (SN65HVD230 on FDCAN2, switched 24 V on J_STIR_PWR; both bundled into the SU harness ending at J_SU on the right pillar).
- **Dock power**: Q_DOCK + F_DOCK 2 A switch DOCK_24V onto the pillar pogo block only while DOCK_SENSE_B says the DU is seated.
- **Safety I/O**: relay cutting AC to the heating unit, E-stop, pot reed (J_SAFE), two 12 V exhaust fans (J_FAN); USB-C DFU, SWD and a debug UART for the bench.

There is no temperature sensor on either board: the heating unit (rice-cooker class, replaced the induction surface on 2026-11-04 with no board change) runs its own loop and reports over CAN.

## Contents

```
PCB/
├── README.md                     This file
├── 01-Unified-PCB-Design.md      Unified PCB v4.0 (Dispensing Unit board): block diagrams, RV1106
│                                 interface, power tree, dock, connector pinouts, BOM, checklist
├── 02-Cooking-Surface-Controller.md
│                                 Cooking Surface Controller v1.0 (CS board): STM32G0B1RE pin map,
│                                 both CAN ports, dock power/interlock, safety I/O, fans, connectors
├── 03-PCB-Design-Rules.md        Stackup, track widths, clearances, creepage, net classes,
│                                 JLCPCB limits, DRC policy
├── 05-Schematic-Review-v1.1.md   Review of schematic rev 1.1 (2026-03-21): open issues by severity
├── _archive/                     Superseded 2-board (Controller + Driver) design docs
└── Unified_v1.3/                 KiCad 9 project
    ├── Unified_v1.3.kicad_pro / .kicad_pcb / .kicad_prl
    ├── Unified_v1.3.kicad_sch    Root sheet
    ├── Controller.kicad_sch      MCU, power, CAN
    ├── Driver.kicad_sch          Actuator drivers
    ├── Connector.kicad_sch       Off-board connectors
    ├── libraries/                Project symbol (.kicad_sym) and footprint (.pretty) libraries
    ├── sym-lib-table, fp-lib-table
    ├── BoM/Unified_v1.3.csv      KiCad grouped BOM export (multiple refs per row)
    ├── Schematic_PDF/            PDF export of all sheets
    └── Unified_v1.3-backups/     KiCad auto-backups (ignored by git)
```

## Status: schematic lags the design documents

The KiCad project `Unified_v1.3/` **predates the three-unit split**: it is still one board holding both the STM32 side and the RV1106 side, and must be **split into two projects** — a DU board per `01-Unified-PCB-Design.md` v4.0 (no STM32, dock pads, battery-first power, NAU7802 load cells) and a new CSC project per `02-Cooking-Surface-Controller.md` v1.0 (STM32G0B1RE, both CAN transceivers, relay/E-stop/fans, Q_DOCK). Until then the notes below describe where `Unified_v1.3/` stands against the previous v3.x target.

`01-Unified-PCB-Design.md` v3.1 was the target for the next schematic revision. The KiCad project in `Unified_v1.3/` still reflects the v2.x design (P-ASD pneumatics, single-cell battery, PWM stirring motor) with two exceptions: U3 on `Controller.kicad_sch` is already the STM32G0B1RETx with the G0 pin map (I2C1_SCL PB6, FDCAN1 PB8/PB9, VBUS PA0, battery PA4, buzzer PA8, BOOT0 on PA14, USART2 PA2/PA3 service link), with the retired v2.x pump/motor lines parked on PA15/PD0/PD1; and the RV1106 side is in place (2026-11-04): J_AP1 and J_LCD1 with the BOOT0/NRST series resistors and touch pull-ups on `Connector.kicad_sch` (replacing J_CM5), and the RV1106 CAN node (U_CAN1 MCP2518FD, Y3 40 MHz, U_ISO2 ISO1050DUB #2 on the isolated bus with U6) on `Driver.kicad_sch`. J_AP header pins for SPI0_CS1, nINT, TP_RST, BOOT0 and NRST (11/13/15/31/33) are provisional until layout.

Still to do in KiCad (v3.x items that carry over, now split across the two boards):

- Dispenser Servo Block (PCA9685, Q_DSB rail switch, J_DSB1-4) replacing the P-ASD and CID actuator drivers — DU board
- Regular CAN port (SN65HVD230, J_CAN_STIR) and Q_STIR / J_STIR_PWR replacing the PWM/EN/DIR motor lines — CSC
- 2S battery path (J_BAT, TP5100, VSYS diode-OR) replacing the 12 V UPS input and the single-cell micro-UPS — DU board
- Removal of the MLX90614 IR thermometer and its connector

New for v4.0 / CSC v1.0:

- Split `Unified_v1.3/` into a DU project (keep `Connector.kicad_sch` J_AP/J_LCD and the `Driver.kicad_sch` RV1106 CAN node; drop U3 and the STM32-only parts) and a CSC project (U3 STM32G0B1RETx, U6/ISO1050 #1, B0505S, SN65HVD230, relay, fans, USB, SWD)
- DU board: J_DOCK pad pattern, INA219 #2 on BAT+, TCA9543A + 2x NAU7802, Q_LED white-ring driver, DOCK_PWR_OK / DOCK_SENSE_A sense, 6.0 V rail from VSYS, re-pinned J_SLD
- CSC: J_DOCK 8-pin header, Q_DOCK + F_DOCK, DOCK_SENSE_B on PB3, Q_STIR enable on PB4, J_DBG, MP1584EN 5 V instead of the TPS54531

## Board stackup (4-layer)

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (only break: CAN isolation slot) | 1 oz |
| L3 | Split power plane: 5 V / 3.3 V, 12 V, 24 V / VSYS | 1 oz |
| L4 (Bottom) | Signal + power traces, passives under the Pico Zero | 2 oz |

FR-4 (Tg ≥ 150 °C), 1.6 mm, ENIG finish. Fabrication target is JLCPCB; its capability limits are in `03-PCB-Design-Rules.md` §9.

## Critical design constraints

- **CAN isolation slot** (both boards): ≥ 6 mm milled cutout under the ISO1050DUB transceiver (and the B0505S output on the CSC) per IEC 60664-1 reinforced isolation. No copper crosses the boundary; the isolated domain also crosses the dock (pins 3–6), so the pad/header spacing must hold the creepage. The regular motor CAN port on the SN65HVD230 (CSC) is not isolated and never shares termination or routing with the isolated bus.
- **Mains relay AC traces** (CSC): ≥ 2.5 mm clearance (IEC 60335-1).
- **High-current nets** (pump, solenoid, fan, 24 V motor feed, DOCK_24V, BAT+): 1.5 mm track, 0.5 mm clearance, 1.0 / 0.5 mm via pad / drill.
- **USB differential pair** (CSC): 90 Ω, length-matched within 0.5 mm.
- **Dock pads** (DU board): eight 2.54 mm-pitch gold pads on the bottom layer at the board edge over the CS left pillar, mask-free.

## Net classes

| Net class | Track (mm) | Clearance (mm) | Via pad / drill (mm) |
|-----------|-----------|----------------|----------------------|
| Default | 0.20 | 0.20 | 0.6 / 0.3 |
| SPI, I2C, CAN | 0.25 | 0.20 | 0.6 / 0.3 |
| Power_3V3, Power_5V, Power_GND | 0.50 | 0.30 | 0.8 / 0.4 |
| Power_12V | 0.80 | 0.30 | 0.8 / 0.4 |
| Power_24V | 1.00 | 0.30 | 1.0 / 0.5 |
| USB | 0.20 | 0.20 | 0.6 / 0.3 |
| HighCurrent | 1.50 | 0.50 | 1.0 / 0.5 |

Full definitions, including net membership, are in `03-PCB-Design-Rules.md` §10 (v1.2, aligned with the v3.x net names).

## Working with the project

- Open `Unified_v1.3/Unified_v1.3.kicad_pro` in KiCad 9 (`/Applications/KiCad/KiCad.app` on the design Mac).
- Exports and checks from the command line:

```bash
KICAD=/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli
cd Unified_v1.3
$KICAD sch export pdf Unified_v1.3.kicad_sch -o Schematic_PDF/Unified_v1.3.pdf
$KICAD sch export bom Unified_v1.3.kicad_sch -o BoM/Unified_v1.3.csv --group-by Value,Footprint
$KICAD sch erc Unified_v1.3.kicad_sch
$KICAD pcb drc Unified_v1.3.kicad_pcb
```

- Check `05-Schematic-Review-v1.1.md` before raising an issue; many are already logged there.
- Design changes go into `01-Unified-PCB-Design.md` (DU board) or `02-Cooking-Surface-Controller.md` (CSC) first (connectors, pin maps, power tree, BOM), then propagate to the epi-Spec architecture docs: [`epi-Spec/docs/02-Hardware/01-Epicura-Architecture.md`](../../epi-Spec/docs/02-Hardware/01-Epicura-Architecture.md) and the subsystem docs under `epi-Spec/docs/05-Subsystems/`.

## Design history

3-board stack (CM5IO + Controller + Driver) → 2-board stack (CM5IO + Unified, v2.x, the current KiCad files) → single board hosting the Luckfox Pico Zero in a slot (design doc v3.0, 2026-11-04) → per-axis dispenser servos and D_DSB4 (v3.1, 2026-11-04) → three-unit architecture: the Unified PCB becomes the Dispensing Unit board (v4.0, no MCU, dock pads, battery-first) and the STM32 side moves to the new Cooking Surface Controller (CSC v1.0), both 2026-11-04. The repo moved from the standalone `manasXP/epicura-pcb` into `Epicura-Design/PCB` on 2026-11-04 with its history.
