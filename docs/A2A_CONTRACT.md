# A2A Contract for AI Orchestration Engine

**Version:** 1.0  
**Status:** Draft  
**Owner:** ideabosque  
**Last Updated:** 2025-04-30

---

## 1. Overview

This document defines the SilvaEngine message contract between the **AI Orchestration Engine (AOE)** and the **A2A Daemon** for task dispatch and result handling.

Important distinction: `assign_task` is treated as a SilvaEngine daemon wrapper. The daemon should map it to the official A2A protocol shape, preferably A2A `message/send` with `Message.parts`, metadata, task states, artifacts, and `pushNotificationConfig` for callbacks. If the daemon exposes official A2A directly, AOE should call that through `handlers/a2a_client.py` and keep this document as the AOE metadata/schema contract.

Recommended server roles:
- `ai_a2a_daemon_engine`: A2A gateway / registry / routing wrapper.
- `ai_agent_core_engine`: worker-side A2A server endpoint that invokes the actual agent.
- `ai_orchestration_engine`: orchestration owner and callback / push-notification receiver.

It specifies:

- Outbound messages: AOE → A2A (task dispatch)
- Inbound messages: A2A → AOE (step results and events)
- Payload schemas and validation rules
- Error handling and retry semantics

---

## 2. Message Types

### 2.1 Outbound: AOE Dispatches Step

**Action:** `assign_task`

AOE may call the A2A daemon's `assign_task` endpoint to dispatch a step to an agent when the daemon wrapper is the confirmed integration path. This is not the raw A2A method name; the wrapper should translate to A2A `message/send`.

#### Request Payload

```json
{
  "action": "assign_task",
  "endpoint_id": "prod-us-east-1",
  "task_id": "step-session-uuid-abc123",
  "task_type": "orchestration_step",
  "assigned_agent_id": "agent-uuid-xyz789",
  "input_data": {
    "step_session_uuid": "step-session-uuid-abc123",
    "contract_version": "1.0",
    "a2a_protocol_version": "0.3.0",
    "task_run_uuid": "task-run-uuid-def456",
    "step_id": "step_1",
    "agent_uuid": "agent-uuid-xyz789",
    "user_id": "user-123",
    "query": "Summarize the following document",
    "predecessor_outputs": [
      {
        "step_id": "step_0",
        "output": { ... },
        "completed_at": "2025-04-30T12:00:00Z"
      }
    ],
    "files": [
      {
        "file_uuid": "file-abc",
        "filename": "document.pdf",
        "mime_type": "application/pdf"
      }
    ],
    "thread_uuid": "thread-uuid-optional",
    "attempt": 1,
    "callback": {
      "agent_id": "ai_orchestration_engine",
      "endpoint_url": "https://aoe.example.com/a2a/callback"
    }
  }
}
```

#### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | Must be `"assign_task"` |
| `endpoint_id` | string | Yes | Target environment (e.g., `prod-us-east-1`) |
| `task_id` | UUID | Yes | AOE-generated StepSession UUID |
| `task_type` | string | Yes | Must be `"orchestration_step"` |
| `assigned_agent_id` | UUID | Yes | The `agent_uuid` to execute this step |
| `input_data` | object | Yes | Step execution context |
| `input_data.step_session_uuid` | UUID | Yes | Correlation ID for callbacks |
| `input_data.contract_version` | string | Yes | AOE wrapper contract version, e.g. `"1.0"` |
| `input_data.a2a_protocol_version` | string | Yes | A2A protocol version used by the daemon adapter, e.g. `"0.3.0"` |
| `input_data.task_run_uuid` | UUID | Yes | Parent TaskRun reference |
| `input_data.step_id` | string | Yes | Step identifier within Task DAG |
| `input_data.agent_uuid` | UUID | Yes | Same as `assigned_agent_id` |
| `input_data.user_id` | string | Yes | End user identifier |
| `input_data.query` | string | Yes | Primary input text/query |
| `input_data.predecessor_outputs` | array | No | Outputs from upstream steps |
| `input_data.files` | array | No | Input file references |
| `input_data.thread_uuid` | UUID | No | Conversation continuity ID |
| `input_data.attempt` | integer | Yes | Retry attempt counter (1-indexed) |
| `input_data.callback` | object | Yes | AOE callback configuration |
| `input_data.callback.agent_id` | string | Yes | Must be `"ai_orchestration_engine"` |
| `input_data.callback.endpoint_url` | string | Yes | HTTPS URL for A2A callbacks |

#### Response

**Success (200 OK):**

```json
{
  "task_id": "step-session-uuid-abc123",
  "status": "accepted",
  "a2a_task_id": "a2a-task-uuid-123",
  "estimated_completion": "2025-04-30T12:05:00Z"
}
```

**Failure (4xx/5xx):**

```json
{
  "error": {
    "code": "AGENT_NOT_FOUND",
    "message": "Assigned agent not registered with A2A daemon",
    "retryable": false
  }
}
```

---

### 2.2 Inbound: Step Started

**Message Type:** `step.started`

Sent by the agent runtime (AI Agent Core) when step execution begins.

```json
{
  "message_type": "step.started",
  "timestamp": "2025-04-30T12:00:01Z",
  "step_session_uuid": "step-session-uuid-abc123",
  "task_run_uuid": "task-run-uuid-def456",
  "step_id": "step_1",
  "agent_uuid": "agent-uuid-xyz789",
  "agent_thread_uuid": "thread-uuid-from-core",
  "started_at": "2025-04-30T12:00:01Z"
}
```

---

### 2.3 Inbound: Step Result

**Message Type:** `step.result`

Sent when step execution completes successfully.

```json
{
  "message_type": "step.result",
  "timestamp": "2025-04-30T12:05:00Z",
  "step_session_uuid": "step-session-uuid-abc123",
  "task_run_uuid": "task-run-uuid-def456",
  "step_id": "step_1",
  "agent_uuid": "agent-uuid-xyz789",
  "attempt": 1,
  "status": "succeeded",
  "output": {
    "text": "Generated summary text...",
    "tokens_used": 150,
    "model": "gpt-4",
    "metadata": { ... }
  },
  "completed_at": "2025-04-30T12:05:00Z",
  "agent_thread_uuid": "thread-uuid-from-core",
  "agent_async_task_uuid": "async-task-uuid"
}
```

#### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message_type` | string | Yes | Must be `"step.result"` |
| `timestamp` | ISO8601 | Yes | When message was generated |
| `step_session_uuid` | UUID | Yes | Correlation ID |
| `task_run_uuid` | UUID | Yes | Parent TaskRun reference |
| `step_id` | string | Yes | Step identifier |
| `agent_uuid` | UUID | Yes | Executing agent |
| `attempt` | integer | Yes | Which attempt this result is for |
| `status` | string | Yes | `"succeeded"` |
| `output` | object | Yes | Agent output (opaque to AOE) |
| `completed_at` | ISO8601 | Yes | Step completion timestamp |
| `agent_thread_uuid` | UUID | No | AI Agent Core thread reference |
| `agent_async_task_uuid` | UUID | No | AI Agent Core async task reference |

---

### 2.4 Inbound: Step Failure

**Message Type:** `step.failure`

Sent when step execution fails.

```json
{
  "message_type": "step.failure",
  "timestamp": "2025-04-30T12:05:00Z",
  "step_session_uuid": "step-session-uuid-abc123",
  "task_run_uuid": "task-run-uuid-def456",
  "step_id": "step_1",
  "agent_uuid": "agent-uuid-xyz789",
  "attempt": 1,
  "status": "failed",
  "error": {
    "type": "LLM_RATE_LIMIT",
    "message": "OpenAI API rate limit exceeded",
    "code": "RATE_LIMIT_ERROR",
    "retryable": true,
    "retry_after_seconds": 60
  },
  "completed_at": "2025-04-30T12:05:00Z"
}
```

#### Error Types

| Type | Retryable | Description |
|------|-----------|-------------|
| `LLM_RATE_LIMIT` | Yes | LLM provider rate limit hit |
| `LLM_TIMEOUT` | Yes | LLM call timed out |
| `LLM_ERROR` | Yes | Transient LLM failure |
| `AGENT_CONFIG_ERROR` | No | Agent misconfiguration |
| `INVALID_INPUT` | No | Input validation failed |
| `TOOL_ERROR` | Depends | Tool execution failed |
| `UNKNOWN_ERROR` | Yes | Unclassified error |

---

### 2.5 Inbound: Input Required

**Message Type:** `step.input_required`

Sent when agent needs user clarification.

```json
{
  "message_type": "step.input_required",
  "timestamp": "2025-04-30T12:03:00Z",
  "step_session_uuid": "step-session-uuid-abc123",
  "task_run_uuid": "task-run-uuid-def456",
  "step_id": "step_1",
  "agent_uuid": "agent-uuid-xyz789",
  "attempt": 1,
  "status": "waiting_for_input",
  "prompt": {
    "text": "Please clarify: do you want a brief summary or detailed analysis?",
    "options": ["brief", "detailed"],
    "required": true
  },
  "context": {
    "partial_output": { ... },
    "files_processed": 3
  }
}
```

---

### 2.6 Inbound: Step Heartbeat

**Message Type:** `step.heartbeat`

Optional liveness signal (every 30s recommended for long-running steps).

```json
{
  "message_type": "step.heartbeat",
  "timestamp": "2025-04-30T12:02:30Z",
  "step_session_uuid": "step-session-uuid-abc123",
  "task_run_uuid": "task-run-uuid-def456",
  "progress": {
    "percent_complete": 45,
    "current_operation": "Processing file 3 of 5"
  }
}
```

---

## 3. Validation Rules

### 3.1 Inbound Message Validation

AOE validates every inbound message and rejects malformed messages with structured error:

```json
{
  "error": "INVALID_MESSAGE",
  "message": "Missing required field: step_session_uuid",
  "received_at": "2025-04-30T12:00:00Z"
}
```

**Required Validations:**
- `step_session_uuid` must exist in `StepSession` table
- `message_type` must be in allowed set
- `timestamp` must be within 5 minutes of current time (prevent replay attacks)
- `attempt` must match current `StepSession.attempt`
- Message signature must verify (if A2A implements message signing)

### 3.2 Idempotency

All inbound wrapper messages are idempotent by `step_session_uuid` + `message_type` + `attempt`. Timestamp is used for replay-window validation, not as the durable idempotency key, because duplicate deliveries may legitimately arrive with different delivery timestamps.

---

## 4. Retry Semantics

### 4.1 Outbound Retries

AOE retries A2A `assign_task` calls on:

| Scenario | Retry Count | Backoff |
|----------|-------------|---------|
| A2A timeout | 3 | Exponential: 1s, 2s, 4s |
| A2A 5xx | 3 | Exponential: 1s, 2s, 4s |
| A2A 4xx (non-retryable) | 0 | N/A |

### 4.2 Inbound Acknowledgment

AOE must acknowledge inbound messages within 30 seconds:

```json
{
  "status": "acknowledged",
  "received_at": "2025-04-30T12:00:00Z",
  "processing_estimate_ms": 500
}
```

If AOE fails to process within 30s, A2A should retry delivery (up to 3 times).

---

## 5. Security Considerations

### 5.1 Authentication

- AOE ↔ A2A: mTLS with service-to-service certificates
- Callback URL must be HTTPS with valid certificate
- `endpoint_id` must match authenticated identity

### 5.2 Authorization

- AOE can only dispatch to agents in its `procedure.agents` list
- AOE callbacks only accepted from known A2A daemon IPs

### 5.3 Payload Security

- Sensitive data in `input_data.query` may be encrypted at rest
- `files` array contains references only, not actual file content
- `predecessor_outputs` may contain sensitive data; handle per data classification

---

## 6. Versioning

### 6.1 Contract Evolution

This contract follows semantic versioning:

- **Major**: Breaking changes (e.g., required field removed)
- **Minor**: Additive changes (e.g., optional field added)
- **Patch**: Documentation/clarification

### 6.2 Version Negotiation

AOE includes expected contract version in `input_data.contract_version`:

```json
{
  "input_data": {
    "contract_version": "1.0",
    ...
  }
}
```

A2A rejects requests with unsupported versions:

```json
{
  "error": {
    "code": "UNSUPPORTED_VERSION",
    "message": "Contract version 2.0 not supported. Supported: [1.0]"
  }
}
```

---

## 7. Error Handling

### 7.1 A2A Delivery Failures

If A2A cannot deliver a message to AI Agent Core:

1. Retry with exponential backoff (max 3 attempts)
2. After max retries, emit `step.failure` to AOE with `error.type = "A2A_DELIVERY_FAILED"`
3. Mark step as failed; AOE retry policy applies

### 7.2 AOE Callback Failures

If AOE callback endpoint is unavailable:

1. A2A retries with exponential backoff (max 5 attempts)
2. After max retries, store message in dead letter queue
3. Emit CloudWatch alarm
4. Manual intervention required

---

## 8. Testing

### 8.1 Contract Tests

CI must validate:
- All message types serialize/deserialize correctly
- Schema validation rejects malformed messages
- Version negotiation works correctly
- Idempotency keys prevent duplicates

### 8.2 Mock A2A Daemon

For local development (Phase 0.5):

```python
# Mock implementation for testing
class MockA2ADaemon:
    def assign_task(self, payload):
        # Validate payload against schema
        # Echo worker: immediately return result
        # Real worker: async callback after delay
        pass
```

---

## 9. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-04-30 | Initial draft |

---

## 10. References

- [A2A Daemon Documentation](https://github.com/silvaengine/ai_a2a_daemon_engine)
- [AI Agent Core Integration](https://github.com/silvaengine/ai_agent_core_engine)
- [Development Plan](../DEVELOPMENT_PLAN.md)
