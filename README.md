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

### Bedroom Ceiling Fan Control (via Zen30 Dimmer + Template Fan)

This configuration allows a Zooz Zen30 dimmer to control a 6-speed ceiling fan through Home Assistant using RF commands sent by a script.
Both the Home Assistant UI and the physical dimmer paddles remain fully synchronized — no matter where the fan is controlled from.

#### ⚙️ Components
| Entity                                      | Description                                               |
| ------------------------------------------- | --------------------------------------------------------- |
| `input_boolean.bedroom_1_ceiling_fan_power` | Tracks the fan’s on/off state                             |
| `input_number.bedroom_1_ceiling_fan_speed`  | Tracks the fan’s current speed (1–6)                      |
| `light.bedroom_1_dimmer`                    | Zen30 dimmer entity used as the physical fan controller   |
| `fan.bedroom_1_ceiling_fan`                 | Template fan entity combining the helpers above           |
| `script.bedroom_1_ceiling_fan`              | Sends RF commands to the fan via `remote.bedroom_1_remote`|

#### 🧩 Logic Overview

1. Template Fan Entity
  - Defines a 6-speed fan entity whose state and speed are derived from the helper entities.
  - Turning the fan on/off updates `input_boolean.bedroom_1_ceiling_fan_power`.
  - Setting a speed updates `input_number.bedroom_1_ceiling_fan_speed`.
  - The fan entity reflects any helper updates, keeping it in sync with the dimmer and RF control script.

2. Dimmer → Fan Automation
Monitors the Zen30 dimmer (light.bedroom_1_dimmer) and maps its state and brightness to fan speed and power.
  - Turning the dimmer off stops the fan.
  - Turning the dimmer on restores the last known fan speed.
  - Adjusting brightness (0–255) changes the fan speed between 1–6.
  - Built-in conditions prevent feedback loops or redundant updates.

3. Helpers → Dimmer Sync
A companion automation keeps the Zen30 dimmer’s LEDs and brightness aligned with fan activity.
  - If the fan is turned on/off in the UI, the dimmer’s LEDs mirror that state.
  - Adjusting the fan speed from the UI updates the dimmer brightness to match.

4. RF Fan Script
`script.bedroom_1_ceiling_fan` translates the helper states into RF commands using `remote.bedroom_1_remote`, ensuring the physical fan speed and power state always match Home Assistant.

#### 🔄 Behavior Summary
| Action                             | Result                                             |
| ---------------------------------- | -------------------------------------------------- |
| Turn on fan in Home Assistant      | Fan powers on, dimmer LED turns on                 |
| Change fan speed in Home Assistant | Fan speed adjusts (1–6), dimmer brightness matches |
| Press Zen30 paddle **on**          | Fan turns on at last known speed                   |
| Adjust dimmer brightness           | Fan speed updates proportionally                   |
| Press Zen30 paddle **off**         | Fan stops and UI reflects “off”                    |

