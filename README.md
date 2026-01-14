# bm2-esphome
An ESPHome-based handler for BM2 battery monitors including voltage, state of charge and presence.

# Requirements
* [Home Assistant](https://home-assistant.io) and [ESPHome](https://esphome.io)
* BMS battery monitor (like [this](https://www.aliexpress.com/item/1005008058900833.html) or [this](https://amzn.to/3NrcjGY))
* ESP32 device close enough to your BM2 battery monitor to receive the BLE packets

# How to use
* Copy [bm2_aes.h](https://raw.githubusercontent.com/DJBenson/bm2-esphome/refs/heads/main/bm2_aes.h) to your ESPHome configuration directory
* Copy the example code to a new ESP32 device template in ESPHome
* Update the following are update;
  * bm2_mac: The MAC address of the BM2 device to monitor (you can get this from the app for the device)
  * bm2_presence_timeout_s: The time (*in seconds*) since the last broadcast after which we consider the device "away". **Default: 120**.
  * bm2_publish_interval_s: The time (*in seconds*) between each broadcast - intended to slow down the number of updates to Home Assistant. The device itself reports every 10 seconds. **Default: 30**.

# Provided entities 
The code provides the following entities to Home Assistant;
* BM2 Battery Voltage (in volts)
* BM2 Battery Charge (as a percentage)
* BM2 Battery Presence (as a binary sensor)
* BM2 Battery Last Updated (as a timestamp)

#  Example
<img width="322" height="609" alt="image" src="https://github.com/user-attachments/assets/83ca2758-2487-455c-9c9b-4602473417da" />
