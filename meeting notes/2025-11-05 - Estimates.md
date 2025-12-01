Things can go wrong with a estimate:
* Check if robots are avialable, assign
* Problem, cant generaet an estimate, or an demand isn't getting assigned
* For a robot that was previously assigned, the robot lost the order, can we figure out why the robot that was assigned wasnt able to pick up the order
* When a new robot gets assigned, we want to have the information about why it was unassigned
* Need to track the next assignment to make sure the re-assignment is correct
Main issues:
* No observability of assignments, re-assignments, 

## Datadog setup
Use these new logs and metrics to create widgets:

- Key Metric: dispatch-engine.planner.replan.demand_orphaned
	- This metric counts every time a demand is forced to be reassigned.
	- It includes a reason tag (e.g., reason:Offline, reason:Unhealthy) which is perfect for creating timeseries graphs and pie charts that break down why reassignments are happening.
- Key Logs:
	1. [replan] orphaning demand from robot: The most important log. It tells you the exact reason a robot was dropped.
	2. [replan] robot reassignment: Confirms the reassignment and shows the oldDeviceId and newDeviceId.
	3. [estimate] estimate assignment: Shows the very first robot that was assigned to a demand.

## Datadog monitoring
This dashboard will help you quickly understand the health of the dispatching system.

- How to Monitor:
	- Keep an eye on the Total Reassignments count. A sudden spike indicates a problem with our robot fleet.
	- Look at the Reassignments by Reason graph. This tells you if the spike is caused by a network issue (Offline), hardware problems (Needs maintenance), or something else.
- How to Investigate a Specific Delivery:
	1. Find the reassignment log in the dashboard and copy the demandId.
	2. Search for that demandId in Datadog logs.
	3. You will see the complete story in one place: why the original robot was dropped and which new robot took over.