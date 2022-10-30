---
title: "ESP32, BLE, RF Plant Sensor Roundup and Home Assistant Dashboard"
date: 2022-10-16T07:19:55-07:00
draft: false
---

{{< toc >}}

## Overview

My partner and I have quite a few plants. I've been wanting to add a sensor network to them so we could more closely monitor their health. My requirements were  as follows:

* Long battery life. I didn't want wires running to every plant and I didn't want to be changing batteries every week.
* Must-have sensors were soil moisture and ideally fertilization rate/level.

Nice-to-haves:

* Additional sensors for things like illumination, temperature, etc.
* An auto-watering system. 

## Contenders

| Model | Price | Soil Moisture | Soil Fertilization | Other Sensors | Protocols | Notes |
| ---- | ----| ---- | ---- | ---- | ---- | ---- | 
| [Ecowitt WH51 Wireless Soil Moisture Sensor](https://www.ecowitt.com/shop/goodsDetail/19) | `$16.99` | ✅ | ❌ | Temperature, Humidity | RF | Requires a base station. Each base station can only handle 8 sensors. |
| [Xioami Mijia BLE Flower Monitor](https://www.aliexpress.us/item/2255800399707249.html) | `$12-20` | ✅ | ✅ | Temperature, Luminosity | BLE | There are *two* versions of this sensor. It's important to get the `HHCCJCY01` rather than the `HHCCJCY10` |
| [ESP32 Soil Moisture Sensor](https://www.aliexpress.us/item/3256803006424429.html) | `$10-15` | ✅ | ❌ | Temperature, Humidity | WiFi, BLE | Battery doesn't last very long. Max is 5-7 days. Deep sleep support required soldering. |
| [LILYGO TTGO T-Higrow ESP32 Soil Sensor](http://www.lilygo.cn/prod_view.aspx?TypeId=50033&Id=1172) | `$9.85` | ✅ | ✅ | Temperature, Humidity, Luminosity | WiFi, BLE | They offer a nice case and a [water pump attachment](https://www.aliexpress.us/item/3256803314634205.html). |

## Setup 

I decided to go with a blended setup: 

* **3x LILYGO TTGO T-Higrow ESP32s** – These will form the backbone of the system by being the proxies for the BLE sensors and watering the plants. I am running full-time power to these and treating them as "zones" essentially.
* **Xioami Mijia BLE Flower Monitor** – These are powered CR2032 batteries that last a year. They ended up having the most sensors, the most _effective_ sensors, and were pretty cheap. I found them in bulk for `$12` on AliExpress.

I ended up arranging the plants into zones like so: 

* 3x zones with a LILYGO TTGO T-Higrow serving as a BLE hub.
* Xioami Mijia BLE Flower Monitors for all of the other plants in a zone.
* Each zone of plants will be a mixture of plants that require almost no watering (e.g. cactuses) and plants that have similar watering characteristics. 

### Hardware

* [LILYGO® TTGO T-Higrow ESP32 WiFi And Bluetooth Battery And DHT11 Soil Temperature And Humidity Photometric Electrolyte Sensor](https://www.aliexpress.us/item/2251832630345390.html)
* [T-Higrow Motor](https://www.aliexpress.us/item/3256803314634205.html) - You will want the hat/case as well.
* [Xiaomi Mijia Flora Sensor](https://www.aliexpress.us/item/2255800399707249.html) - The BLE signal from these will be monitored by the LILYGO® TTGO T-Higrow ESP32.

### Software

* [Open Plantbook](https://open.plantbook.io/) – You'll need an API key.
* [Alternative Plant Component](https://github.com/Olen/homeassistant-plant) – This sets up plants as devices.
* [Open Plantbook Integration](https://github.com/Olen/home-assistant-openplantbook) - This is a "backend" sensor for the plant device and card.
* [lovelace-flower-card](https://github.com/Olen/lovelace-flower-card/tree/new_plant) - Fancy plant card for Lovelace.
* [ESPHome](https://esphome.io/) – Used to manage the LILYGO® TTGO T-Higrow.

## Assembling the Hardware

### Soldering the Motor Connector

The motor comes with a five pin connector that needs to be soldered to the LILYGO® TTGO T-Higrow ESP32. This connector needs to be soldered to the 5V and GPOI19 pins. See the photo below:

![Motor Pins Highlight](/images/hass/plant-sensor-roundup/motor-pins-highlight.jpeg)

Here is a photo of mine after having soldered the connector to the board:

![Motor Connector Soldered](/images/hass/plant-sensor-roundup/motor-connector.jpeg)

Here is the sensor with the motor mounted using the plastic hat you can purchase from AliExpress:

![Assembled Sensor and Motor](/images/hass/plant-sensor-roundup/assembled-sensor-motor.jpeg)

## Configuring ESPHome

### Base Configuration

The LILYGO® TTGO T-Higrow ESP32 comes in two configurations:

* [BME280](https://www.bosch-sensortec.com/products/environmental-sensors/humidity-sensors-bme280/)
* [DHT11](https://www.adafruit.com/product/386)

Both of them have the following base configuration for soil moisture, soil conductivity, and illuminance.

{{< highlight yml >}}
esphome:
  name: soil-sensor

esp32:
  board: lolin_d32

api:
  encryption:
    key: "0GamOcw/LFuS2HYcvVRkj88OUl17WEH/Grf9G3nWdxI="

ota:
  password: "45c182089a39e49022f4316cad690a2f"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

captive_portal:

logger:

esp32_ble_tracker:

i2c:
  sda: 25
  scl: 26
  scan: True
  id: bus_a

sensor:
  - platform: adc
    pin: GPIO32
    name: "Soil Moisture"
    update_interval: 15min
    attenuation: 11db
    unit_of_measurement: '%'
    filters:
      - calibrate_linear:
          # Map 0.0 (from sensor) to 0.0 (true value)
          - 3.08 -> 0.0
          - 1.42 -> 100.0
  - platform: adc
    pin: GPIO34
    name: "Soil Conductivity"
    update_interval: 15min
    accuracy_decimals: 4    
    unit_of_measurement: 'µS/cm'
    filters:
      - calibrate_linear:
          # Map 0.0 (from sensor) to 0.0 (true value)
          - 0.0 -> 0.0
          - 1.1 -> 100.0
  - platform: bh1750
    i2c_id: bus_a
    name: "BH1750 Illuminance"
    address: 0x23
    update_interval: 15min
    setup_priority: -300
{{< / highlight >}}

### BME280

If you have the LILYGO TTGO T-Higrow with the BME280 sensor add the following to your `sensor` section:

{{< highlight yml >}}
- platform: bme280
  i2c_id: bus_a        
  temperature:
    name: "BME280 Temperature"
    oversampling: 1x
  pressure:
    name: "BME280 Pressure"
  humidity:
    name: "BME280 Humidity"
  address: 0x77
  update_interval: 120s
{{< / highlight >}}

### DHT11

If you have the LILYGO TTGO T-Higrow with the DHT11 sensor add the following to your `sensor` section:

{{< highlight yml >}}
- platform: dht
  pin: 16
  model: dht11
  temperature:
    name: "Temperature"
    unit_of_measurement: "°C"
    icon: "mdi:thermometer"
    device_class: "temperature"
    state_class: "measurement"
    accuracy_decimals: 1            
  humidity:
    name: "Humidity"
    unit_of_measurement: "%"
    icon: "mdi:thermometer"
    device_class: "humidity"
    state_class: "measurement"      
  update_interval: 120s
  setup_priority: -100
{{< / highlight >}}

### Motor/Water Pump

The motor/water pump can be turned on/off with `GPIO19` as a `switch`:

{{< highlight yml >}}
switch:
  - platform: gpio
    pin: GPIO19
    name: "Water Pump"
    id: water
{{< / highlight >}}

## Configuring the Xiaomi Mijia Flora Sensor

The Xiaomi Mijia Flora Sensors emit sensor data via BLE. They have fantastic battery life and report data in a straightforward manner. To configure them, you need to first follow the logs of a configured sensor that has `esp32_ble_tracker` enabled. After a while you will see the `esp32_ble_tracker` logs emit an entry for the Xiaomi Mijia Flora Sensor:

{{< highlight yml >}}
[19:56:41][D][esp32_ble_tracker:809]: Found device C4:7C:8D:6D:5D:F5 RSSI=-102
[19:56:41][D][esp32_ble_tracker:830]:   Address Type: PUBLIC
[19:56:41][D][esp32_ble_tracker:832]:   Name: 'Flower care'
{{< / highlight >}}

Note the MAC address: `C4:7C:8D:6D:5D:F5`. We will use it to define a [`xiaomi_hhccjcy01`](https://esphome.io/components/sensor/xiaomi_ble.html#hhccjcy01) sensor:

{{< highlight yml >}}
sensor:    
  - platform: xiaomi_hhccjcy01
    mac_address: 'C4:7C:8D:6D:5D:F5'
    temperature:
      name: "Xiaomi HHCCJCY01-01 Temperature"
    moisture:
      name: "Xiaomi HHCCJCY01-01 Moisture"
    illuminance:
      name: "Xiaomi HHCCJCY01-01 Illuminance"
    conductivity:
      name: "Xiaomi HHCCJCY01-01 Soil Conductivity"
    battery_level:
      name: "Xiaomi HHCCJCY01-01 Battery Level" 
{{< / highlight >}}

## Configuring a Dashboard

Once you have configured all of the HACS add-ons, you can add a few plants using the Plant Monitor integration. These will show up as `plant` devices. They can be used with the `custom:flower-card`.

### Lovelace Plant Card

{{< highlight yml >}}
- type: custom:flower-card
  entity: plant.philodendron_birkin
  show_bars:
    - illumination
    - moisture
    - conductivity
    - temperature
    - dli
{{< / highlight >}}

### Plant Dashboard

![Plant Dashboard](/images/hass/plant-sensor-roundup/plant-dashboard.jpg)

## Water Pump in Action

{{< youtube 9XwzJZ1Acao >}}
