# System Mechanics Specification - Grammar Reference v1.6

## Overview

This document provides the grammar reference for SMS v1.6, which extends the v1.5 grammar with new declarations and extensions. For the complete base grammar, refer to `../1.5_spec/01-grammar-reference.md`.

**New in v1.6**:
- Storage Binding declarations
- Cache Layer declarations (from PS-01)
- Intent Routing extensions
- Infrastructure declarations
- Client Cache Contract (Experience extension)
- Cache Signal type extension

---

## Grammar Extensions Summary

| Feature | Type | Location |
|---------|------|----------|
| StorageBinding | New top-level declaration | `02-storage-binding.md` |
| CacheLayer | New top-level declaration | `03-unified-caching.md` |
| Intent Routing | Extension to InputIntent | `04-intent-routing.md` |
| ResourceRequirements | New top-level declaration | `05-infrastructure.md` |
| ClientCache | Extension to Experience | `06-client-cache.md` |
| Cache Signal | Extension to Signal | This document |

---

## Signal Type Extension

### Cache Signal (NEW in v1.6)

Extends the base signal types with cache-specific metrics and monitoring.

#### Base Signal Types (v1.5)

```yaml
signal:
  type: capacity | health | latency | saturation | policy | backpressure
  source:
    component: worker | sms | ui | infra
    name: <string>
    region: <string>
  subject:
    intent: <optional>
    dataState: <optional>
  metrics:
    <key>: <number | string>
  severity: info | warn | critical
  timestamp: <iso8601>
```

#### Extended Signal Types (v1.6)

```yaml
signal:
  type: capacity | health | latency | saturation | policy | backpressure | cache
  source:
    component: worker | sms | ui | infra | cache
    name: <string>
    region: <string>
    tier: local | distributed | hybrid  # NEW: For cache signals
  subject:
    intent: <optional>
    dataState: <optional>
    cache: <optional>  # NEW: Cache layer reference
  metrics:
    <key>: <number | string>
  severity: info | warn | critical
  timestamp: <iso8601>
```

---

### Cache Signal Definition

Cache signals provide operational observability for cache layers.

```yaml
signal:
  type: cache
  source:
    component: cache
    name: <cache_layer_name>
    tier: local | distributed | hybrid
    region: <string>
  subject:
    cache: <cache_layer_name>
  metrics:
    # Hit/Miss Rates
    hit_rate: <float>         # 0.0 - 1.0
    miss_rate: <float>        # 0.0 - 1.0
    hit_count: <integer>      # Total hits in interval
    miss_count: <integer>     # Total misses in interval
    
    # Latency Metrics
    latency_p50: <duration>   # 50th percentile latency
    latency_p95: <duration>   # 95th percentile latency
    latency_p99: <duration>   # 99th percentile latency
    latency_avg: <duration>   # Average latency
    
    # Size Metrics
    size: <integer>           # Current entry count
    size_bytes: <integer>     # Current size in bytes
    max_entries: <integer>    # Configured max entries
    max_bytes: <integer>      # Configured max size
    
    # Eviction Metrics
    evictions: <integer>      # Evictions in interval
    eviction_rate: <float>    # Evictions per second
    
    # Pressure Indicator
    pressure: low | medium | high | critical
  severity: info | warn | critical
  timestamp: <iso8601>
```

---

### Pressure Levels

Pressure indicates cache resource utilization and health.

| Level | Capacity Threshold | Hit Rate Threshold | Recommended Action |
|-------|-------------------|-------------------|-------------------|
| `low` | < 50% | > 80% | Normal operation |
| `medium` | 50-75% | 60-80% | Monitor closely |
| `high` | 75-90% | 40-60% | Consider scaling or tuning |
| `critical` | > 90% | < 40% | Immediate attention required |

**Pressure Calculation**:
```
pressure = MAX(capacity_pressure, performance_pressure)

capacity_pressure:
  - low: size < 50% of max
  - medium: size 50-75% of max
  - high: size 75-90% of max
  - critical: size > 90% of max

performance_pressure:
  - low: hit_rate > 80%
  - medium: hit_rate 60-80%
  - high: hit_rate 40-60%
  - critical: hit_rate < 40%
```

---

### Signal Examples

#### Healthy Cache Signal

```yaml
signal:
  type: cache
  source:
    component: cache
    name: ViewHybridCache
    tier: hybrid
    region: us-east-1
  subject:
    cache: ViewHybridCache
  metrics:
    hit_rate: 0.92
    miss_rate: 0.08
    hit_count: 45000
    miss_count: 4000
    latency_p50: 0.1ms
    latency_p99: 2ms
    size: 8500
    max_entries: 10000
    evictions: 120
    pressure: low
  severity: info
  timestamp: "2024-01-15T10:30:00Z"
```

#### Degraded Cache Signal

```yaml
signal:
  type: cache
  source:
    component: cache
    name: SessionCache
    tier: distributed
    region: us-west-2
  subject:
    cache: SessionCache
  metrics:
    hit_rate: 0.45
    miss_rate: 0.55
    latency_p50: 15ms
    latency_p99: 150ms
    size: 95000
    max_entries: 100000
    evictions: 5000
    pressure: critical
  severity: critical
  timestamp: "2024-01-15T10:30:00Z"
```

---

### Cache Signal Emission

Cache layers emit signals based on observability configuration.

```yaml
cacheLayer:
  name: ViewHybridCache
  version: v1
  topology:
    tier: hybrid
    # ... topology config
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

**Emission Rules**:
- Signals emitted at configured interval
- Severity escalates based on pressure level
- `info` for low/medium pressure
- `warn` for high pressure
- `critical` for critical pressure

---

### Signal Subscription

Systems can subscribe to cache signals for monitoring and auto-scaling.

```yaml
scheduler:
  name: CacheAutoScaler
  scope: cache
  authority: true
  observes:
    - signal: cache
      source:
        component: cache
        tier: distributed
      triggers:
        - condition: "metrics.pressure == 'critical'"
          action: scale_out
        - condition: "metrics.hit_rate < 0.5"
          action: investigate
```

---

## Integration with Existing Signal Infrastructure

### Signal Flow

```
┌─────────────────┐
│  Cache Layer    │
│  (L1/L2/Hybrid) │
└────────┬────────┘
         │ emit
         ▼
┌─────────────────┐
│ Signal Channel  │
│ (NATS Subject)  │
│ SMS.SIGNAL.cache│
└────────┬────────┘
         │ distribute
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────┐
│Monitor│ │Scaler │
└───────┘ └───────┘
```

### Subject Pattern

Cache signals are published to a dedicated subject hierarchy:

```
SMS.SIGNAL.cache.<cache_name>.<tier>.<region>

Examples:
- SMS.SIGNAL.cache.ViewHybridCache.hybrid.us-east-1
- SMS.SIGNAL.cache.SessionCache.distributed.us-west-2
```

---

## EBNF Grammar Extension

### Extended Signal Grammar

```ebnf
signal_type     ::= "capacity"
                  | "health"
                  | "latency"
                  | "saturation"
                  | "policy"
                  | "backpressure"
                  | "cache"            (* NEW in v1.6 *)

signal_source   ::= "source:" NEWLINE INDENT
                    "component:" component_type NEWLINE
                    "name:" string NEWLINE
                    ( "region:" string NEWLINE )?
                    ( "tier:" cache_tier NEWLINE )?   (* NEW in v1.6 *)
                    DEDENT

component_type  ::= "worker"
                  | "sms"
                  | "ui"
                  | "infra"
                  | "cache"            (* NEW in v1.6 *)

signal_subject  ::= "subject:" NEWLINE INDENT
                    ( "intent:" identifier NEWLINE )?
                    ( "dataState:" identifier NEWLINE )?
                    ( "cache:" identifier NEWLINE )?  (* NEW in v1.6 *)
                    DEDENT

cache_tier      ::= "local" | "distributed" | "hybrid"

cache_metrics   ::= "metrics:" NEWLINE INDENT
                    ( "hit_rate:" float NEWLINE )?
                    ( "miss_rate:" float NEWLINE )?
                    ( "hit_count:" integer NEWLINE )?
                    ( "miss_count:" integer NEWLINE )?
                    ( "latency_p50:" duration NEWLINE )?
                    ( "latency_p95:" duration NEWLINE )?
                    ( "latency_p99:" duration NEWLINE )?
                    ( "latency_avg:" duration NEWLINE )?
                    ( "size:" integer NEWLINE )?
                    ( "size_bytes:" integer NEWLINE )?
                    ( "max_entries:" integer NEWLINE )?
                    ( "max_bytes:" integer NEWLINE )?
                    ( "evictions:" integer NEWLINE )?
                    ( "eviction_rate:" float NEWLINE )?
                    ( "pressure:" pressure_level NEWLINE )?
                    DEDENT

pressure_level  ::= "low" | "medium" | "high" | "critical"
```

---

## Validation Rules

### Structural Violations (MUST reject)

1. Cache signal without source.tier
2. Cache signal without cache subject reference
3. Pressure level not in valid set
4. Hit rate or miss rate outside 0.0-1.0 range
5. Negative latency values
6. Negative size or count values

### Semantic Violations (MUST reject)

1. Cache signal referencing non-existent cache layer
2. Signal interval shorter than 1 second
3. Pressure level inconsistent with metrics (warning)

---

## Conformance

An implementation is conformant with Cache Signal semantics if it:

1. Emits cache signals at configured intervals
2. Calculates pressure levels correctly
3. Includes all configured metrics
4. Publishes to correct subject pattern
5. Escalates severity based on pressure
6. Provides accurate hit/miss rates
7. Reports latency percentiles correctly
8. Tracks eviction metrics

---

## Specification Metadata

```yaml
specMetadata:
  feature: cache-signal
  version: v1.6
  status: normative
  extends: signal (v1.5)
  origin: PS-01
  dependencies:
    - signal
    - cacheLayer
  changelog:
    v1.6:
      - "Added cache signal type"
      - "Extended source with tier field"
      - "Extended subject with cache field"
      - "Cache-specific metrics (hit_rate, miss_rate, latency, evictions)"
      - "Pressure level indicator"
      - "Cache signal emission configuration"
```
