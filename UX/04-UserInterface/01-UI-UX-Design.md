---
created: 2026-02-15
modified: 2026-11-04
version: 1.2
status: Draft
---

# UI/UX Design

> [!warning] Screen layouts pending redesign for 480x320
> The application processor is now the Rockchip RV1106G3 (Luckfox Pico Zero) and the primary display is a 4" 480x320 SPI LCD rendered by LVGL 9. **All screen layouts in Section 2 were designed for the previous 10" 1280x800 display and are pending redesign for 480x320.** They are kept as design intent only; do not implement them pixel-for-pixel. Their *content* (dispenser positions, temperature source, stir and E-stop semantics) was brought up to the current hardware on 2026-11-04: three dispensers (FID Box1–6 and CID Slot1–5 on one servo block, SLD oil/water with load cells), a commercial induction module that runs its own temperature loop and is the only temperature source, and a CAN servo stirring motor. The platform constraints in Sections 1, 5 and 6 are current. The Vite/React mockups in `epi-Design/UX/mocks` remain design-intent mockups of the companion app, not an implementation of the device UI.

## 1. Display Overview

### 1.1 Primary Display
- **Type:** 4" IPS LCD, ST7796S controller
- **Resolution:** 480x320 (landscape)
- **Interface:** SPI0 from the Luckfox Pico Zero (RV1106G3), fbtft driver, 50 MHz with DMA; backlight on PWM10
- **Touch:** FT6336 capacitive touch on I2C3, evdev
- **Rendering:** LVGL 9, software rendered (the RV1106 has no GPU), 20 Hz tick. A full-frame refresh over SPI takes ~50 ms, so the UI relies on LVGL partial redraws and avoids full-screen animations, screen-slide transitions and scrolling backgrounds
- **Mounting:** Integrated into Epicura enclosure top panel

### 1.2 Secondary Display
- **Native companion mobile apps** (iOS: SwiftUI, Android: Jetpack Compose)
- **Communication:** Cloud REST/MQTT (the device is a client of the cloud, not a server); BLE for pairing and Wi-Fi provisioning; live view over RTSP on the same LAN or via cloud relay
- **Features:** Remote recipe browsing, on-demand live pot view (H.264 720p), cooking notifications

---

## 2. Screen Layouts

### 2.1 Home Screen

```
┌──────────────────────────────────────────────┐
│ ☰  Epicura              12:30 PM    ⚙ WiFi   │
├──────────────────────────────────────────────┤
│                                              │
│   Good Afternoon, Manas!                     │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │   Dal    │  │  Paneer  │  │   Rice   │    │
│  │  Tadka   │  │  Butter  │  │  Pulao   │    │
│  │  🕐 35m  │  │  🕐 45m   │  │  🕐 40m  │    │
│  │  ★★★★☆   │  │  ★★★★★   │  │  ★★★☆☆   │    │
│  └──────────┘  └──────────┘  └──────────┘    │
│                                              │
│  ┌──────────┐                                │
│  │  Aloo    │                                 │
│  │  Gobi    │   Recently Cooked:              │
│  │  🕐 30m  │   Dal Tadka - Yesterday        │
│  │  ★★★★☆   │   Paneer Butter - 2 days ago    │
│  └──────────┘                                │
│                                              │
│        [    Browse All Recipes    ]          │
│                                              │
├──────────────────────────────────────────────┤
│  Ready  │  Pot: Not Detected  │  Surf 25°C   │
└──────────────────────────────────────────────┘
```

**Elements:**
- Status bar: hamburger menu, brand name, time, settings gear, WiFi indicator
- Greeting with user name (from preferences)
- 3-4 quick recipe cards (favorites or suggestions)
- Recent cooking history
- "Browse All Recipes" prominent button
- Bottom status bar: device state, pot detection and surface temperature. Both come from the induction module's HEAT_STATUS frame (pot_detected, surface_temp); Epicura has no temperature sensor of its own

---

### 2.2 Recipe Selection

```
┌──────────────────────────────────────────────┐
│ ◄ Back         Browse Recipes        🔍      │
├──────────────────────────────────────────────┤
│                                              │
│ [All] [Vegan] [Healthy] [Vegetarian] [Pro..] │
│ [Indian] [Italian] [American] [Chinese] [..] │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │ ┌────────┐  Dal Tadka               │    │
│  │ │ (food  │  35 min · Easy ●○○       │    │
│  │ │  bowl) │  P:18g C:42g F:8g 320cal │    │
│  │ └────────┘                           │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │ ┌────────┐  Dal Makhni              │    │
│  │ │ (food  │  40 min · Med ●●○        │    │
│  │ │  bowl) │  P:14g C:38g F:12g 350cal│    │
│  │ └────────┘                           │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │ ┌────────┐  Sambar                  │    │
│  │ │ (food  │  45 min · Med ●●○        │    │
│  │ │  bowl) │  P:12g C:48g F:6g 290cal │    │
│  │ └────────┘                           │    │
│  └──────────────────────────────────────┘    │
│                                              │
│         [ Load More... ]                     │
└──────────────────────────────────────────────┘
```

**Elements:**
- **Filter chips (top row):** All Recipes, Vegan, Healthy, Vegetarian, Protein Rich, Stir Fry, Gluten Free, Quick Recipe (horizontally scrollable)
- **Cuisine tabs (second row):** Indian, Italian, American, Chinese, Mexican, Korean, Thai, Asian, Global
- Search bar with text input
- Recipe cards in list layout:
  - Food image in bowl on the **left side** of the card
  - Recipe name, time, difficulty on the right
  - **Nutrition per serving** displayed below: Protein (g), Carbs (g), Fats (g), Calories (kcal)
- Difficulty indicator: Easy/Medium/Hard with dots
- Scrollable list, "Load More" or infinite scroll

---

### 2.3 Recipe Detail

```
┌──────────────────────────────────────────────┐
│ ◄ Back          Dal Tadka            ♡       │
├──────────────────────────────────────────────┤
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │                                      │    │
│  │        [ Recipe Photo ]              │    │
│  │                                      │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  🕐 35 min  │  Serves 4  │  Easy ●○○         │
│                                              │
│  Spice Level: [──────●────────] Medium       │
│                                              │
│  ⚠ Allergens: Contains mustard seeds         │
│                                              │
│  Ingredients:                                │
│  ├─ SLD Oil:    Oil (30 mL)                  │
│  ├─ FID Box1:   Turmeric powder (1 tsp)      │
│  ├─ FID Box2:   Chili powder (2 tsp)         │
│  ├─ FID Box4:   Salt (¾ tsp)                 │
│  ├─ FID Box5:   Garam masala (1¼ tsp)        │
│  ├─ CID Slot1:  Onions + Tomatoes (150 g)    │
│  ├─ CID Slot2:  Toor dal (200 g)             │
│  └─ SLD Water:  Water (400 mL)               │
│                                              │
│  Pre-loaded in pot: Cooked toor dal (400 g)  │
│                                              │
│       [     Start Cooking     ]              │
│                                              │
└──────────────────────────────────────────────┘
```

**Elements:**
- Recipe photo (from local storage or placeholder)
- Time estimate, servings, difficulty badge
- Spice level slider (adjustable, saved to preferences)
- Allergen warnings (highlighted, configurable in settings)
- Ingredient list mapped to dispenser positions: FID Box1–Box6 (quantities in teaspoons; the recipe stores ¼-tsp doses and grams are derived for logging only), CID Slot1–Slot5 in ascending order of use (the whole slot is dispensed, so the quantity is what the user pre-measures), SLD Oil / Water in mL
- Any pre-loading instructions (items placed directly in pot)
- Prominent "Start Cooking" button

---

### 2.4 Ingredient Loading

```
┌──────────────────────────────────────────────┐
│          Load Ingredients: Dal Tadka         │
├──────────────────────────────────────────────┤
│                                              │
│   Step 6 of 8                                │
│   [████████████████████░░░░░░░] 75%          │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │  FID boxes        CID slots     SLD  │    │
│  │  1  2  3  4  5  6   1  2  3  4  5    │    │
│  │  ✓  ✓  ·  ✓  ✓  ·   ✓  ►  ·  ·  ·    │    │
│  │                     Oil ✓   Water !  │    │
│  │  (carousel at COVER, gate closed)    │    │
│  └──────────────────────────────────────┘    │
│                                              │
│   ► CID Slot2: Toor dal — 200 g              │
│     Pre-measure into the slot; the whole     │
│     slot empties when dispensed.             │
│                       [ ✓ Slot2 loaded ]     │
│                                              │
│   SLD  Oil   182 g  (need 28 g)  ✓           │
│        Water  44 g  (need 400 g) ! Refill    │
│                                              │
│   [ ◄ Previous ]        [ Next ► ]           │
│                                              │
├──────────────────────────────────────────────┤
│   [ Cancel ]     [ All Loaded — Start ]      │
└──────────────────────────────────────────────┘
```

**Elements:**
- Step counter and progress bar; one step per FID box, CID slot and SLD channel the recipe uses
- Top-view dispenser diagram (FID Box1–6, CID Slot1–5, SLD oil/water) with the current position highlighted and checkmarks on confirmed ones. Carousel positions come from DISPENSER_STATUS (0x239): the screen shows the CID carousel parked at its COVER sector and the FID steam gate closed before asking the user to load
- **FID and CID steps are tap-to-confirm.** Neither carousel senses fill level, so the user confirms each box/slot visually (spec §10.3). CID slots are assigned in ascending order of use (Slot1 first) because every slot that passes the aperture empties; the screen enforces that order, and a slot designated for raw protein is flagged with a cleaning reminder after the cook
- **SLD steps show live reservoir weight** from the dedicated load cells (LOADCELLS 0x311, or QUERY_WEIGHT 0x235 on demand) against the recipe requirement; below the low-level threshold (default 50 g remaining, reported as SLD_LOW_LEVEL in STATUS 0x312) the screen asks for a refill. Long-press on a reservoir runs TARE (0x237)
- Hold-time timer starts when perishables are loaded (alert at 90 min, spec §11.1)
- Previous/Next navigation
- "All Loaded — Start" button, enabled when PREFLIGHT (0x236) reports every required subsystem loaded/ready and both SLD channels are above the recipe minimum; otherwise disabled with the failing subsystem named

---

### 2.5 Cooking Progress

```
┌──────────────────────────────────────────────┐
│  Cooking: Dal Tadka          Stage 3 of 6    │
├──────────────────────────────────────────────┤
│                                              │
│  Current Stage: Saute Onions                 │
│  [████████████████░░░░░░░░░░░] 62%           │
│                                              │
│  ┌────────────┐  ┌────────────┐  ┌─────────┐ │
│  │ Surface    │  │ Time Left  │  │  Pot    │ │
│  │ 148°C      │  │ 3:12       │  │  ✓      │ │
│  │ Set 150    │  │ of 5:00    │  │ 60% pwr │ │
│  └────────────┘  └────────────┘  └─────────┘ │
│                                              │
│  Stir: ● Continuous, 60 RPM (meas. 58)       │
│  CV:   ● Monitoring (golden_brown: 0.42)     │
│                                              │
│              [ View Pot ]                    │
│                                              │
│  [ ⏸ Pause ]              [ ⏹ Emergency Stop ]│
│                                              │
└──────────────────────────────────────────────┘
```

**Elements:**
- Stage counter in header
- Current stage name and overall progress bar
- Surface temperature tile: the module-reported surface_temp from HEAT_STATUS (0x180, 10 Hz) against the stage setpoint the `heat` thread is sending (HEAT_SET_TEMP 0x100). The module regulates; the UI only displays. Shown as "Surface", never "Food", since the module measures the pot bottom through the glass
- Pot / power tile: pot_detected and power_pct from HEAT_STATUS; a missing pot shows "No pot" and the module stops heating on its own
- Time remaining countdown
- Stir status indicator: pattern and commanded RPM from the recipe step, measured RPM from the CAN servo's feedback (TELEMETRY 0x310 motor_rpm)
- CV monitoring status (what it is looking for, current confidence)
- "View Pot" button opens the on-demand pot preview (§6.3); the cooking screen itself carries no permanent video
- Pause button (holds current state; the heater is held at reduced power while paused)
- Emergency Stop button (prominent, red, always accessible). It broadcasts E_STOP (0x001) on the CAN bus, which the STM32 (AC relay open, stir off, dispensers off) and the induction module both act on

---

### 2.6 Cooking Complete

```
┌──────────────────────────────────────────────┐
│              Cooking Complete!               │
├──────────────────────────────────────────────┤
│                                              │
│              ✓                               │
│                                              │
│         Dal Tadka is Ready!                  │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │  Summary:                            │    │
│  │                                      │    │
│  │  Total Time:     34 min 22 sec       │    │
│  │  Stages:         6 of 6 completed    │    │
│  │  Peak Surface:   182°C               │    │
│  │  CV Transitions: 5 auto, 1 timer     │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  Rate This Cook:                             │
│  [ ★ ] [ ★ ] [ ★ ] [ ★ ] [ ☆ ]               │
│                                              │
│  Notes: [________________________]           │
│                                              │
│  [ Cook Again ]        [ Home ]              │
│                                              │
└──────────────────────────────────────────────┘
```

**Elements:**
- Success checkmark and message
- Cooking summary: total time, stages completed, peak surface temperature (module-reported), CV vs timer transitions
- Star rating (1-5, saved to cooking_logs)
- Optional notes text field
- "Cook Again" button (same recipe) and "Home" button

---

### 2.7 Settings

```
┌──────────────────────────────────────────────┐
│ ◄ Back              Settings                 │
├──────────────────────────────────────────────┤
│                                              │
│  Language              [ English       ▼ ]   │
│                                              │
│  WiFi Network          [ EpicuraHome   ▼ ]   │
│                                              │
│  ──── Cooking Preferences ────               │
│  Default Spice Level   [────●──────] Med     │
│  Default Servings      [ 4          ▼ ]      │
│                                              │
│  ──── Allergen Profile ────                  │
│  [ ] Nuts     [ ] Dairy    [✓] Mustard       │
│  [ ] Gluten   [ ] Soy      [ ] Sesame        │
│                                              │
│  ──── Display ────                           │
│  Brightness            [──────────●] 80%     │
│  Theme                 [ Light     ▼ ]       │
│                                              │
│  ──── Dispensers ────                        │
│  FID Box Contents      [ Edit ▸ ]            │
│  SLD Reservoirs        Oil 182 g · Water 44 g│
│  [ Home Dispensers ]   [ Rinse SLD Lines ]   │
│                                              │
│  ──── Dispensers ────                        │
│  FID Box Contents      [ Edit ▸ ]            │
│  SLD Reservoirs        Oil 182 g · Water 44 g│
│  [ Home Dispensers ]   [ Rinse SLD Lines ]   │
│                                              │
│  ──── System ────                            │
│  Firmware Version      RV1106: 1.0.0         │
│                        STM32: 1.0.0          │
│  [ Check for Updates ]                       │
│  [ Factory Reset ]                           │
│                                              │
│  ──── About ────                             │
│  Device ID: EPIC-001                         │
│  Serial: SN2026020001                        │
│                                              │
└──────────────────────────────────────────────┘
```

**Elements:**
- Language dropdown (see Multi-Language Support below)
- WiFi network selection and connection status
- Cooking preferences: default spice level, default servings
- Allergen profile checkboxes (flagged recipes show warnings)
- Display: brightness slider, light/dark theme
- Dispensers: FID box contents (which spice is in Box1–Box6 and its bulk density, user-editable per spec §4.2 and §9), live SLD reservoir weights, "Home Dispensers" (DISPENSER_HOME 0x238: CID to COVER, FID gate closed) and the SLD rinse cycle (spec §12.2; command ID to be allocated in Controller Software Architecture §7.2)
- System: firmware versions for both processors, update check, factory reset
- Device identification

---

## 3. Multi-Language Support

### 3.1 Supported Languages

| Language | Script | Locale Code | Priority |
|----------|--------|-------------|----------|
| English | Latin | en_IN | Primary |
| Hindi | Devanagari (हिन्दी) | hi_IN | Primary |
| Tamil | Tamil (தமிழ்) | ta_IN | Secondary |
| Telugu | Telugu (తెలుగు) | te_IN | Secondary |
| Kannada | Kannada (ಕನ್ನಡ) | kn_IN | Secondary |
| Malayalam | Malayalam (മലയാളം) | ml_IN | Secondary |
| Bengali | Bengali (বাংলা) | bn_IN | Secondary |
| Marathi | Devanagari (मराठी) | mr_IN | Secondary |

### 3.2 Implementation

**LVGL i18n Workflow (`lv_i18n`):**
1. Mark all UI strings with the `_()` macro
2. Run `lv_i18n extract` to collect strings into per-locale YAML translation files
3. Translate the YAML files (Poedit-style tooling or by hand)
4. Run `lv_i18n compile` to generate `lv_i18n.c`, which is built into `epicurad` (no runtime file loading)
5. Call `lv_i18n_set_locale()` at startup from the `user_preferences.language` row in SQLite

```c
/* In C files */
#include "lv_i18n.h"

lv_label_set_text(label,    _("Cooking in Progress..."));
lv_label_set_text(greeting, _("Good Afternoon!"));
lv_label_set_text(start_lbl, _("Start Cooking"));
lv_label_set_text_fmt(temp_label, _("Temperature: %d°C"), current_temp);
```

**Font Requirements:**
- Noto Sans family covers all target scripts
- Converted to LVGL bitmap fonts with `lv_font_conv` (per script and size, glyph ranges subset per language) and stored on the `/oem` partition
- Fallback font chain for missing glyphs (LVGL font `fallback` property)
- Estimated font storage: ~15-20 MB for all scripts on `/oem`; only the active language's fonts are loaded into RAM

---

## 4. UX Goals

### 4.1 Five Core Principles

1. **Easy Recipe Selection (< 3 taps to start)**
   - Home screen shows favorites and recent recipes
   - One tap on recipe card → recipe detail
   - One tap "Start Cooking" → ingredient loading
   - Maximum 3 taps from power-on to cooking

2. **Clear Ingredient Loading Instructions**
   - Visual dispenser diagram (FID boxes, CID slots, SLD reservoirs) with highlighting
   - Tap-to-confirm for FID boxes and CID slots (no fill sensing); live SLD reservoir weight feedback from load cells
   - Step-by-step guided flow
   - Confirmation before proceeding

3. **Transparent Cooking Progress**
   - Pot preview available on demand ("view pot" screen, ≤ 10 fps); the cooking screen shows stage, temperature and timer without a permanent video feed
   - Real-time surface temperature (reported by the induction module), time, and stir status
   - CV confidence indicator (user can see what the system "sees")
   - Stage-by-stage progress with estimated completion

4. **Safety First**
   - Emergency Stop button visible on every cooking screen
   - Red color, large touch target, no confirmation dialog
   - Clear warning messages for module faults (HEAT_FAULT) and the STM32 WARNING / CRITICAL states
   - Pot detection (the induction module's own, reported in HEAT_STATUS) prevents cooking without pot

5. **Offline-Capable**
   - All UI assets stored locally
   - Full recipe library cached on device
   - No internet required for any cooking operation
   - Cloud features (sync, OTA) are optional enhancements

---

## 5. Accessibility

### 5.1 Touch Targets
- Minimum size: 48 x 48 pixels for all interactive elements
- Emergency Stop: 96 x 48 pixels minimum (oversized for safety)
- Spacing between targets: minimum 8 pixels

### 5.2 Visual Accessibility
- **High contrast mode:** Dark backgrounds with bright text (toggle in settings)
- **Adjustable font size:** Small / Medium / Large / Extra-Large
- **Color-blind safe palette:** Avoid red/green only indicators; use shape + color
- **Status indicators:** Use both color and icon (e.g., green checkmark, red X)

### 5.3 Audio Feedback
- **Stage transitions:** Distinct beep pattern when moving to next cooking stage
- **Completion:** Musical chime when cooking is done
- **Warnings:** Rapid beeping for temperature warnings
- **Emergency:** Continuous alarm tone for E-stop or critical errors
- **Touch:** Subtle click sound on button press (configurable)

### 5.4 Text & Typography
- Primary font: Noto Sans (covers all Indian scripts)
- Minimum body text: 16px (on 4" 480x320 display, ~144 ppi; all sizes to be re-validated in the 480x320 redesign)
- Headings: 24-32px
- Temperature/time numbers: 36-48px (large, at-a-glance readable)

---

## 6. LVGL Implementation Notes

### 6.1 Application Architecture

The UI is the `ui` thread of `epicurad` (see [[02-Controller-Software-Architecture|Controller & Software Architecture]]): LVGL 9 rendering into a 480x320 RGB565 double buffer and flushing changed areas to `/dev/fb0` (fbtft, ST7796S on SPI0). Touch arrives through evdev from the FT6336 on I2C3.

```
┌─────────────────────────────────────────────────────────┐
│               ui thread (LVGL 9, 20 Hz tick)            │
│  ┌─────────────────┐  ┌──────────────────────────────┐  │
│  │   header_bar    │  │        status_bar            │  │
│  └─────────────────┘  └──────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐   │
│  │      Screen stack (lv_screen_load, no anim)      │   │
│  │  ┌────────────────────────────────────────────┐  │   │
│  │  │ home_scr │ recipe_scr │ cooking_scr │ ...  │  │   │
│  │  └────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │  pot_preview (ISP main → RGA 480x320 → /dev/fb0) │   │
│  │  on demand only, ≤ 10 fps                        │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 6.2 Key LVGL Screens and Widgets

| Component | File | Purpose |
|-----------|------|---------|
| HomePage | `home_screen.c` | Greeting, quick recipes, recent history |
| RecipeBrowser | `recipe_browser_screen.c` | Category filters, list view, search |
| RecipeDetail | `recipe_detail_screen.c` | Ingredients, spice slider, allergens |
| IngredientLoader | `ingredient_loader_screen.c` | Step-by-step dispenser loading (FID boxes, CID slots, SLD reservoirs) |
| CookingProgress | `cooking_progress_screen.c` | Gauges, progress, E-stop, "view pot" button |
| CookingComplete | `cooking_complete_screen.c` | Summary, rating, notes |
| SettingsPage | `settings_screen.c` | Language, WiFi, preferences, firmware |
| PotPreview | `pot_preview.c` | On-demand ISP main path → RGA → framebuffer, ≤ 10 fps |
| TempGauge | `temp_gauge.c` | Circular surface-temperature gauge (`lv_arc` + label) fed from HEAT_STATUS |
| DispenserMap | `dispenser_map.c` | Top-view FID/CID/SLD diagram driven by DISPENSER_STATUS and LOADCELLS |
| RecipeCard | `recipe_card.c` | Reusable card with photo, name, time |

### 6.3 Pot Preview

There is no video widget. While the "view pot" screen is open, the RGA scales the always-live ISP main path (1280x720 NV12) to 480x320 RGB565 and writes it straight into the framebuffer region that LVGL is told not to repaint. Frames used for inference never pass through the UI (ISP self-path → NPU).

```c
/* pot_preview.c — on demand only. Memory-pressure fallback (free RAM < 32 MB)
 * drops this preview before anything else. */
int pot_preview_start(ui_ctx_t *ui)
{
    /* ISP main path 1280x720 NV12 → RGA scale/convert → 480x320 RGB565 → /dev/fb0 */
    rga_bind(&ui->rga, &ui->cv->vi, ISP_PATH_MAIN, 480, 320, PIX_RGB565);
    lv_obj_add_flag(ui->preview_area, LV_OBJ_FLAG_HIDDEN);   /* LVGL leaves the region alone */
    ui->preview_timer = lv_timer_create(pot_preview_tick, 100 /* ms, ≤ 10 fps */, ui);
    return 0;
}

void pot_preview_stop(ui_ctx_t *ui)
{
    lv_timer_delete(ui->preview_timer);
    rga_unbind(&ui->rga);
    lv_obj_remove_flag(ui->preview_area, LV_OBJ_FLAG_HIDDEN);
    lv_obj_invalidate(lv_screen_active());                   /* repaint over the last frame */
}
```

### 6.4 UI Data Sources

The `ui` thread never touches the CAN bus. It reads the lock-free ring buffer the `can` thread fills with STM32 telemetry and induction-module status, and the engine's state (see [[04-MPU-Functional-Specification|MPU Functional Specification]] §4.4). Commands the UI raises go through the engine onto the can command queue; only the `heat` thread sends heater setpoints.

| UI element | Source | Frame (CAN ID) |
|------------|--------|----------------|
| Surface temperature, power %, pot detected | Induction module via `can` thread | HEAT_STATUS (0x180), 10 Hz |
| Heater setpoint shown as "Set" | `heat` thread | HEAT_SET_TEMP (0x100) / HEAT_SET_POWER (0x103) |
| Module fault banner | Induction module | HEAT_FAULT (0x181), fault_flags in HEAT_STATUS |
| Stir pattern / commanded RPM | Engine (recipe step) | SET_STIR (0x202) |
| Measured stir RPM, STM32 safety state, battery flags | STM32 | TELEMETRY (0x310), 20 Hz |
| STM32 fault / SLD low level | STM32 | STATUS (0x312), on change |
| SLD oil / water weights | STM32 (HX711 load cells) | LOADCELLS (0x311), QUERY_WEIGHT (0x235) |
| Dispenser diagram positions | STM32 | DISPENSER_STATUS (0x239) |
| "All Loaded — Start" gate | STM32 | PREFLIGHT (0x236) |
| Tare / Home Dispensers buttons | STM32 | TARE (0x237), DISPENSER_HOME (0x238) |
| Emergency Stop button | Broadcast by the RV1106 | E_STOP (0x001), no ACK |
| Link-loss overlay | `can` thread watchdogs | No TELEMETRY for 300 ms or no HEAT_STATUS for 500 ms → heater off, PAUSED |

### 6.5 Styling

**Design Language:** Material Design inspired, warm food-friendly palette

| Element | Color | Hex |
|---------|-------|-----|
| Primary | Warm Orange | #E65100 |
| Secondary | Deep Green | #2E7D32 |
| Background | Cream White | #FFF8E1 |
| Surface | White | #FFFFFF |
| Text Primary | Dark Brown | #3E2723 |
| Text Secondary | Medium Brown | #6D4C41 |
| Emergency Stop | Bright Red | #D50000 |
| Success | Green | #43A047 |
| Warning | Amber | #FFB300 |

```c
/* theme.h - App-wide styling constants */
#define THEME_PRIMARY         lv_color_hex(0xE65100)
#define THEME_SECONDARY       lv_color_hex(0x2E7D32)
#define THEME_BACKGROUND      lv_color_hex(0xFFF8E1)
#define THEME_SURFACE         lv_color_hex(0xFFFFFF)
#define THEME_TEXT_PRIMARY    lv_color_hex(0x3E2723)
#define THEME_TEXT_SECONDARY  lv_color_hex(0x6D4C41)
#define THEME_EMERGENCY       lv_color_hex(0xD50000)
#define THEME_SUCCESS         lv_color_hex(0x43A047)
#define THEME_WARNING         lv_color_hex(0xFFB300)

#define FONT_SIZE_BODY        16
#define FONT_SIZE_HEADING     24
#define FONT_SIZE_DISPLAY     36
#define TOUCH_TARGET_MIN      48
```

---

## 7. Companion App Design

### 7.1 Native Mobile Architecture

Epicura uses native mobile development: **SwiftUI** (iOS) and **Jetpack Compose** (Android), following the MVVM pattern. For complete architecture details, project structure, and platform-specific implementation guides, see:

- [[01-Mobile-Architecture|Mobile Architecture]] - MVVM layers, networking, BLE manager, design system
- [[02-iOS-App|iOS App]] - Swift/SwiftUI project structure, Core Bluetooth, APNs
- [[03-Android-App|Android App]] - Kotlin/Compose project structure, CompanionDeviceManager, FCM

### 7.2 Key Screens

1. **Recipe Browse** - List layout with food bowl image on left, tag filters (Vegan/Healthy/Vegetarian/Protein Rich/Stir Fry/Gluten Free/Quick), cuisine filters (Indian/Italian/American/Chinese/Mexican/Korean/Thai/Asian/Global), nutrition per serving (Protein/Carbs/Fats/Calories)
2. **Recipe Detail** - Ingredients, spice customization, allergen flags, nutrition breakdown, remote start
3. **Live Cook View** - On-demand RTSP H.264 live view (same LAN, or via cloud relay), surface-temperature gauge (module-reported), stir status, stage progress, notifications
4. **History** - List of past cooks with date, rating, duration, recipe name
5. **Profile / Settings** - Food preferences, device pairing, allergen profile, notifications
6. **Food Preferences** (within Profile tab) — Dedicated section for dietary and taste customization:

```
┌──────────────────────────────────────────────┐
│ ◄ Back            Profile                    │
├──────────────────────────────────────────────┤
│                                              │
│  ──── Food Preferences ────                 │
│                                              │
│  Diet                                        │
│  [ Vegetarian | Vegan | Pescatarian | ●None ]│
│                                              │
│  Preferred Cuisines                          │
│  [●Indian] [●Italian] [●Thai] [Chinese]     │
│  [American] [●Mexican] [Korean] [Global]    │
│                                              │
│  Seasoning Levels                            │
│  Spice  [─────────●─────────] 3             │
│  Salt   [─────────●─────────] 3             │
│  Oil    [─────────●─────────] 3             │
│                                              │
│  Typical Servings     [ − ]  2  [ + ]       │
│                                              │
│  ──── Allergen Profile ────                 │
│  [ ] Nuts     [ ] Dairy    [✓] Mustard      │
│  [ ] Gluten   [ ] Soy      [ ] Sesame       │
│                                              │
│  ──── Device & Account ────                 │
│  Paired Device        EPIC-001 (Online)     │
│  Notifications        [  ON  ]              │
│  Theme                [ Light     ▼ ]       │
│                                              │
└──────────────────────────────────────────────┘
```

**Food Preferences Fields:**

| Control | Type | Options / Range | Default |
|---------|------|-----------------|---------|
| Diet | Single-select segmented | Vegetarian, Vegan, Pescatarian, No Restrictions | No Restrictions |
| Preferred Cuisines | Multi-select chip group | Indian, Italian, Thai, Chinese, American, Mexican, Korean, Global | Indian, Italian, Thai, Mexican |
| Spice Level | 5-point discrete slider | 1 (Mild) – 5 (Hot) | 3 |
| Salt Level | 5-point discrete slider | 1 (Low) – 5 (High) | 3 |
| Oil Level | 5-point discrete slider | 1 (Light) – 5 (Rich) | 3 |
| Typical Servings | Stepper (spinner) | 1 – 4 | 2 |
6. **Device Pairing** - BLE scan, WiFi provisioning, cloud account linking
7. **Device Status** - Firmware versions, connection status, SLD reservoir levels, dispenser and induction-module health

### 7.3 Communication

| Channel | Protocol | Data |
|---------|----------|------|
| Recipe browsing | REST (HTTP GET) | JSON recipe list and details |
| Start cooking | REST (HTTP POST) | Recipe ID, customization params |
| Live camera | RTSP/H.264 from the device on the same LAN (`rtsp://<device>:554/pot`) or via cloud relay; started by the cloud over MQTT `live_view_start`, on demand only | 1280x720 @ 15 fps, 1.5 Mbps, single viewer |
| Cooking status | WebSocket | Real-time surface temp, heater power, stir RPM, stage, progress |
| Settings sync | REST (HTTP GET/PUT) | User preferences JSON |
| Push notifications | FCM (Android) / APNs (iOS) | Cooking complete, errors |
| Device pairing | BLE GATT services | WiFi provisioning, device identification |

### 7.4 Offline Recipe Cache

- iOS: Recipes cached locally using SwiftData; images cached using Nuke
- Android: Recipes cached locally using Room; images cached using Coil
- Offline mode: browse and select recipes, queue for when device is reachable
- Last sync timestamp displayed in app

---

## 8. Related Documentation

- [[__Workspaces/Appliance/Epicura/epi-Spec/docs/01-Overview/01-Project-Overview|Project Overview]]
- [[02-Controller-Software-Architecture|Controller & Software Architecture]]
- [[01-Tech-Stack|Tech Stack]]

#epicura #ui-ux #user-interface #touchscreen #mobile-app #lvgl #swift #kotlin #native-mobile #accessibility #multi-language

---

## 9. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-15 | Manas Pradhan | Initial document creation |
| 1.1 | 2026-11-04 | Manas Pradhan | Application processor changed to RV1106G3 (Luckfox Pico Zero): primary display is a 4" 480x320 SPI LCD with LVGL 9 (software rendering, no GPU); Kivy implementation notes replaced with LVGL; phone live view is on-demand RTSP; Section 2 layouts flagged as pending redesign for 480x320 |
| 1.2 | 2026-11-04 | Manas Pradhan | Brought screen content up to the actuator topology of epi-Spec commit 137c0e2 and the CAN host bus: three dispensers (FID Box1–6 and CID Slot1–5 on one servo block, SLD oil/water), ingredient loading as tap-to-confirm for FID/CID and live load-cell weight for SLD with DISPENSER_STATUS / PREFLIGHT gating, surface temperature and pot detection sourced from the induction module's HEAT_STATUS (no Epicura temperature sensor), CAN servo stir feedback, E-stop as a CAN E_STOP broadcast, dispenser settings group, new §6.4 UI data-source table. 480x320 redesign still pending. |