# PCB Design Rules

Design rules reference for the Epicura Unified PCB (v5.2: Luckfox Pico Zero slot, DU STM32G0B1 controller with its UART host link and a non-isolated CAN node, dispenser/liquid drivers and the 2S battery path on a 4-layer board) and, by extension, the Cooking Surface Controller. Rules are enforced via KiCad DRC in the `.kicad_pro` file. Net names below follow [[01-Unified-PCB-Design]] v5.2 and [[02-Cooking-Surface-Controller]]; the v2.x KiCad project in `Unified_v1.3/` still carries some older names until the next schematic pass.

**Related:** [[01-Unified-PCB-Design]]

---

## 1. Applicable Standards

| Standard | Scope |
|----------|-------|
| **IPC-2221B Class 2** | Dedicated-service electronics — track width, clearance, via sizing |
| **IPC-7351C** | Land pattern geometry (footprints) |
| **IEC 60335-1** | Household appliance safety — creepage & clearance for mains isolation |
| **JLCPCB DFM** | Fabrication capability limits for prototype and production runs |

---

## 2. Board Stackup

| Parameter | Value |
|-----------|-------|
| Layer count | 4 (F.Cu, GND plane, split power plane, B.Cu) |
| Board thickness | 1.6 mm |
| Substrate | FR-4 (Tg ≥ 150°C) |
| Copper weight (outer) | 2 oz (70 µm) — required for power traces |
| Copper weight (inner) | 1 oz (35 µm) — planes |
| Surface finish | ENIG (lead-free) |
| Solder mask | Green LPI, both sides |
| Silkscreen | White, both sides |
| Board dimensions | Unified PCB v5.3: annular sector R 135–172 mm × ≈ 150° (TBC with the DU CAD; was 160 × 90 mm) |
| Layer 1 (Top) | Signal + Components (2 oz) |
| Layer 2 (Inner1) | GND Plane — continuous (1 oz) |
| Layer 3 (Inner2) | Split Power Plane: 5V/3.3V, 12V, 24V (1 oz) |
| Layer 4 (Bottom) | Signal + Power Traces (2 oz) |

---

## 3. Track Width by Current

Based on IPC-2221B external layer, 10°C temperature rise, 1 oz copper.

| Application | Width (mm) | Current Rating | Usage |
|-------------|-----------|----------------|-------|
| Signal (default) | 0.20 | < 0.5 A | GPIO, logic, misc signals |
| Bus (I2C/SPI/CAN) | 0.25 | < 0.5 A | Controlled-impedance bus signals |
| Light load | 0.30 | ~ 0.7 A | LED drivers, low-power rails |
| 1 A | 0.50 | 1.0 A | 5V rail, moderate loads |
| 2 A | 0.80 | 2.0 A | 12V distribution, motor enable |
| 3 A | 1.00 | 3.0 A | 24V main rail |
| 5 A | 1.50 | 5.0 A | High-current motor/actuator feeds |

> [!tip] For currents above 5 A, use copper pours or parallel traces rather than single tracks.

---

## 4. Clearance Rules

| Context | Clearance (mm) | Rationale |
|---------|---------------|-----------|
| Signal-to-signal | 0.20 | IPC-2221B Class 2 minimum |
| Power-to-signal | 0.30 | Noise margin for DC rails |
| High-current-to-any | 0.50 | Thermal and voltage margin |
| CAN bus isolation (ISO1050, CSC only) | 6.00 | IEC 60664-1 reinforced isolation; the Unified PCB has no isolated net since v5.2 |
| Mains relay AC traces | 2.50 | IEC 60335-1 basic insulation (250 VAC) |
| Copper-to-board-edge | 0.30 | JLCPCB minimum + margin |

---

## 5. Via Specifications

| Type | Pad Dia (mm) | Drill (mm) | Annular Ring (mm) | Usage |
|------|-------------|-----------|-------------------|-------|
| Standard | 0.60 | 0.30 | 0.15 | Signal routing, bus lines |
| Power | 0.80 | 0.40 | 0.20 | 5V, 12V, GND stitching |
| High-current | 1.00 | 0.50 | 0.25 | 24V, motor/actuator feeds |

> [!note] Use multiple vias in parallel for high-current layer transitions (e.g., 3× power vias for 24V feed).

---

## 6. Copper Pour

| Parameter | Value |
|-----------|-------|
| Bottom layer | GND flood fill (continuous ground plane) |
| Top layer | Selective pours for power rails where needed |
| Zone clearance | 0.50 mm (to pads and traces) |
| Thermal relief | Spoke width 0.5 mm, gap 0.5 mm |
| Min spoke count | 2 |
| Isolated copper | Remove (DRC warning → manual review) |

---

## 7. Creepage & Clearance (IEC 60335-1)

These rules apply to the **unified board** where mains-referenced signals exist (relay AC switching, heating unit CAN interface).

| Insulation Type | Min Distance (mm) | Application |
|----------------|-------------------|-------------|
| Functional | 1.5 | Low-voltage signal separation |
| Basic (250 VAC) | 2.5 | Relay AC trace to DC traces |
| Reinforced | 6.0 | ISO1050 CAN bus isolation slots (CSC only) |

- Route an **isolation slot** (milled cutout) under each ISO1050 CAN transceiver on the **CSC**, ≥ 6 mm wide. The **Unified PCB has no slot since v5.2**: its DU STM32 drives a non-isolated SN65HVD230, dock pad 5 (`GND_ISO`) is tied to `GND` with a 0R next to the pads, pad 6 (`5V_ISO`) is not connected, and the dock pad row uses ordinary SELV (functional) spacing
- The CSC has **two** such slots on opposite board edges: under ISO1050DUB #1 + the B0505S #1 output (heater / dock bus, `GND_ISO`) and under ISO1050DUB #3 + the B0505S #2 output (stir bus, `GND_SU` — the floating Stirring Unit domain, never joined to `GND` or `GND_ISO`); the SU island also carries `24V_SU` (J_SU_IN → F_SU → J_STIR_PWR) and `5V_ISO2`
- No copper pour or traces may cross the isolation boundary
- Keep mounting holes and connectors ≥ 3 mm from isolation boundaries

---

## 8. Silkscreen

| Parameter | Min Value |
|-----------|-----------|
| Text height | 0.80 mm |
| Text line width | 0.15 mm |
| Clearance to pads | 0.15 mm |
| Reference designators | All components — F.Silkscreen |

---

## 9. JLCPCB Capability Limits

Absolute manufacturing minimums — our design rules are set above these for margin.

| Parameter | JLCPCB Min | Our Design Min |
|-----------|-----------|----------------|
| Trace width | 0.127 mm (5 mil) | 0.15 mm |
| Trace spacing | 0.127 mm (5 mil) | 0.20 mm |
| Drill diameter | 0.30 mm | 0.30 mm |
| Annular ring | 0.13 mm | 0.13 mm |
| Via pad diameter | 0.50 mm | 0.60 mm |
| Board-edge clearance | 0.20 mm | 0.30 mm |
| Silkscreen line | 0.10 mm | 0.15 mm |
| Silkscreen text | 0.80 mm height | 0.80 mm |

---

## 10. Net Class Definitions (Unified Board)

| Net Class | Track (mm) | Clearance (mm) | Via Pad/Drill (mm) | Nets |
|-----------|-----------|----------------|--------------------:|------|
| Default | 0.20 | 0.20 | 0.6 / 0.3 | All unassigned signals, GPIO |
| SPI | 0.25 | 0.20 | 0.6 / 0.3 | `SPI0_*` from J_AP to J_LCD (display; CS0 only) |
| UART | 0.25 | 0.20 | 0.6 / 0.3 | `HOST_TX`, `HOST_RX` (J_AP UART2 ↔ DU STM32 USART1, 33R series), `BOOT0`, `NRST`; keep away from the 6V_DSP switch node |
| I2C | 0.25 | 0.20 | 0.6 / 0.3 | `I2C1_SCL`, `I2C1_SDA` (DU STM32 → PCA9685, PCF8574 #2, INA219 ×2); `I2C3_*` from J_AP to J_LCD touch only |
| CAN | 0.25 | 0.20 | 0.6 / 0.3 | Bus #1 (isolated at the CSC port only): `CANH_HEAT`, `CANL_HEAT`, `FDCAN1_*` (DU STM32 → SN65HVD230 on the Unified PCB, referenced to `GND`; CSC STM32 → ISO1050 #1 on the CSC); isolated stir bus (CSC only, SU domain referenced to `GND_SU`): `CANH_STIR`, `CANL_STIR`, `FDCAN2_*`. The two buses never share routing, termination or DC-DC |
| Power_3V3 | 0.50 | 0.30 | 0.8 / 0.4 | `+3V3`, `VDDA` |
| Power_5V | 0.50 | 0.30 | 0.8 / 0.4 | `+5V` (TPS54531 output: Pico Zero, LCD, STM32, CAN domain, LED ring) |
| Power_12V | 0.80 | 0.30 | 0.8 / 0.4 | `12V_OUT` (Unified PCB: TP5100 charger input only; the pumps are on 6V_DSP) |
| Power_12V_IN | 1.00 | 0.30 | 1.0 / 0.5 | `12V_IN` (CSC board supply from its own Mean Well LRS-35-12: J_12V_IN → F1 → SS34 → C_IN 100 µF → fans, K_SAFE / K_SU coils, MP1584EN input; ~1.4 A peak, ≥ 1 mm to the fan MOSFETs per CSC §15.2) |
| Power_24V | 1.00 | 0.30 | 1.0 / 0.5 | `24V_INT`, `VSYS`, `BAT+` (Unified PCB: diode-OR node and 2S battery path); `24V_DU` (CSC: J_DU_IN → F_DU → Q_DOCK → F_DOCK → `DOCK_24V`, up to 1.6 A — the CSC has no other 24 V net on its ground side) |
| Power_GND | 0.50 | 0.30 | 0.8 / 0.4 | `GND` (uses L2 plane primarily) |
| USB | 0.20 | 0.20 | 0.6 / 0.3 | `USB_DP`, `USB_DM` — 90R differential impedance, length-matched within 0.5mm |
| HighCurrent | 1.50 | 0.50 | 1.0 / 0.5 | `24V_SU` (floating Stirring Unit feed on the CSC SU island, J_SU_IN → F_SU → J_STIR_PWR, up to 3 A), `6V5_DSB` (switched servo rail, up to 2.5 A stall), `PUMP_O*`, `SOL*_D`, `FAN*_D` |

---

## 11. DRC Severity Policy

Rules configured in each `.kicad_pro` — key severity overrides from KiCad defaults:

| Check | Severity | Rationale |
|-------|----------|-----------|
| `clearance` | **error** | Hard constraint — must not violate |
| `track_width` | **error** | Must meet current-carrying minimums |
| `annular_width` | **error** | Manufacturing reliability |
| `hole_to_hole` | **error** | Upgraded from warning — structural integrity |
| `silk_over_copper` | **error** | Upgraded from warning — readability |
| `missing_courtyard` | **warning** | Upgraded from ignore — placement checking |
| `copper_sliver` | **warning** | Manufacturing defect risk |
| `isolated_copper` | **warning** | Review and remove manually |
| `footprint_type_mismatch` | **warning** | Upgraded from ignore — catch wrong footprints |

---

## 12. Design Settings Summary (`.kicad_pro`)

Global DRC minimums for the unified board:

```
min_clearance:             0.20 mm
min_track_width:           0.15 mm
min_via_annular_width:     0.13 mm
min_via_diameter:          0.60 mm
min_through_hole_diameter: 0.30 mm
min_copper_edge_clearance: 0.30 mm
min_hole_clearance:        0.25 mm
min_hole_to_hole:          0.25 mm
min_silk_clearance:        0.15 mm
min_text_height:           0.80 mm
min_text_thickness:        0.08 mm
```

---

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.7 | 2026-11-05 | Manas Pradhan | Aligned to CSC v1.6: `Power_12V_HU` / `12V_HU` renamed `Power_12V_IN` / `12V_IN` — the CSC's board supply is now a dedicated Mean Well LRS-35-12 on J_12V_IN (C_IN 100 µF) instead of the heating-unit tap; same width and clearance |
| 1.6 | 2026-11-05 | Manas Pradhan | Aligned to CSC v1.5: new `Power_12V_HU` class (1.00 mm) for `12V_HU`, the CSC's board supply tapped from the heating unit (J_HU_PWR → fans, relay coils, MP1584EN); `24V_DU` (J_DU_IN → Q_DOCK) added to Power_24V as the CSC's only ground-referenced 24 V net; `24V_SU` HighCurrent rule unchanged; `12V_OUT` / `24V_INT` marked as Unified PCB nets |
| 1.5 | 2026-11-04 | Manas Pradhan | Aligned to Unified PCB v5.2 / CSC v1.4: the DU CAN node is a non-isolated SN65HVD230, so the isolation-slot and reinforced-creepage rules apply to the CSC only (one island per port there); the Unified PCB has no isolated net, pad 5 `GND_ISO` ties to `GND` and pad 6 is unused; CAN net class reworded |
| 1.4 | 2026-11-04 | Manas Pradhan | Aligned to CSC v1.3: `24V_STIR` → `24V_SU` in HighCurrent; CAN class names the stir bus as the CSC's second isolated port (SU domain, `GND_SU`); §7 notes the CSC's second isolation slot under ISO1050DUB #3 + B0505S #2 and the floating SU island (`24V_SU`, `GND_SU`, `5V_ISO2`) |
| 1.3 | 2026-11-04 | Manas Pradhan | Net classes aligned with Unified PCB v5.0: MCP2518FD removed from the SPI and CAN classes (SPI0 is LCD-only), UART class added for the RV1106 ↔ DU STM32 host link (HOST_TX/RX, BOOT0, NRST), I2C class is the DU STM32 I2C1 plus the touch-only I2C3; intro updated for the two-board scope |
| 1.2 | 2026-11-04 | Manas Pradhan | Net classes aligned with design doc v3.x: SPI class is the Pico Zero SPI0 (MCP2518FD, J_LCD) instead of the retired STM32 SPI2 bridge; CAN class covers both the isolated heater bus and the regular stir-motor bus; `12V_UPS` and `LACT_O*` (linear actuator) removed; `VSYS`/`BAT+` added to Power_24V; `24V_STIR` and `6V5_DSB` added to HighCurrent; intro updated for the single-board v3.x scope |
| 1.1 | 2026-03-09 | Manas Pradhan | Added USB differential pair net class (90R impedance, length-matched) for USB-C programming port |
| 1.0 | 2026-03-06 | Manas Pradhan | Initial design rules document for unified PCB — stackup, track widths, clearances, via specs, net classes, DRC policy, JLCPCB limits |
