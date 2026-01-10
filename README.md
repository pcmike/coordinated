# Coordinated Cooler Control Package v1.0

A Home Assistant package that coordinates AC and standalone dehumidifier control based on humidity levels.

## How It Works

1. **Normal operation**: Thermostat runs at your chosen temperature
2. **High humidity (>50%)**: Lowers to floor temperature (default 70°F) to dehumidify using AC
3. **AC can't clear humidity**: If temperature reaches floor but humidity persists, standalone dehumidifier activates
4. **Humidity normal (<47%)**: Returns to previous temperature, turns off standalone dehumidifier

## Features

- Automatic coordination between AC and standalone dehumidifier
- Manual override detection with suppression to prevent flapping
- Startup sync to handle Home Assistant restarts gracefully
- Mobile notifications for state changes
- Dashboard cards with expandable settings

## Files

| File | Description |
|------|-------------|
| `coordinated.yaml` | Main package with all automations, entities, and climate config |
| `coordinated_cards.yaml` | Combined dashboard card with expandable settings |
| `coordinated_temperature_card.yaml` | Standalone temperature card |
| `coordinated_humidity_card.yaml` | Standalone humidity card |
| `coordinated_graph.yaml` | ApexCharts graph showing temperature, humidity, and device states |

## Installation

1. Copy `coordinated.yaml` to your Home Assistant packages directory
2. Add to `configuration.yaml`:
   ```yaml
   homeassistant:
     packages:
       coordinated: !include packages/coordinated.yaml
   ```
3. Restart Home Assistant
4. Add dashboard cards using the YAML from the card files

## Configuration

All user-configurable options are at the top of `coordinated.yaml`:

| Setting | Default | Range | Description |
|---------|---------|-------|-------------|
| `coordinated_humidity_on_threshold` | 50% | 40-70% | Humidity level that triggers dehumidification |
| `coordinated_humidity_off_threshold` | 47% | 35-65% | Humidity level that exits dehumidification |
| `coordinated_humidity_floor_temp` | 70°F | 65-75°F | Temperature to cool to when dehumidifying |

These can also be adjusted via the expandable settings in the dashboard card.

## Dashboard Cards

### Combined Card (Recommended)
Use `coordinated_cards.yaml` for a compact view with:
- Humidity and temperature display side by side
- Tap to expand settings sliders
- Color-coded icons indicating current state

### Separate Cards
Use `coordinated_temperature_card.yaml` and `coordinated_humidity_card.yaml` if you prefer individual cards.

### Graph
Use `coordinated_graph.yaml` for a 24-hour view showing:
- Temperature line (blue)
- Humidity line (green)
- AC running for temperature (light blue shading)
- AC running for humidity (light orange shading)
- Standalone dehumidifier running (light red shading)

## Icon Colors

**Temperature Card:**
- Grey: Climate off
- Orange: Humidity control active
- Blue: Normal cooling

**Humidity Card:**
- Grey: Idle
- Orange: AC running for humidity
- Red: Standalone dehumidifier running

## Required Entities

Update these in `coordinated.yaml` to match your setup:
- `switch.meross_cool_switch` - AC control switch
- `sensor.third_reality_inc_3rths24bz_temperature` - Temperature sensor
- `sensor.max_indoor_humidity_sensor` - Humidity sensor
- `humidifier.151732605035846_humidifier` - Standalone dehumidifier
- `notify.mobile_app_midnight` - Mobile notification service

## Required Custom Cards

- [Mushroom Cards](https://github.com/piitaya/lovelace-mushroom)
- [ApexCharts Card](https://github.com/RomRider/apexcharts-card)
- [Expander Card](https://github.com/Alia5/lovelace-expander-card)

## Automations

1. **Humidity High**: Triggers dehumidification mode when humidity exceeds ON threshold
2. **Humidity Normal**: Exits dehumidification mode when humidity drops below OFF threshold
3. **Standalone Fallback**: Activates standalone dehumidifier when AC can't reduce humidity further
4. **Manual Override**: Detects manual changes and exits humidity mode with suppression
5. **Clear Suppression**: Re-enables humidity control after humidity drops below threshold
6. **Startup Sync**: Synchronizes state after Home Assistant restart

## License

MIT
