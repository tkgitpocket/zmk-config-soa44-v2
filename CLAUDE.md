# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

ZMK firmware configuration for the SOA44 — a 44-key wireless split keyboard with a PMW3610 trackball on the right half and WS2812 LEDs on both halves. Built on XIAO BLE (nRF52840) controllers.

## Building the Firmware

Firmware is built via GitHub Actions — there is no local build step. Push to the `main` branch (or trigger `workflow_dispatch`) to kick off `.github/workflows/build.yml`, which uses the standard ZMK user-config workflow.

Three artifacts are produced per build:
- `soa44_R` — right half (central), built with `snippet: studio-rpc-usb-uart` for ZMK Studio support
- `soa44_L` — left half (peripheral)
- `settings_reset` — resets ZMK settings flash

The keymap diagram in `keymap-drawer/` is updated by a separate workflow (`draw.yml`), which must be triggered manually via `workflow_dispatch`.

## File Layout

```
config/
  soa44.keymap       # Main keymap: all layers, combos, macros, behaviors
  layers.dtsi        # Layer number #defines shared by keymap and R overlay
  west.yml           # ZMK module dependencies (external drivers/modules)

boards/shields/soa44/
  soa44.dtsi         # Shared hardware: matrix transform, kscan, LED strip, battery
  soa44_L.overlay    # Left half: col-gpios, SPI1 pin-control for LED
  soa44_R.overlay    # Right half: col-gpios, SPI0 for trackball, SPI1 for LED, layer input-processors
  soa44_L.conf       # Left half Kconfig: BLE, battery, WS2812 widget settings
  soa44_R.conf       # Right half Kconfig: PMW3610 driver, ZMK Studio, JIS layout shift
  Kconfig.defconfig  # Sets KEYBOARD_NAME and SPLIT/CENTRAL roles
  Kconfig.shield     # Shield selection symbols (SHIELD_SOA44_R / SHIELD_SOA44_L)
```

## External ZMK Modules (`config/west.yml`)

| Module | Purpose |
|---|---|
| `zmk` (zmkfirmware/zmk@main) | Core ZMK firmware |
| `zmk-feature-non-lipo-battery-management` (sekigon-gonnoc) | Ni-MH battery support with custom ADC voltage curves |
| `zmk-pmw3610-driver` (badjeff) | PMW3610 trackball sensor driver |
| `zmk-ws2812-driver` (gohanda11) | WS2812 LED widget (layer color, battery level, BT status) |
| `zmk-scroll-snap` (kot149@v1) | Scroll-snap input processor for smooth scrolling |
| `zmk-layout-shift` (kot149@v1) | JIS keyboard layout shift (overrides `&kp` behavior) |

## Architecture Notes

### Split roles
- **Right half = central** (`ZMK_SPLIT_ROLE_CENTRAL=y`). It processes the trackball, runs ZMK Studio, and proxies the left half's battery level.
- **Left half = peripheral**. Has no trackball; its LED shows the same status colors as the right.

### Trackball input processing (`soa44_R.overlay`)
Layer-conditional input-processors are defined on `trackball_listener`:
- `SLOWTRACKBALL_L` → `zip_xy_scaler 1 4` (1/4 speed)
- `SCROLL_L` → `zip_xy_to_scroll_mapper` + snap + Y-invert + `1/8` speed
- `FASTRSCROLL_L` → same pipeline at `1/4` speed

### Battery monitoring
Both halves use a custom `zmk,non-lipo-battery` node (not the standard LiPo). The ADC sees a voltage-divided Ni-MH cell:
- Divider: R2=1MΩ, R3=470kΩ → ratio ≈ 0.32
- `ZMK_NON_LIPO_MIN_MV=352` (→ 1100mV actual, 0%), `MAX=448` (→ 1400mV, 100%)

### WS2812 LED widget
Both halves enable `CONFIG_WS2812_WIDGET`. Colors and timing are configured in the `.conf` files. Right half additionally defines per-layer persistent colors (`LAYER_0_COLOR` … `LAYER_6_COLOR`). LED power is gated by a P-ch MOSFET on P0.15 (`led_power` node in `soa44.dtsi`).

### JIS layout
The right half enables `CONFIG_LAYOUT_SHIFT_TARGET_JIS=y` (zmk-layout-shift module). The keymap overrides `&kp` with `zmk,behavior-layout-shift-key-press` and keeps the original as `&original_key_press` for ZMK internal use. JIS key aliases (e.g. `JP_DQUOTE`, `JP_LBRACE`) are defined at the top of `config/soa44.keymap`.

### Layers
Defined in `config/layers.dtsi` and referenced by both the keymap and `soa44_R.overlay`:

| # | Name | Activated by |
|---|---|---|
| 0 | DEFAULT | base |
| 1 | WINDOWS_L | (reserved) |
| 2 | MAC_L | (reserved) |
| 3 | IOS_L | (reserved) |
| 4 | LEFT_L | hold left thumb `INT_HENKAN` |
| 5 | RIGHT_L | hold right thumb `INT_MUHENKAN` |
| 6 | TRACKBALL_L | auto-mouse (unused in current keymap) |
| 7 | SLOWTRACKBALL_L | hold bottom-right key |
| 8 | SCROLL_L | hold `MINUS` key |
| 9 | FASTRSCROLL_L | hold `JP_UNDERSCORE` key |
| 10 | CENTER_L | hold center `SPACE` |
| 11 | CENTER2_L | (mac center, currently same bindings) |
| 12 | BT_L | hold `SPACE` inside CENTER_L |

### Bluetooth connection macros
`out_bt_0` … `out_bt_4` in the keymap disconnect all profiles then select one, ensuring a clean single-device connection rather than using the plain `BT_SEL` behavior.
