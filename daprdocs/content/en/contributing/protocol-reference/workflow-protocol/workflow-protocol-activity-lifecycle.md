---
type: docs
title: "Workflow Protocol - Activity Lifecycle"
linkTitle: "Activity Lifecycle"
weight: 400
description: "Low-level description of the Workflow building block internals."
---

# Activity Lifecycle

Activities are the basic units of work in a Dapr Workflow. Unlike orchestrations, activities are not replayed and do 
not need to be deterministic. They are executed exactly once per "schedule" (though retries may occur).

## Execution Flow

1.  **Scheduling**: An orchestration requests an activity by sending a `ScheduleTask` action to the Dapr engine.
2.  **Work Item Dispatch**: The Dapr engine enqueues an activity task. When an activity worker (SDK) is available, the engine sends an `ActivityWorkItem` via the `GetWorkItems` stream.
3.  **Execution**: The SDK receives the `ActivityWorkItem`, which contains:
    *   `name`: The name of the activity to execute.
    *   `input`: The input data for the activity.
    *   `instance_id`: The ID of the workflow instance that scheduled the activity.
    *   `task_id`: A unique identifier for this specific activity execution.
4.  **Reporting**: After the activity logic finishes, the SDK sends a `CompleteActivityTask` request back to Dapr.
    *   **Success**: The SDK provides the serialized output in the `result` field.
    *   **Failure**: The SDK provides `failure_details` (error message, type, stack trace).

## Retries

Dapr handles activity retries based on the policy defined in the orchestration (if the SDK supports defining retry 
policies in the `ScheduleTask` action). If an activity fails and a retry policy is in place, the engine will re-enqueue 
the activity task after the specified delay.

From the activity worker's perspective, a retry is simply a new `ActivityWorkItem` with the same name and input, but 
potentially a different `task_id` (or the same, depending on the backend implementation).

## Idempotency

Because activities might be executed more than once (e.g., if the worker crashes after execution but before reporting 
completion), it is recommended that activity logic be idempotent where possible.

## Comparison with Orchestrations

| Feature | Orchestration | Activity |
| :--- | :--- | :--- |
| **Execution Style** | Replay-based (Deterministic) | Direct execution |
| **State** | Managed via History Events | No internal workflow state |
| **Side Effects** | Forbidden (must use activities) | Allowed (IO, Database, etc.) |
| **Lifetime** | Can be long-running (days/months) | Usually short-lived |
| **Connectivity** | Connected via `GetWorkItems` | Connected via `GetWorkItems` |
