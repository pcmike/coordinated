# Coordinated Cooler v4.3 - Testing Guide

**Last Updated:** January 2026  
**Package Version:** 4.3

---

## Table of Contents

- [What's New in v4.3 Testing](#whats-new-in-v43-testing)
- [Pre-Flight Checks](#pre-flight-checks)
- [Test 1: Basic ON/OFF Control](#test-1-basic-onoff-control)
- [Test 2: Day Mode Temperature Floor](#test-2-day-mode-temperature-floor)
- [Test 3: Night Mode Temperature Floor](#test-3-night-mode-temperature-floor)
- [Test 4: Critical Humidity Response](#test-4-critical-humidity-response)
- [Test 5: Day/Night Mode Transitions](#test-5-daynight-mode-transitions)
- [Test 6: Standalone Fallback](#test-6-standalone-fallback)
- [Test 7: Emergency Shutoff Protection](#test-7-emergency-shutoff-protection)
- [Test 8: Manual Override Detection](#test-8-manual-override-detection)
- [Test 9: Startup Recovery](#test-9-startup-recovery)
- [Test 10: Anti-Short-Cycle Protection](#test-10-anti-short-cycle-protection)
- [Validation Checklist](#validation-checklist)
- [Common Issues During Testing](#common-issues-during-testing)
- [Final Validation](#final-validation)

---

## What's New in v4.3 Testing

### New Tests Required

1. **Day mode floor testing** - Verify 70°F flat floor allows AC during daytime
2. **Night mode floor testing** - Verify calculated floor protects sleep comfort
3. **Critical humidity mode** - Test dual-device (AC + standalone) operation
4. **Mode transitions** - Verify smooth switching between day/night modes
5. **Critical override safety** - Ensure emergency buffer prevents dangerous cooling

### Key Scenarios to Validate

- **Post-shower spike:** Day mode + critical humidity → Both devices activate
- **Night comfort:** Night mode blocks AC when temp too cold
- **Emergency safety:** Critical mode respects emergency shutoff buffer

---

## Pre-Flight Checks

### 1. Verify Entity IDs Configured
```
Developer Tools → States
```

Search for each entity and verify all exist:
- `sensor.third_reality_inc_3rths24bz_temperature`
- `sensor.third_reality_inc_3rths24bz_humidity`
- `sensor.max_indoor_humidity_sensor`
- `switch.meross_cool_switch`
- `humidifier.151732605035846_humidifier`

---

### 2. Verify New v4.3 Entities Created
```
Developer Tools → States
Search: "coordinated"
```

**New in v4.3 - should see:**
- `input_number.coordinated_critical_humidity` (default 55)
- `input_number.coordinated_day_mode_start` (default 6)
- `input_number.coordinated_night_mode_start` (default 21)
- `input_number.coordinated_day_floor_temp` (default 70)

---

### 3. Check Configuration Validity
```
Developer Tools → YAML → Check Configuration
```

Expected: "Configuration valid!" ✅

---

### 4. Verify Current Day/Night Mode
```
Developer Tools → Template

{% set current_hour = now().hour %}
{% set day_start = 6 %}
{% set night_start = 21 %}
{% set is_night = current_hour >= night_start or current_hour < day_start %}

Current Hour: {{ current_hour }}
Mode: {{ 'NIGHT' if is_night else 'DAY' }}
Day Floor: 70°F
Night Floor: [calculated based on setpoint]
```

---

## Test 1: Basic ON/OFF Control

### Test 1A: AC Turn ON

**Steps:**
1. `Developer Tools → Services`
2. Service: `switch.turn_on`
3. Target: `switch.coordinated_cooler_switch`
4. **CALL SERVICE**

**Expected Results:**
- ✅ Physical AC turns on
- ✅ Log: "❄️ AC turned ON (temp=off, hum=off)"

**Verification:**
```
Developer Tools → States → switch.meross_cool_switch
state: on
```

---

### Test 1B: AC Turn OFF

**Steps:**
1. `Developer Tools → Services`
2. Service: `switch.turn_off`
3. Target: `switch.coordinated_cooler_switch`
4. **CALL SERVICE**

**Expected Results:**
- ✅ Physical AC stops
- ✅ Timer starts (15 minutes)
- ✅ Log: "🔌 AC turned OFF (both requests satisfied), 15-min timer started"

**Verification:**
```
Developer Tools → States
- switch.meross_cool_switch: off
- timer.coordinated_cooler_min_cycle: active
```

---

## Test 2: Day Mode Temperature Floor

### Test 2A: Verify Day Mode Active

**Setup:**
- Perform this test between 6 AM - 9 PM (default day mode hours)
- Or temporarily adjust `coordinated_day_mode_start` to current hour

**Verify mode:**
```
Developer Tools → Template

{% set current_hour = now().hour %}
{% set day_start = states('input_number.coordinated_day_mode_start') | int %}
{% set night_start = states('input_number.coordinated_night_mode_start') | int %}
{% set is_night = current_hour >= night_start or current_hour < day_start %}

Current: {{ 'NIGHT' if is_night else 'DAY' }} mode
```

Should show: "DAY mode"

---

### Test 2B: Day Floor Allows AC (The Shower Scenario)

**Purpose:** Verify AC can run during day even when thermostat is set warm

**Setup:**
1. Set thermostat to **76°F**
2. Wait for temp to stabilize at **72-74°F** (cooler than setpoint)
3. Manually increase humidity to **52%** (steam from kettle)

**Expected Floor:**
- Day mode floor: **70°F** (flat, from `coordinated_day_floor_temp`)

**Expected Results:**
- ✅ Current temp (73°F) > Floor (70°F) → AC allowed
- ✅ Humidity (52%) > Target (49%) → Humidity request ON
- ✅ AC turns ON
- ✅ Log: "💧 Humidity ON: 52.0% > 49.0%, temp 73.0°F > floor 70.0°F (DAY mode)"
- ✅ Notification: "High humidity (52%)! AC activated for dehumidification."

**Verification:**
```
Developer Tools → States
- input_boolean.coordinated_cooling_hum_request: on
- input_boolean.coordinated_cooling_temp_request: off
- switch.meross_cool_switch: on
```

**Let AC run and monitor:**
- AC should continue running down to 70°F
- At 69.9°F, humidity request should turn OFF
- Log: "🌡️ Humidity OFF: Temp 69.9°F ≤ floor 70.0°F (too cold for AC dehumid, DAY mode)"

---

### Test 2C: Day Floor Reached

**Continue from Test 2B:**

**When temp drops to 70°F:**
- ✅ Temp at floor edge
- ✅ AC may continue briefly if humidity still high

**When temp drops to 69.5°F:**
- ✅ Temp below floor
- ✅ Humidity request forced OFF
- ✅ AC stops
- ✅ Log: "🌡️ Humidity OFF: Temp 69.5°F ≤ floor 70.0°F"
- ✅ Timer starts

**If humidity still high (>49%):**
- ✅ Standalone should activate (see Test 6)

---

## Test 3: Night Mode Temperature Floor

### Test 3A: Verify Night Mode Active

**Setup:**
- Perform this test between 9 PM - 6 AM (default night mode hours)
- Or temporarily adjust `coordinated_night_mode_start` to current hour

**Verify mode:**
```
Developer Tools → Template

{% set current_hour = now().hour %}
Current Hour: {{ current_hour }}
Night Mode Active: {{ current_hour >= 21 or current_hour < 6 }}
```

Should show: "True"

---

### Test 3B: Night Floor Calculation

**Setup:**
1. Set thermostat to **73°F** (sleep mode)
2. Wait for temp to stabilize at **73-74°F**

**Calculate expected floor:**
```
Developer Tools → Template

{% set setpoint = 73 %}
{% set cutoff = setpoint - 0.5 %}
{% set buffer = 0.5 %}
{% set floor = cutoff - buffer + 0.2 %}

Setpoint: {{ setpoint }}°F
Cutoff: {{ cutoff }}°F
Buffer: {{ buffer }}°F (cool mode, cutoff < 75)
Floor: {{ floor }}°F
```

Should show: "Floor: 72.2°F"

---

### Test 3C: Night Floor Blocks AC (Comfort Mode)

**Setup:**
1. Thermostat at **73°F** (sleep mode)
2. Wait for temp to drop to **68-70°F** (below floor of 72.2°F)
3. Manually increase humidity to **52%**

**Expected Behavior:**
- ✅ Night mode: Calculated floor 72.2°F
- ✅ Current temp (68°F) < Floor (72.2°F) → AC blocked
- ✅ Humidity (52%) < Critical (55%) → Normal mode (not emergency)
- ✅ Humidity request stays OFF
- ✅ Log: "🌡️ Humidity OFF: Temp 68.0°F ≤ floor 72.2°F (too cold for AC dehumid, NIGHT mode)"
- ✅ AC does NOT turn on
- ✅ Standalone should activate instead

**Verification:**
```
Developer Tools → States
- input_boolean.coordinated_cooling_hum_request: off (blocked by floor)
- switch.meross_cool_switch: off
- humidifier.151732605035846_humidifier: on (fallback active)
```

**This protects sleep comfort** - AC won't make you cold at night.

---

## Test 4: Critical Humidity Response

### Test 4A: Critical Mode During Day

**Purpose:** Verify both AC and standalone activate for emergency humidity

**Setup:**
1. Ensure **DAY mode** active (6 AM - 9 PM)
2. Set thermostat to **76°F**
3. Current temp: **73°F**
4. Manually increase humidity to **57%** (above 55% critical threshold)

**Expected Behavior:**
```
Humidity: 57% ≥ 55% (CRITICAL threshold)
Day Mode Floor: 70°F
Current Temp: 73°F > 70°F

Decision: EMERGENCY MODE
```

**Expected Results:**
- ✅ Humidity (57%) ≥ Critical (55%) → CRITICAL MODE
- ✅ Temp (73°F) > Floor (70°F) → AC allowed
- ✅ Humidity request ON
- ✅ AC turns ON
- ✅ Log: "💧 Humidity ON: 57.0% > 49.0%, temp 73.0°F > floor 70.0°F (DAY mode, CRITICAL)"
- ✅ Standalone ALSO turns ON (critical mode)
- ✅ Log: "🔧 Standalone ON: 57.0% > 49.0% (CRITICAL - dual device mode)"
- ✅ Notification: "🚨 CRITICAL humidity 57%! AC activated for emergency dehumidification."
- ✅ **Both devices running together**

**Verification:**
```
Developer Tools → States
- input_boolean.coordinated_cooling_hum_request: on
- switch.meross_cool_switch: on
- humidifier.151732605035846_humidifier: on (BOTH DEVICES!)
```

**Let both run until humidity drops to 46%:**
- ✅ AC stops when humidity request turns OFF
- ✅ Standalone stops when humidity < 46%
- ✅ Both notifications sent

---

### Test 4B: Critical Mode at Night (Below Floor)

**Purpose:** Verify critical mode safety limits

**Setup:**
1. Ensure **NIGHT mode** active (9 PM - 6 AM)
2. Set thermostat to **73°F**
3. Current temp: **68°F** (below calculated floor of 72.2°F)
4. Manually increase humidity to **57%** (critical)

**Expected Behavior:**
```
Humidity: 57% ≥ 55% (CRITICAL)
Night Mode Floor: 72.2°F
Current Temp: 68°F < 72.2°F
Emergency Threshold: 66°F
Safety Buffer: 66 + 4 = 70°F

Critical Override Check:
68°F > 70°F safety buffer? NO → TOO COLD for AC even in critical mode
```

**Expected Results:**
- ✅ Humidity (57%) ≥ Critical (55%) → CRITICAL MODE
- ✅ Temp (68°F) < Floor (72.2°F) → Would normally block AC
- ✅ Temp (68°F) < Safety buffer (70°F) → CRITICAL OVERRIDE BLOCKED (too dangerous)
- ✅ AC stays OFF (safety wins)
- ✅ Standalone turns ON (only safe option)
- ✅ Log: "🔧 Standalone ON: 57.0% > 49.0% (CRITICAL - dual device mode)"
- ✅ Notification: "🚨 CRITICAL humidity 57%! Standalone activated..."

**Verification:**
```
Developer Tools → States
- input_boolean.coordinated_cooling_hum_request: off (safety blocked AC)
- switch.meross_cool_switch: off (correct - too cold)
- humidifier.151732605035846_humidifier: on (only device safe to use)
```

**This is CORRECT behavior** - Safety > humidity in extreme cold scenarios.

---

### Test 4C: Critical Override Within Safe Range

**Setup:**
1. **NIGHT mode** active
2. Set thermostat to **73°F**
3. Current temp: **71°F** (below floor 72.2°F, but above safety buffer 70°F)
4. Humidity: **57%** (critical)

**Expected Behavior:**
```
Humidity: 57% ≥ 55% (CRITICAL)
Night Floor: 72.2°F
Current: 71°F < 72.2°F (normally blocks)
Safety Buffer: 70°F
71°F > 70°F → Within safe override range
```

**Expected Results:**
- ✅ Critical mode active
- ✅ Temp above safety buffer → CRITICAL OVERRIDE ALLOWED
- ✅ Humidity request forced ON despite being below floor
- ✅ AC turns ON
- ✅ Standalone turns ON
- ✅ Log: "🚨 CRITICAL HUMIDITY OVERRIDE: 57.0% ≥ 55.0% - AC forced ON despite temp 71.0°F ≤ floor 72.2°F"
- ✅ **Both devices running** (maximum power)

**This balances safety and effectiveness.**

---

## Test 5: Day/Night Mode Transitions

### Test 5A: Setup Transition Test

**Purpose:** Verify smooth mode changes at 6 AM and 9 PM

**Method 1: Wait for natural transition**
- Monitor system at 5:59 AM or 8:59 PM
- Watch logs at transition time

**Method 2: Force transition with time change**
- Temporarily set transition times to current hour
- Change `coordinated_day_mode_start` or `coordinated_night_mode_start`

---

### Test 5B: Night → Day Transition (6 AM)

**Setup:**
- Time: 5:55 AM
- Thermostat: 73°F
- Current temp: 71°F (below night floor of 72.2°F)
- Humidity: 51%
- AC: OFF (blocked by night floor)
- Standalone: ON (handling humidity)

**At 6:00 AM:**
- ✅ Mode switches to DAY
- ✅ Floor changes: 72.2°F → 70°F
- ✅ Current temp (71°F) now > Floor (70°F)
- ✅ Humidity still > target → Humidity request turns ON
- ✅ AC turns ON
- ✅ Log: "💧 Humidity ON: 51.0% > 49.0%, temp 71.0°F > floor 70.0°F (DAY mode)"
- ✅ Standalone continues running (humidity still high)

**System now more aggressive due to day mode.**

---

### Test 5C: Day → Night Transition (9 PM)

**Setup:**
- Time: 8:55 PM
- Thermostat: 76°F
- Current temp: 70.5°F (above day floor of 70°F)
- Humidity: 51%
- AC: ON (humidity dehumidification)

**At 9:00 PM:**
- ✅ Mode switches to NIGHT
- ✅ Floor changes: 70°F → calculated floor
- ✅ Calculate new floor: (76 - 0.5) - 1.5 + 0.2 = 74.2°F
- ✅ Current temp (70.5°F) < New floor (74.2°F)
- ✅ Humidity request forced OFF
- ✅ AC stops
- ✅ Log: "🌡️ Humidity OFF: Temp 70.5°F ≤ floor 74.2°F (too cold for AC dehumid, NIGHT mode)"
- ✅ Standalone takes over if humidity still > 49%

**System now protects sleep comfort.**

---

## Test 6: Standalone Fallback

### Test 6A: Normal Fallback (Non-Critical)

**Setup:**
1. Temp below floor (AC blocked)
2. Humidity above target (needs dehumidification)
3. Humidity below critical threshold

**Steps:**
1. Set thermostat to **73°F**
2. Wait for temp to drop to **68°F**
3. Humidity at **51%** (above 49%, below 55%)

**Expected Results:**
- ✅ Humidity request OFF (temp floor blocking AC)
- ✅ Temp request OFF (not hot)
- ✅ Humidity (51%) > Target (49%) → Standalone should activate
- ✅ Log: "🔧 Standalone ON: 51.0% > 49.0% (AC disabled by temp floor)"
- ✅ Notification: "High humidity (51%)! Standalone activated (AC too cold to help)."

**Verification:**
```
Developer Tools → States
- input_boolean.coordinated_cooling_hum_request: off
- input_boolean.coordinated_cooling_temp_request: off
- switch.meross_cool_switch: off
- humidifier.151732605035846_humidifier: on
```

---

### Test 6B: Critical Fallback (Dual Device)

**Already covered in Test 4A** - Standalone runs WITH AC in critical mode.

---

### Test 6C: Standalone Stops When Humidity Satisfied

**Continue from Test 6A:**

**Let standalone run until humidity drops to 46%:**
- ✅ Humidity drops below target - dry_tolerance
- ✅ Standalone stops
- ✅ Log: "🔧 Standalone OFF: 46.0% < 46.0%"
- ✅ Notification: "Humidity normal (46%)! Standalone dehumidifier turned off."

---

## Test 7: Emergency Shutoff Protection

### ⚠️ WARNING

**This test triggers emergency shutoff. You'll need to manually re-enable the system.**

---

### Test 7A: Setup Emergency Test

**Steps:**
1. **Temporarily** set `coordinated_emergency_temp` to **72°F** (test value)
2. Set thermostat to **76°F**
3. Manually turn AC ON
4. Monitor temperature drop

---

### Test 7B: Monitor Drop to Emergency Threshold

**As temp drops:**
- Normal operation above 72°F
- System waits 2 minutes after reaching 72°F

**After 2 minutes at or below 72°F:**
- ✅ Emergency automation triggers
- ✅ AC forced OFF
- ✅ Both requests OFF
- ✅ Master switch OFF (entire system disabled)
- ✅ Timer starts
- ✅ Log: "🚨 EMERGENCY SHUTOFF: Temp 72.0°F - system disabled"
- ✅ Notification: "🚨 AC Emergency Shutoff - Temperature dropped to 72°F!"

**Verification:**
```
Developer Tools → States
- switch.meross_cool_switch: off
- input_boolean.coordinated_system_enabled: off (SYSTEM DISABLED)
- All requests: off
```

---

### Test 7C: Recovery

**Manual recovery:**
1. Turn ON: `input_boolean.coordinated_system_enabled`
2. Reset: `coordinated_emergency_temp` → 66
3. System resumes

---

## Test 8: Manual Override Detection

### Test 8A: Manual ON

**Setup:**
- System idle (both requests OFF, AC OFF)

**Steps:**
1. Manually turn on AC via Meross app or physical switch
2. Wait 10 seconds

**Expected Results:**
- ✅ Both requests turn ON
- ✅ Log: "👤 Manual override: AC turned ON, both requests activated"

---

### Test 8B: Manual OFF

**Setup:**
- AC running for humidity (temp OFF, humidity ON)

**Steps:**
1. Manually turn off AC
2. Wait 10 seconds

**Expected Results:**
- ✅ Both requests turn OFF
- ✅ Timer starts (15 min)
- ✅ Log: "👤 Manual override: AC turned OFF, both requests deactivated, timer started"

---

## Test 9: Startup Recovery

### Test 9A: Restart with AC Running

**Setup:**
1. AC actively running
2. **Restart Home Assistant**

**After restart (within 5 minutes):**
- ✅ Log: "🔧 Coordinated System: All sensors loaded successfully"
- ✅ Log: "🔧 Startup: AC=on, Temp=XX°F, Humidity=XX%"
- ✅ Log: "✅ Startup: Reclaimed by [temperature/humidity] control"
- ✅ Appropriate request stays ON

---

### Test 9B: Restart with AC Off

**Setup:**
1. AC off, timer active
2. **Restart Home Assistant**

**After restart:**
- ✅ Log: "✅ Startup: AC off - no action needed"
- ✅ System remains idle

---

## Test 10: Anti-Short-Cycle Protection

### Test 10A: Rapid ON/OFF Attempt

**Steps:**
1. Turn AC ON
2. Wait 5 seconds
3. Turn AC OFF
4. Immediately try to turn ON again

**Expected Results:**
- ✅ First OFF: Timer starts (15 min)
- ✅ Attempted ON: Request turns ON but physical AC stays OFF
- ✅ After 15 min: Timer expires, AC starts if request still active

---

## Validation Checklist

After completing all tests, verify:

### Day/Night Mode Features
- [ ] Day mode uses 70°F flat floor
- [ ] Night mode uses calculated floor
- [ ] Mode transitions happen at configured times
- [ ] Logs clearly indicate current mode (DAY/NIGHT)

### Critical Humidity Response
- [ ] Both AC and standalone activate when humidity ≥55%
- [ ] Critical notifications sent
- [ ] Safety buffer prevents dangerous cooling
- [ ] Both devices stop when humidity satisfied

### Temperature Floor
- [ ] Day floor allows AC when thermostat set warm
- [ ] Night floor blocks AC when too cold for comfort
- [ ] Floor enforcement logged clearly
- [ ] Standalone takes over when AC blocked

### Safety Features
- [ ] Emergency shutoff works at 66°F (after test reset)
- [ ] Critical override respects safety buffer
- [ ] 15-minute timer prevents short-cycling
- [ ] Manual overrides detected

### User Experience
- [ ] All new settings adjustable via UI
- [ ] Notifications indicate critical vs normal mode
- [ ] Logs use DAY/NIGHT/CRITICAL labels
- [ ] System recovers after restart

---

## Common Issues During Testing

### Issue: Day mode not activating during daytime

**Check:**
```
Developer Tools → States
- input_number.coordinated_day_mode_start: Should be 6
- input_number.coordinated_night_mode_start: Should be 21
```

**Verify current mode:**
```
Developer Tools → Template
Current Hour: {{ now().hour }}
Should be DAY mode: {{ 6 <= now().hour < 21 }}
```

---

### Issue: Critical mode not activating at 55%

**Check:**
```
Developer Tools → States → input_number.coordinated_critical_humidity
```

Should be: 55

**Verify humidity sensor:**
```
Developer Tools → States → sensor.max_indoor_humidity_sensor
```

Must be reading ≥55%

---

### Issue: AC running despite night mode floor

**Check logs:**
```
Search: "CRITICAL HUMIDITY OVERRIDE"
```

**If found:** Humidity is in critical mode, overriding floor (correct behavior)

**If not found:** Check automation is using correct floor calculation

---

## Final Validation

### Comprehensive Status Template
```
Developer Tools → Template

=== MODE STATUS ===
{% set hour = now().hour %}
{% set day_start = 6 %}
{% set night_start = 21 %}
{% set is_night = hour >= night_start or hour < day_start %}
Current Hour: {{ hour }}
Active Mode: {{ 'NIGHT' if is_night else 'DAY' }}
Day Start: {{ day_start }}:00
Night Start: {{ night_start }}:00

=== TEMPERATURE FLOOR ===
{% set setpoint = 73 %}
{% set cutoff = setpoint - 0.5 %}
{% set day_floor = 70 %}
{% if is_night %}
  {% set buffer = 0.5 %}
  {% set calc_floor = cutoff - buffer + 0.2 %}
  Night Floor (Calculated): {{ calc_floor }}°F
{% else %}
  Day Floor (Flat): {{ day_floor }}°F
{% endif %}

Current Temp: {{ states('sensor.third_reality_inc_3rths24bz_temperature') }}°F
Thermostat: {{ setpoint }}°F

=== HUMIDITY STATUS ===
Current: {{ states('sensor.max_indoor_humidity_sensor') }}%
Target: {{ states('input_number.coordinated_target_humidity') }}%
Critical Threshold: {{ states('input_number.coordinated_critical_humidity') }}%
Is Critical: {{ states('sensor.max_indoor_humidity_sensor') | float >= states('input_number.coordinated_critical_humidity') | float }}

=== SYSTEM STATE ===
Master Enabled: {{ states('input_boolean.coordinated_system_enabled') }}
Temp Request: {{ states('input_boolean.coordinated_cooling_temp_request') }}
Humidity Request: {{ states('input_boolean.coordinated_cooling_hum_request') }}
AC Running: {{ states('switch.meross_cool_switch') }}
Standalone Running: {{ states('humidifier.151732605035846_humidifier') }}
Timer: {{ states('timer.coordinated_cooler_min_cycle') }}

=== SAFETY ===
Emergency Threshold: {{ states('input_number.coordinated_emergency_temp') }}°F
Safety Buffer: {{ states('input_number.coordinated_emergency_temp') | float + 4 }}°F
Distance from Emergency: {{ states('sensor.third_reality_inc_3rths24bz_temperature') | float - states('input_number.coordinated_emergency_temp') | float }}°F
```

**Expected:** All values reasonable and logical.

---

### Success Criteria

**System ready for production when:**

1. ✅ All 10 tests complete without errors
2. ✅ Day mode allows AC with 70°F floor
3. ✅ Night mode protects comfort with calculated floor
4. ✅ Critical mode activates both devices at 55%
5. ✅ Mode transitions smooth at configured times
6. ✅ Safety buffer prevents dangerous cooling
7. ✅ Standalone fallback works in all scenarios
8. ✅ Emergency shutoff tested and recoverable
9. ✅ Manual overrides detected
10. ✅ System recovers after restart
11. ✅ All logs show DAY/NIGHT/CRITICAL labels
12. ✅ Notifications distinguish normal vs critical

---

## Post-Testing Configuration

### Restore Normal Settings

After testing complete:

1. **Emergency threshold:** 66°F (if changed for testing)
2. **Day mode hours:** 6 AM - 9 PM (or your preference)
3. **Day floor temp:** 70°F (or adjust based on comfort)
4. **Critical humidity:** 55% (or adjust based on climate)
5. **Master switch:** ON

---

### Monitor First Week

**Daily:**
- Check logs for mode transitions (6 AM, 9 PM)
- Verify critical mode activations make sense
- Confirm no emergency shutoffs

**Weekly:**
- Review humidity response times
- Check if day floor needs adjustment
- Verify night mode comfort acceptable

---

**Testing complete! System v4.3 validated and ready for production.**

**For support, see README.md troubleshooting section.**
