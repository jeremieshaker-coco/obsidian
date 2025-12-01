---
tags:
  - testing
  - integration
  - doordash
---
# DoorDash Integration Testing

To test the DoorDash integration, you should primarily use the following script:

[coco-dd-v3-demo](https://github.com/cocorobotics/coco-dd-v3-demo)

This script provides handy functionality to hit our DoorDash integrations endpoints as DoorDash normally would to create/assign, cancel, or initiate a handoff for a delivery.

## Usage

```python
# format:
python3 staging.py -v <API_VERSION> -r <REGION> -a <ACTION>

# example:
# creating and assigning a delivery
python3 staging.py -v 3 -r US -a create
python3 staging.py -v 3 -r US -a assign
```

## Gotchas

-   By default, the script hits the following merchant on staging: [https://mission-control.delivery-stage.cocodelivery.com/v3/merchants/44de43cb-5e74-4890-865a-38dbc19022f2](https://mission-control.delivery-stage.cocodelivery.com/v3/merchants/44de43cb-5e74-4890-865a-38dbc19022f2)
-   If you are unable to create a delivery, make sure there is a healthy robot deployed to the merchant’s parking lot and an available pilot (you may need to log in to the Pilot UI).
