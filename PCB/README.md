# epi-Design/PCB (Unified PCB)

KiCad 9 project and design documents for the Epicura Unified PCB, the single custom board of the [Epicura](../../README.md) autonomous kitchen robot. Part of the `epi-Design` workspace (GitHub `manasXP/Epicura-Design`), alongside [`../UX/`](../UX/README.md).

## Overview

The Unified PCB is a 4-layer 160 x 90 mm board that carries:

- **STM32G0B1RET6** MCU (Cortex-M0+ @ 64 MHz, 2x FDCAN) running FreeRTOS: dispensers, sensors, heater off-switch, safety watchdog. Replaced the STM32G474RE on 2026-11-04 since no control loop runs on the MCU.
- **J_AP slot** (2x20 socket + standoffs) for the Luckfox Pico Zero (RV1106G3) application processor, with the display broken out again on J_LCD.
- **Dispenser Servo Block (DSB)**: PCA9685 on I2C1 driving four servos on a switched 6.5 V rail (J_DSB1-4: C1 CID index DS3218 270°, F1/F2 FID index and dose MG996R, F3 steam gate MG90S through D_DSB4).
- **SLD liquid drivers**: TB6612FNG for two peristaltic pumps, two solenoid MOSFETs, two HX711 load-cell front ends on J_SLD.
- **Two CAN ports**: isolated J_CAN_HEAT to the cooking surface (2x ISO1050DUB + B0505S; STM32 FDCAN1 and the RV1106's MCP2518FD share the bus) and regular J_CAN_STIR to the 24 V CAN servo stirring motor (SN65HVD230 on FDCAN2, switched 24 V on J_STIR_PWR).
- **Power**: 24 V input, MP1584EN bucks (12 V, 6.5 V DSB rail), TPS54531 VSYS → 5 V, AMS1117-3.3, and a 2-cell Li-ion backup (J_BAT) charged by a TP5100 with a VSYS diode-OR so the logic domain rides through outages.

There is no temperature sensor on the board: the induction surface runs its own loop and reports over CAN.

## Contents

```
PCB/
├── README.md                     This file
├── 01-Unified-PCB-Design.md      Master design document (v3.1): block diagrams, pin allocation,
│                                 power tree, connector pinouts, BOM, verification checklist
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

## Status: schematic lags the design document

`01-Unified-PCB-Design.md` v3.1 is the target for the next schematic revision. The KiCad project in `Unified_v1.3/` still reflects the v2.x design (CM5 stack, P-ASD pneumatics, single-cell battery) with one exception: U3 on `Controller.kicad_sch` is already the STM32G0B1RETx with the G0 pin map (I2C1_SCL PB6, FDCAN1 PB8/PB9, VBUS PA0, battery PA4, buzzer PA8, BOOT0 on PA14), with the retired v2.x pump/motor lines parked on PA15/PD0/PD1.

Still to do in KiCad for v3.x:

- J_AP slot and J_LCD break-out, MCP2518FD + second ISO1050DUB for the RV1106 CAN node
- Dispenser Servo Block (PCA9685, Q_DSB rail switch, J_DSB1-4, D_DSB4) replacing the P-ASD and CID actuator drivers
- Regular CAN port (SN65HVD230, J_CAN_STIR) and Q_STIR / J_STIR_PWR replacing the PWM/EN/DIR motor lines
- 2S battery path (J_BAT, TP5100, VSYS diode-OR) replacing the 12 V UPS input and the single-cell micro-UPS
- Removal of the MLX90614 IR thermometer and its connector

## Board stackup (4-layer)

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (only break: CAN isolation slot) | 1 oz |
| L3 | Split power plane: 5 V / 3.3 V, 12 V, 24 V / VSYS | 1 oz |
| L4 (Bottom) | Signal + power traces, passives under the Pico Zero | 2 oz |

FR-4 (Tg ≥ 150 °C), 1.6 mm, ENIG finish. Fabrication target is JLCPCB; its capability limits are in `03-PCB-Design-Rules.md` §9.

## Critical design constraints

- **CAN isolation slot**: ≥ 6 mm milled cutout under both ISO1050DUB transceivers and the B0505S output (IEC 60664-1 reinforced isolation). No copper crosses the boundary. The regular motor CAN port on the SN65HVD230 is not isolated and never shares termination or routing with the isolated bus.
- **Mains relay AC traces**: ≥ 2.5 mm clearance (IEC 60335-1).
- **High-current nets** (pump, solenoid, fan, 24 V motor feed): 1.5 mm track, 0.5 mm clearance, 1.0 / 0.5 mm via pad / drill.
- **USB differential pair**: 90 Ω, length-matched within 0.5 mm.

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

Full definitions, including net membership, are in `03-PCB-Design-Rules.md` §10. The `LACT_O*` nets listed there belong to the v2.x linear actuator and go away in v3.x.

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
- Design changes go into `01-Unified-PCB-Design.md` first (connectors, pin maps, power tree, BOM), then propagate to the epi-Spec architecture docs: [`epi-Spec/docs/02-Hardware/01-Epicura-Architecture.md`](../../epi-Spec/docs/02-Hardware/01-Epicura-Architecture.md) and the subsystem docs under `epi-Spec/docs/05-Subsystems/`.

## Design history

3-board stack (CM5IO + Controller + Driver) → 2-board stack (CM5IO + Unified, v2.x, the current KiCad files) → single board hosting the Luckfox Pico Zero in a slot (design doc v3.0, 2026-11-04) → per-axis dispenser servos and D_DSB4 (v3.1, 2026-11-04). The repo moved from the standalone `manasXP/epicura-pcb` into `Epicura-Design/PCB` on 2026-11-04 with its history.
