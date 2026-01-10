# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Coordinated Cooler Control System v4.3** - A Home Assistant package that intelligently coordinates a single AC unit between temperature and humidity control, with automatic fallback to a standalone dehumidifier.

### Core Problem Solved
Prevents AC from overcooling while chasing humidity, handles post-shower humidity spikes quickly, and protects compressor from short-cycling.

## File Structure

| File | Purpose |
|------|---------|
| `coordinated.yaml` | Main HA package: automations, entities, input_numbers, climate config |
| `coordinated_sensors.yaml` | Helper template sensors and dashboard collapse state booleans |
| `coordinated_dashboard.yaml` | Mushroom card dashboard configuration |
| `README.md` | User documentation and configuration guide |
| `TESTING.md` | Validation test procedures |

## Architecture

```
Physical Layer:  AC (switch.meross_cool_switch) + Standalone Dehumidifier (humidifier.*)
       ↑
Logical Layer:   input_boolean.coordinated_cooling_temp_request (OR logic)
                 input_boolean.coordinated_cooling_hum_request
                 Day/Night floor calculation, Critical humidity override
       ↑
UI Layer:        climate.coordinated_indoor_thermostat (generic_thermostat)
                 input_number.* sliders for all thresholds
```

### Key Logic
- **AC ON:** temp_request OR hum_request (either demands cooling)
- **AC OFF:** NOT temp_request AND NOT hum_request (both satisfied)
- **Day mode (6AM-9PM):** Flat 70°F floor for aggressive dehumidification
- **Night mode (9PM-6AM):** Calculated floor based on setpoint for comfort
- **Critical humidity (≥55%):** Both AC and standalone run together

## Configuration Points

Entity IDs must be configured in **two places** in `coordinated.yaml`:
1. `input_text` section (~line 57) - for automations
2. `climate` section (~line 411) - for generic_thermostat `target_sensor`

Key configurable thresholds (all via UI after install):
- `coordinated_target_humidity` (default: 49%)
- `coordinated_critical_humidity` (default: 55%)
- `coordinated_day_floor_temp` (default: 70°F)
- `coordinated_emergency_temp` (default: 66°F)
- `coordinated_day_mode_start` / `coordinated_night_mode_start`

## Installation

1. Copy YAML files to `/config/packages/`
2. Add to `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Edit entity IDs in `coordinated.yaml` (both places)
4. Restart Home Assistant
5. Add dashboard from `coordinated_dashboard.yaml`

## Dashboard Requirements

Uses custom Lovelace cards:
- `custom:mushroom-template-card`
- `custom:mushroom-chips-card`
- `custom:mushroom-climate-card`
- `custom:mushroom-number-card`
- `custom:mushroom-entity-card`
- `custom:stack-in-card`
- `card-mod` (for styling)

## Key Automations

| Automation ID | Purpose |
|--------------|---------|
| `initial_sync_coordinated_cooler` | Startup recovery after HA restart |
| `invoke_coordinated_cooler_actions` | Core OR/AND coordination logic |
| `sync_real_cooler_to_proxies` | Manual override detection |
| `unified_humidity_control_with_temp_floor` | Day/night floor logic + critical response |
| `standalone_dehumidifier_fallback` | Dehumidifier takeover when AC blocked |
| `emergency_shutoff_coordinated` | Safety protection at 66°F |

## Testing

Run tests from `TESTING.md`. Key scenarios:
- Day mode allows AC at 70°F floor
- Night mode uses calculated floor
- Critical humidity (≥55%) activates both devices
- Emergency shutoff at configured threshold
- 15-minute anti-short-cycle timer
