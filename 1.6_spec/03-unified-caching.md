# System Mechanics Specification - Unified Caching Semantics v1.6

## Overview

Unified Caching provides declarative configuration for multi-tier caching strategy. This specification defines cache layer semantics, enabling specification of local, distributed, and hybrid caches with configurable invalidation strategies and observability.

**Origin**: This specification incorporates semantics from PS-01 (Unified Caching Proposal).

**Design Principle**: Cache layers declare caching STRATEGY (topology, eviction, invalidation). Distributed cache backing storage is specified via resource references to `resourceRequirements`, maintaining the intent-first principle.

---

## Cache Layer Declaration

### CacheLayer

Defines a unified cache layer configuration.

```yaml
cacheLayer:
  name: <string>
  version: <vN>
  description: <string>
  topology:
    tier: local | distributed | hybrid
    local:
      max_entries: <integer>
      ttl: <duration>
      eviction: lru | lfu | fifo
      max_size: <size_expression>
    distributed:
      resource: <resource_ref>     # References resourceRequirements.entityStores[].name
      ttl: <duration>              # Cache-layer TTL override (optional)
    hybrid:
      promotion: on_miss | on_access
  invalidation:
    strategy: explicit | ttl | watch | hybrid
    ttl: <duration>
    watch_keys: [<key_pattern>, ...]
    propagate: true | false
  warmup:
    enabled: true | false
    strategy: lazy | eager | scheduled
    on_startup: true | false
    queries: [<query_expression>, ...]
  observability:
    emit_signals: true | false
    metrics: [hit_rate, miss_rate, latency, size, evictions]
    interval: <duration>
```

---

## Cache Topologies

### Local Cache (L1)

In-process cache for lowest latency access.

```yaml
cacheLayer:
  name: ViewLocalCache
  version: v1
  topology:
    tier: local
    local:
      max_entries: 10000
      ttl: 5m
      eviction: lru
      max_size: 100MB
  invalidation:
    strategy: ttl
```

**Characteristics**:
- Sub-microsecond access latency
- Process-local, not shared across instances
- Lost on process restart
- Suitable for frequently accessed, immutable or slowly-changing data

**Eviction Policies**:

| Policy | Description | Best For |
|--------|-------------|----------|
| `lru` | Least Recently Used | General purpose, access-pattern sensitive |
| `lfu` | Least Frequently Used | Hot-spot workloads |
| `fifo` | First In First Out | Time-ordered data |

---

### Distributed Cache (L2)

Shared cache across all instances using a resource-backed store.

```yaml
cacheLayer:
  name: ViewDistributedCache
  version: v1
  topology:
    tier: distributed
    distributed:
      resource: ViewCache       # References resourceRequirements.entityStores[].name
      ttl: 30m                  # Cache-layer TTL (may override resource default)
  invalidation:
    strategy: watch
    watch_keys:
      - "view.{{entity_id}}"
    propagate: true
```

**Resource Reference**:

The `resource` field references an entityStore defined in `resourceRequirements`:

```yaml
resourceRequirements:
  entityStores:
    - name: ViewCache
      purpose: cache
      consistency: eventual
      durability: persistent
      expiration:
        enabled: true
        ttl: 1h
      replication:
        min_replicas: 3
```

**Runtime Backend Selection**:

The runtime provisioner selects an appropriate backend based on the resource's requirements:

| Resource Consistency | Resource Durability | Runtime May Select |
|---------------------|---------------------|-------------------|
| `strong_per_key` | `persistent` | NATS KV, Redis Cluster |
| `eventual` | `persistent` | NATS KV, Redis, Memcached |
| `eventual` | `ephemeral` | Memcached, in-memory distributed |

**Characteristics**:
- Millisecond access latency (network hop)
- Shared across all instances
- Survives process restarts (if resource is persistent)
- Suitable for computed/derived data with moderate freshness requirements

---

### Hybrid Cache (L1 + L2)

Two-tier caching with automatic promotion.

```yaml
cacheLayer:
  name: ViewHybridCache
  version: v1
  topology:
    tier: hybrid
    local:
      max_entries: 1000
      ttl: 1m
      eviction: lru
    distributed:
      resource: ViewCache
      ttl: 30m
    hybrid:
      promotion: on_miss
  invalidation:
    strategy: hybrid
    propagate: true
```

**Promotion Policies**:

| Policy | Description | Use Case |
|--------|-------------|----------|
| `on_miss` | Promote to L1 on L1 miss | Conservative memory usage |
| `on_access` | Promote on every access | Aggressive hot-spot optimization |

**Lookup Order**:
1. Check L1 (local)
2. On L1 miss, check L2 (distributed)
3. On L2 hit, optionally promote to L1
4. On L2 miss, compute from source

---

## Invalidation Strategies

### Explicit Invalidation

Cache entries invalidated via explicit API calls.

```yaml
invalidation:
  strategy: explicit
```

**Use Case**: Application controls cache lifecycle precisely
**Trade-off**: Requires careful invalidation logic

### TTL-Based Invalidation

Cache entries expire after configured time-to-live.

```yaml
invalidation:
  strategy: ttl
  ttl: 5m
```

**Use Case**: Data with predictable staleness tolerance
**Trade-off**: May serve stale data within TTL window

### Watch-Based Invalidation

Cache entries invalidated when watched keys change.

```yaml
invalidation:
  strategy: watch
  watch_keys:
    - "entity.{{entity_id}}"
    - "view.{{view_name}}.{{entity_id}}"
  propagate: true
```

**Use Case**: Real-time consistency with source data
**Trade-off**: Requires pub/sub infrastructure

### Hybrid Invalidation

Combines TTL with explicit/watch invalidation.

```yaml
invalidation:
  strategy: hybrid
  ttl: 30m  # Maximum staleness
  watch_keys:
    - "entity.{{entity_id}}"
  propagate: true
```

**Use Case**: Balance between consistency and performance
**Behavior**: TTL provides backstop; watch provides real-time updates

---

## Cache Warming

### Eager Warming

Pre-populate cache on startup or schedule.

```yaml
warmup:
  enabled: true
  strategy: eager
  on_startup: true
  queries:
    - "SELECT * FROM active_accounts"
    - "SELECT * FROM recent_transactions LIMIT 1000"
```

### Lazy Warming

Populate cache on first access.

```yaml
warmup:
  enabled: true
  strategy: lazy
```

### Scheduled Warming

Periodic cache refresh.

```yaml
warmup:
  enabled: true
  strategy: scheduled
  queries:
    - "REFRESH_VIEW AccountSummaryView"
```

---

## Observability

### Cache Signals

Cache layers emit signals for monitoring and auto-scaling.

```yaml
observability:
  emit_signals: true
  metrics:
    - hit_rate
    - miss_rate
    - latency
    - size
    - evictions
  interval: 10s
```

**Signal Schema**:

```yaml
signal:
  type: cache
  source:
    component: cache
    name: ViewHybridCache
    tier: local | distributed | hybrid
  metrics:
    hit_rate: 0.85
    miss_rate: 0.15
    latency_p50: 0.1ms
    latency_p99: 5ms
    size: 1000
    evictions: 50
    pressure: low | medium | high | critical
  timestamp: <iso8601>
```

**Pressure Levels**:

| Level | Condition | Action |
|-------|-----------|--------|
| `low` | < 50% capacity | Normal operation |
| `medium` | 50-75% capacity | Consider scaling |
| `high` | 75-90% capacity | Scale recommended |
| `critical` | > 90% capacity | Immediate attention |

---

## Binding to Views

Cache layers are associated with Materializations:

```yaml
materialization:
  name: AccountSummaryView
  source: AccountState
  targetState: AccountSummaryState
  cache:
    layer: ViewHybridCache
    key: "account.{{account_id}}"
```

**Rules**:
- Materialized views SHOULD have a cache layer for performance
- Singleton views MAY use local-only caching
- High-cardinality views SHOULD use distributed or hybrid caching

---

## Resource Compatibility

### Required Resource Properties

When a cache layer references an entityStore resource, the resource SHOULD have:

| Cache Requirement | Resource Property |
|------------------|-------------------|
| Distributed cache | `purpose: cache` or `purpose: derived_state` |
| Watch invalidation | Resource supports pub/sub (runtime capability) |
| TTL override | `expiration.enabled: true` in resource |

### Example Resource Definition

```yaml
resourceRequirements:
  name: CacheResources
  version: v1
  
  entityStores:
    # View cache store
    - name: ViewCache
      purpose: cache
      access_pattern: key_value
      consistency: eventual
      durability: persistent
      history:
        enabled: false
      expiration:
        enabled: true
        ttl: 1h
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 10MB
    
    # Session cache store
    - name: SessionCache
      purpose: session
      access_pattern: key_value
      consistency: strong_per_key
      durability: ephemeral
      history:
        enabled: false
      expiration:
        enabled: true
        ttl: 24h
      replication:
        min_replicas: 3
      capacity:
        max_entry_size: 100KB
```

---

## Key Pattern Grammar

```ebnf
cache_key       ::= key_segment ( "." key_segment )*

key_segment     ::= static_segment
                  | dynamic_segment

static_segment  ::= identifier

dynamic_segment ::= "{{" field_ref "}}"

field_ref       ::= identifier ( "." identifier )*
```

**Examples**:
```yaml
# Simple entity key
key: "account.{{id}}"

# Composite key with view name
key: "view.{{view_name}}.{{entity_id}}"

# Nested field reference
key: "customer.{{id}}.{{status.code}}"
```

---

## Validation Rules

### Structural Violations (MUST reject)

1. Cache layer without topology tier
2. Local cache without eviction policy
3. Distributed cache without resource reference
4. Hybrid cache without both local and distributed config
5. Watch invalidation without watch_keys
6. Eviction policy not in valid set

### Semantic Violations (MUST reject)

1. Local TTL greater than distributed TTL in hybrid
2. Watch keys referencing non-existent entities
3. Warmup queries on lazy strategy
4. Propagate enabled on local-only cache

### Reference Violations (MUST reject)

1. Distributed resource referencing non-existent entityStore
2. Resource with `purpose: authoritative_state` used for cache (warning)
3. Resource without `expiration.enabled` when cache layer specifies TTL

---

## EBNF Grammar

```ebnf
cache_layer     ::= "cacheLayer:" NEWLINE INDENT
                    "name:" identifier NEWLINE
                    ( "version:" version NEWLINE )?
                    ( "description:" string NEWLINE )?
                    topology_config
                    invalidation_config?
                    warmup_config?
                    observability_config?
                    DEDENT

topology_config ::= "topology:" NEWLINE INDENT
                    "tier:" cache_tier NEWLINE
                    local_config?
                    distributed_config?
                    hybrid_config?
                    DEDENT

cache_tier      ::= "local" | "distributed" | "hybrid"

local_config    ::= "local:" NEWLINE INDENT
                    ( "max_entries:" integer NEWLINE )?
                    ( "ttl:" duration NEWLINE )?
                    ( "eviction:" eviction_policy NEWLINE )?
                    ( "max_size:" size_expression NEWLINE )?
                    DEDENT

eviction_policy ::= "lru" | "lfu" | "fifo"

distributed_config ::= "distributed:" NEWLINE INDENT
                       "resource:" identifier NEWLINE
                       ( "ttl:" duration NEWLINE )?
                       DEDENT

hybrid_config   ::= "hybrid:" NEWLINE INDENT
                    ( "promotion:" promotion_policy NEWLINE )?
                    DEDENT

promotion_policy ::= "on_miss" | "on_access"

invalidation_config ::= "invalidation:" NEWLINE INDENT
                        "strategy:" invalidation_strategy NEWLINE
                        ( "ttl:" duration NEWLINE )?
                        ( "watch_keys:" key_list NEWLINE )?
                        ( "propagate:" boolean NEWLINE )?
                        DEDENT

invalidation_strategy ::= "explicit" | "ttl" | "watch" | "hybrid"

key_list        ::= "[" cache_key ( "," cache_key )* "]"

warmup_config   ::= "warmup:" NEWLINE INDENT
                    ( "enabled:" boolean NEWLINE )?
                    ( "strategy:" warmup_strategy NEWLINE )?
                    ( "on_startup:" boolean NEWLINE )?
                    ( "queries:" query_list NEWLINE )?
                    DEDENT

warmup_strategy ::= "lazy" | "eager" | "scheduled"

observability_config ::= "observability:" NEWLINE INDENT
                         ( "emit_signals:" boolean NEWLINE )?
                         ( "metrics:" metric_list NEWLINE )?
                         ( "interval:" duration NEWLINE )?
                         DEDENT

metric_list     ::= "[" cache_metric ( "," cache_metric )* "]"

cache_metric    ::= "hit_rate" | "miss_rate" | "latency" | "size" | "evictions"
```

---

## Performance Targets

### L1 (Local) Cache

| Metric | Target |
|--------|--------|
| Lookup latency | < 10μs |
| Insert latency | < 10μs |
| Eviction latency | < 100μs |

### L2 (Distributed) Cache

| Metric | Target |
|--------|--------|
| Lookup latency | < 5ms |
| Insert latency | < 10ms |
| Invalidation propagation | < 100ms |

### Hybrid Cache

| Metric | Target |
|--------|--------|
| L1 hit latency | < 10μs |
| L1 miss + L2 hit | < 5ms |
| Promotion latency | < 1ms |

---

## Conformance

An implementation is conformant with Unified Caching semantics if it:

1. Parses and validates all cache layer declarations
2. Resolves resource references to resourceRequirements
3. Creates cache infrastructure using appropriate backend for resource
4. Respects topology tier configuration
5. Enforces eviction policies correctly
6. Applies invalidation strategies as declared
7. Propagates invalidation across instances when configured
8. Emits cache signals at configured intervals
9. Warms cache according to warmup configuration
10. Meets performance targets for cache operations
11. Promotes entries in hybrid cache per policy
12. Validates resource compatibility (purpose, expiration)

---

## Specification Metadata

```yaml
specMetadata:
  feature: unified-caching
  version: v1.6
  status: normative
  origin: PS-01
  invariant_compliance:
    principle: "Intent-First, Runtime-Decided"
    application: "Cache strategy is declarative; backing store resolved via resource reference"
  dependencies:
    - materialization
    - signal
    - resourceRequirements
  changelog:
    v1.6:
      - "Initial unified caching semantics (from PS-01)"
      - "Local, distributed, and hybrid cache topologies"
      - "Eviction policies (LRU, LFU, FIFO)"
      - "Invalidation strategies (explicit, TTL, watch, hybrid)"
      - "Cache warming (lazy, eager, scheduled)"
      - "Cache signals and observability"
      - "Performance targets"
      - "Resource reference for distributed cache (intent-first)"
      - "Removed technology-specific backend field"
```
