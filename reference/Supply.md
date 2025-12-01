---
tags:
  - concept
  - dispatch-engine
---
# Supply

**Supply** in the [[Dispatch Engine]] represents the pool of available resources that can be used to fulfill a [[Demand]].

The primary resources are:
- [[Robot|Robots]]
- Pilots

The `SupplyModule` is responsible for managing these resources. It interacts with the `ResourceService` to get the real-time status, location, and availability of all pilots and [[Robot|robots]], which is then fed into the [[PlannerModule]] for [[Continuous Replanning]].
