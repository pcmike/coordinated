# Coordinated Cooler Control Package v6.1

A Home Assistant package that coordinates AC and standalone dehumidifier control based on humidity levels.

## How It Works

### Humidity Control
1. **Normal operation**: Thermostat runs at your chosen temperature
2. **High humidity (>49%)**: Lowers to floor temperature (default 70°F) to dehumidify using AC
3. **AC can't clear humidity**: If temperature reaches floor but humidity persists, standalone dehumidifier activates
4. **Humidity normal (<47%)**: Returns to previous temperature, turns off standalone dehumidifier

### Day/Night Scheduling
1. **Day mode**: At configured time or sunrise, sets thermostat to day temperature
2. **Night mode**: At configured time or sunset, sets thermostat to night temperature
3. **Sun tracking**: Optionally use sunrise/sunset with adjustable offset (-120 to +120 minutes)
4. **Humidity priority**: Schedule skips if humidity control is active
5. **Toggle**: Can be disabled entirely via dashboard

## Features

- Automatic coordination between AC and standalone dehumidifier
- Smart humidity alerts: only notifies if humidity isn't clearing after 45 minutes
- Day/night temperature scheduling with humidity-aware logic
- Sunrise/sunset tracking with configurable offset
- Manual override detection with suppression to prevent flapping
- Startup sync to handle Home Assistant restarts gracefully
- All events logged for debugging (filtered by "Coordinated:" prefix)
- Dashboard cards with expandable settings and real-time status

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

### Humidity Control

| Setting | Default | Range | Description |
|---------|---------|-------|-------------|
| `coordinated_humidity_enabled` | On | On/Off | Enable/disable automatic humidity control |
| `coordinated_humidity_on_threshold` | 49% | 40-70% | Humidity level that triggers dehumidification |
| `coordinated_humidity_off_threshold` | 47% | 35-65% | Humidity level that exits dehumidification |
| `coordinated_humidity_floor_temp` | 70°F | 65-75°F | Temperature to cool to when dehumidifying |

### Day/Night Schedule

| Setting | Default | Description |
|---------|---------|-------------|
| `coordinated_schedule_enabled` | On | Enable/disable day/night automation |
| `coordinated_use_sun` | On | Use sunrise/sunset instead of fixed times |
| `coordinated_day_temp` | 76°F | Temperature during daytime hours |
| `coordinated_night_temp` | 73°F | Temperature during nighttime hours |
| `coordinated_day_sun_offset` | 0 min | Minutes before (-) or after (+) sunrise |
| `coordinated_night_sun_offset` | -30 min | Minutes before (-) or after (+) sunset |
| `coordinated_day_start` | 6:00 AM | Time to switch to day temperature (when using fixed times) |
| `coordinated_night_start` | 9:00 PM | Time to switch to night temperature (when using fixed times) |

**Note:** All settings persist across Home Assistant restarts. Defaults are only applied on first installation. These can also be adjusted via the expandable settings in the dashboard card.

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

### Humidity Control
1. **Humidity High**: Triggers dehumidification mode when humidity exceeds ON threshold
2. **Humidity Normal**: Exits dehumidification mode when humidity drops below OFF threshold
3. **Standalone Fallback**: Activates standalone dehumidifier when AC reaches floor temp but humidity persists
4. **Manual Override**: Detects manual changes and exits humidity mode with suppression
5. **Clear Suppression**: Re-enables humidity control after humidity drops below threshold
6. **Startup Sync**: Synchronizes state after Home Assistant restart, including setting correct day/night temperature based on schedule

### Day/Night Schedule
7. **Day Mode**: Switches to day temperature at scheduled time or sunrise (skips if humidity control active)
8. **Night Mode**: Switches to night temperature at scheduled time or sunset (skips if humidity control active)

## Notifications & Logging

All events are logged at WARNING level with "Coordinated:" prefix for easy filtering.

**Critical phone notifications (iOS)** are sent only when there's an actual problem:

| Alert | When | Message Variants |
|-------|------|------------------|
| 🚨 Humidity Not Clearing | 45 min after trigger, humidity hasn't dropped at least 3% | "Humidity at X% after 45 min (started at Y%). [AC cooling to 70°F / Standalone dehumidifier running / Both AC and standalone running] but not clearing effectively." |

**The notification intelligently reports what's actually running** so you know the system status at a glance.

**Examples:**
- Trigger at 52%, drops to 48% in 45 min → No notification (dropped 4% ≥ 3%)
- Trigger at 52%, still at 50.5% in 45 min → Notification (dropped only 1.5%)
- Standalone activates automatically at 30 seconds or 30 minutes → No notification (working as designed)
- Humidity clears before 45 min → No notification

## License

MIT
