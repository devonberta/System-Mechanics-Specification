# SMS v1.6 EBNF Grammar

> **Authoritative Formal Grammar for System Mechanics Specification v1.6**

This document provides the complete, integrated formal grammar in Extended Backus-Naur Form (EBNF) for all v1.6 extensions to the System Mechanics Specification. This grammar extends the v1.5 base grammar with new declarations and extensions for storage binding, unified caching, intent routing, resource requirements, and client-side caching.

**Related Documents:**
- **SMS-v1.5-EBNF-Grammar.md**: Base grammar (v1.5) that this document extends
- **01-grammar-reference.md**: v1.6 grammar reference overview
- **02-storage-binding.md**: Storage binding semantics
- **03-unified-caching.md**: Unified caching semantics
- **04-intent-routing.md**: Intent routing semantics
- **05-infrastructure.md**: Resource requirements semantics
- **06-client-cache.md**: Client cache contract semantics

---

## Document Organization

| Section | Purpose |
|---------|---------|
| 1. Grammar Extensions Summary | Overview of v1.6 additions |
| 2. Extended Lexical Grammar | New tokens for v1.6 |
| 3. Extended Document Structure | New top-level declarations |
| 4. Storage Binding Grammar | Storage configuration declarations |
| 5. Cache Layer Grammar | Unified caching declarations |
| 6. Intent Routing Grammar | Transport and routing extensions |
| 7. Resource Requirements Grammar | Abstract infrastructure declarations |
| 8. Client Cache Grammar | Client-side caching extensions |
| 9. Extended Signal Grammar | Cache signal type |
| 10. Validation Rules | Semantic constraints for v1.6 |
| 11. Conformance | Parser and runtime requirements |

---

# 1. GRAMMAR EXTENSIONS SUMMARY

## 1.1 New Top-Level Declarations (v1.6)

| Declaration | Purpose | Origin |
|-------------|---------|--------|
| `storageBinding` | Declarative storage configuration | v1.6 |
| `cacheLayer` | Unified caching configuration | PS-01 |
| `resourceRequirements` | Abstract infrastructure requirements | v1.6 |

## 1.2 Extensions to Existing Declarations (v1.6)

| Declaration | Extension | Origin |
|-------------|-----------|--------|
| `inputIntent` | `routing` block for transport configuration | v1.6 |
| `experience` | `clientCache` block for client-side caching | PS-01 |
| `signal` | `cache` signal type with cache metrics | PS-01 |
| `dataState` | `storage` reference to storageBinding | v1.6 |
| `materialization` | `cache` reference to cacheLayer | v1.6 |

---

# 2. EXTENDED LEXICAL GRAMMAR

These productions extend the v1.5 lexical grammar with new tokens required for v1.6.

## 2.1 New Keywords (v1.6)

```ebnf
(* Storage Binding Keywords *)
storage_binding_keyword = "storageBinding" ;
kv_keyword              = "kv" ;
stream_keyword          = "stream" ;
object_store_keyword    = "object_store" ;

(* Cache Layer Keywords *)
cache_layer_keyword     = "cacheLayer" ;
topology_keyword        = "topology" ;
invalidation_keyword    = "invalidation" ;
warmup_keyword          = "warmup" ;
observability_keyword   = "observability" ;

(* Resource Requirements Keywords *)
resource_requirements_keyword = "resourceRequirements" ;
event_logs_keyword      = "eventLogs" ;
entity_stores_keyword   = "entityStores" ;
object_stores_keyword   = "objectStores" ;
provisioning_keyword    = "provisioning" ;

(* Intent Routing Keywords *)
routing_keyword         = "routing" ;
transport_keyword       = "transport" ;
nats_keyword            = "nats" ;
http_keyword            = "http" ;
grpc_keyword            = "grpc" ;

(* Client Cache Keywords *)
client_cache_keyword    = "clientCache" ;
strategies_keyword      = "strategies" ;
prefetch_keyword        = "prefetch" ;
offline_keyword         = "offline" ;
```

## 2.2 Extended Enumerations

```ebnf
(* Storage Types *)
storage_type            = "kv" | "stream" | "object_store" ;

(* Cache Tiers *)
cache_tier              = "local" | "distributed" | "hybrid" ;

(* Eviction Policies *)
eviction_policy         = "lru" | "lfu" | "fifo" ;

(* Invalidation Strategies *)
invalidation_strategy   = "explicit" | "ttl" | "watch" | "hybrid" ;

(* Cache Strategies *)
cache_strategy          = "cache_first" | "network_first" | "stale_while_revalidate" ;

(* Transport Types *)
transport_type          = "nats" | "http" | "grpc" ;

(* Delivery Modes *)
delivery_mode           = "jetstream" | "core" ;

(* Response Modes *)
response_mode           = "request_reply" | "async" | "fire_and_forget" ;

(* Serialization Formats *)
serialization_type      = "json" | "protobuf" | "msgpack" ;

(* Retention Policies *)
retention_policy        = "limits" | "interest" | "workqueue" ;

(* Storage Medium *)
storage_medium          = "file" | "memory" ;

(* Read Policies *)
read_policy             = "local" | "replicated" | "eventual" ;

(* Write Consistency *)
write_consistency       = "strong" | "eventual" ;

(* Ordering Guarantees *)
ordering_guarantee      = "strict" | "causal" | "none" ;

(* Delivery Semantics *)
delivery_semantic       = "at_least_once" | "at_most_once" | "exactly_once" ;

(* Durability Types *)
durability_type         = "persistent" | "ephemeral" ;

(* Store Consistency Levels *)
store_consistency       = "strong_per_key" | "strong_global" | "eventual" ;

(* Provisioning Modes *)
provisioning_mode       = "auto" | "validate" | "strict" ;

(* Migration Strategies *)
migration_strategy      = "rolling" | "blue_green" ;

(* Conflict Resolution Strategies *)
conflict_strategy       = "last_write_wins" | "server_wins" | "client_wins" | "manual" ;

(* Pressure Levels *)
pressure_level          = "low" | "medium" | "high" | "critical" ;
```

---

# 3. EXTENDED DOCUMENT STRUCTURE

## 3.1 Extended Declaration List

The document declaration list is extended to include v1.6 top-level declarations:

```ebnf
(* Extended from v1.5 - NEW declarations in v1.6 *)
declaration             = (* v1.5 declarations *)
                          system_decl
                        | data_decl
                        | ui_decl
                        | input_decl
                        | sms_decl
                        | wts_decl
                        | policy_decl
                        | signal_decl
                        | realm_decl
                        | work_unit_contract_decl
                        | relationship_decl
                        | authority_decl
                        | presentation_composition_decl
                        | experience_decl
                        | search_intent_decl
                        | webhook_receiver_decl
                        | external_dependency_decl
                        | graph_query_decl
                        | consent_record_decl
                        | erasure_request_decl
                        | legal_hold_decl
                        | device_capability_decl
                        | device_telemetry_decl
                        | actuator_command_decl
                        (* NEW in v1.6 *)
                        | storage_binding_decl
                        | cache_layer_decl
                        | resource_requirements_decl ;
```

---

# 4. STORAGE BINDING GRAMMAR

Storage Binding provides declarative configuration for data persistence.

## 4.1 StorageBinding Declaration

```ebnf
storage_binding_decl    = "storageBinding:" , NEWLINE , INDENT ,
                          "name:" , identifier , NEWLINE ,
                          "version:" , version , NEWLINE ,
                          "type:" , storage_type , NEWLINE ,
                          ( kv_config | stream_config | object_store_config ) ,
                          [ access_config ] ,
                          DEDENT ;
```

## 4.2 Key-Value Storage Configuration

```ebnf
kv_config               = "kv:" , NEWLINE , INDENT ,
                          "bucket:" , string , NEWLINE ,
                          [ "key_pattern:" , string , NEWLINE ] ,
                          [ "serialization:" , serialization_type , NEWLINE ] ,
                          [ "history:" , integer , NEWLINE ] ,
                          [ "ttl:" , duration , NEWLINE ] ,
                          [ "replicas:" , integer , NEWLINE ] ,
                          DEDENT ;
```

## 4.3 Stream Storage Configuration

```ebnf
stream_config           = "stream:" , NEWLINE , INDENT ,
                          "name:" , string , NEWLINE ,
                          "subjects:" , subject_list , NEWLINE ,
                          [ "retention:" , retention_policy , NEWLINE ] ,
                          [ "max_msgs:" , integer , NEWLINE ] ,
                          [ "max_bytes:" , size_expression , NEWLINE ] ,
                          [ "max_age:" , duration , NEWLINE ] ,
                          [ "storage:" , storage_medium , NEWLINE ] ,
                          [ "replicas:" , integer , NEWLINE ] ,
                          DEDENT ;

subject_list            = "[" , subject_pattern , { "," , subject_pattern } , "]" ;

subject_pattern         = string ;
```

## 4.4 Object Store Configuration

```ebnf
object_store_config     = "object_store:" , NEWLINE , INDENT ,
                          "bucket:" , string , NEWLINE ,
                          [ "description:" , string , NEWLINE ] ,
                          [ "max_object_size:" , size_expression , NEWLINE ] ,
                          [ "replicas:" , integer , NEWLINE ] ,
                          DEDENT ;
```

## 4.5 Access Configuration

```ebnf
access_config           = "access:" , NEWLINE , INDENT ,
                          [ "read_policy:" , read_policy , NEWLINE ] ,
                          [ "write_consistency:" , write_consistency , NEWLINE ] ,
                          DEDENT ;
```

## 4.6 Key Pattern Grammar

```ebnf
key_pattern             = key_segment , { "." , key_segment } ;

key_segment             = static_segment
                        | dynamic_segment ;

static_segment          = identifier ;

dynamic_segment         = "{{" , field_ref , "}}" ;

field_ref               = identifier , { "." , identifier } ;
```

---

# 5. CACHE LAYER GRAMMAR

Unified Caching provides declarative configuration for multi-tier caching.

## 5.1 CacheLayer Declaration

```ebnf
cache_layer_decl        = "cacheLayer:" , NEWLINE , INDENT ,
                          "name:" , identifier , NEWLINE ,
                          [ "version:" , version , NEWLINE ] ,
                          [ "description:" , string , NEWLINE ] ,
                          topology_config ,
                          [ invalidation_config ] ,
                          [ warmup_config ] ,
                          [ cache_observability_config ] ,
                          DEDENT ;
```

## 5.2 Topology Configuration

```ebnf
topology_config         = "topology:" , NEWLINE , INDENT ,
                          "tier:" , cache_tier , NEWLINE ,
                          [ local_cache_config ] ,
                          [ distributed_cache_config ] ,
                          [ hybrid_cache_config ] ,
                          DEDENT ;

local_cache_config      = "local:" , NEWLINE , INDENT ,
                          [ "max_entries:" , integer , NEWLINE ] ,
                          [ "ttl:" , duration , NEWLINE ] ,
                          [ "eviction:" , eviction_policy , NEWLINE ] ,
                          [ "max_size:" , size_expression , NEWLINE ] ,
                          DEDENT ;

distributed_cache_config = "distributed:" , NEWLINE , INDENT ,
                           "resource:" , identifier , NEWLINE ,
                           [ "ttl:" , duration , NEWLINE ] ,
                           DEDENT ;

hybrid_cache_config     = "hybrid:" , NEWLINE , INDENT ,
                          [ "promotion:" , promotion_policy , NEWLINE ] ,
                          DEDENT ;

promotion_policy        = "on_miss" | "on_access" ;
```

## 5.3 Invalidation Configuration

```ebnf
invalidation_config     = "invalidation:" , NEWLINE , INDENT ,
                          "strategy:" , invalidation_strategy , NEWLINE ,
                          [ "ttl:" , duration , NEWLINE ] ,
                          [ "watch_keys:" , key_list , NEWLINE ] ,
                          [ "propagate:" , boolean , NEWLINE ] ,
                          DEDENT ;

key_list                = "[" , cache_key , { "," , cache_key } , "]" ;

cache_key               = string ;
```

## 5.4 Warmup Configuration

```ebnf
warmup_config           = "warmup:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "strategy:" , warmup_strategy , NEWLINE ] ,
                          [ "on_startup:" , boolean , NEWLINE ] ,
                          [ "queries:" , query_list , NEWLINE ] ,
                          DEDENT ;

warmup_strategy         = "lazy" | "eager" | "scheduled" ;

query_list              = "[" , query_expression , { "," , query_expression } , "]" ;

query_expression        = string ;
```

## 5.5 Cache Observability Configuration

```ebnf
cache_observability_config = "observability:" , NEWLINE , INDENT ,
                             [ "emit_signals:" , boolean , NEWLINE ] ,
                             [ "metrics:" , cache_metric_list , NEWLINE ] ,
                             [ "interval:" , duration , NEWLINE ] ,
                             DEDENT ;

cache_metric_list       = "[" , cache_metric , { "," , cache_metric } , "]" ;

cache_metric            = "hit_rate" | "miss_rate" | "latency" | "size" | "evictions" ;
```

---

# 6. INTENT ROUTING GRAMMAR

Intent Routing provides declarative configuration for InputIntent transport.

## 6.1 Routing Extension to InputIntent

```ebnf
(* Extension to inputIntent declaration from v1.5 *)
input_intent_decl       = "inputIntent:" , NEWLINE , INDENT ,
                          "name:" , identifier , NEWLINE ,
                          "version:" , version , NEWLINE ,
                          "proposes:" , proposes_spec , NEWLINE ,
                          "supplies:" , supplies_spec , NEWLINE ,
                          "constraints:" , input_constraints , NEWLINE ,
                          "transitions:" , transition_target ,
                          [ NEWLINE , "delivery:" , intent_delivery ] ,
                          [ NEWLINE , "response:" , intent_response ] ,
                          [ NEWLINE , intent_routing ] ,  (* NEW in v1.6 *)
                          DEDENT ;
```

## 6.2 Intent Routing Block

```ebnf
intent_routing          = "routing:" , NEWLINE , INDENT ,
                          "transport:" , transport_type , NEWLINE ,
                          transport_config ,
                          [ response_config ] ,
                          DEDENT ;

transport_config        = nats_config
                        | http_config
                        | grpc_config ;
```

## 6.3 NATS Transport Configuration

```ebnf
nats_config             = "nats:" , NEWLINE , INDENT ,
                          "subject_pattern:" , string , NEWLINE ,
                          [ "delivery:" , delivery_mode , NEWLINE ] ,
                          [ "stream:" , identifier , NEWLINE ] ,
                          [ "queue:" , identifier , NEWLINE ] ,
                          [ "dead_letter:" , string , NEWLINE ] ,
                          DEDENT ;
```

## 6.4 HTTP Transport Configuration

```ebnf
http_config             = "http:" , NEWLINE , INDENT ,
                          "path:" , string , NEWLINE ,
                          [ "method:" , http_method , NEWLINE ] ,
                          [ "timeout:" , duration , NEWLINE ] ,
                          DEDENT ;

http_method             = "GET" | "POST" | "PUT" | "PATCH" | "DELETE" ;
```

## 6.5 gRPC Transport Configuration

```ebnf
grpc_config             = "grpc:" , NEWLINE , INDENT ,
                          "service:" , identifier , NEWLINE ,
                          "method:" , identifier , NEWLINE ,
                          [ "streaming:" , boolean , NEWLINE ] ,
                          DEDENT ;
```

## 6.6 Response Configuration

```ebnf
response_config         = "response:" , NEWLINE , INDENT ,
                          "mode:" , response_mode , NEWLINE ,
                          [ "timeout:" , duration , NEWLINE ] ,
                          [ "callback:" , string , NEWLINE ] ,
                          DEDENT ;
```

## 6.7 Subject Pattern Grammar

```ebnf
routing_subject_pattern = subject_segment , { "." , subject_segment } ;

subject_segment         = static_subject_segment
                        | dynamic_subject_segment
                        | wildcard_segment ;

static_subject_segment  = identifier ;

dynamic_subject_segment = "{{" , variable_name , "}}" ;

variable_name           = "domain"
                        | "intent_name"
                        | "realm"
                        | "version"
                        | "entity_id"
                        | "correlation_id"
                        | identifier ;

wildcard_segment        = "*"       (* single token wildcard *)
                        | ">" ;     (* multi-token wildcard *)
```

---

# 7. RESOURCE REQUIREMENTS GRAMMAR

Resource Requirements provides abstract infrastructure specification.

## 7.1 ResourceRequirements Declaration

```ebnf
resource_requirements_decl = "resourceRequirements:" , NEWLINE , INDENT ,
                             "name:" , identifier , NEWLINE ,
                             [ "version:" , version , NEWLINE ] ,
                             [ "description:" , string , NEWLINE ] ,
                             [ event_logs_config ] ,
                             [ entity_stores_config ] ,
                             [ object_stores_config ] ,
                             [ provisioning_config ] ,
                             DEDENT ;
```

## 7.2 Event Logs Configuration

```ebnf
event_logs_config       = "eventLogs:" , NEWLINE , INDENT ,
                          { event_log_decl } ,
                          DEDENT ;

event_log_decl          = "- name:" , identifier , NEWLINE , INDENT ,
                          [ "purpose:" , event_log_purpose , NEWLINE ] ,
                          [ "ordering:" , ordering_guarantee , NEWLINE ] ,
                          [ "delivery:" , delivery_semantic , NEWLINE ] ,
                          [ "durability:" , durability_type , NEWLINE ] ,
                          [ retention_config ] ,
                          [ replication_config ] ,
                          DEDENT ;

event_log_purpose       = "intent_ingestion" | "event_sourcing" | "audit" | "dead_letter" ;

retention_config        = "retention:" , NEWLINE , INDENT ,
                          [ "strategy:" , retention_strategy , NEWLINE ] ,
                          [ "max_age:" , duration , NEWLINE ] ,
                          [ "max_size:" , size_expression , NEWLINE ] ,
                          [ "max_count:" , integer , NEWLINE ] ,
                          DEDENT ;

retention_strategy      = "bounded" | "unbounded" ;

replication_config      = "replication:" , NEWLINE , INDENT ,
                          [ "min_replicas:" , integer , NEWLINE ] ,
                          [ "consistency:" , replication_consistency , NEWLINE ] ,
                          DEDENT ;

replication_consistency = "strong" | "eventual" ;
```

## 7.3 Entity Stores Configuration

```ebnf
entity_stores_config    = "entityStores:" , NEWLINE , INDENT ,
                          { entity_store_decl } ,
                          DEDENT ;

entity_store_decl       = "- name:" , identifier , NEWLINE , INDENT ,
                          [ "purpose:" , entity_store_purpose , NEWLINE ] ,
                          [ "access_pattern:" , access_pattern , NEWLINE ] ,
                          [ "consistency:" , store_consistency , NEWLINE ] ,
                          [ "durability:" , durability_type , NEWLINE ] ,
                          [ history_config ] ,
                          [ expiration_config ] ,
                          [ replication_config ] ,
                          [ capacity_config ] ,
                          DEDENT ;

entity_store_purpose    = "authoritative_state" | "derived_state" | "cache" | "session" ;

access_pattern          = "key_value" | "range_scan" | "full_scan" ;

history_config          = "history:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "depth:" , integer , NEWLINE ] ,
                          DEDENT ;

expiration_config       = "expiration:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "ttl:" , duration , NEWLINE ] ,
                          DEDENT ;

capacity_config         = "capacity:" , NEWLINE , INDENT ,
                          [ "max_entry_size:" , size_expression , NEWLINE ] ,
                          [ "max_entries:" , integer , NEWLINE ] ,
                          DEDENT ;
```

## 7.4 Object Stores Configuration

```ebnf
object_stores_config    = "objectStores:" , NEWLINE , INDENT ,
                          { object_store_decl } ,
                          DEDENT ;

object_store_decl       = "- name:" , identifier , NEWLINE , INDENT ,
                          [ "purpose:" , object_store_purpose , NEWLINE ] ,
                          [ "durability:" , durability_type , NEWLINE ] ,
                          [ replication_config ] ,
                          [ object_capacity_config ] ,
                          DEDENT ;

object_store_purpose    = "documents" | "media" | "assets" | "archives" ;

object_capacity_config  = "capacity:" , NEWLINE , INDENT ,
                          [ "max_object_size:" , size_expression , NEWLINE ] ,
                          DEDENT ;
```

## 7.5 Provisioning Configuration

```ebnf
provisioning_config     = "provisioning:" , NEWLINE , INDENT ,
                          [ "mode:" , provisioning_mode , NEWLINE ] ,
                          [ migrations_config ] ,
                          DEDENT ;

migrations_config       = "migrations:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "strategy:" , migration_strategy , NEWLINE ] ,
                          DEDENT ;
```

---

# 8. CLIENT CACHE GRAMMAR

Client Cache Contract provides client-side caching configuration for Experiences.

## 8.1 ClientCache Extension to Experience

```ebnf
(* Extension to experience declaration from v1.5 *)
experience_decl         = "experience:" , NEWLINE , INDENT ,
                          "name:" , identifier , NEWLINE ,
                          "version:" , version , NEWLINE ,
                          [ "description:" , string , NEWLINE ] ,
                          "entry_point:" , identifier , NEWLINE ,
                          "includes:" , "[" , identifier_list , "]" , NEWLINE ,
                          [ "policy_set:" , identifier , NEWLINE ] ,
                          [ "unauthenticated:" , unauthenticated_spec , NEWLINE ] ,
                          [ "on_unauthorized:" , unauthorized_action , NEWLINE ] ,
                          [ "session:" , session_spec , NEWLINE ] ,
                          [ "devices:" , devices_spec , NEWLINE ] ,
                          [ "offline:" , v15_offline_spec , NEWLINE ] ,
                          [ "navigation:" , experience_navigation , NEWLINE ] ,
                          [ client_cache_config ] ,  (* NEW in v1.6 *)
                          DEDENT ;
```

## 8.2 Client Cache Configuration

```ebnf
client_cache_config     = "clientCache:" , NEWLINE , INDENT ,
                          "enabled:" , boolean , NEWLINE ,
                          client_storage_config ,
                          [ client_strategies_config ] ,
                          [ client_prefetch_config ] ,
                          [ client_offline_config ] ,
                          [ version_policy_config ] ,
                          DEDENT ;
```

## 8.3 Client Storage Configuration

```ebnf
client_storage_config   = "storage:" , NEWLINE , INDENT ,
                          "type:" , client_storage_type , NEWLINE ,
                          [ "max_size:" , size_expression , NEWLINE ] ,
                          [ "encryption:" , boolean , NEWLINE ] ,
                          DEDENT ;

client_storage_type     = "memory" | "persistent" | "hybrid" ;
```

## 8.4 Client Strategies Configuration

```ebnf
client_strategies_config = "strategies:" , NEWLINE , INDENT ,
                           [ primary_strategy_config ] ,
                           [ related_strategy_config ] ,
                           DEDENT ;

primary_strategy_config = "primary:" , NEWLINE , INDENT ,
                          "strategy:" , cache_strategy , NEWLINE ,
                          [ "ttl:" , duration , NEWLINE ] ,
                          DEDENT ;

related_strategy_config = "related:" , NEWLINE , INDENT ,
                          "strategy:" , cache_strategy , NEWLINE ,
                          [ "ttl:" , duration , NEWLINE ] ,
                          [ "lazy_cache:" , boolean , NEWLINE ] ,
                          DEDENT ;
```

## 8.5 Client Prefetch Configuration

```ebnf
client_prefetch_config  = "prefetch:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "views:" , view_list , NEWLINE ] ,
                          [ "trigger:" , prefetch_trigger , NEWLINE ] ,
                          DEDENT ;

prefetch_trigger        = "on_navigate" | "on_idle" | "on_connect" ;

view_list               = "[" , view_ref , { "," , view_ref } , "]" ;

view_ref                = identifier ;
```

## 8.6 Client Offline Configuration

```ebnf
client_offline_config   = "offline:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "required_views:" , view_list , NEWLINE ] ,
                          [ "sync_on_reconnect:" , boolean , NEWLINE ] ,
                          [ "conflict_resolution:" , conflict_strategy , NEWLINE ] ,
                          [ intent_sync_config ] ,
                          [ governance_sync_config ] ,
                          DEDENT ;

intent_sync_config      = "intent_sync:" , NEWLINE , INDENT ,
                          [ "on_cas_failure:" , cas_failure_action , NEWLINE ] ,
                          [ "retry_with_refresh:" , boolean , NEWLINE ] ,
                          DEDENT ;

cas_failure_action      = "reject_and_notify" | "queue_for_retry" ;

governance_sync_config  = "governance:" , NEWLINE , INDENT ,
                          [ "validate_on_sync:" , boolean , NEWLINE ] ,
                          [ "on_violation:" , violation_action , NEWLINE ] ,
                          DEDENT ;

violation_action        = "reject" | "queue_for_review" | "notify_and_proceed" ;
```

## 8.7 Version Policy Configuration

```ebnf
version_policy_config   = "version_policy:" , NEWLINE , INDENT ,
                          [ "on_upgrade:" , upgrade_action , NEWLINE ] ,
                          [ "grace_period:" , duration , NEWLINE ] ,
                          [ "migration:" , cache_migration_strategy , NEWLINE ] ,
                          DEDENT ;

upgrade_action          = "invalidate_all" | "invalidate_incompatible" | "keep_compatible" ;

cache_migration_strategy = "eager" | "lazy" ;
```

---

# 9. EXTENDED SIGNAL GRAMMAR

Cache Signals provide operational observability for cache layers.

## 9.1 Extended Signal Type

```ebnf
(* Extended from v1.5 signal grammar *)
signal_type             = "capacity"
                        | "health"
                        | "latency"
                        | "saturation"
                        | "policy"
                        | "backpressure"
                        | "error"
                        | "cache" ;          (* NEW in v1.6 *)
```

## 9.2 Extended Component Type

```ebnf
(* Extended from v1.5 *)
component_type          = "worker"
                        | "sms"
                        | "ui"
                        | "infra"
                        | "cache" ;          (* NEW in v1.6 *)
```

## 9.3 Extended Signal Source

```ebnf
(* Extended from v1.5 *)
signal_source           = "source:" , NEWLINE , INDENT ,
                          "component:" , component_type , NEWLINE ,
                          "name:" , identifier , NEWLINE ,
                          [ "region:" , identifier , NEWLINE ] ,
                          [ "tier:" , cache_tier , NEWLINE ] ,   (* NEW in v1.6 *)
                          DEDENT ;
```

## 9.4 Extended Signal Subject

```ebnf
(* Extended from v1.5 *)
signal_subject          = "subject:" , NEWLINE , INDENT ,
                          [ "intent:" , identifier , NEWLINE ] ,
                          [ "dataState:" , identifier , NEWLINE ] ,
                          [ "cache:" , identifier , NEWLINE ] ,  (* NEW in v1.6 *)
                          DEDENT ;
```

## 9.5 Cache Signal Metrics

```ebnf
cache_signal_metrics    = "metrics:" , NEWLINE , INDENT ,
                          [ "hit_rate:" , float , NEWLINE ] ,
                          [ "miss_rate:" , float , NEWLINE ] ,
                          [ "hit_count:" , integer , NEWLINE ] ,
                          [ "miss_count:" , integer , NEWLINE ] ,
                          [ "latency_p50:" , duration , NEWLINE ] ,
                          [ "latency_p95:" , duration , NEWLINE ] ,
                          [ "latency_p99:" , duration , NEWLINE ] ,
                          [ "latency_avg:" , duration , NEWLINE ] ,
                          [ "size:" , integer , NEWLINE ] ,
                          [ "size_bytes:" , integer , NEWLINE ] ,
                          [ "max_entries:" , integer , NEWLINE ] ,
                          [ "max_bytes:" , integer , NEWLINE ] ,
                          [ "evictions:" , integer , NEWLINE ] ,
                          [ "eviction_rate:" , float , NEWLINE ] ,
                          [ "pressure:" , pressure_level , NEWLINE ] ,
                          DEDENT ;

float                   = integer , "." , integer ;
```

---

# 10. EXTENDED DATA GRAMMAR

Extensions to data-related declarations for v1.6.

## 10.1 DataState Storage Reference

```ebnf
(* Extension to dataState declaration from v1.5 *)
datastate_decl          = "dataState:" , NEWLINE , INDENT ,
                          "name:" , identifier , NEWLINE ,
                          "type:" , identifier , NEWLINE ,
                          "lifecycle:" , lifecycle_type , NEWLINE ,
                          [ "constraintPolicy:" , constraint_policy , NEWLINE ] ,
                          [ "evolutionPolicy:" , evolution_policy , NEWLINE ] ,
                          [ "governance:" , governance_spec , NEWLINE ] ,
                          [ "storage:" , storage_reference , NEWLINE ] ,  (* NEW in v1.6 *)
                          DEDENT ;

storage_reference       = NEWLINE , INDENT ,
                          "binding:" , identifier , NEWLINE ,   (* References storageBinding *)
                          DEDENT ;
```

## 10.2 Materialization Cache Reference

```ebnf
(* Extension to materialization declaration from v1.5 *)
materialization_decl    = "materialization:" , NEWLINE , INDENT ,
                          "name:" , identifier , NEWLINE ,
                          "source:" , source_spec , NEWLINE ,
                          "targetState:" , identifier , NEWLINE ,
                          "freshness:" , freshness_spec , NEWLINE ,
                          "evolution:" , mat_evolution_spec ,
                          [ NEWLINE , "retrieval:" , mat_retrieval_spec ] ,
                          [ NEWLINE , "temporal:" , mat_temporal_spec ] ,
                          [ NEWLINE , "search_index:" , search_index_spec ] ,
                          [ NEWLINE , "external_source:" , external_source_spec ] ,
                          [ NEWLINE , "cache:" , mat_cache_reference ] ,  (* NEW in v1.6 *)
                          DEDENT ;

mat_cache_reference     = NEWLINE , INDENT ,
                          "layer:" , identifier , NEWLINE ,     (* References cacheLayer *)
                          "key:" , string , NEWLINE ,           (* Cache key pattern *)
                          DEDENT ;
```

---

# 11. VALIDATION RULES

## 11.1 Storage Binding Validation

### Structural Violations (MUST reject)

```
1. storageBinding without name
2. storageBinding without type
3. storageBinding with type="kv" but missing kv.bucket
4. storageBinding with type="stream" but missing stream.name
5. storageBinding with type="object_store" but missing object_store.bucket
6. Invalid key_pattern expression (unclosed {{ or }})
7. serialization not in valid set (json, protobuf, msgpack)
8. replicas less than 1
9. retention not in valid set (limits, interest, workqueue)
10. storage not in valid set (file, memory)
```

### Semantic Violations (MUST reject)

```
1. Strong write_consistency with read_policy="local" only
2. ttl specified on stream with retention="interest"
3. history specified on stream storage (history is KV-only)
4. key_pattern referencing fields not in associated dataType
```

## 11.2 Cache Layer Validation

### Structural Violations (MUST reject)

```
1. cacheLayer without name
2. cacheLayer without topology.tier
3. topology.tier="local" without eviction policy when max_entries specified
4. topology.tier="distributed" without resource reference
5. topology.tier="hybrid" without both local and distributed config
6. invalidation.strategy="watch" without watch_keys
7. eviction not in valid set (lru, lfu, fifo)
8. invalidation.strategy not in valid set (explicit, ttl, watch, hybrid)
```

### Semantic Violations (MUST reject)

```
1. local.ttl greater than distributed.ttl in hybrid cache
2. watch_keys referencing non-existent entities
3. warmup.queries specified with warmup.strategy="lazy"
4. propagate=true on local-only cache (tier="local")
```

### Reference Violations (MUST reject)

```
1. distributed.resource referencing non-existent entityStore in resourceRequirements
2. Resource with purpose="authoritative_state" used for cache (warning)
3. Resource without expiration.enabled when cache layer specifies TTL
```

## 11.3 Intent Routing Validation

### Structural Violations (MUST reject)

```
1. routing block without transport type
2. transport="nats" without nats.subject_pattern
3. transport="http" without http.path
4. transport="grpc" without grpc.service or grpc.method
5. response.mode="async" without response.callback
6. Invalid subject_pattern syntax (unclosed {{ or }}, invalid wildcard placement)
7. http.method not in valid set (GET, POST, PUT, PATCH, DELETE)
```

### Semantic Violations (MUST reject)

```
1. nats.delivery="jetstream" without nats.stream reference
2. response.mode="fire_and_forget" with response.mode="request_reply" (contradiction)
3. response.timeout specified with response.mode="fire_and_forget"
4. nats.queue specified with wildcard-only subject pattern
5. grpc.streaming=true with response.mode="fire_and_forget"
```

### Invariant Violations (MUST reject - I9 Transport Scope)

```
1. Transport bindings specified within sms.flows steps
2. Transport bindings specified on work_unit declarations
3. Transport bindings specified on atomic groups
```

## 11.4 Resource Requirements Validation

### Structural Violations (MUST reject)

```
1. resourceRequirements without name
2. eventLog without name
3. entityStore without name
4. objectStore without name
5. ordering not in valid set (strict, causal, none)
6. delivery not in valid set (at_least_once, at_most_once, exactly_once)
7. durability not in valid set (persistent, ephemeral)
8. consistency not in valid set (strong_per_key, strong_global, eventual)
9. min_replicas less than 1
10. provisioning.mode not in valid set (auto, validate, strict)
```

### Semantic Violations (MUST reject)

```
1. Duplicate resource names across eventLogs, entityStores, objectStores
2. delivery="exactly_once" with durability="ephemeral"
3. history.enabled=true with purpose="cache"
4. consistency="strong_global" with durability="ephemeral"
5. purpose="authoritative_state" with consistency="eventual" (warning)
```

### Reference Violations (MUST reject)

```
1. storageBinding referencing non-existent resource
2. cacheLayer referencing non-existent entityStore
3. intentRouting referencing non-existent eventLog
```

## 11.5 Client Cache Validation

### Structural Violations (MUST reject)

```
1. clientCache without storage.type
2. offline.enabled=true without required_views
3. prefetch.enabled=true without views list
4. strategies.primary.strategy not in valid set (cache_first, network_first, stale_while_revalidate)
5. conflict_resolution not in valid set (last_write_wins, server_wins, client_wins, manual)
6. max_size not a valid size_expression
7. prefetch.trigger not in valid set (on_navigate, on_idle, on_connect)
```

### Semantic Violations (MUST reject)

```
1. storage.type="memory" with encryption=true (memory is volatile, encryption meaningless)
2. offline.enabled=true with storage.type="memory" (offline requires persistent storage)
3. sync_on_reconnect=true without offline.enabled=true
4. required_views referencing non-existent presentationView
5. intent_sync.on_cas_failure set to any auto-resolve value (violates I4)
6. governance.on_violation="notify_and_proceed" for intents with enforcement="strict" (violates I7)
7. version_policy.on_upgrade="keep_compatible" with breaking schema changes between versions
```

## 11.6 Cache Signal Validation

### Structural Violations (MUST reject)

```
1. signal.type="cache" without source.tier
2. signal.type="cache" without subject.cache reference
3. pressure not in valid set (low, medium, high, critical)
4. hit_rate or miss_rate outside 0.0-1.0 range
5. Negative latency values
6. Negative size or count values
```

### Semantic Violations (MUST reject)

```
1. Cache signal referencing non-existent cacheLayer
2. observability.interval shorter than 1 second
3. Pressure level inconsistent with metrics (warning only)
```

---

# 12. CONFORMANCE

## 12.1 Parser Conformance

A parser implementation is conformant with SMS v1.6 EBNF grammar if it:

1. Accepts all syntactically valid v1.6 documents
2. Rejects all syntactically invalid documents with meaningful error messages
3. Correctly parses all new v1.6 declarations (storageBinding, cacheLayer, resourceRequirements)
4. Correctly parses all v1.6 extensions (inputIntent.routing, experience.clientCache, signal.type="cache")
5. Produces an Abstract Syntax Tree (AST) representing all grammar elements
6. Handles all enumeration values defined in this grammar
7. Validates structural constraints at parse time
8. Supports incremental parsing for IDE integration
9. Maintains backward compatibility with v1.5 documents

## 12.2 Validator Conformance

A validator implementation is conformant with SMS v1.6 semantics if it:

1. Enforces all structural violations listed in Section 11
2. Enforces all semantic violations listed in Section 11
3. Enforces all reference violations listed in Section 11
4. Validates I9 (Transport Scope Boundary) invariant
5. Validates resource references between declarations
6. Produces clear error messages with location information
7. Distinguishes between errors (MUST reject) and warnings
8. Supports progressive validation for large documents

## 12.3 Runtime Conformance

A runtime implementation is conformant with SMS v1.6 if it:

### Storage Binding Conformance
1. Parses and validates all storageBinding declarations
2. Creates storage resources matching specifications
3. Enforces key patterns on writes
4. Respects read policies for lookups
5. Honors write consistency guarantees
6. Maintains replication factor
7. Applies retention policies correctly
8. Enforces TTL expiration
9. Preserves history depth for KV stores

### Cache Layer Conformance
1. Parses and validates all cacheLayer declarations
2. Resolves resource references to resourceRequirements
3. Creates cache infrastructure using appropriate backend for resource
4. Respects topology tier configuration
5. Enforces eviction policies correctly (L1 lookup < 10μs, L2 lookup < 5ms)
6. Applies invalidation strategies as declared (propagation < 100ms)
7. Propagates invalidation across instances when configured
8. Emits cache signals at configured intervals
9. Warms cache according to warmup configuration
10. Promotes entries in hybrid cache per policy

### Intent Routing Conformance
1. Parses and validates all routing declarations
2. Resolves subject patterns correctly at runtime
3. Routes intents via declared transport
4. Applies delivery mode guarantees (JetStream vs Core)
5. Respects response mode semantics
6. Enforces timeouts as declared
7. Delivers async callbacks correctly
8. Distributes load via queue groups
9. Routes failed intents to dead-letter when configured
10. Preserves I9 invariant (transport only at intent ingress)

### Resource Requirements Conformance
1. Parses and validates all resourceRequirements declarations
2. Maps abstract requirements to appropriate backend implementations
3. Provisions resources with equivalent or stronger guarantees than specified
4. Honors ordering guarantees per requirement
5. Implements delivery semantics correctly
6. Respects durability requirements
7. Enforces retention policies
8. Maintains replication minimums
9. Respects provisioning mode on startup
10. Reports provisioning mappings accurately

### Client Cache Conformance
1. Parses and validates all clientCache declarations
2. Creates client storage matching type specification
3. Enforces max_size limits with appropriate eviction
4. Applies encryption when specified
5. Implements caching strategies correctly (memory < 1ms, persistent < 10ms)
6. Prefetches views according to trigger configuration
7. Maintains required views for offline access
8. Syncs on reconnect when configured
9. Applies conflict resolution strategy to view cache only
10. Respects I4 (CAS Safety) for offline intent synchronization
11. Validates governance constraints on sync when configured
12. Invalidates cache appropriately on experience version changes

---

# 13. GRAMMAR SUMMARY

## 13.1 Complete Production Index

| Production | Section | Type |
|------------|---------|------|
| storage_binding_decl | 4.1 | Top-level |
| kv_config | 4.2 | Storage |
| stream_config | 4.3 | Storage |
| object_store_config | 4.4 | Storage |
| access_config | 4.5 | Storage |
| cache_layer_decl | 5.1 | Top-level |
| topology_config | 5.2 | Cache |
| invalidation_config | 5.3 | Cache |
| warmup_config | 5.4 | Cache |
| cache_observability_config | 5.5 | Cache |
| intent_routing | 6.2 | Extension |
| nats_config | 6.3 | Transport |
| http_config | 6.4 | Transport |
| grpc_config | 6.5 | Transport |
| response_config | 6.6 | Transport |
| resource_requirements_decl | 7.1 | Top-level |
| event_logs_config | 7.2 | Resource |
| entity_stores_config | 7.3 | Resource |
| object_stores_config | 7.4 | Resource |
| provisioning_config | 7.5 | Resource |
| client_cache_config | 8.2 | Extension |
| client_storage_config | 8.3 | Client |
| client_strategies_config | 8.4 | Client |
| client_prefetch_config | 8.5 | Client |
| client_offline_config | 8.6 | Client |
| version_policy_config | 8.7 | Client |
| cache_signal_metrics | 9.5 | Signal |

## 13.2 Enumeration Values Summary

| Enumeration | Values |
|-------------|--------|
| storage_type | kv, stream, object_store |
| cache_tier | local, distributed, hybrid |
| eviction_policy | lru, lfu, fifo |
| invalidation_strategy | explicit, ttl, watch, hybrid |
| promotion_policy | on_miss, on_access |
| warmup_strategy | lazy, eager, scheduled |
| transport_type | nats, http, grpc |
| delivery_mode | jetstream, core |
| response_mode | request_reply, async, fire_and_forget |
| http_method | GET, POST, PUT, PATCH, DELETE |
| serialization_type | json, protobuf, msgpack |
| retention_policy | limits, interest, workqueue |
| storage_medium | file, memory |
| read_policy | local, replicated, eventual |
| write_consistency | strong, eventual |
| ordering_guarantee | strict, causal, none |
| delivery_semantic | at_least_once, at_most_once, exactly_once |
| durability_type | persistent, ephemeral |
| store_consistency | strong_per_key, strong_global, eventual |
| entity_store_purpose | authoritative_state, derived_state, cache, session |
| event_log_purpose | intent_ingestion, event_sourcing, audit, dead_letter |
| object_store_purpose | documents, media, assets, archives |
| provisioning_mode | auto, validate, strict |
| migration_strategy | rolling, blue_green |
| cache_strategy | cache_first, network_first, stale_while_revalidate |
| client_storage_type | memory, persistent, hybrid |
| prefetch_trigger | on_navigate, on_idle, on_connect |
| conflict_strategy | last_write_wins, server_wins, client_wins, manual |
| cas_failure_action | reject_and_notify, queue_for_retry |
| violation_action | reject, queue_for_review, notify_and_proceed |
| upgrade_action | invalidate_all, invalidate_incompatible, keep_compatible |
| pressure_level | low, medium, high, critical |
| cache_metric | hit_rate, miss_rate, latency, size, evictions |

---

# 14. SPECIFICATION METADATA

```yaml
specMetadata:
  document: formal-grammar-ebnf
  version: v1.6
  status: normative
  extends: SMS-v1.5-EBNF-Grammar
  origin: PS-01 (Unified Caching Proposal)
  dependencies:
    - lexical-grammar (v1.5 base)
    - document-structure (v1.5 base)
    - core-grammar (v1.5 base)
    - presentation-grammar (v1.5 base)
    - execution-grammar (v1.5 base)
  invariants:
    - "I9. Transport Scope Boundary"
    - "I4. CAS Safety (offline intent sync)"
    - "I7. Governance enforcement (offline sync)"
  changelog:
    v1.6:
      - "Added storageBinding top-level declaration"
      - "Added cacheLayer top-level declaration"
      - "Added resourceRequirements top-level declaration"
      - "Extended inputIntent with routing block"
      - "Extended experience with clientCache block"
      - "Extended signal with cache type and metrics"
      - "Extended dataState with storage reference"
      - "Extended materialization with cache reference"
      - "Added comprehensive validation rules for all v1.6 features"
      - "Added conformance requirements for parser, validator, and runtime"
      - "Added I9 Transport Scope Boundary invariant"
```
