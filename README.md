# Tibs ThermAlarm
This repository includes the necessary code and 3D model for the Tibs ThermAlarm. A DIY project combining a thermostat and an alarm panel! The ThermAlarm is meant to be **used with Home Assistant (HA)** and is created using **ESPHome** and **PlatformIO**.

It includes the following components:
- 1x 4x4 matrix keypad
- 1x 0.96 OLED display SSD1306 128x64
- 1x Rotary encoder KY-040 (rotation + push button)
- 2x Wemos D1 Mini

It's possible to use a single NodeMCU to complete this project, which would involve some custom component code in ESPHome as a keypad is not yet integrated in the core.

# Usage
The goal of the project is to make a DIY thermostat and alarm panel in one. It is heavily inspired by [HASS-YAAP](https://github.com/paviro/HASS-YAAP) and [3ative's thermostat](https://github.com/3ative/thermostat-project-v3). The code for the alarm panel is identical to the original project, but instead of connecting it to an external display, the code runs headless. The OLED display indicates the alarm state, but does not show when a code is being entered.

The **alarm panel** uses MQTT to control an alarm created within HA. Check out the original project for an in-dept description. The way it's being used here is simply to arm and disarm the alarm.

The **thermostat** is used to control a simple central heating system (on/off). Note that the control for the thermostat is performed by an external switch, also integrated in HA. It is also only used for heating, not for cooling.

## Pinout diagram
The D1 mini itself is powered by a regular 5V adapter plugged into the wall.

![tibs-thermalarm-pinout](https://user-images.githubusercontent.com/45207725/192397898-814ff18a-ca77-42ee-82e9-7bd8cc70ea5c.png)


## Final Result
![tibs-thermalarm-front](https://user-images.githubusercontent.com/45207725/192398301-f4d7239a-c979-443a-ac46-957cadabbeb0.jpg)
![tibs-thermalarm-wiring](https://user-images.githubusercontent.com/45207725/192398211-86e45e32-7fc2-43e4-942f-7c82de3798de.png)



## Credits

- The [3D model for the rotary encoder knob](https://www.thingiverse.com/thing:1283248) is from Thingiverse user tardomatic.
- [The alarm panel code](https://github.com/paviro/HASS-YAAP) is by GitHub user [@paviro](https://github.com/paviro).
- The ESPHome code for the thermostat is inspired by [Thermostat Project V3](https://github.com/3ative/thermostat-project-v3) by GitHub user [@3ative](https://github.com/3ative).
