# Home Assistant Configuration

This is my [Home Assistant](https://www.home-assistant.io) configuration for my smart home!  I've been a Home Assistant user since 2019 and have continued to grow my home's capabilities as more devices are added.

#### Table of Contents
1. [Platform](#platform)
1. [Integrations](#integrations)
  1. [Remote](#remote)
  1. [Template](#template)

## Platform

My Home Assistant instance is deployed on Docker.  I switched over from the OVA image after moving other services from virtual machines to containers.

## Devicess

| Device                                                   | Entities                  |
| -------------------------------------------------------- |-------------------------- |
| [Broadlink RM4 Pro IR/RF Remote](https://a.co/d/j1mcWRS) | `remote.bedroom_1_remote` |
| Zooz Zen30 Dimmer Switch                                 | `light.bedroom_1_dimmer`  |

## Automations

Here are a few interesting automations used in this instance.

### Ceiling Fan Control via RF Commands and a Wall Dimmer Switch

This configuration allows a Zooz Zen30 dimmer to control a 6-speed ceiling fan through Home Assistant using RF commands sent by a script.
Both the Home Assistant UI and the physical dimmer paddles remain fully synchronized regardless of where the fan is controlled.

#### Components
| Entity                                      | Description                                               |
| ------------------------------------------- | --------------------------------------------------------- |
| `input_boolean.bedroom_1_ceiling_fan_power` | Tracks the fan’s on/off state                             |
| `input_number.bedroom_1_ceiling_fan_speed`  | Tracks the fan’s current speed (1–6)                      |
| `light.bedroom_1_dimmer`                    | Zen30 dimmer entity used as the physical fan controller   |
| `fan.bedroom_1_ceiling_fan`                 | Template fan entity combining the helpers above           |
| `script.bedroom_1_ceiling_fan`              | Sends RF commands to the fan via `remote.bedroom_1_remote`|

#### Logic Overview

1. Template Fan Entity
  - Defines a 6-speed fan entity whose state and speed are derived from the helper entities.
  - Turning the fan on/off updates `input_boolean.bedroom_1_ceiling_fan_power`.
  - Setting a speed updates `input_number.bedroom_1_ceiling_fan_speed`.
  - The fan entity reflects any helper updates, keeping it in sync with the dimmer and RF control script.

2. Dimmer to Fan Automation
Monitors the Zen30 dimmer (light.bedroom_1_dimmer) and maps its state and brightness to fan speed and power.
  - Turning the dimmer off stops the fan.
  - Turning the dimmer on restores the last known fan speed.
  - Adjusting brightness (0–255) changes the fan speed between 1–6.
  - Built-in conditions prevent feedback loops or redundant updates.

3. Helpers to Dimmer Sync
A companion automation keeps the Zen30 dimmer’s LEDs and brightness aligned with fan activity.
  - If the fan is turned on/off in the UI, the dimmer’s LEDs mirror that state.
  - Adjusting the fan speed from the UI updates the dimmer brightness to match.

4. RF Fan Script
`script.bedroom_1_ceiling_fan` translates the helper states into RF commands using `remote.bedroom_1_remote`, ensuring the physical fan speed and power state always match Home Assistant.

#### Behavior Summary
| Action                             | Result                                             |
| ---------------------------------- | -------------------------------------------------- |
| Turn on fan in Home Assistant      | Fan powers on, dimmer LED turns on                 |
| Change fan speed in Home Assistant | Fan speed adjusts (1–6), dimmer brightness matches |
| Press Zen30 paddle **on**          | Fan turns on at last known speed                   |
| Adjust dimmer brightness           | Fan speed updates proportionally                   |
| Press Zen30 paddle **off**         | Fan stops and UI reflects “off”                    |

### Virtual Thermostat

[`automations/living_room_thermostat_control_for_virtual_thermostat.yaml`](./automations/living_room_thermostat_control_for_virtual_thermostat.yaml)

This automation and supporting components controls a single Honeywell T6 thermostat on a single zone HVAC
system.  It uses temperature sensors throughout the house to actuate the physical T6 thermostat to more
effectively regulate the temeprature.

#### Components

| Component                      | Entity / Name                                                       | Purpose                                                                      |
| ------------------------------ | ------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Virtual Thermostat             | `climate.virtual_thermostat`                                        | Primary controller that determines desired HVAC behavior (mode and setpoint) |
| Physical Thermostat (Actuator) | `climate.living_room_thermostat`                                    | Honeywell T6 Pro used strictly to start/stop heating or cooling              |
| Virtual Temperature Sensor     | `sensor.virtual_thermostat_temperature`                             | Aggregated or selected room temperature used for control decisions           |
| Room Selector                  | `input_select.virtual_thermostat_temperature_select`                | Allows the user to choose which room/floor temperature drives the system     |
| Deadband (Normal)              | `input_number.virtual_thermostat_deadband`                          | Temperature buffer used during normal operation                              |
| Deadband (Eco)                 | `input_number.virtual_thermostat_eco_deadband`                      | Larger buffer used when Eco mode is enabled                                  |
| Eco Mode Toggle                | `input_boolean.virtual_thermostat_eco_mode`                         | Enables relaxed temperature control to reduce cycling and energy use         |
| Min Runtime (Heat)             | `input_number.virtual_thermostat_heat_runtime_minimum`              | Minimum heating run time before the system may turn off                      |
| Max Runtime (Heat)             | `input_number.virtual_thermostat_heat_runtime_maximum`              | Maximum allowed continuous heating run                                       |
| Min Runtime (Cool)             | `input_number.virtual_thermostat_cool_runtime_minimum`              | Minimum cooling run time before the system may turn off                      |
| Max Runtime (Cool)             | `input_number.virtual_thermostat_cool_runtime_maximum`              | Maximum allowed continuous cooling run                                       |
| Lockout Time                   | `input_number.virtual_thermostat_lockout_time`                      | Minimum off-time between completed runs to prevent short cycling             |
| Run End Timestamp Sensor       | `sensor.living_room_thermostat_last_change`                         | Records when the thermostat transitions to `off` (end of a run)              |
| Control Automation             | Automation: *Living Room Thermostat Control for Virtual Thermostat* | Executes the control logic and issues commands to the T6                     |

#### Logic Overview

This setup implements a **virtual thermostat–driven HVAC control system** where Home Assistant owns all decision-making, and the Honeywell T6 Pro functions solely as an actuator.

The virtual thermostat provides:

* Desired HVAC mode (`heat`, `cool`, or `off`)
* Desired setpoint temperature
* Eco mode state

A virtual temperature sensor supplies the actual temperature used for control, which may represent a specific room or aggregated area selected via a dashboard helper.

The automation continuously evaluates:

* Current temperature vs setpoint
* Active HVAC mode
* Deadband (normal or eco)
* Minimum and maximum run-time limits
* Lockout status based on the last completed run

When heating or cooling is needed, the automation:

* Turns the T6 **on** in the appropriate mode
* Forces the T6 to an extreme setpoint (high for heat, low for cool)

When conditions are met to stop (setpoint reached, max runtime exceeded, or virtual thermostat turned off), the automation:

* Turns the T6 **off**
* Starts a lockout timer using the timestamp sensor

The system never relies on `hvac_action`. Instead, it uses **explicit mode changes** (`on → off`) as authoritative run boundaries.

#### Behavior Summary

| Action                                                                                                      | Result                                                                                                      |
| ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Virtual thermostat is set to **heat** and the selected room temperature falls below `(setpoint − deadband)` | The living room thermostat is turned **on in heating mode** and forced to a high setpoint to deliver heat   |
| Virtual thermostat is set to **cool** and the selected room temperature rises above `(setpoint + deadband)` | The living room thermostat is turned **on in cooling mode** and forced to a low setpoint to deliver cooling |
| Temperature reaches the virtual thermostat setpoint **and** the minimum runtime has elapsed                 | The living room thermostat is turned **off**, ending the current run                                        |
| Heating or cooling runtime exceeds the configured **maximum runtime**                                       | The living room thermostat is **forcibly turned off** to prevent excessive run times                        |
| Living room thermostat transitions to **off**                                                               | A run is considered complete and the **lockout timer starts**                                               |
| Lockout timer is active after a completed run                                                               | The system **will not start** a new heating or cooling cycle                                                |
| Eco mode is enabled                                                                                         | A larger deadband is applied, reducing HVAC cycling and allowing greater temperature variation              |
| Virtual thermostat mode is set to **off** while the T6 is running                                           | The thermostat is turned **off** once the minimum runtime is satisfied                                      |
| Minimum runtime has not yet elapsed                                                                         | The thermostat **continues running**, even if the setpoint has already been reached                         |
| Lockout period has expired and heating or cooling is needed                                                 | The system is allowed to **start a new run**                                                                |
