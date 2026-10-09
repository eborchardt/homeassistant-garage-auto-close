# Home Assistant Blueprints

Home Assistant automation blueprints by cmstone.

## Included blueprints

### Garage Auto Close

A Home Assistant automation blueprint for automatically closing a garage door after a visible timer expires.

Activity sensors reset the countdown timer. Obstruction sensors prevent auto-close if active and can trigger the same optional notification action used for successful close notifications.

### Indoor Camera Privacy Manager

A Home Assistant automation blueprint for managing indoor camera privacy based on presence and sleep state.

Privacy is enabled when any selected presence entity is present and the household is awake, disabled when all selected presence entities are absent, and disabled while sleeping so the camera can record overnight. Sleep mode is determined by both a configured time window and at least one active sleep indicator, such as a white noise machine. Optional notifications can be sent whenever the blueprint changes privacy mode, including what changed and what triggered it.

## Features

### Garage Auto Close

- Auto-close a garage door after a timer helper finishes
- Uses an existing Home Assistant `timer` helper for visible countdown state
- Activity sensors reset/restart the timer while the garage door is open
- Obstruction sensors block auto-close when active
- Hold-open helper pauses auto-close indefinitely until it is turned off
- Optional repeating reminder while the door is held open
- Optional notification action for obstruction, successful auto-close, and hold-open reminder events
- Works with standard Home Assistant `cover` entities, including garage door integrations such as ratgdo

### Indoor Camera Privacy Manager

- Enables camera privacy when any selected presence entity is present and the household is awake
- Disables camera privacy only when all selected presence entities are absent
- Disables privacy while sleeping so recording can run overnight
- Requires both a sleep time window and an active sleep indicator
- Supports a white noise machine, sleep helper, bed sensor, fan, or similar entity as the sleep indicator
- Supports optional manual override, guest mode, and vacation mode helpers
- Supports optional notifications for privacy mode changes
- Includes notification variables for privacy state, recording state, trigger reason, trigger entity, from/to state, and timestamp
- Uses action selectors so it can work with many camera platforms and helper entities

## Repository structure

```text
blueprints/
  automation/
    cmstone/
      garage_auto_close.yaml
      indoor_camera_privacy_manager.yaml
.github/
  ISSUE_TEMPLATE/
    bug_report.yml
    feature_request.yml
.gitignore
LICENSE
README.md
```

## Installation

### Option 1: Import from GitHub

Use Home Assistant's blueprint import feature with this repository URL:

```text
https://github.com/cmstone/homeassistant-garage-auto-close
```

### Option 2: Manual install

Copy the desired blueprint file into your Home Assistant configuration directory:

```text
blueprints/automation/cmstone/garage_auto_close.yaml
blueprints/automation/cmstone/indoor_camera_privacy_manager.yaml
```

Then reload automations or restart Home Assistant.

## Garage Auto Close required helpers

Create a timer helper before using the garage auto-close blueprint:

1. Go to **Settings → Devices & services → Helpers**.
2. Create a **Timer** helper.
3. Set the timer duration to the desired auto-close delay.
4. Select that helper in the blueprint automation.

The blueprint starts this timer whenever the door opens or activity is detected while the door remains open.

## Garage Auto Close hold-open

The blueprint requires an `input_boolean` helper that holds the door open:

1. Go to **Settings → Devices & services → Helpers**.
2. Create a **Toggle** (`input_boolean`) helper, for example `Garage Hold Open`.
3. Select it as the **Hold Open Helper** in the blueprint automation.

While the helper is **on**, auto-close is paused and the door stays open
indefinitely. Turn it **off** to resume auto-close; if the door is still open
at that point the countdown restarts. Expose the helper as a dashboard toggle,
or flip it from a physical button with your own automation.

Set **Hold Open Reminder (minutes)** above `0` to run the notification action
when the door is still open that many minutes after the hold engaged, repeating
every interval until the hold is released or the door closes. Activity sensors
do not reset this reminder, so it measures the absolute time the door has been
open. Leave it at `0` to disable the reminder.

> Upgrading from an earlier version: the **Hold Open Helper** is a new required
> input. After updating the blueprint, open the automation and select the
> helper, or the automation will report a missing input.

### Driving the hold from a button

The hold state lives in the `input_boolean`, because a button entity is
momentary and cannot hold state. To control it from a button, point the button
at the helper.

On a dashboard, a toggle or button card targeting the helper is enough:

```yaml
type: button
entity: input_boolean.garage_hold_open
name: Hold Garage Open
icon: mdi:garage-alert
tap_action:
  action: toggle
```

To drive it from a physical button (any device that exposes a `button`,
`binary_sensor`, or event), add a small automation that toggles the helper:

```yaml
alias: Garage hold-open button
trigger:
  - platform: state
    entity_id: binary_sensor.garage_hold_button
    to: "on"
action:
  - service: input_boolean.toggle
    target:
      entity_id: input_boolean.garage_hold_open
mode: single
```

Replace the entity IDs with your own. Use `input_boolean.turn_on` instead of
`toggle` if you want the button to only engage the hold and rely on release
elsewhere.

## Indoor Camera Privacy Manager inputs

| Input | Description |
| --- | --- |
| Presence Entities | People, device trackers, or binary sensors. Any selected entity active means someone is home/present. All selected entities must be inactive to be considered away/absent |
| Sleep Start Time | Start of the sleep window |
| Sleep End Time | End of the sleep window |
| Sleep Indicator Entities | White noise machine, sleep helper, bed sensor, fan, or similar |
| Enable Privacy Action | Action to black out, cover, disable, or otherwise make the indoor camera private |
| Disable Privacy Action | Action to uncover, enable, or otherwise make the camera available |
| Enable Recording Action | Optional action to enable recording, detections, or alerts |
| Disable Recording Action | Optional action to disable recording, detections, or alerts |
| Notification Action | Optional action sequence for notifications |
| Away Delay | Delay after all selected presence entities are absent before privacy is disabled |
| Manual Override Helper | Optional input_boolean that pauses automation while on |
| Guest Mode Helper | Optional input_boolean that always keeps privacy enabled while on |
| Vacation Mode Helper | Optional input_boolean that always disables privacy and enables recording while on |

## Indoor camera behavior

| State | Privacy | Recording |
| --- | --- | --- |
| Any presence entity present and awake | Enabled | Disabled, if configured |
| All presence entities absent | Disabled | Enabled, if configured |
| Sleeping | Disabled | Enabled, if configured |
| Guest mode | Enabled | Disabled, if configured |
| Vacation mode | Disabled | Enabled, if configured |

## Indoor camera notification action

The privacy blueprint exposes these variables before running the optional notification action:

| Variable | Description |
| --- | --- |
| `notification_title` | Suggested title for the notification |
| `notification_event` | Machine-readable event name, such as `home_awake`, `away`, `sleeping`, `guest_mode`, or `vacation_mode` |
| `notification_message` | Human-readable notification text |
| `privacy_state` | Final privacy state, either `enabled` or `disabled` |
| `recording_state` | Final recording state, either `enabled` or `disabled` |
| `trigger_reason` | Reason category, such as `any_present`, `all_absent`, `sleep`, `guest_mode`, or `vacation_mode` |
| `trigger_entity` | Entity ID that triggered the automation, when available |
| `trigger_name` | Friendly name of the triggering entity, when available |
| `trigger_from` | Previous state of the triggering entity, when available |
| `trigger_to` | New state of the triggering entity, when available |
| `notification_timestamp` | Timestamp when the notification variables were created |

Example action:

```yaml
- service: notify.mobile_app_your_phone
  data:
    title: "{{ notification_title }}"
    message: >
      {{ notification_message }}

      Privacy: {{ privacy_state }}
      Recording: {{ recording_state }}
      Reason: {{ trigger_reason }}
      Trigger: {{ trigger_name }}{% if trigger_entity %} ({{ trigger_entity }}){% endif %}
      From: {{ trigger_from }}
      To: {{ trigger_to }}
      Time: {{ notification_timestamp }}
```

## Garage notification action

The garage blueprint exposes a `notification_message` variable before running the optional notification action.

Example action:

```yaml
- service: notify.mobile_app_your_phone
  data:
    title: Garage Door
    message: "{{ notification_message }}"
```

The same notification action is used for:

- Auto-close prevented by obstruction
- Garage door automatically closed
- Garage door still held open (reminder interval)

## Safety note

Garage doors can cause property damage or injury. Use the garage blueprint only with properly functioning garage door safety sensors and obstruction detection. Test thoroughly before relying on the automation.

For indoor cameras, verify the selected privacy and recording actions against your specific camera integration before relying on the automation.

## License

MIT
