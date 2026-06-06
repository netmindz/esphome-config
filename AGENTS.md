# AGENTS.md - ESPHome Devices Repository

## Project Overview

This is a collection of ESPHome YAML configuration files defining firmware for
ESP32/ESP8266 microcontrollers integrated with Home Assistant. There is no
traditional application code -- the project is YAML configs with embedded C++
lambdas for custom logic.

## Build Commands

ESPHome CLI is the only build tool. There are no Makefiles, package managers,
or CI pipelines.

```bash
# Validate a config without compiling
esphome config <device>.yaml

# Compile firmware for a specific device
esphome compile <device>.yaml

# Upload previously compiled firmware to a device (OTA)
esphome upload <device>.yaml

# Compile and upload in one step (preferred for deploying changes)
esphome compile <device>.yaml && esphome upload <device>.yaml

# Compile, upload, and stream logs (use only when logs are needed)
esphome run <device>.yaml

# View device logs without uploading
esphome logs <device>.yaml
```

There are no lint, test, or formatting commands. Validation is done via
`esphome config`.

## Validation

**Always run `esphome config` after making changes** to any device or shared YAML file.
Display devices that share `common-display.yaml` must both be validated:

```bash
esphome config kitchen-display.yaml
esphome config kitchen-full-display.yaml
```

A successful validation prints the resolved config without a `Failed config` block.
Warnings about strapping pins are expected and harmless.

## Directory Structure

```
├── common-display.yaml      # Shared display config (included by display devices)
├── common-sensors.yaml      # Shared sensor definitions (HA entities, energy, etc.)
├── kitchen-display.yaml     # WT32-SC01-Plus touchscreen display
├── beoplay.yaml             # Bang & Olufsen speaker (ES8388 DAC)
├── hottub-esp32.yaml        # Hot tub controller variants
├── cyd-01.yaml              # Cheap Yellow Display device
├── tv-unit-screen.yaml      # TV unit screen
├── t-display.yaml           # LilyGO T-Display variants
├── wled-bridge.yaml         # WLED bridge/matter devices
├── fonts/                   # Custom fonts (e.g. MemoryIcons-Regular.otf)
├── secrets.yaml             # (gitignored) WiFi/API credentials
└── start.sh                 # NAS mount + symlink helper
```

## Code Style Guidelines

### YAML Conventions

- **File naming**: lowercase with hyphens (`kitchen-display.yaml`, `common-sensors.yaml`)
- **Indentation**: 2 spaces, no tabs
- **Entity/component IDs**: `snake_case` (e.g. `electricity_power`, `backlight_output`)
- **Substitution variables**: `snake_case` (e.g. `device_name`, `friendly_name`)
- **Friendly names**: Title Case with spaces (e.g. `"Kitchen Display - WT32-SC01-Plus"`)

### YAML Import Patterns

Use these mechanisms to share configuration:

```yaml
# Include shared configs via packages
packages:
  common_display: !include common-display.yaml

# Pull external components from GitHub
external_components:
  - source: github://pr#12256
    components: [ audio ]

# Reference secrets (never hardcode credentials)
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# Use substitutions for parameterized values
substitutions:
  device_name: kitchen-display
  friendly_name: "Kitchen Display"
```

### Embedded C++ Lambda Style

Inline C++ is used via `!lambda` for runtime logic in sensors and displays.

```yaml
lambda: |-
  static char buf[32];
  if (isnan(x)) {
    snprintf(buf, sizeof(buf), "--");
  } else {
    snprintf(buf, sizeof(buf), "%.1f", x);
  }
  return std::string(buf);
```

Key patterns:
- **Always check `isnan(x)`** before using sensor values -- display `"--"` for unavailable data
- **Use `static char buf[N]`** with `snprintf` for string formatting (safe, avoids heap allocation)
- **Use C types**: `float`, `int`, `uint8_t`, `uint16_t`, `std::string`
- **Guard string comparisons**: check for empty strings or known states before processing
- **Keep lambdas short** -- complex logic should be in external components, not inline

### Error Handling

- Check `isnan()` on all sensor float values before use
- Display placeholder text (`"--"`) when data is unavailable
- Check string state values (e.g. `"Ready"`, `"Inactive"`, `""`) before acting on them
- No try/catch -- this is embedded C++, keep error handling simple and defensive

### Secrets Management

- **Never commit credentials** -- all sensitive values go in `secrets.yaml` (gitignored)
- Reference secrets with `!secret <key_name>`
- Required secrets include: `wifi_ssid`, `wifi_password`, `api_password`, `ota_password`

## Common Shared Components

### common-sensors.yaml
Shared Home Assistant sensor definitions (electricity, gas, energy prices,
dishwasher state). Included by display devices that show this data.

### common-display.yaml
Shared display configuration for LVGL-based screens. Contains layout,
widget definitions, and sensor display logic.

## Adding a New Device

1. Create `<device-name>.yaml` at the project root (lowercase, hyphenated)
2. Define `substitutions:` for `device_name` and `friendly_name`
3. Include common configs with `packages: !include` where applicable
4. Add `wifi:`, `api:`, `ota:` sections referencing `!secret` values
5. Validate with `esphome config <device-name>.yaml`
6. Compile and flash with `esphome compile <device-name>.yaml && esphome upload <device-name>.yaml`

## Important Notes

- The `.esphome/` directory is auto-generated build cache -- never edit it
- External components are cached in `.esphome/external_components/`
- LVGL is used for display UIs on touchscreen devices
- `start.sh` mounts a NAS share and creates symlinks -- run it on the host machine
- No CI/CD exists; validation is manual via `esphome config`
