# Coordinated Cooler Control System

**Version 4.3** | Advanced HVAC coordination with Day/Night Floor Logic

---

## Table of Contents

- [Overview](#overview)
- [What's New in v4.3](#whats-new-in-v43)
- [Features](#features)
- [How It Works](#how-it-works)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)
- [Dashboard Setup](#dashboard-setup)
- [Troubleshooting](#troubleshooting)
- [Advanced Tuning](#advanced-tuning)
- [FAQ](#faq)

---

## Overview

This package intelligently coordinates a single AC unit between **temperature control** and **humidity control**, with automatic fallback to a **standalone dehumidifier** when indoor temperatures make AC-based dehumidification inefficient.

**NEW IN v4.3:** Smart day/night temperature floor logic and critical humidity response for real-world scenarios like post-shower humidity spikes.

### The Problem It Solves

**Without coordination:**
- AC runs for temperature, stops, then humidity rises
- AC runs for humidity, overcools the house trying to dehumidify
- In cool weather, AC makes house freezing while chasing humidity
- Post-shower humidity spikes aren't cleared fast enough (mold risk)
- Short-cycling damages compressor

**With this system:**
- ✅ AC activates if **either** temperature **or** humidity demands it
- ✅ AC only stops when **both** are satisfied
- ✅ **Day mode** (6 AM - 9 PM): Aggressive 70°F floor for rapid humidity clearing
- ✅ **Night mode** (9 PM - 6 AM): Calculated floor based on setpoint for comfort
- ✅ **Critical humidity mode** (≥55%): Both AC and standalone run together
- ✅ Standalone dehumidifier takes over when house is too cold for AC
- ✅ 15-minute anti-short-cycle protection
- ✅ Emergency shutoff protection against dangerous overcooling
- ✅ Comprehensive logging and notifications

---

## What's New in v4.3

### Day/Night Temperature Floor Modes

**Problem:** Morning shower creates 59% humidity spike. House is 73°F, thermostat set to 76°F. Old system calculated floor of 74.2°F, blocking AC. Only slow standalone could run - not fast enough to prevent mold risk.

**Solution:**
- **Day Mode (6 AM - 9 PM):** Flat 70°F floor allows aggressive AC dehumidification
  - Handles shower/cooking/laundry humidity spikes quickly
  - Thermostat can be set to 76°F, but AC still allowed down to 70°F for humidity
  - Nobody gets uncomfortable from a few degrees of cooling
  
- **Night Mode (9 PM - 6 AM):** Calculated floor based on thermostat setpoint
  - Prevents overcooling during sleep
  - Prioritizes comfort over aggressive dehumidification
  - Standalone handles moderate humidity without waking you up cold

### Critical Humidity Response

**New threshold** (default 55%, configurable):
- Above this level, system enters **emergency mode**
- AC runs if temperature allows (even if below normal restrictions)
- Standalone **ALWAYS** runs (even if AC is already running)
- **Both devices work together** for maximum dehumidification power
- Prevents mold growth from large humidity spikes

### User-Configurable Settings

All new features are adjustable via UI (no YAML editing):
- **Day mode start hour** (default: 6 AM)
- **Night mode start hour** (default: 9 PM)
- **Day floor temperature** (default: 70°F)
- **Critical humidity threshold** (default: 55%)

---

## Features

### Core Functionality
- **Dual Control**: Temperature OR humidity can activate AC (OR logic)
- **Smart Coordination**: AC only stops when BOTH temp and humidity are satisfied (AND logic)
- **Day/Night Temperature Floor**: 
  - Day: Permissive flat floor (70°F) for fast humidity clearing
  - Night: Calculated floor based on setpoint for sleep comfort
- **Critical Humidity Response**: Dual-device mode for emergency dehumidification
- **Standalone Fallback**: Automatic handoff when AC would overcool
- **Anti-Short-Cycle**: 15-minute minimum off time protects compressor
- **Manual Override Detection**: Respects manual AC control
- **Startup Recovery**: Correctly resumes state after Home Assistant restarts
- **Emergency Shutoff**: Safety protection if temperature drops below 66°F

### User Experience
- **Dashboard Control**: Single thermostat card + humidity slider
- **Mobile Notifications**: Alerts for humidity events and critical modes
- **Comprehensive Logging**: All actions logged at WARNING level with emoji filtering
- **Master On/Off Switch**: Disable entire system for maintenance
- **Fully Configurable**: All key parameters adjustable via UI

---

## How It Works

### Architecture Overview
```
┌─────────────────────────────────────────────────────────────┐
│                    PHYSICAL LAYER                            │
│  • AC Unit (via smart switch)                                │
│  • Standalone Dehumidifier (5-gal portable)                  │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │
┌─────────────────────────────────────────────────────────────┐
│                    LOGICAL LAYER                             │
│  • Temperature Request Boolean (on/off)                      │
│  • Humidity Request Boolean (on/off)                         │
│  • Day/Night Floor Logic (time-based)                        │
│  • Critical Humidity Override (emergency mode)               │
│  • OR Logic: AC ON if either is on                           │
│  • AND Logic: AC OFF if both are off                         │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │
┌─────────────────────────────────────────────────────────────┐
│                 USER INTERFACE LAYER                         │
│  • Generic Thermostat (temp control)                         │
│  • Input Number Sliders (humidity targets, thresholds)       │
│  • Unified Humidity Automation (direct control)              │
└─────────────────────────────────────────────────────────────┘
```

### Day/Night Temperature Floor Logic

**The temperature floor is the minimum indoor temperature at which AC-based dehumidification is allowed.**

#### Day Mode (6 AM - 9 PM by default)
```
Floor: 70°F (flat, configurable)

Example scenario:
- Thermostat: 76°F
- Current temp: 73°F
- Humidity: 59% (post-shower spike)

Decision:
✅ 73°F > 70°F floor → AC allowed!
✅ 59% ≥ 55% (critical) → AC + Standalone both run
✅ Maximum dehumidification power
✅ House cools to 70°F if needed (nobody notices 3°F drop)
```

#### Night Mode (9 PM - 6 AM by default)
```
Floor: Calculated based on setpoint

Formula:
  floor = (setpoint - 0.5) - buffer + 0.2
  
  If (setpoint - 0.5) ≥ 75°F: buffer = 1.5°F
  If (setpoint - 0.5) < 75°F:  buffer = 0.5°F

Example scenario:
- Thermostat: 73°F (sleep mode)
- Calculated floor: 72.2°F
- Current temp: 68°F
- Humidity: 52%

Decision:
❌ 68°F < 72.2°F floor → AC blocked (too cold)
✅ Standalone handles moderate humidity
✅ No uncomfortable overcooling during sleep
```

### Critical Humidity Response

**Threshold: 55% (configurable)**

When humidity exceeds this level:
```
NORMAL MODE (humidity 49-54%):
- AC runs if temp > floor
- Standalone runs if AC can't help

CRITICAL MODE (humidity ≥55%):
🚨 AC forced ON (if temp > emergency threshold - 4°F)
🚨 Standalone ALWAYS ON (even if AC running)
🚨 Both devices work together
🚨 Notifications indicate "CRITICAL"
🚨 Maximum dehumidification power
```

### Decision Flow
```
Humidity above target?
  ├─ NO → Nothing to do
  └─ YES → Is it CRITICAL (≥55%)?
      ├─ YES → EMERGENCY MODE
      │   ├─ Check temp > (emergency shutoff + 4°F buffer)?
      │   │   ├─ YES → Force AC ON + Standalone ON
      │   │   └─ NO  → Only Standalone ON (too cold for AC)
      │   └─ Notification: "CRITICAL humidity!"
      └─ NO → NORMAL MODE
          ├─ What time is it?
          │   ├─ DAY (6 AM - 9 PM) → Use 70°F floor
          │   └─ NIGHT (9 PM - 6 AM) → Use calculated floor
          ├─ Is current temp > floor?
          │   ├─ YES → AC ON
          │   └─ NO  → Standalone ON
          └─ Notification: "High humidity"
```

---

## Installation

### Prerequisites

**Required Hardware:**
- Smart switch/relay controlling AC unit
- At least one temperature sensor
- At least one humidity sensor (two recommended for better control)
- Standalone/portable dehumidifier with Home Assistant integration

**Required Software:**
- Home Assistant 2023.6+
- Mobile app for notifications (optional but recommended)

### Installation Steps

1. **Download Files**
   - Download `coordinated.yaml` and `README.md`
   - Place `coordinated.yaml` in `/config/packages/`

2. **Enable Packages in configuration.yaml**
```yaml
   homeassistant:
     packages: !include_dir_named packages
```

3. **Configure Entity IDs**
   
   **⚠️ CRITICAL: Edit entity IDs in TWO places in coordinated.yaml:**
   
   **Place 1: input_text section (~line 32)**
```yaml
   coordinated_master_temp_sensor:
     initial: "sensor.YOUR_TEMP_SENSOR"
   coordinated_master_hum_sensor:
     initial: "sensor.YOUR_HUMIDITY_SENSOR"
   coordinated_max_hum_sensor:
     initial: "sensor.YOUR_MAX_HUMIDITY_SENSOR"
   coordinated_ac_switch:
     initial: "switch.YOUR_AC_SWITCH"
   coordinated_standalone_dehumidifier:
     initial: "humidifier.YOUR_DEHUMIDIFIER"
   coordinated_notification_service:
     initial: "notify.mobile_app_YOUR_DEVICE"
```
   
   **Place 2: climate section (~line 405)**
```yaml
   climate:
     - platform: generic_thermostat
       target_sensor: sensor.YOUR_TEMP_SENSOR  # Must match above
       cold_tolerance: 0.5  # Must match coordinated_cold_tolerance
```

4. **Verify Configuration**
```
   Developer Tools → YAML → Check Configuration
```

5. **Restart Home Assistant**

6. **Verify Entities Created**
```
   Developer Tools → States
   Search: "coordinated"
```

7. **Configure Day/Night Settings** (via UI, no restart needed)
   - Adjust day mode hours if needed (default 6 AM - 9 PM works for most)
   - Adjust day floor temperature (default 70°F)
   - Adjust critical humidity threshold (default 55%)

---

## Configuration

### Sensor Strategy

| Sensor | Purpose | Recommended Source |
|--------|---------|-------------------|
| **Master Temperature** | Thermostat control + temp floor | Bedroom (priority area) |
| **Master Humidity** | Standalone trigger | Bedroom (where standalone located) |
| **Max Humidity** | AC trigger + critical detection | MAX of multiple sensors |

**Creating a max humidity sensor** (recommended for multi-sensor setups):
```yaml
# In configuration.yaml
template:
  - sensor:
      - name: "Max Indoor Humidity"
        state: >
          {{ [
            states('sensor.bedroom_humidity') | float(0),
            states('sensor.living_room_humidity') | float(0),
            states('sensor.bathroom_humidity') | float(0)
          ] | max }}
        unit_of_measurement: "%"
```

### Key Settings (All Adjustable via UI)

#### Humidity Control

| Setting | Default | Purpose |
|---------|---------|---------|
| **Target Humidity** | 49% | Desired humidity level |
| **Wet Tolerance** | 0% | Turn ON offset (0 = immediately at target) |
| **Dry Tolerance** | 3% | Turn OFF offset (turn off at 46%) |
| **Critical Threshold** | 55% | Emergency mode activation |

**Example:** With defaults, system turns ON at 49%, turns OFF at 46%, enters critical mode at 55%.

#### Day/Night Mode

| Setting | Default | Purpose |
|---------|---------|---------|
| **Day Mode Start** | 6 AM | When permissive floor begins |
| **Night Mode Start** | 9 PM | When comfort floor begins |
| **Day Floor Temp** | 70°F | Minimum temp for AC during day |

**Recommendations:**
- **Early risers:** Set day start to 5 AM
- **Night owls:** Set night start to 11 PM or midnight
- **Cold-sensitive:** Increase day floor to 71-72°F
- **Humid climates:** Keep day floor at 70°F or lower

#### Emergency Safety

| Setting | Default | Purpose |
|---------|---------|---------|
| **Emergency Shutoff** | 66°F | System disables if temp drops below this |

**Recommendation:** Set at least 6°F below typical indoor temperature.

---

## Usage Examples

### Example 1: Morning Shower (The Problem That Inspired v4.3)

**Scenario:**
- Time: 7:42 AM (day mode)
- Indoor temp: 73°F
- Thermostat: 76°F (normal daytime setting)
- Humidity: 59% (post-shower spike)

**OLD BEHAVIOR (v4.2 and earlier):**
```
❌ Calculated floor: 74.2°F
❌ 73°F < 74.2°F → AC blocked
❌ Only standalone runs (slow)
❌ 30+ minutes to clear humidity
❌ Mold risk during delay
❌ Manual intervention required (lower thermostat to 70°F)
```

**NEW BEHAVIOR (v4.3):**
```
✅ Day mode: Flat 70°F floor
✅ 73°F > 70°F → AC allowed!
✅ 59% ≥ 55% (critical) → EMERGENCY MODE
✅ AC turns ON (fast cooling)
✅ Standalone turns ON (extra dehumidification)
✅ Both devices work together
✅ Notification: "🚨 CRITICAL humidity 59%! AC activated for emergency dehumidification."
✅ Log: "💧 Humidity ON: 59% > 49%, temp 73°F > floor 70°F (DAY mode, CRITICAL)"
✅ Log: "🔧 Standalone ON: 59% > 49% (CRITICAL - dual device mode)"

Results after 34 minutes:
- Humidity: 59% → 48% ✅
- Temperature: 73°F → 70°F ✅
- No manual intervention needed ✅
- Both devices stopped automatically ✅
```

---

### Example 2: Summer Day (Normal Temperature + Humidity)

**Scenario:**
- Time: 2 PM (day mode)
- Indoor: 79°F, 55% humidity
- Thermostat: 76°F
- Target humidity: 49%

**What Happens:**
```
1. Temperature: 79°F > 77.5°F → Temp request ON
2. Day mode: Floor 70°F, current 79°F → Allows AC
3. Humidity: 55% ≥ 55% (critical) → Humidity request ON
4. Critical mode: Standalone also ON

System status: AC + Standalone (dual device mode)

AC runs until:
- Temp drops to 75.5°F → Temp request OFF
- Humidity drops to 46% → Humidity request OFF

Standalone continues until:
- Humidity drops to 46%

Result: House at 75.5°F and 46% humidity
```

**Logs:**
```
❄️ AC turned ON (temp=on, hum=on)
💧 Humidity ON: 55% > 49%, temp 79°F > floor 70°F (DAY mode, CRITICAL)
🔧 Standalone ON: 55% > 49% (CRITICAL - dual device mode)
💧 Humidity OFF: 46% < 46% (satisfied)
🔌 AC turned OFF (both requests satisfied), 15-min timer started
🔧 Standalone OFF: 46% < 46%
```

---

### Example 3: Cool Evening (Moderate Humidity, Normal Mode)

**Scenario:**
- Time: 7 PM (day mode)
- Indoor: 72°F, 51% humidity
- Thermostat: 73°F (night)
- Target humidity: 49%

**What Happens:**
```
1. Temperature: 72°F < 74.5°F → Temp request OFF (not hot)
2. Day mode: Floor 70°F, current 72°F → Allows AC
3. Humidity: 51% < 55% → Normal mode (not critical)
4. Humidity: 51% > 49% → Humidity request ON
5. AC turns ON (no standalone needed yet)

AC runs, temp drops to 70°F:
6. Temp: 70°F = Floor → At edge
7. Humidity still 50% → AC continues

AC runs, temp drops to 69.5°F:
8. Temp: 69.5°F < Floor 70°F → Humidity request forced OFF
9. AC stops
10. Standalone takes over (humidity still 50%)

Standalone runs until 46%

Result: House at 69-70°F, humidity cleared without overcooling
```

---

### Example 4: Night Mode (Sleep Comfort Priority)

**Scenario:**
- Time: 11 PM (night mode)
- Indoor: 68°F, 52% humidity
- Thermostat: 73°F (sleep mode)
- Target humidity: 49%

**What Happens:**
```
1. Temperature: 68°F < 74.5°F → Temp request OFF
2. Night mode: Calculated floor 72.2°F
3. Current temp: 68°F < 72.2°F → AC blocked
4. Humidity: 52% < 55% → Normal mode (not critical)
5. Humidity request stays OFF (temp floor protection)
6. Standalone activates (master bedroom 52% > 49%)
7. Notification: "High humidity (52%)! Standalone activated (AC too cold to help)."

Standalone runs quietly until 46%

Result: Sleep not disturbed by cold AC, moderate humidity cleared by standalone
```

---

### Example 5: Critical Override at Night

**Scenario:**
- Time: 2 AM (night mode)
- Indoor: 68°F, 58% humidity (someone took a shower)
- Thermostat: 73°F (sleep mode)
- Critical threshold: 55%

**What Happens:**
```
1. Night mode: Calculated floor 72.2°F
2. Current temp: 68°F < 72.2°F → Normally blocks AC
3. Humidity: 58% ≥ 55% → CRITICAL MODE OVERRIDE
4. Emergency check: 68°F > (66°F emergency + 4°F buffer) = 70°F?
   ❌ 68°F < 70°F → Too cold for safe AC override
5. Standalone turns ON (only device safe to use)
6. Notification: "🚨 CRITICAL humidity 58%! Standalone activated..."

If temp was 71°F instead:
✅ 71°F > 70°F buffer → AC would be forced ON despite night mode
✅ Both AC and standalone would run (critical override)

Result: Critical humidity handled safely without dangerous overcooling
```

---

## Dashboard Setup

### Single-Card Status Infographic
```yaml
type: custom:mushroom-template-card
primary: >
  {{ states('sensor.third_reality_inc_3rths24bz_temperature') }}°F · 
  {{ states('sensor.max_indoor_humidity_sensor') }}%
secondary: >
  {% if not is_state('input_boolean.coordinated_system_enabled', 'on') %}
    ⚠️ System Disabled
  {% elif is_state('switch.meross_cool_switch', 'on') %}
    {% if is_state('input_boolean.coordinated_cooling_temp_request', 'on') and is_state('input_boolean.coordinated_cooling_hum_request', 'on') %}
      🔥💧 Cooling & Dehumidifying
    {% elif is_state('input_boolean.coordinated_cooling_temp_request', 'on') %}
      🔥 Cooling for Temperature
    {% elif is_state('input_boolean.coordinated_cooling_hum_request', 'on') %}
      💧 Dehumidifying Only
    {% else %}
      ❄️ AC Running
    {% endif %}
  {% elif is_state('humidifier.151732605035846_humidifier', 'on') %}
    🌬️ Standalone Dehumidifying
  {% elif is_state('timer.coordinated_cooler_min_cycle', 'active') %}
    ⏱️ Cooldown Active
  {% else %}
    ✅ System Idle
  {% endif %}
icon: mdi:home-thermometer-outline
icon_color: >
  {% if not is_state('input_boolean.coordinated_system_enabled', 'on') %}red
  {% elif is_state('switch.meross_cool_switch', 'on') %}blue
  {% elif is_state('humidifier.151732605035846_humidifier', 'on') %}purple
  {% elif is_state('timer.coordinated_cooler_min_cycle', 'active') %}orange
  {% else %}green{% endif %}
badge_icon: >
  {% if not is_state('input_boolean.coordinated_system_enabled', 'on') %}mdi:power-off
  {% elif is_state('input_boolean.coordinated_cooling_temp_request', 'on') and is_state('input_boolean.coordinated_cooling_hum_request', 'on') %}mdi:fire-circle
  {% elif is_state('input_boolean.coordinated_cooling_temp_request', 'on') %}mdi:fire
  {% elif is_state('input_boolean.coordinated_cooling_hum_request', 'on') %}mdi:water
  {% elif is_state('humidifier.151732605035846_humidifier', 'on') %}mdi:air-humidifier
  {% elif is_state('timer.coordinated_cooler_min_cycle', 'active') %}mdi:timer-sand
  {% endif %}
badge_color: >
  {% if not is_state('input_boolean.coordinated_system_enabled', 'on') %}red
  {% elif is_state('input_boolean.coordinated_cooling_temp_request', 'on') and is_state('input_boolean.coordinated_cooling_hum_request', 'on') %}purple
  {% elif is_state('input_boolean.coordinated_cooling_temp_request', 'on') %}red
  {% elif is_state('input_boolean.coordinated_cooling_hum_request', 'on') %}blue
  {% elif is_state('humidifier.151732605035846_humidifier', 'on') %}purple
  {% elif is_state('timer.coordinated_cooler_min_cycle', 'active') %}orange
  {% endif %}
tap_action:
  action: more-info
  entity: climate.coordinated_indoor_thermostat
hold_action:
  action: more-info
  entity: input_number.coordinated_target_humidity
```

### Full Control Dashboard
```yaml
type: vertical-stack
cards:
  # Temperature Control
  - type: thermostat
    entity: climate.coordinated_indoor_thermostat
    name: "Temperature"
  
  # Humidity Control
  - type: entities
    title: "Humidity Control"
    entities:
      - entity: input_number.coordinated_target_humidity
        name: "Target Humidity"
      - entity: input_number.coordinated_critical_humidity
        name: "Critical Threshold"
      - entity: sensor.max_indoor_humidity_sensor
        name: "Current Humidity"
        secondary_info: last-changed
  
  # Day/Night Configuration
  - type: entities
    title: "Day/Night Mode"
    entities:
      - entity: input_number.coordinated_day_mode_start
        name: "Day Starts At"
      - entity: input_number.coordinated_night_mode_start
        name: "Night Starts At"
      - entity: input_number.coordinated_day_floor_temp
        name: "Day Floor Temperature"
  
  # System Status
  - type: entities
    title: "System Status"
    state_color: true
    entities:
      - entity: input_boolean.coordinated_system_enabled
        name: "Master Switch"
      - entity: input_boolean.coordinated_cooling_temp_request
        name: "🌡️ Temp Demanding"
        tap_action:
          action: none
      - entity: input_boolean.coordinated_cooling_hum_request
        name: "💧 Humidity Demanding"
        tap_action:
          action: none
      - entity: switch.meross_cool_switch
        name: "AC Running"
        tap_action:
          action: none
      - entity: humidifier.151732605035846_humidifier
        name: "Standalone Running"
        tap_action:
          action: none
```

---

## Troubleshooting

### Viewing Logs
```
Settings → System → Logs
```

Search for emoji to filter:
- 🔧 = System events
- ❄️ = AC on/off
- 💧 = Humidity control
- 🌡️ = Temperature floor
- 🚨 = Critical mode
- 👤 = Manual override
- ⚠️ = Warnings/errors

### Common Issues

#### Issue: AC Not Running During Day Despite High Humidity

**Check:**
```
Settings → System → Logs
Search: "💧 Humidity"
```

**If seeing:** "🌡️ Humidity OFF: Temp X°F ≤ floor Y°F"
- Temp is still too cold even for day mode
- **Solution:** Lower `coordinated_day_floor_temp` from 70°F to 68-69°F

**If seeing:** Nothing in logs
- Check `input_boolean.coordinated_system_enabled` is ON
- Verify humidity sensor is working
- Check automation is enabled

---

#### Issue: Critical Mode Not Activating

**Check critical threshold:**
```
Developer Tools → States → input_number.coordinated_critical_humidity
```

**If set too high (e.g., 65%):**
- Lower to 55% for more aggressive response
- Or check current humidity is actually above threshold

**Check logs:**
```
Search: "CRITICAL"
```

Should see: "🚨 CRITICAL HUMIDITY OVERRIDE" or "CRITICAL - dual device mode"

---

#### Issue: System Overcooling at Night

**Check night mode settings:**
```
Developer Tools → States
- input_number.coordinated_night_mode_start: Should be 21 (9 PM)
- input_number.coordinated_day_mode_start: Should be 6 (6 AM)
```

**Check floor calculation:**
```
Developer Tools → Template

{% set setpoint = 73 %}
{% set cutoff = setpoint - 0.5 %}
{% set buffer = 0.5 %}
{% set floor = cutoff - buffer + 0.2 %}
Floor: {{ floor }}°F
```

**If floor too low:**
- Increase `coordinated_cool_buffer` from 0.5 to 0.7-1.0
- This raises the floor, preventing AC from running when too cold

---

## Advanced Tuning

### Adjusting Day Mode Aggressiveness

**More aggressive (faster humidity clearing, more cooling):**
```
Day Floor Temperature: 68-69°F
Critical Humidity: 52-54%
```

**Less aggressive (less cooling, slower response):**
```
Day Floor Temperature: 71-72°F
Critical Humidity: 58-60%
```

### Seasonal Adjustments

**Summer (hot & humid):**
```
Target Humidity: 45%
Day Floor: 70°F
Critical Threshold: 55%
Day Mode Hours: 5 AM - 10 PM (longer day mode)
```

**Winter (cool & dry):**
```
Target Humidity: 52%
Day Floor: 72°F
Critical Threshold: 60%
Day Mode Hours: 7 AM - 8 PM (shorter day mode)
```

### Custom Day/Night Schedules

**Early risers:**
```
Day Mode Start: 5 AM
Night Mode Start: 9 PM
```

**Night owls:**
```
Day Mode Start: 8 AM
Night Mode Start: 11 PM
```

**Shift workers (sleep during day):**
```
Day Mode Start: 6 PM
Night Mode Start: 8 AM
(Day mode is when you're awake)
```

---

## FAQ

### Q: Why 70°F for day floor instead of calculated?

**A:** Real-world testing showed:
- Post-shower humidity spikes (55-60%) need fast response
- 70°F allows aggressive AC without discomfort
- Nobody notices 3-4°F cooling over 30 minutes
- Calculated floor often blocks AC when you need it most

---

### Q: When should I adjust critical humidity threshold?

**A:** 
- **55% (default):** Good for humid climates, prevents mold
- **60%:** Moderate response, less aggressive
- **50-52%:** Very aggressive, for extremely humid environments
- **65%:** Conservative, only for emergencies

---

### Q: Can standalone and AC run at the same time?

**A:** YES! In critical humidity mode (≥55%), both devices run together for maximum dehumidification power. This is intentional and safe.

---

### Q: What if my schedule is irregular?

**A:** Day/night mode is based on time, not occupancy. If your schedule varies:
- Set day mode hours to when you're typically awake
- Or use two automations to adjust mode times based on occupancy
- System will still work, just may be slightly less optimal

---

### Q: Does day mode use more electricity?

**A:** Possibly slightly more during the day, but:
- Prevents mold/property damage (expensive)
- Clears humidity faster (AC runs shorter overall)
- Night mode still optimizes for efficiency
- Most users report lower bills due to reduced runtime

---

### Q: Emergency shutoff triggered during critical mode - is that safe?

**A:** YES. Emergency shutoff is the ultimate safety. If temp drops to 66°F even in critical mode, safety wins. Critical mode has a 4°F buffer above emergency temp to prevent this, but if it happens, system prioritized your safety over humidity control (correct behavior).

---

### Q: Can I have different critical thresholds for day/night?

**A:** Not currently in v4.3, but you could add a second `input_number` and update the automation. Generally not needed - critical is critical regardless of time.

---

**Version:** 4.3  
**Last Updated:** January 2026  
**License:** MIT
