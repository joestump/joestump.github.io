---
title: "Per-Cushion Couch Presence"
date: 2023-05-07T07:19:55-07:00
draft: false
---

{{< toc >}}

## Overview

The main part of my house is a large, open space with various lights throughout. When my partner and I sit on the couch to watch TV or game, some of these lights are annoying on our periphery. To date, I've not had luck with motion sensors. I find they're either too sensitive (e.g. tripping when birds fly by or plants move) or not sensitive enough. This area of my house is also very high traffic so using motion as the sole signal for automation results in the automations running too frequently.

Thankfully, with a little solder and some car seat sensors from AliExpress, I was able to get a decent setup going.

### Hardware

* [CITALL Universal Car Seat Pressure Sensor](https://www.aliexpress.us/item/2255799962608724.html)
* [Aqara Zigbee Door and Window Sensor](https://www.amazon.com/Aqara-MCCGQ11LM-Window-Sensor-White/dp/B07D37VDM3/)

## Assembling the Hardware

### Soldering the Sensor

All we need to do is solder the black and red wire to either side of the contact switch on the Aqara Zigbee sensor. I followed these steps when assembling mine:

 1. Open up the Aqara sensor with a small flathead screwdriver. Pull out the sensor/battery from the white case.
 2. Cut the black connector off of the car seat sensor.
 3. Strip back about 1/4" on both wires coming off the car seat sensor.
 4. Make small hooks with the exposed wire and hook them around either side of the contact sensor of the now bare Aqara sensor.
 5. Solder the wires. I positioned them running along either side of the board so they are pointed out the side of the case opposite of the small white Zigbee connect/reset button on the Aqara sensor's case.
 6. Drill a 1/8" hole on the opposite side as the small white button of the Aqara sensor's case.
 7. Use a utility knife to cut out the top of the hole to create a small channel for the wires.
 8. Snap everything back together.

![Soldered Aqara Sensor](/images/hass/couch-cushions/soldering-layout.jpeg)

It doesn't matter which side the red/black wires go on. Once you've put everything back together you should have something that looks like this:

![Soldered Aqara Sensor](/images/hass/couch-cushions/finished-sensor.jpeg)

### Placing Sensors in the Cushions

In each of my couch cushions I unzipped the cushion and put the sensor between the top fabric and the cushion itself. My couch's fabric is thick enough that you can't tell that the sensor is under it without feeling around for it. Visually, I can't detect them at all.

![Cushion Sensor](/images/hass/couch-cushions/sensor-in-cushion.jpeg)

## Configuring Home Assistant

### Pairing the Sensor

The Aqara Zigbee sensor pairs with Home Assistant like any other Zigbee device. Just press the pairing button and add it using your preferred Zigbee integration with Home Assistant.

### Create Binary Template Sensors

The default contact switch settings make it appear as though you are away when sitting on the cushion and present when not. To correct this, I created template binary sensors for each of the four cushions on my couch.

{{< highlight yml >}}
# In my templates.yaml file
- binary_sensor:
    - name: "Couch Presence 1"
      device_class: occupancy
      icon: mdi:account
      unique_id: couch_presence_1
      state: >-
        {% if states('binary_sensor.couch_cushion_1') == 'on' %}off{% else %}on{% endif %}
- binary_sensor:
    - name: "Couch Presence 2"
      device_class: occupancy
      icon: mdi:account      
      unique_id: couch_presence_2
      state: >-
        {% if states('binary_sensor.couch_cushion_2') == 'on' %}off{% else %}on{% endif %}
- binary_sensor:
    - name: "Couch Presence 3"
      device_class: occupancy
      icon: mdi:account      
      unique_id: couch_presence_3
      state: >-
        {% if states('binary_sensor.couch_cushion_3') == 'on' %}off{% else %}on{% endif %}
- binary_sensor:
    - name: "Couch Presence 4"
      device_class: occupancy
      icon: mdi:account      
      unique_id: couch_presence_4
      state: >-
        {% if states('binary_sensor.couch_cushion_4') == 'on' %}off{% else %}on{% endif %}
{{< / highlight >}}

After saving `templates.yaml` reload the YAML via the Developer menu in Home Assistant. Once HASS has reloaded the template entities, you should see a new entity for your couch cushion presence that correctly displays presence when pushing down on the car seat sensor.

![Cushion Presence](/images/hass/couch-cushions//couch-presense-template.png)

### Count of People on the Couch
 
The last sensor I wanted was a simple count of how many cushions detected presence. The following YAML in my `templates.yaml` creates an entity that returns 0-4 depending on how many of the car seat sensors in my cushions detect presence.

{{< highlight yml >}}
# In my templates.yaml file
- sensor:
    - name: "Number of People on the Couch"
      unique_id: number_of_people_on_the_couch
      icon: mdi:account-group      
      state: >-
        {{ [
          states('binary_sensor.couch_presence_1') == 'on',
          states('binary_sensor.couch_presence_2') == 'on', 
          states('binary_sensor.couch_presence_3') == 'on', 
          states('binary_sensor.couch_presence_4') == 'on', 
        ] | sum}}
{{< / highlight >}}

### Automating the TV

I already have my TVs wired into Home Assistant. I use the [Universal Media Player](https://www.home-assistant.io/integrations/universal/) to tie my SONOS, Apple TV, and LG TV together into a single compoenent that I use in my various HASS automations. Once the `sensor.number_of_people_on_the_couch` was working, it was trivial to turn on my TV when I sat down at the couch with the following automation:

{{< highlight yml >}}
- id: sat_on_the_couch
  alias: Someone sat on the couch
  trigger:
    platform: numeric_state
    entity_id: sensor.number_of_people_on_the_couch
    above: 0
  action:
    - service: media_player.turn_on
      data:
        entity_id: media_player.left_tv
{{< / highlight >}}
