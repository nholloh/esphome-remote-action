# ESPHome Remote Action

ESPHome Remote Action enables one ESPHome node to query state and trigger actions on another node via the built-in web server REST API, without routing requests through Home Assistant.

## When to use this

When you…

- Don’t mind a latency of up to ~1 second before triggering  
- Prefer not to set up a UDP-based solution (since it is a lot more repetitive work on target and source device)  
- Want to avoid routing everything via Home Assistant (thus eliminating a potential point of failure)

## Minimal setup

Below is a high-level overview of the steps needed to get up and running. For the sake of simplicity, we'll assume in this setup that one device is designated as a **target** (which responds to HTTP requests) and another device as a **source** (which issues those HTTP requests). In a real-world scenario, both devices may be source and target at the same time.

### Target: Enable Web Server

```yaml
# Enable the web server
web_server:
    port: 80  # Make sure this is set to 80 for default usage
```

### Source: Import Package

```yaml
# Add the remote_action package
packages:
  remote_action: 
    url: github://nholloh/esphome-remote-action/remote-action.yml@latest
```

### Source: Configure Automation

```yaml
# Within any automation, run the script:
on_press:
  - script.execute:
      id: remote_action # id of the script, keep this.
      device_name: "livingroom-cover" # the name of your device from the esphome section.
      domain: "cover" # the type of component: [cover, light, switch, fan, select, button, number]
      component_name: "Livingroom Cover" # the name of the component to trigger
      action: "close" # the action to run. Available actions depend on the domain.
      variables: "" # additional variables for the action. Pass "" if no variables are required.
```

## Detailed example: Running an action

In the more detailed example below we will assume that the **source** device is a Shelly 2 PM dedicated to reading the states of a physical cover switch with two buttons (up, down). Each of those buttons is a `binary_sensor` which can be On or Off.

The **target** device is an Shelly 2 PM hidden in the fuse box from where the covor motor is powered.

Please note that specifics of GPIO as well as readings of resistance through i2c are specific to the Shelly devices I am using and may not be accurate for you. I recommend checking the [ESPHome Device Configuration Repository](https://devices.esphome.io) for details on your specific device.

### Source device

<details>
<summary>Source device full configuration</summary>

```yml
substitutions:
  devicename: "groundfloor-cover" # I use this switch to trigger all covers on the ground floor.

esphome:
  name: ${devicename}

esp32:
  board: esp32doit-devkit-v1
  framework:
    type: arduino

packages:
  remote_package:
    url: !secret github_url
    files:
      - remote-action.yml
    refresh: 0s

logger:
api:
ota:
  - platform: esphome

safe_mode:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_pass

binary_sensor:
  - platform: gpio
    name: "${devicename} Switch Down"
    pin: GPIO5
    filters:
      - delayed_on_off: 50ms
    on_press:
      then:
        - script.execute:
            id: remote_action                           # Name of the script. Keep as is!
            device_name: "livingroom-cover"             # Name of the device.
            domain: "cover"                             # The device is a cover.
            component_name: "livingroom-cover Cover"    # The component is called "${devicename} Cover".
            action: "close"                             # "close" because the Down Switch was pressed (ON).
            variables: ""                               # No additional variables required.
    on_release:
      then:
        - script.execute:
            id: remote_action                           # Name of the script. Keep as is!
            device_name: "livingroom-cover"             # Name of the device.
            domain: "cover"                             # The device is a cover.
            component_name: "livingroom-cover Cover"    # The component is called "${devicename} Cover".
            action: "stop"                              # "stop" because the Down Switch was released (OFF).
            variables: ""                               # No additional variables required.
  - platform: gpio
    name: "${devicename} Switch Up"
    pin: GPIO18
    filters:
      - delayed_on_off: 50ms
    on_press:
      then:
        - script.execute:
            id: remote_action                           # Name of the script. Keep as is!
            device_name: "livingroom-cover"             # Name of the device.
            domain: "cover"                             # The device is a cover.
            component_name: "livingroom-cover Cover"    # The component is called "${devicename} Cover".
            action: "open"                              # "open" because the Up Switch was pressed (ON).
            variables: ""                               # No additional variables required.
    on_release:
      then:
        - script.execute:
            id: remote_action                           # Name of the script. Keep as is!
            device_name: "livingroom-cover"             # Name of the device.
            domain: "cover"                             # The device is a cover.
            component_name: "livingroom-cover Cover"    # The component is called "${devicename} Cover".
            action: "stop"                              # "stop" because the Up Switch was released (OFF).
            variables: ""                               # No additional variables required.
```

</details>

### Target device

<details>
<summary>Target device full configuration</summary>

```yml
substitutions:
  devicename: "livingroom-cover"

esphome:
  name: ${devicename}

esp32:
  board: esp32doit-devkit-v1
  framework:
    type: arduino

logger:
api:
ota:
  - platform: esphome

safe_mode:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_pass

web_server:
  port: 80

i2c:
  sda: GPIO26
  scl: GPIO25

output:
  - platform: gpio
    id: "relay_output_1"
    pin: GPIO12
  - platform: gpio
    id: "relay_output_2"
    pin: GPIO13

switch:
  - platform: output
    id: "relay_1"
    name: "${devicename} Relay Down"
    output: "relay_output_1"
    internal: true

  - platform: output
    id: "relay_2"
    name: "${devicename} Relay Up"
    output: "relay_output_2"
    internal: true

cover:
  - platform: current_based
    name: "livingroom-cover Cover"
    id: cover1

    open_sensor: current_channel_2
    open_moving_current_threshold: 0.2
    open_obstacle_current_threshold: 1.2
    open_action:
      - switch.turn_on: relay_2
    open_duration: 23s

    close_sensor: current_channel_1
    close_moving_current_threshold: 0.2
    close_obstacle_current_threshold: 1.2
    close_action:
      - switch.turn_on: relay_1
    close_duration: 23s
    start_sensing_delay: 1s

    stop_action:
      - switch.turn_off: relay_2
      - switch.turn_off: relay_1

sensor:
  # Power Sensor
  - platform: ade7953_i2c
    irq_pin: GPIO27
    voltage:
      name: "${devicename} Voltage"
      entity_category: 'diagnostic'
    current_a:
      name: "${devicename} Relay Up Current"
      id: current_channel_2
      entity_category: 'diagnostic'
    active_power_a:
      name: "${devicename} Relay Up Power"
      id: power_channel_2
      entity_category: 'diagnostic'
      filters:
        - multiply: -1
    current_b:
      name: "${devicename} Relay Down Current"
      id: current_channel_1
      entity_category: 'diagnostic'
    active_power_b:
      name: "${devicename} Relay Down Power"
      id: power_channel_1
      entity_category: 'diagnostic'
      filters:
        - multiply: -1
    update_interval: 0.5s
```

</details>

## Querying State

To read the state from the target device, call:

```yaml
script:
  - id: get_living_room_cover_status
    then:
      - script.execute:
          id: remote_status
          device_name: "livingroom-cover"
          domain: "cover"
          component_name: "livingroom-cover Cover"
```

The response payload is stored in a global variable `remote_status_body` for later use or logging. The status code is available in `remote_status_http_status_code`.

## Supported Devices and Available Actions

Most ESPHome domains exposed via the [Web Server REST API](https://esphome.io/web-api) can be queried with a GET request to retrieve their state, and some can be controlled with a POST request. Here’s a quick summary:

| **Device**       | **GET**                                  | **POST (Actions)**                                                            | **Reference**                                                           |
|------------------|-------------------------------------------|-------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| **Sensor**       | Returns JSON with `"state"` and `"value"` | *No settable actions*                                                         | [Sensor Docs](https://esphome.io/web-api#sensor)                        |
| **Binary Sensor**| Returns JSON with `"state": ON/OFF`       | *No settable actions*                                                         | [Binary Sensor Docs](https://esphome.io/web-api#binary-sensor)         |
| **Switch**       | Returns JSON (similar to binary sensor)   | `turn_on`, `turn_off`, `toggle`                                               | [Switch Docs](https://esphome.io/web-api#switch)                        |
| **Light**        | Returns JSON with `"state"`, `"brightness"`, `"color"`, etc. | `turn_on`, `turn_off`, `toggle`, plus optional query parameters like `brightness`, `color_temp`, etc. | [Light Docs](https://esphome.io/web-api#light)                          |
| **Fan**          | Returns JSON with `"state"`, `"speed_level"`, `"oscillation"` | `turn_on`, `turn_off`, `toggle`, plus optional query parameters `speed_level`, `oscillation` | [Fan Docs](https://esphome.io/web-api#fan)                              |
| **Cover**        | Returns JSON with `"state"`, `"value"`, `"current_operation"`, `"tilt"` | `open`, `close`, `stop`, `toggle`, `set` (with `position`, `tilt` parameters) | [Cover Docs](https://esphome.io/web-api#cover)                          |
| **Select**       | Returns JSON with `"state"`, optional `"option"` array | `set` (with `?option=<value>`)                                               | [Select Docs](https://esphome.io/web-api#select)                        |
| **Button**       | Returns JSON (`"state"` is often `UNAVAILABLE` for buttons) | `press`                                                                       | [Button Docs](https://esphome.io/web-api#button)                        |
| **Number**       | Returns JSON with `"value"`               | `set` (with `?value=<float>`)                                                | [Number Docs](https://esphome.io/web-api#number)                        |

For more details, consult the official [ESPhome Web API documentation](https://esphome.io/web-api).

## Limitations

- **Latency**: This method depends on HTTP polling and can have a noticeable delay (up to ~1 second). See [this discussion](https://github.com/esphome/feature-requests/issues/52) for why a more direct communication (e.g. via Proto/Native API) may be preferable, though it’s currently not straightforward or supported.
- **SSL Validation**: SSL certificate validation is disabled so the component can work on both Arduino and ESP-IDF frameworks.
- **Insecure Transport**: Currently uses plain HTTP on port 80, meaning requests and responses are not encrypted.
- **Port 80**: The component assumes the web server is running on port 80 and uses `device_name.local` as the hostname. If you run the web server on another port or need a different DNS, changes to these scripts will be necessary.
- **Parallel state queries**: Since the response of a state query is saved in global state, state queries can only be run sequentelly as to not lose the response or result in undefined behaviour.
