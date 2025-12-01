---
tags:
  - dispatch-engine
  - planning
  - architecture
  - service
---
# Planner Service

The `PlannerService` is the central orchestration component of the [[Dispatch Engine]]. Its primary responsibility is to create and continuously update delivery plans for the entire fleet of robots. It handles the complex logic of matching available robots and pilots to incoming delivery demands, calculating routes and timings, and responding to real-time changes in the environment.

## Core Responsibilities

The service's responsibilities can be broken down into three main functions:

1.  **[[PlannerService - Estimate]]**: Provides a quote for a potential delivery, including timing and robot assignment, without committing to the plan.
2.  **[[PlannerService - Create]]**: Commits to a delivery plan by creating the necessary [[Demand]] and [[Trip]] records in the system.
3.  **[[PlannerService - Replan]]**: The core, continuously running loop that adapts to change, recalculates all plans, and activates new tasks. This is a form of [[Continuous Replanning]].

## Service Interactions

The `PlannerService` acts as an orchestrator and relies on several other services to perform its duties. See [[PlannerService - Service Interactions]] for a detailed breakdown and a diagram of its dependencies.
