# epi-Design/PCB (Unified PCB + Cooking Surface Controller)

KiCad 9 project and design documents for the two custom boards of the [Epicura](../../README.md) autonomous kitchen robot: the **Unified PCB** (v5.3, the Dispensing Unit board) and the **Cooking Surface Controller** (CSC, v1.6, in the Cooking Surface base). Part of the `epi-Design` workspace (GitHub `manasXP/Epicura-Design`), alongside [`../UX/`](../UX/README.md).

## Overview

Epicura is three units — Cooking Surface (CS, mains), Stirring Unit (SU, on its own 24 V PSU in the CS rear, switched by the CSC) and Dispensing Unit (DU, lifts off, battery-powered) — and each custom board belongs to one of them.

**Unified PCB v5.3** (DU, 4-layer; since v5.3 an annular-sector board in the outer service ring of the Ø350 mm DU disc — was 160 x 90 mm) carries:

- **J_AP slot** (2x20 socket + standoffs) for the Luckfox Pico Zero (RV1106G3) application processor, with the display broken out on J_LCD (SPI0 + I2C3 touch). The RV1106 keeps only camera, LCD, Wi-Fi and a UART host link (UART2 ↔ USART1, 921,600 baud, BOOT0/NRST on GPIO 32/38) to the board's own MCU.
- **DU STM32G0B1RET6** (FreeRTOS, same part as the CSC): dispensers, SLD tank / pump Hall sensing, LED ring, buzzer, battery/dock monitoring on its I2C1 / timer PWM / GPIO, and the DU's CAN node (FDCAN1, non-isolated) acting as a transparent gateway for the RV1106. SWD on J_SWD; flashed by the RV1106 through the ST system bootloader.
- **Dispenser Servo Block (DSB)**: PCA9685 on the DU STM32's I2C1 driving four servos on a switched 6.0 V rail (J_DSB1-4: C1 CID index DS3218 270°, F1/F2 FID index and dose MG996R, F3 steam gate MG90S).
- **SLD liquid drivers** (robot-mop pattern since v5.3): TB6612FNG for the two 6 V pumps (peristaltic oil, diaphragm water) and six Hall-switch inputs (pump tachs, tank present, tank low) on a 12-pin J_SLD; no solenoids, no load cells.
- **DU STM32 CAN node** (FDCAN1 + a non-isolated SN65HVD230 since v5.2; ISO1050 #2, the DU isolation island and the MCP2518FD are gone) on the 500 kbps bus that the CSC isolates at its own port, reaching the CS over the **dock** (J_DOCK, 8 pogo pads: DOCK_24V, GND, CAN_H/L, GND_ISO tied to the DU ground as the bus reference, 5V_ISO reserved, DOCK_SENSE_A/B).
- **Power**: the 2-cell Li-ion pack (J_BAT, TP5100 1 A charger, INA219 monitor) is the primary supply; VSYS = DOCK_24V ⊕ BAT+ feeds a TPS54531 (5 V), AMS1117-3.3 and an MP1584EN (6.0 V dispenser rail); a second MP1584EN makes 12 V for the charger only.

**Cooking Surface Controller v1.6** (CS, 4-layer ~100 x 70 mm) carries:

- **STM32G0B1RET6** MCU (Cortex-M0+ @ 64 MHz, 2x FDCAN) running FreeRTOS: heater off-switch, safety watchdog, stir motor control, fans, dock interlock. Replaced the STM32G474RE on 2026-11-04 since no control loop runs on the MCU.
- **Two isolated CAN ports** (the only isolation in the system; port #1's barrier is a ground-loop barrier since the DU ground meets the CS ground on dock pin 2, so the heating unit's CAN port must itself be SELV): J_CAN_HEAT to the electric heating unit (ISO1050DUB #1 + B0505S #1; the same bus continues to the DU's non-isolated node over J_DOCK pins 3–5, pin 6 reserved) and J_CAN_STIR to the 24 V CAN servo stirring motor (ISO1050DUB #3 + B0505S #2 on FDCAN2, in the floating Stirring Unit domain GND_SU). The SU's own Mean Well LRS-75-24 #2 passes through the board on a floating island (J_SU_IN → F_SU → J_STIR_PWR, 24V_SU / GND_SU never bonded to the CS ground) and is cut by the pilot relay K_SU on its AC input; CAN and 24 V are bundled into the SU harness ending at J_SU on the right pillar.
- **Dock power**: Q_DOCK + F_DOCK 2 A switch DOCK_24V onto the pillar pogo block only while DOCK_SENSE_B says the DU is seated.
- **Safety I/O**: two pilot relays cutting AC to the heating unit on its inlet loom (K_SAFE; the OEM unit is not opened, Epicura uses only its CAN port) and to the SU's PSU (K_SU), E-stop, pot reed (J_SAFE, 2x3), two 12 V exhaust fans (J_FAN); USB-C DFU, SWD and a debug UART for the bench.
- **Power**: the board, both relay coils and the fans run from 12V_IN, a dedicated Mean Well LRS-35-12 (12 V, 3 A) in the CS rear on J_12V_IN (XT30; F1 2 A, SS34, SMBJ12A, 100 µF); one MP1584EN makes 5 V, an AMS1117 3.3 V. The LRS-75-24 #1 enters on J_DU_IN only for the dock feed (24V_DU → Q_DOCK); neither 24 V PSU powers the board and nothing is tapped from the heating unit. 12 connectors.

There is no temperature sensor on either board: the heating unit (rice-cooker class, replaced the induction surface on 2026-11-04 with no board change) runs its own loop and reports over CAN.

## Contents

```
PCB/
├── README.md                     This file
├── 01-Unified-PCB-Design.md      Unified PCB v5.0 (Dispensing Unit board): block diagrams, RV1106 + DU STM32
│                                 interface, power tree, dock, connector pinouts, BOM, checklist
├── 02-Cooking-Surface-Controller.md
│                                 Cooking Surface Controller v1.6 (CS board): STM32G0B1RE pin map,
│                                 12 V board supply from its own LRS-35-12 (J_12V_IN), two isolated CAN
│                                 ports, SU supply island + K_SU, J_DU_IN → dock power/interlock,
│                                 safety I/O, fans, 12 connectors
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

The KiCad project `Unified_v1.3/` **predates the three-unit split**: it is still one board holding both the STM32 side and the RV1106 side, and must be **split into two projects** — a DU board per `01-Unified-PCB-Design.md` v5.3 (RV1106 slot + DU STM32G0B1RE on a UART host link, non-isolated SN65HVD230 CAN node, dock pads, battery-first power, SLD Hall inputs, annular-sector outline) and a new CSC project per `02-Cooking-Surface-Controller.md` v1.6 (STM32G0B1RE on its own LRS-35-12 via J_12V_IN, two isolated CAN ports, SU supply island + K_SU, K_SAFE relay/E-stop/fans, J_DU_IN → Q_DOCK). Until then the notes below describe where `Unified_v1.3/` stands against the previous v3.x target.

`01-Unified-PCB-Design.md` v3.1 was the target for the next schematic revision. The KiCad project in `Unified_v1.3/` still reflects the v2.x design (P-ASD pneumatics, single-cell battery, PWM stirring motor) with two exceptions: U3 on `Controller.kicad_sch` is already the STM32G0B1RETx with the G0 pin map (I2C1_SCL PB6, FDCAN1 PB8/PB9, VBUS PA0, battery PA4, buzzer PA8, BOOT0 on PA14, USART2 PA2/PA3 service link), with the retired v2.x pump/motor lines parked on PA15/PD0/PD1; and the RV1106 side is in place (2026-11-04): J_AP1 and J_LCD1 with the BOOT0/NRST series resistors and touch pull-ups on `Connector.kicad_sch` (replacing J_CM5), and the RV1106 CAN node (U_CAN1 MCP2518FD, Y3 40 MHz, U_ISO2 ISO1050DUB #2 on the isolated bus with U6) on `Driver.kicad_sch`. J_AP header pins for SPI0_CS1, nINT, TP_RST, BOOT0 and NRST (11/13/15/31/33) are provisional until layout.

Still to do in KiCad (v3.x items that carry over, now split across the two boards):

- Dispenser Servo Block (PCA9685, Q_DSB rail switch, J_DSB1-4) replacing the P-ASD and CID actuator drivers — DU board
- Isolated CAN port #2 (ISO1050DUB #3 + B0505S #2, J_CAN_STIR), the floating SU island (J_SU_IN → F_SU → J_STIR_PWR) and the K_SU relay replacing the PWM/EN/DIR motor lines — CSC
- 2S battery path (J_BAT, TP5100, VSYS diode-OR) replacing the 12 V UPS input and the single-cell micro-UPS — DU board
- Removal of the MLX90614 IR thermometer and its connector

New for v5.2 / CSC v1.6:

- Split `Unified_v1.3/` into a DU project (keep `Connector.kicad_sch` J_AP/J_LCD with the BOOT0/NRST and UART2 lines, keep U3 STM32G0B1RETx as the DU STM32 with FDCAN1 → a new SN65HVD230; drop U_CAN1 MCP2518FD + Y3, U_ISO2 ISO1050DUB #2 and the DU isolation slot, the USB-C lot and the CS-side parts; tie dock pad 5 GND_ISO to GND, leave pad 6 unconnected) and a CSC project (its own STM32G0B1RETx, U6/ISO1050 #1 + ISO1050 #3, two B0505S, two G5V-2 relays, fans, USB, SWD)
- DU board: J_DOCK pad pattern, INA219 #2 on BAT+, six SLD Hall inputs on PC0–PC5 (no NAU7802 / TCA9543A / solenoid MOSFETs), Q_LED white-ring driver, DOCK_PWR_OK / DOCK_SENSE_A sense (to the DU STM32), 6.0 V rail from VSYS, 12-pin J_SLD, annular-sector outline; DU STM32 pin map per the v5.3 doc (USART1 PA9/PA10 host link, I2C1 PB6/PB7, TIM3 PA6/PA7 pumps, PA8 LED, PA4 buzzer, PB0 OE, PB3/PB5 dock sense, J_SWD)
- CSC: J_DOCK 8-pin header, Q_DOCK + F_DOCK, DOCK_SENSE_B on PB3, K_SU relay drive on PB4 (J_SAFE 2x3), J_SU_IN, J_DBG, MP1584EN 5 V instead of the TPS54531
- CSC v1.6 power: J_12V_IN (XT30) 12 V input chain (F1 2 A, SS34, SMBJ12A, C_IN 100 µF → 12V_IN) from a dedicated Mean Well LRS-35-12 feeding the fans, both G5V-2 coils and the 5 V buck; the 24 V input chain and the 24 → 12 V MP1584EN #1 go; J_24V_IN becomes J_DU_IN (F_DU 3 A, SS54, SMBJ24A → 24V_DU, to Q_DOCK only, LRS #1 −V bonded to GND there); K_SAFE contact documented on the heating unit's AC inlet loom (J_SAFE pin 1); no IRM-20-12 footprint — the rear bay holds three PSUs off-board (two LRS-75-24, one LRS-35-12, placement TBC)

## Board stackup (4-layer)

| Layer | Function | Copper |
|-------|----------|--------|
| L1 (Top) | Components + signal | 2 oz |
| L2 | Continuous GND plane (CSC: broken only by its two CAN isolation slots; Unified PCB: no slot since v5.2) | 1 oz |
| L3 | Split power plane: 5 V / 3.3 V, 12 V, 24 V / VSYS | 1 oz |
| L4 (Bottom) | Signal + power traces, passives under the Pico Zero | 2 oz |

FR-4 (Tg ≥ 150 °C), 1.6 mm, ENIG finish. Fabrication target is JLCPCB; its capability limits are in `03-PCB-Design-Rules.md` §9.

## Critical design constraints

- **CAN isolation slots** (CSC only): ≥ 6 mm milled cutout under each ISO1050DUB transceiver and its B0505S output per IEC 60664-1 reinforced isolation; no copper crosses the boundary. The Unified PCB has no slot since v5.2 (non-isolated SN65HVD230; dock pad 5 GND_ISO tied to GND, pad 6 unused, SELV spacing on the pad row). The CSC has a second slot under ISO1050DUB #3 and the B0505S #2 output for the stir port (floating SU domain, GND_SU); the two isolated buses never share termination, DC-DC or routing.
- **Mains relay AC traces** (CSC): ≥ 2.5 mm clearance (IEC 60335-1).
- **High-current nets** (pump, fan, 24 V motor feed, DOCK_24V, BAT+): 1.5 mm track, 0.5 mm clearance, 1.0 / 0.5 mm via pad / drill.
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

3-board stack (CM5IO + Controller + Driver) → 2-board stack (CM5IO + Unified, v2.x, the current KiCad files) → single board hosting the Luckfox Pico Zero in a slot (design doc v3.0, 2026-11-04) → per-axis dispenser servos and D_DSB4 (v3.1, 2026-11-04) → three-unit architecture: the Unified PCB becomes the Dispensing Unit board (v4.0, no MCU, dock pads, battery-first) and the STM32 side moves to the new Cooking Surface Controller (CSC v1.0), both 2026-11-04 → MCU back in the DU: a second STM32G0B1RE on the Unified PCB (v5.0, 2026-11-04) takes dispensing and the DU CAN node, with a UART host link to the RV1106 (the MCP2518FD is dropped) → both CSC CAN ports isolated and the Stirring Unit on its own LRS-75-24 #2 behind the relay K_SU (CSC v1.3, 2026-11-04) → the DU CAN node made non-isolated (SN65HVD230, Unified PCB v5.0 / CSC v1.4, 2026-11-04): isolation lives on the CSC's two ports only → the CSC, its relay coils and the fans powered from the heating unit's own 12 V SELV rail (J_HU_PWR), the LRS-75-24 #1 left to the dock feed alone (J_DU_IN) and K_SAFE moved onto the element loom (CSC v1.5, 2026-11-05) → the tap withdrawn the same day: the CSC, its relay coils and the fans on a dedicated Mean Well LRS-35-12 (J_12V_IN), K_SAFE back on the heating unit's AC inlet loom, three PSUs in the CS rear with one customer each (CSC v1.6, 2026-11-05). The repo moved from the standalone `manasXP/epicura-pcb` into `Epicura-Design/PCB` on 2026-11-04 with its history.
