# System Mechanics Specification - Intent Routing Semantics v1.6

## Overview

Intent Routing provides declarative configuration for how InputIntents are transported and delivered. This extends SMS with explicit transport layer semantics, enabling specification of NATS, HTTP, and gRPC routing with configurable delivery modes and response handling.

---

## Invariant Clarification: Transport Rule Scope (v1.6)

### Background

The SMS base specification (v1.5 and earlier) establishes a normative principle in Part X:

> **"Transport is not encoded in the grammar"**
> - Local-first execution is preferred when valid
> - Brokered execution is used when required

And in the Core Principles:

> **"Intent-First, Runtime-Decided"**
> - Execution strategy, transport, locality, and optimization are runtime responsibilities

### v1.6 Extension: External API Contract vs. Internal Flow Execution

Version 1.6 explicitly **scopes** the transport invariant to distinguish between two distinct concerns:

| Concern | Scope | Transport Encoding | Invariant |
|---------|-------|-------------------|-----------|
| **SMS Flow Execution** | Internal | **NOT PERMITTED** | Flows remain logical graphs; runtime decides transport |
| **Intent Delivery Contract** | External API | **PERMITTED** | InputIntent may declare transport bindings for external clients |

### Rationale

**SMS Flows** (internal execution):
- Describe ordering, dependency, and atomicity
- Runtime MUST choose execution paths that satisfy constraints
- Transport remains a runtime decision
- The v1.5 invariant is **preserved unchanged**

**InputIntent Routing** (external API contract):
- Defines how external clients (web, mobile, services) submit intents to the system
- Represents an **API boundary** that requires stable, documented contracts
- Clients need to know: "How do I invoke this intent?"
- Transport binding is part of the **public interface**, not internal execution

### Formal Invariant (v1.6)

**I9. Transport Scope Boundary**

> Transport bindings SHALL be specified only at system ingress points (InputIntent routing).
> Transport SHALL NOT be specified within SMS flow definitions.
> Flow steps, atomic groups, and work unit execution SHALL remain transport-agnostic.

**Corollary**:
- An `inputIntent.routing` block defines the **external contract** for intent submission
- An `sms.flows` block defines the **internal execution graph** without transport
- The runtime bridges these concerns: accepting intents via declared transport, executing flows via optimal internal paths

### Example: Correct Separation

```yaml
# CORRECT: Transport binding at intent ingress (external contract)
inputIntent:
  name: InitiateTransfer
  version: v1
  routing:
    transport: nats
    nats:
      subject_pattern: "SMS.INTENT.banking.InitiateTransfer"
      delivery: jetstream

# CORRECT: Flow remains transport-agnostic (internal execution)
sms:
  flows:
    - name: TransferFlow
      triggeredBy:
        inputIntent: InitiateTransfer
      steps:
        - work_unit: ValidateTransfer      # No transport specified
        - work_unit: ExecuteTransfer       # Runtime decides how to invoke
        - work_unit: NotifyParties         # May be local, remote, or async
```

### Non-Conformant Example

```yaml
# INCORRECT: Transport binding inside flow step (violates invariant)
sms:
  flows:
    - name: TransferFlow
      steps:
        - work_unit: ValidateTransfer
          transport: nats                   # VIOLATION: Transport in flow
          subject: "SMS.WORK.validate"      # VIOLATION: Subject in flow
```

---

## Intent Routing Declaration

### IntentRouting

Defines routing configuration for an InputIntent.

```yaml
inputIntent:
  name: <string>
  version: <vN>
  proposes:
    dataState: <DataState>
  supplies:
    required: [<field>, ...]
    optional: [<field>, ...]
  routing:
    transport: nats | http | grpc
    nats:
      subject_pattern: <pattern_expression>
      delivery: jetstream | core
      stream: <stream_name>
      queue: <queue_group>
    http:
      path: <path_pattern>
      method: POST | PUT | PATCH
      timeout: <duration>
    grpc:
      service: <service_name>
      method: <method_name>
      streaming: true | false
    response:
      mode: request_reply | async | fire_and_forget
      timeout: <duration>
      callback: <subject_pattern | url>
```

---

## Transport Types

### NATS Transport

Native transport using NATS messaging.

```yaml
inputIntent:
  name: InitiateTransfer
  version: v1
  proposes:
    dataState: TransferRequest
  routing:
    transport: nats
    nats:
      subject_pattern: "SMS.INTENT.banking.InitiateTransfer.{{realm}}"
      delivery: jetstream
      stream: SMS_INTENTS
      queue: transfer-workers
    response:
      mode: request_reply
      timeout: 30s
```

**Subject Pattern Expressions**:
- `{{domain}}` - Domain/bounded context
- `{{intent_name}}` - Intent name
- `{{realm}}` - Tenant realm
- `{{version}}` - Intent version
- Static segments: `SMS.INTENT.banking`

**Delivery Modes**:

| Mode | Description | Guarantees |
|------|-------------|------------|
| `jetstream` | JetStream persistent delivery | At-least-once, durable |
| `core` | Core NATS ephemeral delivery | At-most-once, fire-and-forget |

**Queue Groups**:
- Enable load balancing across worker instances
- Only one subscriber in queue receives each message
- Omit for broadcast to all subscribers

---

### HTTP Transport

RESTful HTTP transport for external/web clients.

```yaml
inputIntent:
  name: InitiateTransfer
  version: v1
  proposes:
    dataState: TransferRequest
  routing:
    transport: http
    http:
      path: "/api/v1/transfers"
      method: POST
      timeout: 30s
    response:
      mode: request_reply
      timeout: 30s
```

**Path Patterns**:
- Static paths: `/api/v1/transfers`
- Parameterized: `/api/v1/accounts/{account_id}/transfers`
- Version prefix: `/api/{{version}}/transfers`

**HTTP Methods**:

| Method | Use Case |
|--------|----------|
| `POST` | Create new resource (default) |
| `PUT` | Replace resource |
| `PATCH` | Partial update |

---

### gRPC Transport

High-performance RPC transport.

```yaml
inputIntent:
  name: InitiateTransfer
  version: v1
  proposes:
    dataState: TransferRequest
  routing:
    transport: grpc
    grpc:
      service: banking.TransferService
      method: InitiateTransfer
      streaming: false
    response:
      mode: request_reply
      timeout: 30s
```

**Streaming Modes**:

| Streaming | Description |
|-----------|-------------|
| `false` | Unary RPC (single request, single response) |
| `true` | Streaming RPC (for large payloads or real-time) |

---

## Response Modes

### Request-Reply

Synchronous request with immediate response.

```yaml
response:
  mode: request_reply
  timeout: 30s
```

**Semantics**:
- Client blocks waiting for response
- Timeout triggers error
- Best for: User-initiated actions requiring feedback

**NATS Implementation**: Uses request-reply pattern with inbox subject

---

### Async

Asynchronous processing with callback notification.

```yaml
response:
  mode: async
  timeout: 5m
  callback: "SMS.CALLBACK.{{correlation_id}}"
```

**Semantics**:
- Client receives acknowledgment immediately
- Result delivered via callback when complete
- Best for: Long-running operations

**Callback Patterns**:
- NATS subject: `SMS.CALLBACK.{{correlation_id}}`
- HTTP webhook: `https://api.example.com/callbacks/{{correlation_id}}`

---

### Fire and Forget

No response expected.

```yaml
response:
  mode: fire_and_forget
```

**Semantics**:
- Client receives no acknowledgment
- No delivery guarantee with core NATS
- At-least-once with JetStream
- Best for: Events, audit logs, non-critical notifications

---

## Subject Pattern Grammar

```ebnf
subject_pattern ::= subject_segment ( "." subject_segment )*

subject_segment ::= static_segment
                  | dynamic_segment
                  | wildcard

static_segment  ::= identifier

dynamic_segment ::= "{{" variable_name "}}"

variable_name   ::= "domain"
                  | "intent_name"
                  | "realm"
                  | "version"
                  | "entity_id"
                  | identifier

wildcard        ::= "*"   (* single token *)
                  | ">"   (* multiple tokens *)
```

**Standard Subject Pattern**:
```
SMS.INTENT.{{domain}}.{{intent_name}}
SMS.INTENT.{{domain}}.{{intent_name}}.{{realm}}
```

**Examples**:
```yaml
# Domain-scoped intent
subject_pattern: "SMS.INTENT.banking.InitiateTransfer"

# Realm-scoped intent
subject_pattern: "SMS.INTENT.banking.InitiateTransfer.{{realm}}"

# Versioned intent
subject_pattern: "SMS.INTENT.banking.InitiateTransfer.v{{version}}"

# Wildcard subscription (worker)
subject_pattern: "SMS.INTENT.banking.>"
```

---

## Transport Selection

### Selection Guidelines

| Scenario | Recommended Transport | Reason |
|----------|----------------------|--------|
| Internal microservices | NATS (JetStream) | Native, durable, low latency |
| External API clients | HTTP | Universal compatibility |
| High-performance RPC | gRPC | Efficient binary protocol |
| Browser clients | HTTP | Browser-native |
| Mobile apps | HTTP or gRPC | Platform support |
| Event-driven | NATS (core) | Lightweight, ephemeral |

### Multi-Transport Support

An intent MAY support multiple transports:

```yaml
inputIntent:
  name: InitiateTransfer
  version: v1
  routing:
    primary:
      transport: nats
      nats:
        subject_pattern: "SMS.INTENT.banking.InitiateTransfer"
        delivery: jetstream
    fallback:
      transport: http
      http:
        path: "/api/v1/transfers"
        method: POST
```

---

## Routing Examples

### High-Throughput Processing

```yaml
inputIntent:
  name: ProcessTransaction
  version: v1
  routing:
    transport: nats
    nats:
      subject_pattern: "SMS.TXN.process.{{realm}}"
      delivery: jetstream
      stream: SMS_TRANSACTIONS
      queue: txn-processors
    response:
      mode: fire_and_forget
```

### User-Facing Action

```yaml
inputIntent:
  name: CreateAccount
  version: v1
  routing:
    transport: http
    http:
      path: "/api/v1/accounts"
      method: POST
      timeout: 10s
    response:
      mode: request_reply
      timeout: 10s
```

### Long-Running Workflow

```yaml
inputIntent:
  name: ApproveApplication
  version: v1
  routing:
    transport: nats
    nats:
      subject_pattern: "SMS.WORKFLOW.approval.{{application_id}}"
      delivery: jetstream
      stream: SMS_WORKFLOWS
    response:
      mode: async
      timeout: 24h
      callback: "SMS.CALLBACK.approval.{{correlation_id}}"
```

---

## Validation Rules

### Structural Violations (MUST reject)

1. Routing without transport type
2. NATS routing without subject_pattern
3. HTTP routing without path
4. gRPC routing without service or method
5. Async response without callback
6. Invalid subject pattern syntax
7. Invalid HTTP method

### Semantic Violations (MUST reject)

1. JetStream delivery without stream reference
2. Fire-and-forget with request_reply response mode
3. Timeout with fire_and_forget mode
4. Queue group with wildcard-only subject
5. Streaming gRPC with fire_and_forget

---

## EBNF Grammar

```ebnf
intent_routing  ::= "routing:" NEWLINE INDENT
                    "transport:" transport_type NEWLINE
                    transport_config
                    response_config?
                    DEDENT

transport_type  ::= "nats" | "http" | "grpc"

transport_config ::= nats_config
                   | http_config
                   | grpc_config

nats_config     ::= "nats:" NEWLINE INDENT
                    "subject_pattern:" string NEWLINE
                    ( "delivery:" delivery_mode NEWLINE )?
                    ( "stream:" identifier NEWLINE )?
                    ( "queue:" identifier NEWLINE )?
                    DEDENT

delivery_mode   ::= "jetstream" | "core"

http_config     ::= "http:" NEWLINE INDENT
                    "path:" string NEWLINE
                    ( "method:" http_method NEWLINE )?
                    ( "timeout:" duration NEWLINE )?
                    DEDENT

http_method     ::= "GET" | "POST" | "PUT" | "PATCH" | "DELETE"

grpc_config     ::= "grpc:" NEWLINE INDENT
                    "service:" identifier NEWLINE
                    "method:" identifier NEWLINE
                    ( "streaming:" boolean NEWLINE )?
                    DEDENT

response_config ::= "response:" NEWLINE INDENT
                    "mode:" response_mode NEWLINE
                    ( "timeout:" duration NEWLINE )?
                    ( "callback:" string NEWLINE )?
                    DEDENT

response_mode   ::= "request_reply" | "async" | "fire_and_forget"
```

---

## Runtime Behavior

### Subject Resolution

Subject patterns are resolved at runtime:

```go
// Template: SMS.INTENT.{{domain}}.{{intent_name}}.{{realm}}
// Context: { domain: "banking", intent_name: "InitiateTransfer", realm: "acme" }
// Result: SMS.INTENT.banking.InitiateTransfer.acme
```

### Timeout Handling

| Timeout Type | Default | Behavior |
|--------------|---------|----------|
| Response timeout | 30s | Error returned to caller |
| Async callback | 5m | Callback with timeout error |

### Error Routing

Failed intents MAY be routed to dead-letter subjects:

```yaml
routing:
  transport: nats
  nats:
    subject_pattern: "SMS.INTENT.banking.InitiateTransfer"
    delivery: jetstream
    dead_letter: "SMS.DLQ.banking.InitiateTransfer"
```

---

## Conformance

An implementation is conformant with Intent Routing semantics if it:

### Functional Conformance

1. Parses and validates all routing declarations
2. Resolves subject patterns correctly at runtime
3. Routes intents via declared transport
4. Applies delivery mode guarantees (JetStream vs Core)
5. Respects response mode semantics
6. Enforces timeouts as declared
7. Delivers async callbacks correctly
8. Distributes load via queue groups
9. Routes failed intents to dead-letter when configured

### Invariant Conformance (I9)

10. Transport bindings are accepted ONLY on `inputIntent.routing` declarations
11. Transport bindings are REJECTED within `sms.flows` definitions
12. Flow steps, atomic groups, and work units do NOT contain transport configuration
13. The runtime bridges intent ingress (transport-bound) to flow execution (transport-agnostic)

---

## Specification Metadata

```yaml
specMetadata:
  feature: intent-routing
  version: v1.6
  status: normative
  invariant_extension:
    extends: "Transport is not encoded in the grammar"
    scope: "External API contract (InputIntent ingress only)"
    preserves: "SMS flow execution remains transport-agnostic"
    formal_invariant: "I9. Transport Scope Boundary"
  dependencies:
    - inputIntent
    - storageBinding
    - infrastructure
  changelog:
    v1.6:
      - "Initial intent routing semantics"
      - "NATS, HTTP, and gRPC transport support"
      - "Subject pattern expressions"
      - "Delivery modes (JetStream, Core)"
      - "Response modes (request_reply, async, fire_and_forget)"
      - "Queue groups for load balancing"
      - "Timeout configuration"
      - "Dead-letter routing"
      - "Formal invariant I9: Transport Scope Boundary"
      - "Explicit scoping of transport rule to external vs. internal concerns"
```
