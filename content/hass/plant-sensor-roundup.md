---
title: "ESP32, BLE, RF Plant Sensor Roundup and Home Assistant Dashboard"
date: 2022-10-16T07:19:55-07:00
draft: true
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


