# HomeAssistant automation for iD4 OctopusEnergy  
HomeAssistant automation to determine the % charge necessary to reach the target charge of a VW iD4 and set the Octopus Energy IO Charge to add

## My setup
- iD4 GTx
- Myenergi Zappi charger
- OctopusEnergy Intelligent Octopus Go tarriff with Zappi Smart Charging (beta)

## Goal
Use HomeAssistant to update the `Charge to add` percentage value at Octopus when the car returns home based on the delta between the current charge % and the target charge % as set in the car.

## Why
Allow Octopus to more accurately map out a Smart Charging schedule based on an actual % value to charge rather than simply always requesting 80% (for instance) and letting the car cut off the charge when it's full.

## How

- Install and setup [this](https://github.com/mitch-dc/volkswagen_we_connect_id) custom Volskwagen Connect integration through HACS 
- Install and setup the OctopusEnergy and Myenergi integrations through HACS

### Create a custom sensor:

This sensor grabs the charge % values from the car and subtracts the current charge % from the target charge %. It divides the delta by 10, rounds the value and plusses 1. This increases the the requested charge % value by up to 10% so we have a round number to send to Octopus rather than trying to send them 7%, for instance that wouldn't be requested. The minimum % is also 10%, so we account for that too. The car will cut off/not attempt a charge if full, so it's safe.

```bash
template:
  - sensor:
      - name: "Car Charge Percent Remaining"
        unit_of_measurement: "%"
        state: >
          {% if has_value('sensor.id4_gtx_state_of_charge') and has_value('sensor.id4_gtx_target_state_of_charge') %}
            {% if states('sensor.id4_gtx_state_of_charge') != states('sensor.id4_gtx_target_state_of_charge') %}
              {% set current_charge = states('sensor.id4_gtx_state_of_charge') | int %}
              {% set target_charge = states('sensor.id4_gtx_target_state_of_charge') | int %}
              {{ (((target_charge - current_charge )/10) | round +1) *10 }}
            {% else %}
              {{ 10 }}
            {% endif %}
          {% endif %}
```

### Create the automation

This automaion uses the `device_tracker.id4_gtx_tracker` and triggers when it enters the `zone.home` zone, then updates the `number.octopus_energy_intelligent_charge_limit` based on the above sensor value.

```bash
alias: iD4 Set Octopus Charge Percent
description: ""
trigger:
  - platform: zone
    entity_id: device_tracker.id4_gtx_tracker
    zone: zone.home
    event: enter
condition: []
action:
  - service: number.set_value
    data_template:
      value: "{{ states('sensor.car_charge_percent_remaining') | float }}"
    target:
      entity_id: number.octopus_energy_intelligent_charge_limit
mode: single
```
