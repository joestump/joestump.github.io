---
title: "ESP8266 Thermostat with ESPHome"
date: 2022-10-16T07:19:55-07:00
draft: false
---

{{< toc >}}

## Overview

I've been planning to replace my Nest thermostat with a local-only option since moving into my house. I'd been thinking I would just use a ZWave or Zigbee option, however I also wanted to install a wall-mounted tablet for Home Assistant and the best place to put it was where the Nest was currently mounted. I started poking around for a "headless" thermostat and settled on the [Newgoal 4-Channel Tuya board](https://a.co/d/eiEmhdV).

![Newgoal 4-Channel Tuya board](/images/hass/esp8266-thermostat/esp8266-tuya-board.jpg)

### Parts List

* [Newgoal 4-Channel Tuya board](https://a.co/d/eiEmhdV)
* [HiLetgo CP2102 USB 2.0 Serial Converter](https://a.co/d/h8Unhqz) – This is required to flash ESPHome onto the device over UART. Once flashed with ESPHome, the device can be updated OTA.
* [6 pin male header](https://a.co/d/2PtlZbm) – In order to connect the CP2102 to the board, we will need to solder a 6 pin header to the board.
* [Thermostat wire](https://a.co/d/9gPl1qj)

### Software

* [Home Assistant](https://www.home-assistant.io) – Used for the temperature sensor and to control the thermostat.
* [ESPHome Integration](https://www.home-assistant.io/integrations/esphome/) – We will use this to flash/configure the board.
* [Tuya MCU ESPHome Component](https://esphome.io/components/tuya.html) – The board uses a Tuya MUC. We will use this component to configure the 4 relays as [Tuya switches](https://esphome.io/components/switch/tuya.html).
* [Thermostat ESPHome Component](https://esphome.io/components/climate/thermostat.html) – Manages the relays based on the thermostat readings. This is what gets picked up by Home Assistant as a [`climate` entity](https://developers.home-assistant.io/docs/core/entity/climate/). 
* [CP210x Drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers?tab=downloads) – I use MacOS and used the OSX driver. 

## Wiring the Board for Flashing

The board does not have a boot button, so flashing over USB didn't work. In order to flash the board over UART with the CP2102, I had to do a bit of soldering.

### Solder the Header Pins

The board has a spot to solder 6 pins into `G` (Ground), `V` (Power), `R` (Rx), `T` (Tx), `D`, and `RST` (Reset). I soldered a 6 pin male header to the outlined spot on the board. We will use these pins to connect the CP2102 USB serial converter to.

![Header Pings](/images/hass/esp8266-thermostat/header-pins-esp8266.png)

### Wiring the CP2102

The CP2102 I bought came with a 4 pin wire. This will need to be modified with a jumper from `GND` that we can use to attach to the `D` pin on the board.

![Header Pings](/images/hass/esp8266-thermostat/cp2102-wiring.jpeg)

We can now connect the CP2102 to the board with the following pin out:

* CP2102 `GND` → `G` and `D` pins
* CP2102 `TXD` → `R` pin
* CP2102 `RXD` → `T` pin
* CP2102 `3V3` → `V` pin

**NOTE:** This did not provide me with enough power to flash the device with ESPHome. In addition to the above, I also connected the board to USB power.

## Wiring the Relays to the HVAC System

My HVAC system at home was wired as follows:

* `Rh` – 24V power
* `B` – Neutral
* `Y` – Air Conditioner
* `G` – Fan-only
* `W` – Heat Stage 1
* `B` – Heat Stage 2

Below is a quick and dirty diagram of what my wiring looks like:

![Header Pings](/images/hass/esp8266-thermostat/wiring-diagram.png)

## Flashing ESPHome

Once you've put the solder gun away, it's time to flash ESPHome. I did this in Home Assistant using the ESPHome integration using Chrome. This worked like all of my other ESP32s except there's no boot button to press on connect.

### ESPHome Configuration

The following is a basic configuration that will expose the Tuya MCU once flashed.

{{< highlight yml >}}
esphome:
  name: living-room-thermostat

# Make sure to use an ESP8266 board.
esp8266:
  board: esp8285

api:
  encryption:
    key: "0GamOcw/LFuS2HYcvVRkj88OUl17WEH/Grf9G3nWdxI="

ota:
  password: "45c182089a39e49022f4316cad690a2f"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

captive_portal:

# This MUST be set to 0 in order for the UART bus configured
# below to properly work.
logger:
  baud_rate: 0

# The Tuya MCU communicates over the UART bus.
uart:
  rx_pin: GPIO3
  tx_pin: GPIO1
  baud_rate: 9600

# Register the Tuya MCU connection.
tuya:
{{< / highlight >}}

### Expected Log Output

Once you've flashed the board, you can connect to it normally OTA to get the log output. On boot, you should see the Tuya component output some logs that look something like this:

{{< highlight sh >}}
[15:16:28][C][tuya:033]: Tuya:
[15:16:28][C][tuya:048]:   Datapoint 1: switch (value: OFF)
[15:16:28][C][tuya:048]:   Datapoint 2: switch (value: OFF)
[15:16:28][C][tuya:048]:   Datapoint 3: switch (value: OFF)
[15:16:28][C][tuya:048]:   Datapoint 4: switch (value: OFF)
[15:16:28][C][tuya:050]:   Datapoint 7: int value (value: 0)
[15:16:28][C][tuya:050]:   Datapoint 8: int value (value: 0)
[15:16:28][C][tuya:050]:   Datapoint 9: int value (value: 0)
[15:16:28][C][tuya:050]:   Datapoint 10: int value (value: 0)
[15:16:28][C][tuya:048]:   Datapoint 13: switch (value: OFF)
[15:16:28][C][tuya:048]:   Datapoint 101: switch (value: ON)
[15:16:28][C][tuya:054]:   Datapoint 102: enum (value: 2)
[15:16:28][C][tuya:050]:   Datapoint 103: int value (value: 5)
[15:16:28][C][tuya:062]:   GPIO Configuration: status: pin 13, reset: pin 0
[15:16:28][C][tuya:068]:   Product: '{"p":"waq2wj9pjadcg1qc","v":"1.0.0","m":0}'
{{< / highlight >}}

## Configuring ESPHome Climate Component

The [thermostat climate controller component](https://esphome.io/components/climate/thermostat.html) in ESPHome will require a few things be configured:

* Each relay needs to be exposed as a `switch` component.
* We need a `sensor` that has a `device_class` of `temperature`.
* A `climate` component that calls our relays and consumes our temperature sensor.

**NOTE:** Because I planned to put this device into a recessed receptacle, and it does not appear to have easily accessible GPIO pins, I decided to use the [`homeassistant` component in ESPHome](https://esphome.io/components/sensor/homeassistant.html) to consume temperature sensors I already had. I have a bunch of Zigbee and BLE temperature sensors that I expose using [ha-average](https://github.com/Limych/ha-average) sensor.

{{< highlight yml >}}
# Set up each of our relays. 
switch:
- platform: "tuya"
  id: fan_only
  name: "Fan Only"
  switch_datapoint: 1
- platform: "tuya"
  id: air_cond
  name: "Air Conditioning"
  switch_datapoint: 2
- platform: "tuya"
  id: heat_stage_2
  name: "Heat Stage 2"
  switch_datapoint: 3
- platform: "tuya"
  id: heat_stage_1
  name: "Heat Stage 1"
  switch_datapoint: 4

sensor:
- platform: homeassistant
  id: current_temperature
  entity_id: sensor.upstairs_home_temperature 
  device_class: temperature
  state_class: measurement
  filters:
    # The sensor I consume from HASS is in F. This converts
    # it back to C so it works correctly with the climate component.
    - lambda: return (x - 32) * (5.0/9.0);

# Dual-point configuration entry. This consumes the sensor above
# and uses it to control the relay switches.
climate:
- platform: thermostat
  name: "Living Room Thermostat"
  sensor: current_temperature
  default_mode: heat_cool
  default_target_temperature_low: 20 °C
  default_target_temperature_high: 22 °C
  supplemental_heating_delta: 3 °C
  startup_delay: True
  min_fanning_run_time: 300s
  min_cooling_off_time: 300s
  min_cooling_run_time: 300s
  min_heating_off_time: 300s
  min_heating_run_time: 300s
  max_heating_run_time: 1800s
  min_idle_time: 30s
  min_fanning_off_time: 1800s
  cool_action:
    - switch.turn_on: air_cond
  heat_action:
    - switch.turn_on: heat_stage_1
  supplemental_heating_action:
    - switch.turn_on: heat_stage_2
  fan_only_action:
    - switch.turn_on: fan_only
  idle_action:
    - switch.turn_off: air_cond
    - switch.turn_off: heat_stage_1
  preset:
    - name: away
      default_target_temperature_low: 18 °C 
      default_target_temperature_high: 20 °C
      mode: heat_cool
{{< / highlight >}}
