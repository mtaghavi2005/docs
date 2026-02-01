---
type: docs
title: "Workflow Protocol - Management API"
linkTitle: "Management API"
weight: 100
description: "Low-level description of the Workflow building block internals."
---

# Workflow Management API

The Workflow Management API allows Dapr clients to control the lifecycle of workflow instances. These APIs are exposed 
via the standard Dapr gRPC endpoint.

## gRPC Service Definition

The management APIs are part of the `Dapr` service in `dapr.proto.runtime.v1`. While multiple versions 
(Alpha1, Beta1) may exist, the following describes the current implementation logic.

### StartWorkflow

Starts a new instance of a workflow.

**Request (`StartWorkflowRequest`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | Optional. A unique identifier for the workflow instance. If not provided, Dapr will generate a random UUID. |
| `workflow_component` | `string` | The name of the workflow component to use. Currently, Dapr uses the built-in engine. |
| `workflow_name` | `string` | The name of the workflow definition to execute. |
| `options` | `map<string, string>` | Optional. Component-specific options. |
| `input` | `bytes` | Optional. Input data for the workflow instance, typically a JSON-serialized string. |

**Response (`StartWorkflowResponse`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the started workflow instance. |

### GetWorkflow

Retrieves the current status and metadata of a workflow instance.

**Request (`GetWorkflowRequest`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the workflow instance to query. |
| `workflow_component` | `string` | The name of the workflow component. |

**Response (`GetWorkflowResponse`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the workflow instance. |
| `workflow_name` | `string` | The name of the workflow. |
| `created_at` | `Timestamp` | The time the instance was created. |
| `last_updated_at` | `Timestamp` | The time the instance was last updated. |
| `runtime_status` | `string` | The status (e.g., `RUNNING`, `COMPLETED`, `FAILED`, `TERMINATED`, `PENDING`). |
| `properties` | `map<string, string>` | Additional component-specific metadata. |

### TerminateWorkflow

Forcefully terminates a running workflow instance.

**Request (`TerminateWorkflowRequest`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the workflow instance to terminate. |
| `workflow_component` | `string` | The name of the workflow component. |

### RaiseEventWorkflow

Sends an event to a running workflow instance.

**Request (`RaiseEventWorkflowRequest`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the workflow instance. |
| `workflow_component` | `string` | The name of the workflow component. |
| `event_name` | `string` | The name of the event to raise. |
| `event_data` | `bytes` | The data associated with the event. |

### PauseWorkflow & ResumeWorkflow

Pauses or resumes a workflow instance.

**Request (`PauseWorkflowRequest` / `ResumeWorkflowRequest`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the workflow instance. |
| `workflow_component` | `string` | The name of the workflow component. |

### PurgeWorkflow

Removes all state and history associated with a workflow instance. This can usually only be done for completed, 
failed, or terminated instances.

**Request (`PurgeWorkflowRequest`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `instance_id` | `string` | The ID of the workflow instance. |
| `workflow_component` | `string` | The name of the workflow component. |

## Implementation Details

The sidecar receives these requests and translates them into operations on the underlying `durabletask-go` client. 
For example, `StartWorkflow` calls the backend to create a new orchestration instance and enqueue an 
`ExecutionStarted` event.
