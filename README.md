# Tibs ThermAlarm
This repository includes the necessary code and 3D model for the Tibs ThermAlarm. A DIY project combining a thermostat and an alarm panel! The ThermAlarm is meant to be **used with Home Assistant (HA)** and is created using **ESPHome** and **PlatformIO**. Scroll down to see the [final result](#final-result)!

### Necessary Hardware
It includes the following components:
- 1x 4x4 matrix keypad
- 1x 0.96 OLED display SSD1306 128x64
- 1x Rotary encoder KY-040 (rotation + push button)
- 2x Wemos D1 Mini

It's possible to use a single NodeMCU to complete this project, which would involve some custom component code in ESPHome as a keypad is not yet integrated in the core.

It needs the following components already integrated in HA:
- Temperature sensor
- Switch to control central heating system
- Optional: Dimmable lights

### Usage
The goal of the project is to make a DIY thermostat and alarm panel in one. It is heavily inspired by [paviro's HASS-YAAP](https://github.com/paviro/HASS-YAAP) and [3ative's thermostat](https://github.com/3ative/thermostat-project-v3). The code for the alarm panel used here is identical to the original project, but instead of connecting it to an external display, the code runs headless. The OLED display indicates the alarm state, but does not show when a code is being entered.

The **alarm panel** uses MQTT to control an alarm created within HA. Check out the original project for an in-dept description of the usage. The way it's being used here is simply to arm and disarm the alarm. Additionally it can turn on/off lights in the house if they are integrated in HA.

The **thermostat** is used to control a simple central heating system (on/off). Note that the control for the thermostat is performed by an external switch, also integrated in HA. It is also only used for heating, not for cooling. Additionally it can in/decrease brightness of lights in the house if they are integrated in HA.

The thermostat changes the temperature of the climate entity by turning the rotary encoder. Clicking the rotary encoder button switches to light brightness control and calls HA scripts to perform this action.

## Hardware configuration
The D1 mini itself is powered by a regular 5V adapter plugged into the wall.

![tibs-thermalarm-pinout](https://user-images.githubusercontent.com/45207725/192397898-814ff18a-ca77-42ee-82e9-7bd8cc70ea5c.png)

## ESPHome configuration
The only code you have to adapt to make this work with your system can be found in the substitutions section.
```
substitutions:
  room: Living                                                          # Name of the room thermostat is in
  entity_heater: switch.heating_switch                                  # entity_id of the heating switch in HA
  entity_temp: sensor.temperature_sensor_living_ewe_temperature         # entity_id of the temperature sensor in HA
  entity_alarm: alarm_control_panel.home_alarm                          # entity_id of the alarm control panel in HA
  entity_brightness_increase: script.increase_brightness_living_room    # entity_id of the script in HA to increase lights brightness
  entity_brightness_decrease: script.decrease_brightness_living_room    # entity_id of the script in HA to decrease lights brightness
  entity_timer_alarm_arming: sensor.timer_arming_remaining              # entity_id of the sensor in HA of alarm arming
  entity_timer_alarm_pending: sensor.timer_pending_remaining            # entity_id of the sensor in HA of alarm pending
  default_low: "18.0"                                                   # The default low target temperature for the control algorithm
                                                                        # This can be dynamically set in the frontend later
  min_temp: "15.0"                                                      # Minimum temperature able to set
  max_temp: "25.0"                                                      # Maximum temperature able to set
  temp_step: "0.5"                                                      # Temperature set step
  heat_hysteresis_deadband: "0.3"                                       # Heat deadband, see climate thermostat component documentation
  heat_hysteresis_overrun: "0.1"                                        # Heat overrun,  see climate thermostat component documentation
  ```
Don't forget to place the `_icons` and `_fonts` folders in the config/esphome folder e.g. by using Samba.

## HA configuration
Add the following scripts to HA and change the `light.staanlamp` to your own light entity_id.

### Increase brightness
```
alias: "Lights: Increase brightness - Living Room"
sequence:
  - service: light.turn_on
    data:
      brightness_step_pct: 10
    target:
      entity_id:
        - light.staanlamp
mode: single
icon: mdi:lamps
```
### Decrease brightness
```
alias: "Lights: Decrease brightness - Living Room"
sequence:
  - service: light.turn_on
    data:
      brightness_step_pct: -10
    target:
      entity_id:
        - light.staanlamp
mode: single
icon: mdi:lamps
```
Make sure that you reference to the right script entity_id in the ESPHome configuration.

### Alarm Timer on Display
To add a timer that shows the time until your alarm is armed and the time until it is triggered, you need a few things.

Start by adding two timers with your chosen duration to HA as helpers. Let's call the timer until armed `timer.alarm_arming` and the timer until trigger `timer.alarm_timer`.
Then add the following automations, making sure to replace the entity_id's with your own:

To start the timers when the alarm is arming/pending:
```
alias: "Alarm: Timer starts Alarm Arming"
description: Start timer that tells when alarm will be armed.
trigger:
  - platform: state
    entity_id: alarm_control_panel.home_alarm
    to: arming
condition: []
action:
  - service: timer.start
    data:
      duration: "0"
    target:
      entity_id: timer.alarm_arming
mode: single
```
```
alias: "Alarm: Timer starts Alarm Pending"
description: Start timer that tells when alarm will go off.
trigger:
  - platform: state
    entity_id: alarm_control_panel.home_alarm
    to: pending
condition: []
action:
  - service: timer.start
    data: {}
    entity_id: timer.alarm_timer
mode: single
```
To cancel the timers when the trigger/arm is cancelled:
```
alias: "Alarm: Cancel arming timer"
description: Cancel timer that tells when alarm will be armed.
trigger:
  - platform: state
    entity_id:
      - alarm_control_panel.home_alarm
    from: arming
    to: disarmed
condition: []
action:
  - service: timer.cancel
    data: {}
    target:
      entity_id: timer.alarm_arming
mode: single
```
```
alias: "Alarm: Cancel trigger timer"
description: Cancel timer that tells when alarm will go off.
trigger:
  - platform: state
    entity_id:
      - alarm_control_panel.home_alarm
    from: pending
    to: disarmed
condition: []
action:
  - service: timer.cancel
    data: {}
    target:
      entity_id: timer.alarm_timer
mode: single
```
Because HA does not natively show the remaining time on a timer, you also need to add the following sensors to your configuration.yaml:
```
sensor:
  - platform: template
    sensors:
      timer_arming_remaining:
        friendly_name: 'Timer Arming Remaining'
        value_template: >-
          {% set f = state_attr('timer.alarm_arming', 'finishes_at') %}
          {{ '00:00' if f == None else 
            (as_datetime(f) - now()).total_seconds() | timestamp_custom('%M:%S', false) }}
      timer_pending_remaining:
        friendly_name: 'Timer Pending Remaining'
        value_template: >-
          {% set f = state_attr('timer.alarm_timer', 'finishes_at') %}
          {{ '00:00' if f == None else 
            (as_datetime(f) - now()).total_seconds() | timestamp_custom('%M:%S', false) }}
```            
Then finally, you need to add two more automations to update the remaining time sensors, as this is not done automatically:
```
alias: "Alarm: Update Timer Arming Remaining"
description: ""
trigger:
  - platform: state
    entity_id:
      - timer.alarm_arming
    to: active
  - platform: time_pattern
    seconds: /1
condition:
  - condition: state
    entity_id: timer.alarm_arming
    state: active
action:
  - service: homeassistant.update_entity
    data: {}
    target:
      entity_id: sensor.timer_arming_remaining
mode: single
hide_entity: true
```
```
alias: "Alarm: Update Timer Pending Remaining"
description: ""
trigger:
  - platform: state
    entity_id:
      - timer.alarm_timer
    to: active
  - platform: time_pattern
    seconds: /1
condition:
  - condition: state
    entity_id: timer.alarm_timer
    state: active
action:
  - service: homeassistant.update_entity
    data: {}
    target:
      entity_id: sensor.timer_pending_remaining
mode: single
hide_entity: true
```
Now the current temperature will be replaced everytime your alarm is pending or arming by a timer that counts until the alarm is armed or triggered. Cancelling the action will also cancel the timer and shows the temperature again on the display.

## Tibs Thermostat
If you just want to use the thermostat without the alarm panel, an stl file is included without the keypad cutout.

## Final Result

A [demo video](https://www.reddit.com/r/Esphome/comments/xpxfy0/i_built_a_diy_thermostat_alarm_panel_in_one/?utm_source=share&utm_medium=web2x&context=3) can be found on reddit.
### Front
![IMG_20220927_122309](https://user-images.githubusercontent.com/45207725/192515188-85183305-17e4-4bf2-8464-220cc6d9d71e.jpg)
![PXL_20220927_102025196](https://user-images.githubusercontent.com/45207725/192515153-09560e5b-a001-459a-b3b2-ef0a87bf24d6.jpg)
![PXL_20220927_102140844](https://user-images.githubusercontent.com/45207725/192515177-5636916c-e635-4605-b198-86ba091cd7f3.jpg)
![PXL_20220927_101953073](https://user-images.githubusercontent.com/45207725/192515192-cc9392ba-ff41-497d-b053-7cbe64285019.jpg)

### Alarm Panel States
#### Disarmed & heating off
![disarmed](https://user-images.githubusercontent.com/45207725/192407516-bd44bf3e-0bb9-4a27-817c-ff2faf6ba45d.jpg)

#### Disarmed & heating on
![heating](https://user-images.githubusercontent.com/45207725/192407533-6d4645a0-640e-4655-afb1-bb180df1a270.jpg)

#### Arming
![pending](https://user-images.githubusercontent.com/45207725/192407557-42091be3-69f1-462f-9cd1-bf696e8f9860.jpg)

#### Armed
![armed](https://user-images.githubusercontent.com/45207725/192407702-f562b16b-eb7d-45ed-883a-226eef167815.jpg)

### Wiring
![tibs-thermalarm-wiring](https://user-images.githubusercontent.com/45207725/192398211-86e45e32-7fc2-43e4-942f-7c82de3798de.png)

## Credits

- The [3D model for the rotary encoder knob](https://www.thingiverse.com/thing:2549137/files) is from Thingiverse user tardomatic.
- The [3D model for the OLED display holder](https://www.thingiverse.com/thing:2157801/files) integrated in the thermostat replacement 3D models is from Thingiverse user Nan_1.
- [The alarm panel code](https://github.com/paviro/HASS-YAAP) is by GitHub user [@paviro](https://github.com/paviro).
- The ESPHome code for the thermostat is inspired by [Thermostat Project V3](https://github.com/3ative/thermostat-project-v3) by GitHub user [@3ative](https://github.com/3ative).

## Support
🎁 If you like what I do, let's get a beer/coffee!

[![BMC](https://www.buymeacoffee.com/assets/img/custom_images/white_img.png)](https://www.buymeacoffee.com/tibogeenen)

🍺 As we say in Belgium, schol!
