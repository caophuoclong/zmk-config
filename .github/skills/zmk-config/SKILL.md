---
name: zmk-config
description: Develop and modify ZMK firmware configuration for the Sofle split keyboard (Nice Nano v2). Covers Kconfig (.conf), Devicetree keymap (.keymap), build matrix (build.yaml), and module management (west.yml).
---

# zmk-config

Instructions for developing and modifying ZMK firmware configuration files for this Sofle split keyboard.

## When to use

- Modifying the keyboard layout or layers in `config/sofle.keymap`
- Changing system settings (Bluetooth, RGB, power, debounce, etc.) in `config/sofle.conf`
- Adding new layers, combos, macros, or behaviors
- Configuring the build matrix in `build.yaml`
- Adding external modules or shields via `config/west.yml`
- Updating the visual keymap reference in `KEYMAP_VISUAL.md`

## Workspace Structure

```
build.yaml              # GitHub Actions build matrix (boards + shields)
KEYMAP_VISUAL.md        # Human-readable visual keymap reference
config/
  sofle.conf            # Kconfig settings (system-level config)
  sofle.keymap          # Devicetree keymap (layers, bindings, behaviors)
  west.yml              # West manifest (ZMK source + external modules)
boards/shields/         # Custom shield definitions (currently empty)
zephyr/module.yml       # Declares this repo as a Zephyr module
```

## Hardware Context

- **Keyboard:** Sofle (split, 5x6 + 5 thumb keys per side)
- **Controller:** Nice Nano v2 (nRF52840, Bluetooth + USB)
- **Display:** Nice OLED (via zmk-nice-oled module from mctechnology17)
- **Encoders:** EC11 rotary encoders (one per side)
- **LEDs:** WS2812 RGB underglow
- **Total keys per side:** 30 (6 columns × 4 rows + 5 thumb + 1 encoder button)

## Configuration System Overview

ZMK configuration uses two file types:

### 1. Kconfig Files (`.conf`)

Kconfig controls **global system settings**. The file `config/sofle.conf` applies to both halves of the split keyboard.

**Syntax:** One `CONFIG_<NAME>=<value>` per line, no spaces around `=`.

**Value types:**

- `bool`: `y` (enable) or `n` (disable). Example: `CONFIG_ZMK_SLEEP=y`
- `int`: Integer value. Example: `CONFIG_ZMK_IDLE_TIMEOUT=180000`
- `string`: Double-quoted text. Example: `CONFIG_ZMK_KEYBOARD_NAME="MySofle"`

**Important notes:**

- All configuration is compile-time. After changes, firmware must be rebuilt and flashed.
- For split keyboards, a single `sofle.conf` (without `_left`/`_right` suffix) applies to both halves.
- Changing `CONFIG_ZMK_KEYBOARD_NAME` requires clearing stored settings on the controller.
- `CONFIG_BT_MAX_CONN` and `CONFIG_BT_MAX_PAIRED` should always match; on split keyboards set to desired BT profiles + 1.

**Common Kconfig categories and settings:**

#### General

| Setting                     | Type   | Description                       | Default |
| --------------------------- | ------ | --------------------------------- | ------- |
| `CONFIG_ZMK_KEYBOARD_NAME`  | string | BT advertised name (max 16 chars) | —       |
| `CONFIG_ZMK_WPM`            | bool   | Enable words-per-minute tracking  | n       |
| `CONFIG_HEAP_MEM_POOL_SIZE` | int    | Heap memory pool size             | 8192    |

#### HID

| Setting                               | Type | Description                               | Default |
| ------------------------------------- | ---- | ----------------------------------------- | ------- |
| `CONFIG_ZMK_HID_INDICATORS`           | bool | Enable HID/LED indicator state from host  | n       |
| `CONFIG_ZMK_HID_CONSUMER_REPORT_SIZE` | int  | Simultaneous consumer keys                | 6       |
| `CONFIG_ZMK_HID_REPORT_TYPE_HKRO`     | bool | H-key rollover (default)                  | —       |
| `CONFIG_ZMK_HID_REPORT_TYPE_NKRO`     | bool | Full N-key rollover (may break some BIOS) | —       |

#### USB

| Setting                           | Type | Description                                 | Default |
| --------------------------------- | ---- | ------------------------------------------- | ------- |
| `CONFIG_ZMK_USB`                  | bool | Enable USB keyboard                         | —       |
| `CONFIG_ZMK_USB_BOOT`             | bool | USB Boot protocol (for BitLocker/FileVault) | n       |
| `CONFIG_USB_HID_POLL_INTERVAL_MS` | int  | USB polling interval in ms                  | 1       |

#### Bluetooth

| Setting                               | Type | Description                            | Default |
| ------------------------------------- | ---- | -------------------------------------- | ------- |
| `CONFIG_BT_MAX_CONN`                  | int  | Max simultaneous BT connections        | 5       |
| `CONFIG_BT_MAX_PAIRED`                | int  | Max paired BT devices                  | 5       |
| `CONFIG_ZMK_BLE`                      | bool | Enable BLE keyboard                    | —       |
| `CONFIG_ZMK_BLE_CLEAR_BONDS_ON_START` | bool | Clear bonds on startup                 | n       |
| `CONFIG_ZMK_BLE_PASSKEY_ENTRY`        | bool | Require passkey pairing (experimental) | n       |

#### Power & Sleep

| Setting                         | Type | Description                          | Default |
| ------------------------------- | ---- | ------------------------------------ | ------- |
| `CONFIG_ZMK_SLEEP`              | bool | Enable deep sleep                    | n       |
| `CONFIG_ZMK_IDLE_TIMEOUT`       | int  | Idle timeout in ms before idle state | 30000   |
| `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT` | int  | Idle timeout in ms before deep sleep | 900000  |
| `CONFIG_ZMK_EXT_POWER`          | bool | Enable external power control        | —       |

#### RGB Underglow

| Setting                                  | Type | Description                    | Default |
| ---------------------------------------- | ---- | ------------------------------ | ------- |
| `CONFIG_ZMK_RGB_UNDERGLOW`               | bool | Enable RGB underglow           | n       |
| `CONFIG_ZMK_RGB_UNDERGLOW_HUE_START`     | int  | Starting hue (0-359)           | 0       |
| `CONFIG_ZMK_RGB_UNDERGLOW_SAT_START`     | int  | Starting saturation (0-100)    | 100     |
| `CONFIG_ZMK_RGB_UNDERGLOW_BRT_START`     | int  | Starting brightness (0-100)    | 100     |
| `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE` | bool | Turn off RGB on idle           | n       |
| `CONFIG_WS2812_STRIP`                    | bool | Enable WS2812 LED strip driver | —       |

#### Encoders

| Setting                             | Type | Description                            | Default |
| ----------------------------------- | ---- | -------------------------------------- | ------- |
| `CONFIG_EC11`                       | bool | Enable EC11 rotary encoder             | n       |
| `CONFIG_EC11_TRIGGER_GLOBAL_THREAD` | bool | Use global thread for encoder triggers | n       |

#### Combos

| Setting                               | Type | Description                      | Default |
| ------------------------------------- | ---- | -------------------------------- | ------- |
| `CONFIG_ZMK_COMBO_MAX_COMBOS_PER_KEY` | int  | Max combos that share a key      | 5       |
| `CONFIG_ZMK_COMBO_MAX_KEYS_PER_COMBO` | int  | Max keys in a single combo       | 4       |
| `CONFIG_ZMK_COMBO_MAX_PRESSED_COMBOS` | int  | Max simultaneously active combos | 4       |

#### Logging / Debugging

| Setting                  | Type | Description                | Default |
| ------------------------ | ---- | -------------------------- | ------- |
| `CONFIG_ZMK_USB_LOGGING` | bool | Enable USB CDC ACM logging | n       |
| `CONFIG_ZMK_LOG_LEVEL`   | int  | ZMK log level (0-4)        | 4       |

#### Debounce

| Setting                                | Type | Description                   | Default |
| -------------------------------------- | ---- | ----------------------------- | ------- |
| `CONFIG_ZMK_KSCAN_DEBOUNCE_PRESS_MS`   | int  | Debounce time for key press   | 5       |
| `CONFIG_ZMK_KSCAN_DEBOUNCE_RELEASE_MS` | int  | Debounce time for key release | 5       |

### 2. Devicetree / Keymap Files (`.keymap`)

The keymap file `config/sofle.keymap` defines layers, key bindings, behaviors, combos, macros, and conditional layers using Devicetree syntax.

**Required includes:**

```c
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/rgb.h>
#include <dt-bindings/zmk/ext_power.h>
```

**Layer definition pattern:**

```c
#define BASE 0
#define LOWER 1
#define RAISE 2

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            display-name = "default";
            bindings = <
                // key bindings here, one per physical key
            >;
            sensor-bindings = <&inc_dec_kp C_VOL_UP C_VOL_DN &inc_dec_kp PG_UP PG_DN>;
        };
    };
};
```

**Key binding syntax:**

- Simple key: `&kp <KEYCODE>` (e.g., `&kp A`, `&kp N1`, `&kp LSHFT`)
- Layer momentary: `&mo <LAYER_NUM>` (e.g., `&mo 1`)
- Layer toggle: `&tog <LAYER_NUM>`
- Transparent: `&trans` (passes through to lower layer)
- None: `&none` (blocks key)
- Bluetooth: `&bt BT_CLR`, `&bt BT_SEL 0`
- RGB: `&rgb_ug RGB_TOG`, `&rgb_ug RGB_HUI`, `&rgb_ug RGB_BRI`
- External power: `&ext_power EP_TOG`
- Encoder: `&inc_dec_kp <CW_KEY> <CCW_KEY>`

**Conditional layers:**

```c
conditional_layers {
    compatible = "zmk,conditional-layers";
    adjust_layer {
        if-layers = <LOWER RAISE>;
        then-layer = <ADJUST>;
    };
};
```

**Combos:**

```c
combos {
    compatible = "zmk,combos";
    combo_name {
        timeout-ms = <50>;
        key-positions = <POS1 POS2>;
        bindings = <&kp KEYCODE>;
        layers = <LAYER_NUM>;  // optional: restrict to specific layers
    };
};
```

**Macros:**

```c
macros {
    macro_name: macro_name {
        compatible = "zmk,behavior-macro";
        #binding-cells = <0>;
        bindings = <&kp A &kp B &kp C>;
    };
};
```

**Tap-dance:**

```c
behaviors {
    td_example: td_example {
        compatible = "zmk,behavior-tap-dance";
        #binding-cells = <0>;
        tapping-term-ms = <200>;
        bindings = <&kp A>, <&kp B>;
    };
};
```

**Hold-tap (home row mods):**

```c
behaviors {
    hm: homerow_mods {
        compatible = "zmk,behavior-hold-tap";
        #binding-cells = <2>;
        tapping-term-ms = <200>;
        quick-tap-ms = <150>;
        flavor = "tap-preferred";
        bindings = <&kp>, <&kp>;
    };
};
```

**Changing existing node properties (overlay pattern):**

```c
&kscan0 {
    debounce-press-ms = <0>;
};
```

### 3. Build Configuration (`build.yaml`)

Defines the build matrix for GitHub Actions firmware compilation.

```yaml
include:
  - board: nice_nano_v2
    shield: sofle_left nice_oled # space-separated for multiple shields
  - board: nice_nano_v2
    shield: sofle_right nice_oled
```

Optional properties per build: `snippet`, `cmake-args`, `artifact-name`.

### 4. West Manifest (`config/west.yml`)

Manages ZMK source and external module dependencies.

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
  self:
    path: config
```

To add an external module (e.g., custom shield, display driver):

1. Add a new remote with the GitHub org/user URL base.
2. Add a project entry with name, remote, and revision.

### 5. Snippets

Snippets apply common configuration for specific board variants. Enable via `build.yaml`:

```yaml
- board: nrfmicro@1.3.0/nrf52833
  snippet: nrf52833-nosd
  shield: corne_left
```

**WARNING:** `nrf52833-nosd` and `nrf52840-nosd` snippets erase the Nordic SoftDevice. This can brick the board if later flashed with firmware built without the snippet.

## Instructions for Making Changes

### Modifying Key Bindings

1. Open `config/sofle.keymap`.
2. Locate the target layer by its `display-name`.
3. Edit the `bindings` array — each entry maps to a physical key position.
4. Key positions are ordered left-to-right, top-to-bottom, matching the `kscan` matrix.
5. The Sofle has 60 keys total (30 per side): 6 columns × 4 rows + 5 thumb keys per side.
6. After modifying bindings, update `KEYMAP_VISUAL.md` to reflect the changes.

### Adding a New Layer

1. Define a new layer number: `#define NEWLAYER 4`.
2. Add a new block inside the `keymap` node with `display-name` and `bindings`.
3. Add a way to activate the layer (e.g., `&mo NEWLAYER` on an existing layer).
4. If using encoders, include `sensor-bindings`.
5. Update `KEYMAP_VISUAL.md`.

### Changing System Settings

1. Open `config/sofle.conf`.
2. Add or modify `CONFIG_<NAME>=<value>` lines.
3. Use comments (`#`) to document non-obvious settings.
4. Remember: changes require firmware rebuild. BT name changes require clearing stored settings.

### Adding a Combo

1. Add a `combos` node inside the root `/` node in `sofle.keymap` (if not already present).
2. Define each combo with `timeout-ms`, `key-positions`, and `bindings`.
3. Ensure `CONFIG_ZMK_COMBO_MAX_COMBOS_PER_KEY` in `sofle.conf` is sufficient.

### Adding a Macro

1. Add a `macros` node inside the root `/` node in `sofle.keymap`.
2. Define the macro with `compatible = "zmk,behavior-macro"`.
3. Reference the macro in a layer binding: `&macro_name`.

### Adding an External Module

1. Edit `config/west.yml` to add the remote and project.
2. If needed, update `build.yaml` to reference new shields from the module.

## Current Keymap Summary

This keyboard uses 4 layers with conditional layer activation:

| Layer | Name   | Purpose                           | Activation                        |
| ----- | ------ | --------------------------------- | --------------------------------- |
| 0     | BASE   | Standard QWERTY layout            | Default                           |
| 1     | LOWER  | F-keys, symbols, brackets         | Hold left thumb `mo 1`            |
| 2     | RAISE  | Navigation, arrows, BT management | Hold right thumb `mo 2`           |
| 3     | ADJUST | RGB controls, BT clear, ext power | Hold LOWER + RAISE simultaneously |

**Encoder mappings (all layers):** Left = Volume Up/Down, Right = Page Up/Down.

## Reference Links

- ZMK Configuration Overview: https://zmk.dev/docs/config
- System Configuration: https://zmk.dev/docs/config/system
- Keymap Configuration: https://zmk.dev/docs/config/keymap
- Behaviors: https://zmk.dev/docs/keymaps/behaviors
- Key Codes: https://zmk.dev/docs/codes
- Bluetooth: https://zmk.dev/docs/features/bluetooth
- RGB Underglow: https://zmk.dev/docs/features/underglow
- Combos: https://zmk.dev/docs/keymaps/combos
- Encoders: https://zmk.dev/docs/config/encoders
- New Shield Guide: https://zmk.dev/docs/development/hardware-integration/new-shield
- Modules: https://zmk.dev/docs/features/modules
