# S3 File Tags State Change Process Documentation

## Overview
This document describes the state change process of S3 file object Tags across various service nodes.

## Core Nodes
- **File Sync Service**: File synchronization service responsible for syncing data from data sources to S3
- **DataObject**: Data file objects on S3, managed through Tags for state control
- **Embedding Service**: Downstream vectorization processing service

## S3 Tags Definition

### Core Status Tags
```
- status: "ready" | "processing" | "completed" | "failed"
- retry_count: "0" | "1" | "2" (optional, added during failure retry)
```

## Tags State Change Process

### Scenario 1: File Sync Service Creates New File
```
DataObject Tags Setting:
└── status: "ready"
```

### Scenario 2: File Sync Service Updates File
```
DataObject Tags Update:
└── status: "ready"     # Reset to ready for processing
```

### Scenario 3: Embedding Service Processes File
```
Before processing: status="ready" → status="processing"
After processing: status="processing" → status="completed"
```

### Scenario 4: Processing Failure Retry
```
On failure: status="processing" → status="failed" + retry_count="1"
On retry: status="failed" → status="ready"
```

## Complete Flow Diagram

The complete state transitions and processing logic are shown in the flow diagram below, including normal processing paths and exception retry mechanisms.

### Overall Processing Flow Diagram
```mermaid
graph TD
    A["File Sync Service<br/>Create/Update File"] --> B["DataObject<br/>status: ready"]
    
    B --> C["Embedding Service<br/>Discover File"]
    C --> D["DataObject<br/>status: processing"]
    
    D --> E["Embedding Service<br/>Process File"]
    
    E --> F{"Processing Result"}
    F -->|Success| G["DataObject<br/>status: completed"]
    F -->|Failure| H["DataObject<br/>status: failed<br/>retry_count: +1"]
    
    H --> I{"Retry Decision"}
    I -->|Retryable| J["Reset Status<br/>status: ready"]
    I -->|Exceed Retry Limit| K["Manual Processing Queue"]
    
    J --> C
    
    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style G fill:#e8f5e8
    style H fill:#ffebee
    style K fill:#fff3e0
```

### State Transition Diagram
```mermaid
stateDiagram-v2
    [*] --> ready: File Sync Service<br/>Create/Update File
    
    ready --> processing: Embedding Service<br/>Acquire Processing Rights
    
    processing --> completed: Processing Success
    processing --> failed: Processing Failure
    
    failed --> ready: Retry Mechanism
    failed --> [*]: Exceed Retry Limit<br/>Manual Processing
    
    completed --> ready: File Sync Service<br/>File Updated
    
    note right of ready: status="ready"
    note right of processing: status="processing"
    note right of completed: status="completed"
    note right of failed: status="failed"<br/>retry_count="1/2/3"
```

## Concurrency Control
- Embedding Service acquires tasks through Tags state preemption
- Only instances that successfully update status from "ready" to "processing" gain processing rights
- Regularly clean up timeout processing status files

---

*Document Version: v2.0* 