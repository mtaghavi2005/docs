---
type: docs
title: "Workflow Protocol - Versioning"
linkTitle: "Versioning"
weight: 600
description: "Low-level description of the Workflow building block internals."
---

# Workflow Versioning

Dapr Workflow supports versioning of workflow definitions, allowing you to update workflow logic while existing 
instances continue to run on their original logic.

## Named Workflow Versioning

When registering a workflow with the Dapr engine, you can provide a version name. This allows multiple versions of the 
same workflow to coexist.

*   **Default Version**: One version of a workflow can be marked as the default. If a client starts a workflow by name 
    without specifying a version, the default version is used.
*   **Specific Version**: Clients can request a specific version of a workflow when starting a new instance.

### Registration API

SDKs register versioned workflows using the `AddVersionedOrchestrator` (or similar) method in their task registry.

```go
// Example (Internal Registry API)
registry.AddVersionedOrchestrator("MyWorkflow", "v2", true, MyWorkflowV2)
registry.AddVersionedOrchestrator("MyWorkflow", "v1", false, MyWorkflowV1)
```

## Sidecar Versioning

The Dapr sidecar tracks which version of a workflow an instance is running. This information is stored in the workflow 
history within the `OrchestratorStarted` event's `version` field.

```protobuf
message OrchestrationVersion {
  string name = 1;
  repeated string patches = 2;
}
```

When an instance is resumed (e.g., after an activity completes), the Dapr engine ensures it is dispatched to a worker 
that has the correct version of the workflow registered.
