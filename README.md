# Home Assistant - Frient Smoke Detector Automations

Complete automation system for managing Frient smoke detectors connected via Zigbee2MQTT in Home Assistant.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Required Helpers](#required-helpers)
- [System Architecture](#system-architecture)
- [Scripts](#scripts)
- [Automations](#automations)
- [Operation Flow](#operation-flow)
- [Installation](#installation)
- [Usage](#usage)

## 🎯 Overview

This system enables you to:
- Detect smoke via a group of detectors
- Propagate the alarm to all other detectors on the network
- Manage a temporary silence mode
- Send notifications via Discord and mobile app
- Automatically restart the alarm if smoke persists
- Test detectors individually

## ⚙️ Prerequisites

- Home Assistant with Zigbee2MQTT configured
- Zigbee-compatible Frient smoke detectors
- Discord integration (optional, for notifications)
- Mobile notification integration

## 🔧 Required Helpers

Create the following helpers in Home Assistant:

| Helper | Type | Description |
|--------|------|-------------|
| `input_boolean.alarm_is_active` | Boolean | Indicates the current state of the alarm |
| `input_boolean.smoke_alarm_silence` | Boolean | Enables silence mode (stops the alarm) |
| `input_text.smoke_alarm_source` | Text | Name of the detector that detected smoke |
| `timer.smoke_source_block` | Timer | Duration during which the alarm is blocked |
| `binary_sensor.groupe_detection_de_fumee` | Group | Group containing all smoke sensors |
| `switch.groupe_alarme` | Group | Group containing all alarm actuators |

## 🏗️ System Architecture

The system relies on three main components:

### 1. Centralised Detection
A binary group (`binary_sensor.groupe_detection_de_fumee`) monitors the state of all detectors. As soon as one detector turns "on", the entire group turns "on".

### 2. Alarm Propagation
When one detector detects smoke, all other detectors on the network are activated via MQTT to create a generalised alarm.

### 3. Intelligent Management
- Temporary silence mode (30 minutes by default)
- Automatic restart if smoke persists
- Automatic stop when smoke clears
- Protection against logical inconsistencies

## 📜 Scripts

### `smoke_propagation` - Siren Propagation

**File:** `Scripts/Smoke – Propagation sirènes`

Propagates the alarm to all detectors in the group.

**Parameters:**
- `source_entity` (required): Source detector that detected smoke
- `duration_seconds` (default: 5): Activation duration for secondary detectors

**Operation:**
1. Verifies that silence mode is not active
2. Activates the `alarm_is_active` indicator
3. For each detector in the group:
   - If it's the source detector: can send a START command (disabled by default)
   - If it's a secondary detector: activates the siren in "burglar" mode via MQTT

**MQTT command sent to secondary detectors:**
```json
{
  "warning": {
    "duration": 5,
    "level": "low",
    "mode": "burglar",
    "strobe": false,
    "strobe_duty_cycle": 0
  }
}
```

### `test_detecteur_duree` - Detector Test

**File:** `Scripts/Test détecteur — durée paramétrable`

Allows individual testing of a detector with configurable duration.

**Parameters:**
- `entity_id` (required): Switch entity of the detector to test
- `duration` (default: 00:00:05): Test duration in HH:MM:SS format

**Operation:**
1. Calculates duration in seconds
2. Activates the detector's siren via MQTT
3. Waits for the specified duration
4. Deactivates the alarm
5. Turns off the detector switch

## 🤖 Automations

### 1. `Smoke – Detection via group`

**Trigger:** The group `binary_sensor.groupe_detection_de_fumee` turns "on"

**Actions:**
1. Identifies the source detector (first active detector in the group)
2. Sends notifications:
   - Via the custom notification script
   - Via Discord with user mention and formatted embed
3. Activates `alarm_is_active`
4. Records the source detector in `input_text.smoke_alarm_source`
5. If silence mode is not active: launches the `smoke_propagation` script

**Discord Notification:**
- Title: 🚨 SMOKE ALERT DETECTED
- Red colour (code: 16711680)
- Fields: Location, Time
- User mention for immediate alert

### 2. `Smoke – Silence`

**Trigger:** `input_boolean.smoke_alarm_silence` turns "on"

**Actions:**
1. Starts the timer `timer.smoke_source_block`
2. Sends a Discord notification
3. Deactivates `alarm_is_active`
4. Loops until the timer ends, continuously turning off `switch.groupe_alarme`

**Purpose:** Allows temporary silencing of alarms (e.g., in case of false alarm or if evacuation is in progress).

### 3. `Smoke – Repeat source after 30min`

**Trigger:** The timer `timer.smoke_source_block` finishes

**Conditions:**
- The detection group is still "on"
- A source detector is recorded

**Actions:**
1. Reactivates the source detector's siren via MQTT (60 seconds)
2. Deactivates silence mode
3. Reactivates `alarm_is_active`

**Purpose:** If smoke persists after 30 minutes of silence, the system automatically reactivates the alarm.

### 4. `Smoke – Stop when smoke gone`

**Trigger:** The group `binary_sensor.groupe_detection_de_fumee` turns "off"

**Actions:**
1. Turns off all detectors (`switch.groupe_alarme`)
2. Clears the source detector
3. Deactivates silence mode and active alarm

**Purpose:** Complete automatic shutdown of the system when smoke is no longer present.

### 5. `Smoke – Logical inconsistency`

**Trigger:** `alarm_is_active` remains "on" for 1 minute

**Condition:** The detection group is "off"

**Action:** Deactivates `alarm_is_active`

**Purpose:** Corrects logical inconsistencies where the alarm would be active without smoke detection.

### 6. `Smoke – Watchdog on HA start`

**Trigger:** Home Assistant starts

**Condition:** The detection group is "on"

**Actions:**
1. Identifies the first active detector
2. Records the source
3. Activates `alarm_is_active`
4. If silence mode is not active: reactivates the source detector's siren (60 seconds)

**Purpose:** If Home Assistant restarts during an alert, the system reinitialises correctly.

## 🔄 Operation Flow

### Scenario 1: Normal Smoke Detection

```
1. A detector detects smoke
   ↓
2. binary_sensor.groupe_detection_de_fumee → "on"
   ↓
3. "Detection via group" automation triggers
   ↓
4. Notifications sent (mobile + Discord)
   ↓
5. smoke_propagation script activated
   ↓
6. All detectors sound
   ↓
7. Smoke clears → group → "off"
   ↓
8. "Stop when smoke gone" automation
   ↓
9. Everything stops and resets
```

### Scenario 2: Using Silence Mode

```
1. Alarm in progress (detectors sounding)
   ↓
2. User activates input_boolean.smoke_alarm_silence
   ↓
3. "Silence" automation triggers
   ↓
4. 30-minute timer starts
   ↓
5. Alarms silenced for 30 minutes
   ↓
6. Two possibilities:
   
   A. Smoke clears during silence
      → "Stop when smoke gone" deactivates everything
   
   B. Smoke persists after 30 min
      → "Repeat source after 30min" reactivates source alarm
      → Silence mode deactivated
      → Cycle restarts
```

### Scenario 3: Home Assistant Restart

```
1. Home Assistant restarts during an alert
   ↓
2. "Watchdog on HA start" automation
   ↓
3. Checks if the group is "on"
   ↓
4. If yes: reactivates the source alarm
   ↓
5. Restores system state
```

## 📥 Installation

1. **Create the necessary helpers** (see Required Helpers section)

2. **Create a group for smoke sensors:**
   ```yaml
   binary_sensor:
     - platform: group
       name: "Groupe détection de fumée"
       device_class: smoke
       entities:
         - binary_sensor.detecteur_fumee_1_smoke
         - binary_sensor.detecteur_fumee_2_smoke
         # Add all your detectors
   ```

3. **Create a group for alarm switches:**
   ```yaml
   switch:
     - platform: group
       name: "Groupe alarme"
       entities:
         - switch.detecteur_fumee_1_alarm
         - switch.detecteur_fumee_2_alarm
         # Add all your alarm switches
   ```

4. **Configure the timer:**
   ```yaml
   timer:
     smoke_source_block:
       duration: "00:30:00"  # 30 minutes
   ```

5. **Import the scripts** from the `Scripts/` folder

6. **Import the automations** from the `Automatisation/` folder

7. **Configure your Discord notifications** (optional):
   - Replace `192247198170218497` with your Discord user ID
   - Replace `1459865652679675915` with your Discord channel ID

## 🎮 Usage

### Testing a Detector

Call the `test_detecteur_duree` script with parameters:
```yaml
service: script.test_detecteur_duree
data:
  entity_id: switch.detecteur_fumee_salon_alarm
  duration: "00:00:10"  # 10-second test
```

### Activating Silence Mode

Simply activate the helper:
```yaml
service: input_boolean.turn_on
target:
  entity_id: input_boolean.smoke_alarm_silence
```

Or via the Home Assistant interface, toggle `input_boolean.smoke_alarm_silence` to ON.

### Checking System Status

- **Active alarm:** Check `input_boolean.alarm_is_active`
- **Source detector:** Consult `input_text.smoke_alarm_source`
- **Remaining silence time:** Consult `timer.smoke_source_block`

## 🔐 Security

- The system cannot be completely disabled by accident
- In case of persistent smoke, the alarm reactivates automatically after the silence period
- The startup watchdog ensures that an alert is not lost during a restart
- Logical inconsistency detection prevents prolonged false positives

## 📝 Technical Notes

### MQTT Command Format

Frient detectors use the following Zigbee2MQTT topics:
- **Topic:** `zigbee2mqtt/[DEVICE_NAME]/set`
- **Payload for alarm:**
  ```json
  {
    "warning": {
      "duration": 60,
      "level": "low",
      "mode": "burglar",
      "strobe": false,
      "strobe_duty_cycle": 0
    }
  }
  ```
- **Payload for stop:**
  ```json
  {
    "alarm": "OFF"
  }
  ```

### Available Alarm Modes

- `burglar`: Burglary mode (used here)
- `fire`: Fire mode
- `emergency`: Emergency mode
- `police_panic`: Police panic mode
- `fire_panic`: Fire panic mode
- `emergency_panic`: Emergency panic mode

## 📄 Licence

MIT Licence - See the [LICENSE](LICENSE) file for more details.

## 👤 Author

Rayman223

## 🤝 Contributing

Contributions are welcome! Feel free to open an issue or pull request.

---

**⚠️ Important:** This system is designed to enhance safety, but does not replace a professional fire alarm system. Ensure you comply with local fire safety regulations.
