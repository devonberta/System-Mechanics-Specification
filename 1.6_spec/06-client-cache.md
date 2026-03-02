# System Mechanics Specification - Client Cache Contract v1.6

## Overview

Client Cache Contract provides declarative configuration for client-side (browser/mobile) caching within Experiences. This specification defines how frontend applications cache view data, manage offline scenarios, and synchronize with server-side cache layers.

**Origin**: This specification incorporates semantics from PS-01 (Unified Caching Proposal) client-side extension.

---

## Client Cache Declaration

### ClientCache

Extends `experience` with client-side caching configuration.

```yaml
experience:
  name: <string>
  version: <vN>
  clientCache:
    enabled: true | false
    storage:
      type: memory | persistent | hybrid
      max_size: <size_expression>
      encryption: true | false
    strategies:
      primary:
        strategy: cache_first | network_first | stale_while_revalidate
        ttl: <duration>
      related:
        strategy: cache_first | network_first | stale_while_revalidate
        ttl: <duration>
        lazy_cache: true | false
    prefetch:
      enabled: true | false
      views: [<view_ref>, ...]
      trigger: on_navigate | on_idle | on_connect
    offline:
      enabled: true | false
      required_views: [<view_ref>, ...]
      sync_on_reconnect: true | false
      conflict_resolution: last_write_wins | server_wins | client_wins | manual
      intent_sync:
        on_cas_failure: reject_and_notify | queue_for_retry
        retry_with_refresh: true | false
      governance:
        validate_on_sync: true | false
        on_violation: reject | queue_for_review | notify_and_proceed
    version_policy:
      on_upgrade: invalidate_all | invalidate_incompatible | keep_compatible
      grace_period: <duration>
      migration: eager | lazy
```

---

## Storage Types

### Memory Storage

In-memory cache for session-scoped data.

```yaml
clientCache:
  enabled: true
  storage:
    type: memory
    max_size: 50MB
```

**Characteristics**:
- Fastest access (sub-millisecond)
- Lost on page refresh or app restart
- No encryption needed (volatile)
- Suitable for session-specific, frequently accessed data

---

### Persistent Storage

Browser IndexedDB or mobile SQLite storage.

```yaml
clientCache:
  enabled: true
  storage:
    type: persistent
    max_size: 100MB
    encryption: true
```

**Characteristics**:
- Survives page refresh and app restart
- Requires encryption for sensitive data
- Subject to browser/OS storage quotas
- Suitable for offline-capable applications

**Encryption**:
- When `encryption: true`, all cached data is encrypted at rest
- Encryption key derived from user session
- Key rotation on session refresh

---

### Hybrid Storage

Two-tier client cache with automatic promotion.

```yaml
clientCache:
  enabled: true
  storage:
    type: hybrid
    max_size: 100MB
    encryption: true
```

**Behavior**:
1. Check memory cache first (L1)
2. On miss, check persistent cache (L2)
3. Promote to memory on access
4. Persist frequently accessed views

---

## Caching Strategies

### Cache First

Return cached data immediately; refresh in background.

```yaml
strategies:
  primary:
    strategy: cache_first
    ttl: 5m
```

**Semantics**:
1. If cached data exists and not expired, return immediately
2. Fetch fresh data in background
3. Update cache and notify UI of changes
4. Best for: Read-heavy views with tolerance for slight staleness

**User Experience**: Instant response, eventual consistency

---

### Network First

Always fetch from network; fall back to cache on failure.

```yaml
strategies:
  primary:
    strategy: network_first
    ttl: 5m
```

**Semantics**:
1. Attempt network fetch first
2. On success, update cache and return
3. On failure (timeout/error), return cached data if available
4. Best for: Views requiring fresh data, with offline fallback

**User Experience**: Slight delay, guaranteed freshness when online

---

### Stale While Revalidate

Return stale data immediately while revalidating.

```yaml
strategies:
  primary:
    strategy: stale_while_revalidate
    ttl: 1m
```

**Semantics**:
1. Return cached data immediately (even if stale)
2. Fetch fresh data in background
3. Update cache for next access
4. Optionally update UI with fresh data when available
5. Best for: Performance-critical views with acceptable staleness

**User Experience**: Instant response, background refresh

---

## Strategy Configuration

### Primary Strategy

Applied to the main view data being requested.

```yaml
strategies:
  primary:
    strategy: cache_first
    ttl: 5m
```

### Related Strategy

Applied to related/referenced data (e.g., lookups, linked entities).

```yaml
strategies:
  related:
    strategy: stale_while_revalidate
    ttl: 10m
    lazy_cache: true
```

**Lazy Cache**:
- When `lazy_cache: true`, related data is cached only when accessed
- When `lazy_cache: false`, related data is prefetched with primary

---

## Prefetching

### Prefetch Configuration

Proactively cache views before user navigation.

```yaml
prefetch:
  enabled: true
  views:
    - AccountSummaryView
    - TransactionListView
    - AlertsView
  trigger: on_idle
```

**Trigger Types**:

| Trigger | Description | Use Case |
|---------|-------------|----------|
| `on_navigate` | Prefetch when navigating to related views | Predictive loading |
| `on_idle` | Prefetch during browser idle time | Background optimization |
| `on_connect` | Prefetch on app startup/reconnection | Offline preparation |

---

## Offline Support

### Offline Configuration

Enable offline-first operation.

```yaml
offline:
  enabled: true
  required_views:
    - AccountSummaryView
    - RecentTransactionsView
  sync_on_reconnect: true
  conflict_resolution: server_wins
```

**Required Views**:
- Views that MUST be available offline
- Cached aggressively with longer TTL
- Refreshed on every connection

**Sync on Reconnect**:
- When `true`, automatically sync all pending changes on reconnection
- When `false`, require explicit user action to sync

### Conflict Resolution

Strategies for resolving client-server conflicts after offline period.

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `last_write_wins` | Most recent timestamp wins | Simple, eventual consistency |
| `server_wins` | Server value always wins | Server-authoritative data |
| `client_wins` | Client value always wins | Client-authoritative drafts |
| `manual` | User resolves conflicts | Collaborative editing |

### Authority Invariant Compliance

**Critical Clarification**: Conflict resolution strategies apply **exclusively to cached view data** (materialized/derived state). They do NOT apply to authoritative entity writes.

**Offline Intent Handling**:
- Offline-queued `InputIntent` submissions MUST respect **I4. CAS Safety** on synchronization
- Intent sync failures due to CAS rejection SHALL NOT be resolved via `conflict_resolution`
- CAS-rejected intents MUST be surfaced to the user for re-submission with current context

```yaml
offline:
  conflict_resolution: server_wins  # Applies to VIEW cache only
  intent_sync:
    on_cas_failure: reject_and_notify  # Respects I4
    retry_with_refresh: true           # Re-fetch context, re-prompt user
```

**Invariant Preservation**:

| Invariant | Client Cache Behavior |
|-----------|----------------------|
| I1. Authority Singularity | Cached views are read-only; writes go through server authority |
| I3. No Overlapping Authority | Client cache never accepts writes; only caches server-authoritative views |
| I4. CAS Safety | Offline intents validated via CAS on sync; conflicts rejected, not auto-resolved |

### Governance Validation on Sync

Offline-queued operations MUST respect governance constraints that may have changed during the offline period.

```yaml
offline:
  governance:
    validate_on_sync: true | false
    on_violation: reject | queue_for_review | notify_and_proceed
```

**Behavior**:
- When `validate_on_sync: true`, all queued intents are validated against **current** governance policies
- Intents that violate updated governance are handled per `on_violation` strategy

**I7 Compliance**:

| on_violation | Behavior | I7 Conformance |
|--------------|----------|----------------|
| `reject` | Reject intent, notify user | Fully conformant |
| `queue_for_review` | Hold for manual approval | Conformant (deferred) |
| `notify_and_proceed` | Log violation, proceed | NON-CONFORMANT (use with caution) |

**Note**: `notify_and_proceed` violates I7 and SHOULD only be used for non-critical operations where governance is advisory.

---

## Complete Example

### Banking Experience with Client Cache

```yaml
experience:
  name: CustomerBankingExperience
  version: v1
  description: "Mobile-first banking with offline support"
  
  navigation:
    root: AccountOverview
    tree:
      - name: Accounts
        view: AccountListView
      - name: Transfers
        view: TransferFormView
      - name: History
        view: TransactionHistoryView
  
  clientCache:
    enabled: true
    storage:
      type: hybrid
      max_size: 100MB
      encryption: true
    strategies:
      primary:
        strategy: stale_while_revalidate
        ttl: 5m
      related:
        strategy: cache_first
        ttl: 15m
        lazy_cache: false
    prefetch:
      enabled: true
      views:
        - AccountSummaryView
        - RecentTransactionsView
      trigger: on_connect
    offline:
      enabled: true
      required_views:
        - AccountSummaryView
        - RecentTransactionsView
        - PendingTransfersView
      sync_on_reconnect: true
      conflict_resolution: server_wins
```

---

## Integration with Server Cache

### Cache Hierarchy

```
┌─────────────────────────────────────────────┐
│                 Client                       │
│  ┌─────────┐      ┌─────────────────────┐  │
│  │ Memory  │ ←──→ │ Persistent (IndexDB)│  │
│  │ (L0)    │      │ (L1)                │  │
│  └─────────┘      └─────────────────────┘  │
└─────────────────────┬───────────────────────┘
                      │ Network
┌─────────────────────┴───────────────────────┐
│                 Server                       │
│  ┌─────────┐      ┌─────────────────────┐  │
│  │ Local   │ ←──→ │ Distributed (NATS)  │  │
│  │ (L1)    │      │ (L2)                │  │
│  └─────────┘      └─────────────────────┘  │
└─────────────────────────────────────────────┘
```

### Cache Invalidation Flow

1. Server updates data
2. Server invalidates distributed cache
3. Server publishes invalidation signal
4. Client receives invalidation (WebSocket/SSE)
5. Client invalidates local cache
6. Client fetches fresh data on next access

---

## Version Evolution

### Cache Invalidation on Experience Upgrade

When an Experience version changes, cached data may become incompatible.

```yaml
clientCache:
  version_policy:
    on_upgrade: invalidate_all | invalidate_incompatible | keep_compatible
    grace_period: <duration>
    migration: eager | lazy
```

**Policies**:

| Policy | Description | Use Case |
|--------|-------------|----------|
| `invalidate_all` | Clear entire cache on version change | Breaking changes, security updates |
| `invalidate_incompatible` | Clear only views with schema changes | Incremental evolution |
| `keep_compatible` | Retain backward-compatible cached data | Performance optimization |

**Grace Period**: Duration to allow old-version cache reads during migration

**Migration Strategies**:
- `eager`: Refresh all cached views immediately on upgrade
- `lazy`: Refresh views on next access

### View Version Binding

Cached views are bound to specific view versions:

```yaml
# Cache key includes version
cache_key: "{{experience}}.{{view_name}}.v{{view_version}}.{{entity_id}}"
```

**Zero Downtime Compliance**:
- Runtime MAY serve old-version cached views during upgrade window
- New-version views are populated progressively
- Explicit `invalidate_all` forces immediate refresh (use sparingly)

---

## Validation Rules

### Structural Violations (MUST reject)

1. ClientCache without storage type
2. Offline enabled without required_views
3. Prefetch enabled without views list
4. Strategy not in valid set
5. Conflict resolution not in valid set
6. max_size not a valid size expression

### Semantic Violations (MUST reject)

1. Memory storage with encryption (contradictory)
2. Offline enabled with memory-only storage
3. Sync on reconnect without offline enabled
4. Required views referencing non-existent views
5. `intent_sync.on_cas_failure` set to auto-resolve (violates I4)
6. `governance.on_violation: notify_and_proceed` for intents with `enforcement: strict` (violates I7)
7. `version_policy.on_upgrade: keep_compatible` with breaking schema changes between versions

---

## EBNF Grammar

```ebnf
client_cache    ::= "clientCache:" NEWLINE INDENT
                    "enabled:" boolean NEWLINE
                    storage_config
                    strategies_config?
                    prefetch_config?
                    offline_config?
                    version_policy_config?
                    DEDENT

storage_config  ::= "storage:" NEWLINE INDENT
                    "type:" storage_type NEWLINE
                    ( "max_size:" size_expression NEWLINE )?
                    ( "encryption:" boolean NEWLINE )?
                    DEDENT

storage_type    ::= "memory" | "persistent" | "hybrid"

strategies_config ::= "strategies:" NEWLINE INDENT
                      ( primary_strategy )?
                      ( related_strategy )?
                      DEDENT

primary_strategy ::= "primary:" NEWLINE INDENT
                     "strategy:" cache_strategy NEWLINE
                     ( "ttl:" duration NEWLINE )?
                     DEDENT

related_strategy ::= "related:" NEWLINE INDENT
                     "strategy:" cache_strategy NEWLINE
                     ( "ttl:" duration NEWLINE )?
                     ( "lazy_cache:" boolean NEWLINE )?
                     DEDENT

cache_strategy  ::= "cache_first" | "network_first" | "stale_while_revalidate"

prefetch_config ::= "prefetch:" NEWLINE INDENT
                    ( "enabled:" boolean NEWLINE )?
                    ( "views:" view_list NEWLINE )?
                    ( "trigger:" prefetch_trigger NEWLINE )?
                    DEDENT

prefetch_trigger ::= "on_navigate" | "on_idle" | "on_connect"

view_list       ::= "[" view_ref ( "," view_ref )* "]"

offline_config  ::= "offline:" NEWLINE INDENT
                    ( "enabled:" boolean NEWLINE )?
                    ( "required_views:" view_list NEWLINE )?
                    ( "sync_on_reconnect:" boolean NEWLINE )?
                    ( "conflict_resolution:" conflict_strategy NEWLINE )?
                    ( intent_sync_config )?
                    ( governance_config )?
                    DEDENT

conflict_strategy ::= "last_write_wins" | "server_wins" | "client_wins" | "manual"

intent_sync_config ::= "intent_sync:" NEWLINE INDENT
                       ( "on_cas_failure:" cas_failure_action NEWLINE )?
                       ( "retry_with_refresh:" boolean NEWLINE )?
                       DEDENT

cas_failure_action ::= "reject_and_notify" | "queue_for_retry"

governance_config ::= "governance:" NEWLINE INDENT
                      ( "validate_on_sync:" boolean NEWLINE )?
                      ( "on_violation:" violation_action NEWLINE )?
                      DEDENT

violation_action ::= "reject" | "queue_for_review" | "notify_and_proceed"

version_policy_config ::= "version_policy:" NEWLINE INDENT
                          ( "on_upgrade:" upgrade_action NEWLINE )?
                          ( "grace_period:" duration NEWLINE )?
                          ( "migration:" migration_strategy NEWLINE )?
                          DEDENT

upgrade_action  ::= "invalidate_all" | "invalidate_incompatible" | "keep_compatible"

migration_strategy ::= "eager" | "lazy"
```

---

## Performance Targets

### Client Cache Operations

| Metric | Target |
|--------|--------|
| Memory cache lookup | < 1ms |
| Persistent cache lookup | < 10ms |
| Cache write (memory) | < 1ms |
| Cache write (persistent) | < 50ms |
| Encryption overhead | < 5ms per operation |

### Sync Operations

| Metric | Target |
|--------|--------|
| Initial sync (cold) | < 5s for required views |
| Reconnection sync | < 2s for delta changes |
| Conflict detection | < 100ms |

---

## Conformance

An implementation is conformant with Client Cache Contract semantics if it:

1. Parses and validates all clientCache declarations
2. Creates client storage matching type specification
3. Enforces max_size limits with appropriate eviction
4. Applies encryption when specified
5. Implements caching strategies correctly
6. Prefetches views according to trigger configuration
7. Maintains required views for offline access
8. Syncs on reconnect when configured
9. Applies conflict resolution strategy to view cache only
10. Meets performance targets for cache operations
11. Respects I4 (CAS Safety) for offline intent synchronization
12. Validates governance constraints on sync when configured
13. Invalidates cache appropriately on experience version changes
14. Never auto-resolves CAS conflicts for authoritative writes

---

## Specification Metadata

```yaml
specMetadata:
  feature: client-cache
  version: v1.6
  status: normative
  origin: PS-01
  dependencies:
    - experience
    - presentationView
    - cacheLayer
  changelog:
    v1.6:
      - "Initial client cache contract (from PS-01)"
      - "Memory, persistent, and hybrid storage types"
      - "Caching strategies (cache_first, network_first, stale_while_revalidate)"
      - "Primary and related strategy separation"
      - "Prefetch configuration with triggers"
      - "Offline support with required views"
      - "Sync on reconnect"
      - "Conflict resolution strategies (view cache only)"
      - "Authority invariant compliance (I1, I3, I4)"
      - "Governance validation on sync (I7)"
      - "Version evolution and cache invalidation policies"
      - "Intent sync with CAS safety"
      - "Performance targets"
```
