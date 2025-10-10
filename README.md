# Home Assistant Configuration

This is my [Home Assistant](https://www.home-assistant.io) configuration for my smart home!  I've been a Home Assistant user since 2019 and have continued to grow my home's capabilities as we add more devices.

#### Table of Contents
1. [Platform](#platform)
1. [Integrations](#integrations)
  1. [Remote](#remote)
  1. [Template](#template)

## Platform

Home Assistant is deployed with the following hardware and software:
- Apple Mac Pro 6,1
- VMWare ESXi 7
- [Home Assistant OVA image for VMWare ESXi/vSphere](https://www.home-assistant.io/installation/alternative)

## Devicess

### Remotes
- [Broadlink RM4 Pro IR/RF Remote](https://a.co/d/j1mcWRS)

## Integrations

### Remote
The remote integration is used to expose IR and RF devices to Home Assistant and the Google Assistant.

For example, our [Duraflame electric fireplace](https://a.co/d/fFEitIg) is controlled by calling the `input_button` entities, which then trigger automations to send the base64 encoded commands via the `remote.send_command` call.

The `input_button` entities are also exposed to Google Assistant.  They can then be used in custom routines to be called by a voice command to the Google Assistant.  `input_button` entities from Home Assistant appear as scenes to Google Assistant.

![Custom routine to call the input_button as a scene with the Google Assistant](./docs/images/remote_google_assistant.png)

### Template
Multiple types of sensors are made available using the template integration.  In this configuraiton, multiple files are used to logically separate the template sensors based on their purpose.  This is accomplished using Home Assistant's [ability to split configurations](https://www.home-assistant.io/docs/configuration/splitting_configuration/#advanced-usage).

- [calendars.yaml](./templates/calendars.yaml) Important recurring dates are stored in Local Calendar events and sensors are created to represent the number of days until that particular event.  Previously this was accomplished using [Garbage Collection](https://github.com/bruxy70/Garbage-Collection) by @bruxy70.
