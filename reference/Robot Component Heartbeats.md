---
tags:
---
# Robot Component Heartbeats

Within the `coco-acu` repository, different robot components have different [[Heartbeat]] frequencies.

- **1 second**:
    - Moving Status (`src/hardware_fail_detection/mqtt_reporter/src/mqtt_reporter/moving_status.py`)
- **10 seconds**:
    - Segway Base Status (`src/hardware_fail_detection/mqtt_reporter/src/mqtt_reporter/segway_base_status.py`)
    - GPS Status (`src/hardware_fail_detection/mqtt_reporter/src/mqtt_reporter/gps_status.py`)
    - PCU Driver (`src/hal/pcu_driver/pcu_driver/src/pcu_driver/pcu_status.py`)
    - Emergency Button Status (`src/hardware_fail_detection/mqtt_reporter/src/mqtt_reporter/emergency_button_status.py`)

These are configured using ROS parameters and timers. The different frequencies reflect the criticality of the component's status.

See also: [[Heartbeat Frequencies]]
