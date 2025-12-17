# Home Automation - Temperature Control

This repository contains configuration and documentation for an apartment temperature control system built around Home Assistant, Zigbee thermostats (TRVs), a BEOK display thermostat, and an ESP32‑C3–based fan controller running ESPHome.

Purpose
- Centralize climate control for two radiator TRVs
- Provide a user-facing thermostat (BEOK) for setpoint changes and display
- Improve radiator heat dispersion with two radiator-mounted fans controlled by an ESP32

Hardware (physical devices)
- Sonoff TRVZB thermostatic radiator valves (two units: `TRVZB_vorne`, `TRVZB_hinten`)
- BEOK BOT-R15W thermostat (`climate.beok_thermostat`) — used as a user interface only
- SZ-T04 desk temperature sensor (`sensor.desk_temperature`) — used as the indoor reference sensor
- Two 14cm PWM PC fans (tacho + PWM) mounted on the radiators (powered from 12V supply)
- ESP32‑C3 Super Mini running ESPHome (controls fans and reports temperature/RPM)
- Raspberry Pi 5 running Home Assistant and Zigbee coordinator (Sonoff Zigbee dongle)

Software stack
- Home Assistant (running on Raspberry Pi)
  - Zigbee2MQTT (Zigbee integration for Sonoff TRVs)
  - ESPHome integration (for the ESP32 fan controller)
- Versatile Thermostat integration (VTherm1) — https://github.com/jmcollin78/versatile_thermostat
- ESPHome — https://esphome.io
- Zigbee2MQTT — https://www.zigbee2mqtt.io
- Home Assistant — https://www.home-assistant.io

Entity mapping (in your HA instance)
- `climate.vtherm1` — virtual thermostat (VTherm1) controlling TRV setpoints
- `climate.beok_thermostat` — BEOK BOT-R15W thermostat (user-facing)
- `climate.TRVZB_vorne` and `climate.TRVZB_hinten` — Sonoff TRVZB thermostats (controlled by VTherm1)
- `sensor.desk_temperature` — SZ-T04 indoor sensor used as the temperature reference
- `input_boolean.guard_sync_thermostats` — guard boolean preventing sync loops
- Automations:
  - `sync_thermostats.yaml` — `sync_vtherm1_to_beok` and `sync_beok_to_vtherm1` automations that propagate setpoints between the BEOK thermostat and VTherm1
  - `beok_align_currenttemp.yaml` — calibration automation that aligns BEOK's displayed temperature with the indoor reference sensor

Control logic summary
- VTherm1 (`climate.vtherm1`) is the master for TRV control; VTherm1 controls `TRVZB_vorne` and `TRVZB_hinten`.
- The BEOK device (`climate.beok_thermostat`) is used for user interactions (setpoint display and change). When the BEOK's setpoint changes the sync automation copies it to VTherm1, and vice versa.
- `sensor.desk_temperature` (SZ-T04) is the indoor sensor used as the reference temperature for thermostats. The ESPHome temperature sensor on the ESP32 is NOT used for climate control (it is used for fan control only).
- A future outdoor sensor may be integrated and fed into VTherm1 as required.

Notes about the fans and ESPHome
- The ESP32‑C3 runs `fancontrol/espfan.yaml` for the fan controller; detailed documentation (wiring, substitutions, RPM multiplier, WiFi antenna fix, etc.) is in `fancontrol/README_fancontrol.md`.
- Both fans share the same PWM output (GPIO1) and therefore run at the same speed. 

Links and manuals
- Versatile Thermostat (integration): https://github.com/jmcollin78/versatile_thermostat
- ESPHome documentation: https://esphome.io
- Home Assistant: https://www.home-assistant.io
- Zigbee2MQTT: https://www.zigbee2mqtt.io

Device-specific manuals / references
- Sonoff TRVZB (Zigbee TRV series)
    - https://help.sonoff.tech/docs/trvzb
    - https://www.zigbee2mqtt.io/devices/TRVZB.html

- BEOK BOT-R15W 
    - https://de.beok-controls.com/room-thermostat/gas-boiler-heating-thermostat/wired-gas-boiler-thermostats-bot-r15w-zigbee.html 
    - https://de.beok-controls.com/uploads/34945/files/BOT-R15W-ZIGBEE-Gas-Boiler-Thermostat-Manual.pdf
    - https://www.zigbee2mqtt.io/devices/BOT-R15W.html

- SZ-T04 desk sensor (generic)
    - https://www.zigbee2mqtt.io/devices/SZ-T04.html

- ZTH11 outdoor temperatur/humidity sensor (IP65, no display)
    - indentifed in Z2M as [Tuya TS0601_temperature_humidity_sensor_2](https://www.zigbee2mqtt.io/devices/TS0601_temperature_humidity_sensor_2.html)    
    - https://manuals.plus/ae/1005009682066497

- WSD500A indoor temperatur/humidity sensor (no display)
    - https://www.zigbee2mqtt.io/devices/WSD500A.html#tuya-wsd500a
    - manual: https://ae01.alicdn.com/kf/Seeb674fd88e743d092611304d4d7b03ao.pdf

## VTherm1 Configuration (Overview)
- Type: `over_climate` (Versatile Thermostat controls underlying climate devices)
- Underlying devices: `climate.TRVZB_vorne`, `climate.TRVZB_hinten`
- Reference sensor: `sensor.desk_temperature` (indoor SZ-T04)
- Entity name: `climate.vtherm1`
- Setpoint sync: Handled by automations in `sync_thermostats.yaml` between `climate.vtherm1` and `climate.beok_thermostat`.
- Notes:
  - Physical TRV dial changes are followed by VTherm via the “follow underlying” feature.
  - VTherm regulates TRV setpoints; TRVs are not fed an external temperature directly via Zigbee2MQTT.
  - Time filtering and other VTherm features are configured in HA (UI), not tracked in this repo.


