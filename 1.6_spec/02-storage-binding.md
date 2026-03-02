# System Mechanics Specification - Storage Binding Semantics v1.6

## Overview

Storage Binding provides declarative configuration for how data is persisted and retrieved from underlying storage systems. This extends SMS with explicit storage layer semantics, enabling specification of KV stores, streams, and object storage with configurable consistency and replication.

---

## Storage Binding Declaration

### StorageBinding

Defines storage configuration for data persistence.

```yaml
storageBinding:
  name: <string>
  version: <vN>
  type: kv | stream | object_store
  kv:
    bucket: <string>
    key_pattern: <pattern_expression>
    serialization: json | protobuf | msgpack
    history: <integer>
    ttl: <duration>
    replicas: <integer>
  stream:
    name: <string>
    subjects: [<subject_pattern>, ...]
    retention: limits | interest | workqueue
    max_msgs: <integer>
    max_bytes: <integer>
    max_age: <duration>
    storage: file | memory
    replicas: <integer>
  object_store:
    bucket: <string>
    description: <string>
    max_object_size: <size_expression>
    replicas: <integer>
  access:
    read_policy: local | replicated | eventual
    write_consistency: strong | eventual
```

---

## Storage Types

### Key-Value Storage

For entity-scoped, lookup-oriented data.

```yaml
storageBinding:
  name: AccountStateStorage
  version: v1
  type: kv
  kv:
    bucket: "SMS_ACCOUNT_STATE"
    key_pattern: "account.{{account_id}}"
    serialization: json
    history: 5
    ttl: 0s  # No expiration
    replicas: 3
  access:
    read_policy: local
    write_consistency: strong
```

**Key Pattern Expressions**:
- `{{field_name}}` - Substitute field value from entity
- `{{realm}}.{{entity_id}}` - Composite keys
- Static prefixes: `"account.{{id}}"` → `"account.123"`

**History**:
- Number of previous versions to retain
- Enables rollback and audit capabilities
- `0` = current value only

---

### Stream Storage

For event-sourced and temporal data.

```yaml
storageBinding:
  name: TransactionEventStream
  version: v1
  type: stream
  stream:
    name: "SMS_TRANSACTIONS"
    subjects:
      - "SMS.TXN.{{realm}}.>"
      - "SMS.TXN.*.created"
    retention: limits
    max_msgs: 10000000
    max_bytes: 10GB
    max_age: 365d
    storage: file
    replicas: 3
  access:
    read_policy: eventual
    write_consistency: strong
```

**Retention Policies**:
- `limits`: Retain until limits exceeded (max_msgs, max_bytes, max_age)
- `interest`: Retain only if consumers exist
- `workqueue`: Each message delivered once across consumers

**Subject Patterns**:
- `>` - Wildcard for multiple tokens
- `*` - Wildcard for single token

---

### Object Store Storage

For large binary objects and assets.

```yaml
storageBinding:
  name: DocumentStorage
  version: v1
  type: object_store
  object_store:
    bucket: "SMS_DOCUMENTS"
    description: "Customer documents and attachments"
    max_object_size: 100MB
    replicas: 3
  access:
    read_policy: replicated
    write_consistency: eventual
```

---

## Access Policies

### Read Policies

| Policy | Description | Use Case |
|--------|-------------|----------|
| `local` | Read from local replica only | Low latency, eventual consistency ok |
| `replicated` | Read from any available replica | High availability, eventual consistency |
| `eventual` | Read from any source, may be stale | Maximum availability |

### Write Consistency

| Level | Description | Trade-off |
|-------|-------------|-----------|
| `strong` | Synchronous replication to quorum | Higher latency, guaranteed durability |
| `eventual` | Asynchronous replication | Lower latency, possible data loss on failure |

---

## Binding to DataState

Storage bindings are associated with DataStates:

```yaml
dataState:
  name: AccountState
  type: Account
  lifecycle: persistent
  storage:
    binding: AccountStateStorage  # Reference to storageBinding
```

**Rules**:
- Persistent DataStates MUST have a storage binding
- Materialized DataStates MAY have a storage binding (for caching)
- Intermediate DataStates SHOULD NOT have storage bindings

---

## Key Pattern Grammar

```ebnf
key_pattern     ::= key_segment ( "." key_segment )*

key_segment     ::= static_segment
                  | dynamic_segment

static_segment  ::= identifier

dynamic_segment ::= "{{" field_ref "}}"

field_ref       ::= identifier ( "." identifier )*
```

**Examples**:
```yaml
# Simple entity key
key_pattern: "account.{{id}}"

# Composite key with realm
key_pattern: "{{realm}}.customer.{{customer_id}}"

# Nested field reference
key_pattern: "order.{{id}}.{{status.code}}"
```

---

## Validation Rules

### Structural Violations (MUST reject)

1. Storage binding without type
2. KV binding without bucket
3. Stream binding without name
4. Object store binding without bucket
5. Invalid key pattern expression
6. Unsupported serialization format
7. Replicas less than 1

### Semantic Violations (MUST reject)

1. Strong consistency with local-only reads
2. TTL on stream with interest retention
3. History on stream storage
4. Key pattern referencing non-existent fields

---

## EBNF Grammar

```ebnf
storage_binding ::= "storageBinding:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    "version:" version NEWLINE
                    "type:" storage_type NEWLINE
                    ( kv_config | stream_config | object_store_config )
                    access_config?
                    DEDENT

storage_type    ::= "kv" | "stream" | "object_store"

kv_config       ::= "kv:" NEWLINE INDENT
                    "bucket:" string NEWLINE
                    ( "key_pattern:" string NEWLINE )?
                    ( "serialization:" serialization_type NEWLINE )?
                    ( "history:" integer NEWLINE )?
                    ( "ttl:" duration NEWLINE )?
                    ( "replicas:" integer NEWLINE )?
                    DEDENT

serialization_type ::= "json" | "protobuf" | "msgpack"

stream_config   ::= "stream:" NEWLINE INDENT
                    "name:" string NEWLINE
                    "subjects:" subject_list NEWLINE
                    ( "retention:" retention_policy NEWLINE )?
                    ( "max_msgs:" integer NEWLINE )?
                    ( "max_bytes:" size_expression NEWLINE )?
                    ( "max_age:" duration NEWLINE )?
                    ( "storage:" storage_medium NEWLINE )?
                    ( "replicas:" integer NEWLINE )?
                    DEDENT

retention_policy ::= "limits" | "interest" | "workqueue"

storage_medium  ::= "file" | "memory"

object_store_config ::= "object_store:" NEWLINE INDENT
                        "bucket:" string NEWLINE
                        ( "description:" string NEWLINE )?
                        ( "max_object_size:" size_expression NEWLINE )?
                        ( "replicas:" integer NEWLINE )?
                        DEDENT

access_config   ::= "access:" NEWLINE INDENT
                    ( "read_policy:" read_policy NEWLINE )?
                    ( "write_consistency:" write_consistency NEWLINE )?
                    DEDENT

read_policy     ::= "local" | "replicated" | "eventual"

write_consistency ::= "strong" | "eventual"

subject_list    ::= "[" subject_pattern ( "," subject_pattern )* "]"

subject_pattern ::= string
```

---

## Conformance

An implementation is conformant with Storage Binding semantics if it:

1. Parses and validates all storage binding declarations
2. Creates storage resources matching specifications
3. Enforces key patterns on writes
4. Respects read policies for lookups
5. Honors write consistency guarantees
6. Maintains replication factor
7. Applies retention policies correctly
8. Enforces TTL expiration
9. Preserves history depth for KV stores

---

## Specification Metadata

```yaml
specMetadata:
  feature: storage-binding
  version: v1.6
  status: normative
  dependencies:
    - dataState
    - realm
  changelog:
    v1.6:
      - "Initial storage binding semantics"
      - "KV, stream, and object store support"
      - "Key pattern expressions"
      - "Access policies (read/write)"
```
