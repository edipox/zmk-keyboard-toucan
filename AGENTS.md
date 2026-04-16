# ZMK Keyboard Firmware: beekeeb Toucan

This document captures the complete context of this ZMK firmware configuration so it can be updated or extended without re-reading the entire codebase.

---

## 1. Project Overview

| Property | Value |
|----------|-------|
| Keyboard | beekeeb **Toucan** |
| Keys | 42-key wireless split |
| MCU Board | **Seeed Studio XIAO BLE** (`seeeduino_xiao_ble`) |
| Features | Keys, Cirque trackpad (pointing), nice!view display, RGB LED widget, ZMK Studio |
| ZMK Version | **v0.3** (tracked in `config/west.yml` and `.github/workflows/build.yml`) |

The keyboard is a column-stagger split with an aggressive pinky stagger. It includes a **Cirque Pinnacle trackpad** on the right half and an optional **nice!view display** (via `nice_view_gem` custom shield) on the left half.

---

## 2. Repository Layout

```
.
├── .github/workflows/build.yml          # CI workflow (uses zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3)
├── build.yaml                           # GitHub Actions matrix definition
├── config/
│   ├── west.yml                         # West manifest: pins ZMK v0.3 + external modules
│   ├── toucan.json                      # Physical layout metadata for ZMK Studio
│   ├── toucan.keymap                    # **User keymap** (the file users normally edit)
│   ├── toucan_left.conf                 # DOES NOT EXIST (config is at board/shield level)
│   └── toucan_right.conf                # DOES NOT EXIST
├── boards/shields/
│   ├── toucan/                          # **Core shield definition**
│   │   ├── toucan.dtsi                  # Shared DT: matrix, physical layout, split input definitions
│   │   ├── toucan_left.overlay          # Left half pins + nice_view_spi + glidepoint_listener enable
│   │   ├── toucan_right.overlay         # Right half pins + Cirque SPI device + input-split hookup
│   │   ├── toucan_left.conf             # Left Kconfig overrides
│   │   ├── toucan_right.conf            # Right Kconfig overrides
│   │   ├── toucan.zmk.yml               # Shield metadata (features: keys, display, studio)
│   │   ├── Kconfig.shield               # SHIELD_TOUCAN_LEFT / SHIELD_TOUCAN_RIGHT
│   │   └── Kconfig.defconfig            # Default configs (split, pointing, battery fetching, etc.)
│   └── nice_view_gem/                   # Custom nice!view display shield (modified from nice-view-gem)
│       ├── nice_view_gem.overlay
│       ├── nice_view_gem.conf
│       ├── Kconfig.defconfig / Kconfig.shield
│       ├── CMakeLists.txt
│       ├── custom_status_screen.c
│       ├── assets/
│       └── widgets/
├── toucan_left.conf                     # Duplicate of boards/shields/toucan/toucan_left.conf
├── toucan_right.conf                    # Duplicate of boards/shields/toucan/toucan_right.conf
└── zephyr/module.yml                      # board_root: .
```

**Important:** The *active* `.conf` and `.overlay` files used by the build are the ones inside `boards/shields/toucan/` (and `boards/shields/nice_view_gem/`). The top-level `toucan_left.conf` / `toucan_right.conf` appear to be convenience copies but are not referenced by the build matrix directly.

---

## 3. Build Matrix (`build.yaml`)

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: toucan_left rgbled_adapter nice_view_gem
    snippet: studio-rpc-usb-uart
    cmake-args: -DCONFIG_ZMK_STUDIO=y
  - board: seeeduino_xiao_ble
    shield: toucan_right rgbled_adapter
  - board: seeeduino_xiao_ble
    shield: settings_reset
```

- `toucan_left` gets the display (`nice_view_gem`) and **ZMK Studio** enabled.
- `toucan_right` has no display but hosts the Cirque trackpad.
- `rgbled_adapter` is an external shield provided by `zmk-rgbled-widget` module.
- `settings_reset` is a standard ZMK utility shield for clearing BLE pairings.

---

## 4. Hardware Definitions (DeviceTree)

### 4.1 Shared (`toucan.dtsi`)

**Matrix scan:** `zmk,kscan-gpio-matrix`, `col2row` diode direction.
- 4 rows, 12 columns (6 per half).
- `wakeup-source` is set so the matrix can wake from sleep.

**Physical Layout:** `zmk,physical-layout` named "Default Layout" with 42 keys. Coordinates match `config/toucan.json` (used by ZMK Studio).

**Matrix Transform:**
- 4 rows × 12 columns.
- Left half cols 0-5, right half cols 6-11.
- Right overlay applies `col-offset = <6>`.

**Split Input:**
```dts
split_inputs {
    glidepoint_split: glidepoint_split@0 {
        compatible = "zmk,input-split";
        reg = <0>;
    };
};

glidepoint_listener: glidepoint_listener {
    compatible = "zmk,input-listener";
    status = "disabled";
    device = <&glidepoint_split>;
    input-processors = <&zip_xy_scaler 250 100>;
    scroller {
        layers = <2>;   // SYM layer becomes scroll mode
        input-processors = <
            &zip_xy_to_scroll_mapper
            &zip_scroll_scaler 1 8
            &zip_scroll_transform INPUT_TRANSFORM_Y_INVERT
        >;
    };
};
```

### 4.2 Left Half (`toucan_left.overlay`)

- **Column GPIOs:** `gpio0 9`, `gpio0 4`, `gpio0 10`, `gpio0 5`, `gpio1 11`, `gpio1 12`
- **Row GPIOs:** `gpio0 19`, `gpio0 28`, `gpio1 1`, `gpio0 29`
- **SPI0 (for nice!view):**
  - SCK = P1.13, MOSI = P1.15, MISO = P1.14
  - CS = `gpio0 3` (active high)
- Disables `&xiao_i2c`.
- Enables `&glidepoint_listener` (`status = "okay"`), so the left (central) half receives trackpad events from the right (peripheral) half over BLE split.

### 4.3 Right Half (`toucan_right.overlay`)

- **Column GPIOs:** reversed order compared to left (`gpio1 12`, `gpio1 11`, `gpio0 5`, `gpio0 10`, `gpio0 4`, `gpio0 9`).
- **Row GPIOs:** same as left.
- **SPI0 (for Cirque Pinnacle):**
  - Same pins as left SPI, but **CS** = `gpio0 3` (active low + pull-up to keep unasserted in sleep).
  - `glidepoint@0` compatible = `"cirque,pinnacle"`
  - `dr-gpios = <&gpio0 2 (GPIO_ACTIVE_HIGH)>`
  - `sensitivity = "2x"`
  - `x-invert` and `no-taps` are set.
  - `sleep` is enabled (trackpad sleeps when ZMK idles).
- `&glidepoint_split` has `device = <&glidepoint>`, forwarding local trackpad input to the central side over split.

---

## 5. Kconfig / `.conf` Files

### Left (`toucan_left.conf`)

```
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_MOUSE=y
CONFIG_ZMK_BATTERY_REPORTING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=3600000       # 1 hour
CONFIG_ZMK_IDLE_TIMEOUT=30000               # 30 seconds
CONFIG_ZMK_PM_SOFT_OFF=y
CONFIG_ZMK_POINTING_SMOOTH_SCROLLING=y
CONFIG_ZMK_HID_REPORT_TYPE_NKRO=y
CONFIG_ZMK_HID_INDICATORS=y
CONFIG_ZMK_USB_BOOT=y
```

### Right (`toucan_right.conf`)

Same as left **except** the two `BATTERY_LEVEL_*` options are commented out (peripheral does not proxy/fetch central battery).

### Shield Defaults (`Kconfig.defconfig`)

- `ZMK_KEYBOARD_NAME` defaults to `"Toucan"` (left half only).
- `ZMK_SPLIT_ROLE_CENTRAL` = y (left half).
- Enables `SPI`, `ZMK_SPLIT`, `ZMK_POINTING`.
- `RGBLED_WIDGET_BATTERY_LEVEL_HIGH` defaults to `40`.

---

## 6. Keymap (`config/toucan.keymap`)

**Layers:**
| Index | Name | Purpose |
|-------|------|---------|
| 0 | `BASE` | Default typing layer |
| 1 | `NAV` | Navigation, Bluetooth, arrows |
| 2 | `SYM` | Symbols / scroll mode for trackpad |
| 3 | `ADJ` | Adjust (media, F-keys, toggles) |
| 4 | `LINUX` | Linux-specific base swap |
| 5 | `LINUX_NAV` | Linux navigation (conditional on NAV+LINUX) |

**Conditional Layers:**
- `NAV + SYM → ADJ`
- `NAV + LINUX → LINUX_NAV`

**Custom Behaviors:**
- `bspc_del`: mod-morph — `BSPC` normally, `DEL` when Shift held.
- `ht`: hold-tap (`tap-preferred`, 250 ms) — generic tap/hold for numbers/letters.
- `ht_click`: hold-tap where hold = `&mkp` (mouse key press), tap = `&kp`.
- `ht_mod_click`: hold-tap where hold = `&kp`, tap = `&mkp`.

**Notable bindings:**
- Trackpad is used directly as a pointing device.
- On `SYM` layer (layer 2), the trackpad automatically becomes a **scroller** via the `scroller` node in `toucan.dtsi`.
- Mouse clicks are bound on thumbs: `RCLK` / `LCLK` via `ht_click`/`ht_mod_click`.

---

## 7. nice!view Display Shield (`nice_view_gem`)

This is a **custom shield** based on `M165437/nice-view-gem`. It provides a custom status screen for the 144×168 Sharp memory LCD.

### Files of note

- `nice_view_gem.overlay` — enables `&nice_view_spi`, defines `sharp,ls0xx` display node.
- `Kconfig.defconfig` — forces 1-bit LVGL color depth, dedicated work queue, custom status screen, implies `NICE_VIEW_WIDGET_STATUS`.
- `CMakeLists.txt` — conditionally compiles widgets. **Only the central half** gets the full widget set (layer, profile, screen, sleep). Both halves get battery, output, util widgets.
- `custom_status_screen.c` — entry point `zmk_display_status_screen()`.
- `assets/` — embedded fonts (`quinquefive_24`, `quinquefive_8`) and images.
- `widgets/` — LVGL widgets showing layer, battery (local + peripheral), output status, BLE profile, sleep art.

**Config applied by shield:**
```
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=n
```

---

## 8. External Modules (West Manifest)

`config/west.yml`:

```yaml
manifest:
  defaults:
    revision: v0.3
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: petejohanson
      url-base: https://github.com/petejohanson/
    - name: caksoylar
      url-base: https://github.com/caksoylar
    - name: geeksville
      url-base: https://github.com/geeksville
  projects:
    - name: zmk
      remote: zmkfirmware
      import: app/west.yml
    - name: cirque-input-module
      remote: geeksville
      revision: toucan          # <-- custom branch for Toucan support
    - name: zmk-rgbled-widget
      remote: caksoylar
      revision: v0.3
  self:
    path: config
```

- **`cirque-input-module`** (branch `toucan`) — provides the Cirque Pinnacle driver and input split glue.
- **`zmk-rgbled-widget`** (tag `v0.3`) — provides `rgbled_adapter` shield for battery/status LEDs.

---

## 9. Power Management Notes

This firmware is tuned for low power:

1. **Deep sleep** (`CONFIG_ZMK_SLEEP=y`) after 1 hour of idle (`IDLE_SLEEP_TIMEOUT=3600000`).
2. **Regular idle** (`CONFIG_ZMK_IDLE_TIMEOUT=30000`) after 30 seconds of inactivity. This drops touchpad power draw significantly (~2.9 mA → ~1.7 mA).
3. **Soft-off** (`CONFIG_ZMK_PM_SOFT_OFF=y`) enabled so the Cirque driver receives suspend/resume notifications.
4. **Trackpad sleep** (`sleep` property in right overlay) puts the Pinnacle into sleep mode when ZMK is idle.
5. **SPI sleep pinctrl** has `low-power-enable` and `bias-pull-down` on the right half to minimize leakage.

---

## 10. Common Update Scenarios

### 10.1 Updating the keymap
Edit **`config/toucan.keymap`** only. The overlay/conf files in `boards/shields/toucan/` define hardware; the keymap in `config/` is the user layer.

### 10.2 Changing GPIO pins
Edit `boards/shields/toucan/toucan_left.overlay` or `toucan_right.overlay`. Remember:
- Left and right column GPIO arrays are **mirrored** (right is reversed).
- The right half hosts the Cirque trackpad on SPI0; CS is `gpio0 3` with active-low + pull-up.

### 10.3 Enabling/disabling ZMK Studio
Studio is **only on the left half** in `build.yaml` via `snippet: studio-rpc-usb-uart` and `cmake-args: -DCONFIG_ZMK_STUDIO=y`. The `toucan.zmk.yml` already lists `studio` as a feature, and `toucan.json` provides the physical layout.

### 10.4 Updating ZMK version
Change `revision: v0.3` in `config/west.yml` and the workflow reference `@v0.3` in `.github/workflows/build.yml` together. Be aware:
- `cirque-input-module` is pinned to branch `toucan`; verify compatibility before upgrading ZMK.
- `zmk-rgbled-widget` is pinned to tag `v0.3`.

### 10.5 Changing trackpad behavior
- Sensitivity / inversion / taps: edit the `glidepoint` node in `toucan_right.overlay`.
- Scroll layer mapping: edit the `scroller { layers = <2>; ... }` node in `toucan.dtsi`.
- XY scaling: edit `input-processors` on `glidepoint_listener` in `toucan.dtsi`.

### 10.6 Changing display widgets
Edit files under `boards/shields/nice_view_gem/widgets/` and `custom_status_screen.c`. Rebuild will pick them up automatically because the shield is local.

---

## 11. Reference Commands

If you clone ZMK locally for reference (already done at `/tmp/zmk-reference` in this environment):

```bash
# Reference ZMK v0.3 source
cd /tmp/zmk-reference

# Search for standard shield examples
grep -r "cirque,pinnacle" app/
grep -r "input-split" app/
```

---

## 12. License Summary

- Repo code: **MIT**
- `nice_view_gem` shield: modified from `M165437/nice-view-gem` (MIT)
- ZMK snippets: MIT
- QuinqueFive font: **SIL OFL 1.1**
