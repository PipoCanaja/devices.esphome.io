---
title: "Solis S2_WL_ST"
date-published: 2026-11-14
type: misc
standard: global
board: esp32
difficulty: 4
---

<img width="381" height="492" alt="image" src="https://github.com/user-attachments/assets/36166359-11be-48ff-8203-170543ab5816" />

Solis provides an ESP32 WROVER-E based dongle to connect their inverters (equiped with MODBUS RTU connectivity) to their cloud. 

In order to flash it, pins must be soldered on the board. Then, the current firmware can be dumped using any USB-to-Serial adapter or 3.3V UART. It is always interesting to keep the existing firmware in order to revert the mod. 

The device has:
- Modbus to UART
- Ethernet LAN8720 (with GPIO12 to enable it electrically)
- 1 button (GPIO34)
- 2 LEDs (GPIO13 and GPIO15)
- 1 embedded watchdog (on GPIO4)

I configured it as a MODBUS RTU/TCP Bridge using modbus_bridge component from https://github.com/rosenrot00/esphome_modbus_bridge . It can also be configured to directly poll the MODBUS values and expose them.

It appears that different revisions of the board are available, my board is `Ginlong Technology DAWCE6330-V1.3`. 

## Basic Configuration

```yaml
substitutions:
  device_name: "S2-WL-ST"
  api_key: "xxxxxxxxxxxxx"
  wifi_pass: !secret wifi_password
  wifi_ssid: !secret wifi_ssid

esphome:
  name: s2-wl-st
  friendly_name: ${device_name}

esp32:
  board: esp32dev
  framework:
    type: esp-idf
    advanced:
      minimum_chip_revision: '3.0'
      sram1_as_iram: true

safe_mode:
  num_attempts: 3

# Enable Home Assistant API
api:
  reboot_timeout: 3600s
  encryption:
    key: ${api_key}
ota:
  - platform: esphome 
    allow_partition_access: true
    encryption: 

web_server:
  local: True
  version: 3
  auth:
    username: !secret web_server_username
    password: !secret web_server_password
captive_portal:

network:
  priority:
    - ethernet
    - wifi

ethernet:
  type: LAN8720
  mdc_pin: GPIO23
  mdio_pin: GPIO18
  clk:
    pin: GPIO0
    mode: CLK_EXT_IN
  phy_addr: 0
  power_pin: GPIO5

wifi:
  enable_on_boot: true
  networks:
    - ssid: ${wifi_ssid}
      password: ${wifi_pass}

uart:
  - id: modbus_uart
    tx_pin: GPIO33
    rx_pin: GPIO32
    baud_rate: 9600
    rx_buffer_size: 1152   # default is 256; increase for long RTU responses
    data_bits: 8
    stop_bits: 1
    parity: NONE

modbus:
  - uart_id: modbus_uart
    id: modbus_client
    flow_control_pin: GPIO14
    send_wait_time: 400ms
    turnaround_time: 40ms    


  #GPIO4 - Refresh the Watchdog (10ms)
  #GPIO12 - Power the Ethernet PHY (Always on if using the PHY)
  #GPIO13 - NET / Green LED
  #GPIO15 - COM / Orange LED

  - platform: gpio
    pin:
      number: GPIO34
      inverted: true
    name: "RESET"
    id: button_reset
    filters:
      - delayed_on_off: 30ms
    on_click:
      - min_length: 50ms
        max_length: 6s
        then:
          - logger.log: "Short RESET"
            # Any action on short push
      - min_length: 6s
        max_length: 60s
        then:
          - logger.log: 'Long RESET - initiate reboot'
          - button.press: restart_btn

output:
  - platform: gpio
    pin:
       number: GPIO4
    id: watchdog_feed
switch:
  - platform: gpio
    pin: GPIO12
    id: phy_power
    restore_mode: ALWAYS_ON
    internal: true
      
  - platform: gpio
    name: "NET (green) LED"
    id: led_net
    restore_mode: ALWAYS_OFF
    internal: true
    pin:
      number: GPIO13

  - platform: gpio
    name: "COM (orange) LED"
    id: led_com
    restore_mode: ALWAYS_OFF
    internal: true
    pin:
      number: GPIO15

## Every 60 seconds, if API is connected, refresh the watchdog.
## Watchdog timeout : 15 minutes.
interval:
  - interval: 60s
    then:
      - if:
          condition:
            - api.connected:
          then:
            - output.turn_on: watchdog_feed
            - delay: 10ms
            - output.turn_off: watchdog_feed

```
