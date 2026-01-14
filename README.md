# bm2-esphome
An ESPHome-based handler for BM2 battery monitors including voltage, state of charge and presence.

# Requirements
* [Home Assistant](https://home-assistant.io) and [ESPHome](https://esphome.io)
* BMS battery monitor (like [this](https://www.aliexpress.com/item/1005008058900833.html) or [this](https://amzn.to/3NrcjGY))
* ESP32 device close enough to your BM2 battery monitor to receive the BLE packets
* Tested with (and probably requires) the IDF framework - I've not tested with the Arduino framework

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
* BM2 Battery Last Updated (as a timestamp) - this entity is disabled by default - you'll need to manually enable it if you want to use it.

# Sample configuration

```
substitutions:
  devicename: bm2-ble
  bm2_mac: "AB:CD:EF:00:01:02"
  bm2_presence_timeout_s: "120"
  bm2_publish_interval_s: "30"

esphome:
  name: ${devicename}
  includes:
    - bm2_aes.h

esp32:
  framework:
    type: esp-idf
  board: esp32dev

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
api:
ota:

globals:
  - id: bm2_last_seen_ms
    type: uint32_t
    restore_value: no
    initial_value: "0"
  - id: bm2_last_publish_ms
    type: uint32_t
    restore_value: no
    initial_value: "0"

time:
  - platform: homeassistant
    id: ha_time

esp32_ble_tracker:
  scan_parameters:
    interval: 1100ms
    window: 1100ms
    active: false
  on_ble_advertise:
    then:
      - lambda: |-
          if (x.address_str() != "${bm2_mac}") return;

          auto mfd = x.get_manufacturer_datas();
          if (mfd.empty()) return;

          static const uint8_t BT_KEY[16] = {108,101,97,103,101,110,100,255,254,49,56,56,50,52,54,54};
          static const uint8_t BT_IV[16]  = {0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0};

          for (auto &md : mfd) {
            auto &data = md.data;

            // Only the BM2 payload is 14 bytes. Ignore the 23‑byte iBeacon frame.
            if (data.size() != 14) continue;

            auto u = md.uuid.get_uuid();
            uint16_t mid = 0;
            if (u.len == ESP_UUID_LEN_16) {
              mid = u.uuid.uuid16;
            } else if (u.len == ESP_UUID_LEN_32) {
              mid = u.uuid.uuid32 & 0xFFFF;
            }

            uint8_t in[16];
            in[0] = mid & 0xFF;
            in[1] = (mid >> 8) & 0xFF;
            memcpy(in + 2, data.data(), 14);

            uint8_t out[16];
            if (!bm2_aes_cbc_decrypt_16(BT_KEY, BT_IV, in, out)) return;

            uint8_t vmsb = out[6];
            uint8_t vlsb = out[7];
            float voltage = ((vmsb << 8) | vlsb) / 100.0f;
            uint8_t charge = out[8];

            id(bm2_last_seen_ms) = millis();
            if (voltage >= 10.0f && voltage <= 15.0f && charge > 0 && charge <= 100) {
              uint32_t now = millis();
              if (now - id(bm2_last_publish_ms) < (${bm2_publish_interval_s} * 1000)) {
                continue;
              }
              id(bm2_last_publish_ms) = now;
              id(bm2_voltage).publish_state(voltage);
              id(bm2_charge).publish_state(charge);
              if (id(ha_time).now().is_valid()) {
                id(bm2_last_updated).publish_state(id(ha_time).now().timestamp);
              }
            }
          }

sensor:
  - platform: template
    name: "BM2 Battery Voltage"
    id: bm2_voltage
    unit_of_measurement: "V"
    device_class: voltage
    state_class: measurement
  - platform: template
    name: "BM2 Battery Charge"
    id: bm2_charge
    unit_of_measurement: "%"
    device_class: battery
    state_class: measurement
  - platform: template
    name: "BM2 Battery Last Updated"
    id: bm2_last_updated
    device_class: timestamp
    disabled_by_default: true

binary_sensor:
  - platform: template
    name: "BM2 Battery Presence"
    id: bm2_presence
    device_class: presence

text_sensor:
  - platform: template
    name: "BM2 Battery Last Updated"
    update_interval: 5s
    lambda: |-
      if (id(bm2_last_publish_ms) == 0) {
        return {"never"};
      }
      uint32_t age_s = (millis() - id(bm2_last_publish_ms)) / 1000;
      return {to_string(age_s) + "s ago"};

interval:
  - interval: 5s
    then:
      - lambda: |-
          const uint32_t now = millis();
          bool present = (now - id(bm2_last_seen_ms)) <= (${bm2_presence_timeout_s} * 1000);
          id(bm2_presence).publish_state(present);
```

#  Example
<img width="322" height="609" alt="image" src="https://github.com/user-attachments/assets/83ca2758-2487-455c-9c9b-4602473417da" />
