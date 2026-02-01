---
type: docs
title: "Workflow Protocol - Orchestration Lifecycle"
linkTitle: "Orchestration Lifecycle"
weight: 300
description: "Low-level description of the Workflow building block internals."
---

# Orchestration Lifecycle

This document describes the lifecycle of an orchestration at the protocol level, specifically how the Dapr engine and 
the SDK interact to execute workflow logic reliably.

## Replay-based Execution

Dapr Workflows use **event sourcing** and **replay** to maintain state. Instead of saving the entire state of the 
worker process (stack, variables, etc.), Dapr saves a history of events that have occurred.

### The Replay Loop

1.  **Work Item Arrival**: The Dapr engine sends an `OrchestratorWorkItem` to the SDK via the `GetWorkItems` stream. 
This work item contains the full history of the workflow instance plus any new events (e.g., an activity completion 
or an external event).
2.  **Reconstruction**: The SDK starts executing the orchestration function from the very beginning.
3.  **Deterministic Execution**: As the function executes, it encounters "tasks" (e.g., calling an activity, sleeping).
    *   For each task, the SDK checks the provided **History** to see if that task has already completed.
    *   If the task is in the history, the SDK returns the recorded result immediately without actually re-executing 
        the task logic.
    *   If the task is NOT in the history, the SDK records that this task needs to be scheduled and **suspends**
        execution of the orchestration function (typically by throwing a special exception or returning a pending 
        promise).
4.  **Reporting**: Once the orchestration function is suspended or completes, the SDK sends a 
    `CompleteOrchestratorTask` request to Dapr. This request contains a list of **Actions** (e.g., `ScheduleTask`, 
    `CreateTimer`) that the engine should perform.
5.  **State Commitment**: The Dapr engine receives the actions, updates the workflow history in the state store, and 
    schedules any requested tasks (e.g., by sending work to an activity worker).

## Step-by-Step Example

Imagine a workflow: `Activity A -> Activity B`.

### 1. Workflow Start
*   **Engine**: Enqueues `ExecutionStarted` event.
*   **SDK**: Receives `OrchestratorWorkItem` with `[ExecutionStarted]`.
*   **SDK**: Runs function. Function calls `Activity A`.
*   **SDK**: Checks history. `Activity A` is not there.
*   **SDK**: Suspends. Sends `CompleteOrchestratorTask` with `[ScheduleTask(Activity A)]`.
*   **Engine**: Records `TaskScheduled(Activity A)` in history.

### 2. Activity A Completes
*   **Engine**: Records `TaskCompleted(Activity A, result="foo")` in history.
*   **SDK**: Receives `OrchestratorWorkItem` with `[ExecutionStarted, TaskScheduled(A), TaskCompleted(A)]`.
*   **SDK**: Runs function from start.
*   **SDK**: Function calls `Activity A`. SDK finds `TaskCompleted(A)` in history. Returns `"foo"`.
*   **SDK**: Function calls `Activity B`.
*   **SDK**: Checks history. `Activity B` is not there.
*   **SDK**: Suspends. Sends `CompleteOrchestratorTask` with `[ScheduleTask(Activity B)]`.
*   **Engine**: Records `TaskScheduled(Activity B)`.

### 3. Workflow Completion
*   **Activity B** completes.
*   **SDK**: Receives history with both A and B completed.
*   **SDK**: Runs function. Both A and B return results from history.
*   **SDK**: Function finishes and returns a final result.
*   **SDK**: Sends `CompleteOrchestratorTask` with `[CompleteOrchestration(result="final")]`.
*   **Engine**: Records `OrchestrationCompleted` and marks the instance as `COMPLETED`.

## Critical Requirements for SDK Authors

### 1. Determinism
The orchestration function MUST be deterministic. It cannot use:
*   Random numbers.
*   Current date/time (must use a durable timer or a provided `CurrentUtcDateTime`).
*   Direct IO (must be done in activities).
*   Global state that can change between replays.

### 2. History Management
The SDK must efficiently search the history. Typically, this is done by maintaining a counter of tasks encountered 
during execution and matching them against the sequence of events in the history.

### 3. Graceful Suspension
The SDK needs a mechanism to stop execution of the orchestration function when a task is scheduled but not yet
completed, without losing the ability to restart it later.
