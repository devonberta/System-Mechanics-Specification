# SMS v1.6 EBNF Grammar

> **Authoritative Formal Grammar for System Mechanics Specification v1.6**

This document provides the complete, integrated formal grammar in Extended Backus-Naur Form (EBNF) suitable for parser generation, validation tooling, and formal analysis. All production rules are defined exactly once—no duplicates.

**Version**: 1.6  
**Status**: Production Ready  
**Last Updated**: 2026-02-04

**Related Documents:**
- **SMS-v1.6-Specification.md**: Normative specification with semantic definitions
- **SMS-v1.6-Implementation-Guide.md**: Runtime implementation patterns
- **SMS-v1.6-Reference-Examples.md**: Modeling guidance and worked examples
- **SMS-v1.6-Quick-Reference.md**: Rapid-lookup decision trees and templates

---

## Document Organization

| Section | Purpose |
|---------|---------|
| 1. Lexical Grammar | Base tokens (letters, identifiers, literals) — SINGLE SOURCE |
| 2. Document Structure | Top-level declarations |
| 3. Core Grammar | Data, Authority, Relationship, Invariant |
| 4. Presentation Grammar | UI, Composition, Experience, Navigation |
| 5. Execution Grammar | SMS Flows, WTS Topology, Policy, Signals |
| 6. v1.5 Extensions | Search, Assets, Sessions, Governance, Edge/IoT |
| 7. v1.6 Extensions | Storage Binding, Cache Layer, Resource Requirements, Intent Routing, Client Cache, Cache Signals |
| 8. Validation Rules | Semantic constraints |
| 9. Conformance | Parser, runtime, and tooling requirements |

---

## What's New in v1.6

**6 Major Additions** for infrastructure and caching:

| Declaration | Purpose | Origin |
|-------------|---------|--------|
| `storageBinding` | Declarative storage configuration | v1.6 |
| `cacheLayer` | Unified caching configuration | PS-01 |
| `resourceRequirements` | Abstract infrastructure requirements | v1.6 |

**Extensions to Existing Declarations:**

| Declaration | Extension | Origin |
|-------------|-----------|--------|
| `inputIntent` | `routing` block for transport configuration | v1.6 |
| `experience` | `clientCache` block for client-side caching | PS-01 |
| `signal` | `cache` signal type with cache metrics | PS-01 |
| `dataState` | `storage` reference to storageBinding | v1.6 |
| `materialization` | `cache` reference to cacheLayer | v1.6 |

---

# 1. LEXICAL GRAMMAR

All lexical tokens are defined here exactly once. Every production in subsequent sections references these base definitions.

## 1.1 Characters and Identifiers

```ebnf
(* Characters — BASE DEFINITION *)
letter        = "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m" |
                "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z" |
                "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J" | "K" | "L" | "M" |
                "N" | "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X" | "Y" | "Z" ;

digit         = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;

underscore    = "_" ;

(* Identifiers — BASE DEFINITION *)
identifier    = letter , { letter | digit | underscore } ;
```

## 1.2 Literals

```ebnf
(* Numbers *)
integer       = digit , { digit } ;
decimal       = integer , "." , integer ;
number        = integer | decimal ;

(* Booleans *)
boolean       = "true" | "false" ;

(* Strings *)
string_char   = ? any Unicode character except " and \ ? ;
escape_seq    = "\\" , ( "n" | "t" | "r" | "\"" | "\\" ) ;
string        = "\"" , { string_char | escape_seq } , "\"" ;

(* Durations *)
time_unit     = "ms" | "s" | "m" | "h" | "d" ;
duration      = integer , time_unit ;

(* Versions — BASE DEFINITION *)
version       = "v" , integer ;

(* Size expressions *)
bytes         = integer , [ "KB" | "MB" | "GB" ] ;
size_expression = integer , ( "KB" | "MB" | "GB" ) ;
```

## 1.3 Comments and Whitespace

```ebnf
comment       = "//" , { ? any character ? } , newline ;
newline       = "\n" | "\r\n" ;
whitespace    = " " | "\t" | newline ;
```

## 1.4 Expressions

```ebnf
(* Expression — BASE DEFINITION *)
expression    = comparison_expr
              | logical_expr
              | arithmetic_expr
              | function_call ;

comparison_expr = operand , comparison_op , operand ;
comparison_op = "==" | "!=" | "<" | ">" | "<=" | ">=" | "in" ;

logical_expr  = expression , logical_op , expression
              | "NOT" , expression ;
logical_op    = "AND" | "OR" ;

arithmetic_expr = operand , arithmetic_op , operand ;
arithmetic_op = "+" | "-" | "*" | "/" ;

operand       = identifier | number | string | boolean | field_access ;
field_access  = identifier , { "." , identifier } ;

function_call = identifier , "(" , [ arg_list ] , ")" ;
arg_list      = expression , { "," , expression } ;

(* Common list types *)
identifier_list = identifier , { "," , identifier } ;
expression_list = expression , { "," , expression } ;
string_list   = string , { "," , string } ;

(* Float for metrics *)
float         = integer , "." , integer ;

(* Indentation tokens *)
NEWLINE       = newline ;
INDENT        = ? increase indentation level ? ;
DEDENT        = ? decrease indentation level ? ;
```

## 1.5 v1.6 Extended Keywords

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

## 1.6 v1.6 Extended Enumerations

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
v16_cache_strategy      = "cache_first" | "network_first" | "stale_while_revalidate" ;

(* Transport Types *)
v16_transport_type      = "nats" | "http" | "grpc" ;

(* Delivery Modes *)
delivery_mode           = "jetstream" | "core" ;

(* Response Modes *)
response_mode           = "request_reply" | "async" | "fire_and_forget" ;

(* Serialization Formats *)
serialization_type      = "json" | "protobuf" | "msgpack" ;

(* Retention Policies *)
v16_retention_policy    = "limits" | "interest" | "workqueue" ;

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
v16_migration_strategy  = "rolling" | "blue_green" ;

(* Conflict Resolution Strategies *)
v16_conflict_strategy   = "last_write_wins" | "server_wins" | "client_wins" | "manual" ;

(* Pressure Levels *)
pressure_level          = "low" | "medium" | "high" | "critical" ;

(* Promotion Policies *)
promotion_policy        = "on_miss" | "on_access" ;

(* Warmup Strategies *)
warmup_strategy         = "lazy" | "eager" | "scheduled" ;

(* Cache Metrics *)
cache_metric            = "hit_rate" | "miss_rate" | "latency" | "size" | "evictions" ;

(* Client Storage Types *)
client_storage_type     = "memory" | "persistent" | "hybrid" ;

(* Prefetch Triggers *)
prefetch_trigger        = "on_navigate" | "on_idle" | "on_connect" ;

(* CAS Failure Actions *)
cas_failure_action      = "reject_and_notify" | "queue_for_retry" ;

(* Violation Actions *)
v16_violation_action    = "reject" | "queue_for_review" | "notify_and_proceed" ;

(* Upgrade Actions *)
upgrade_action          = "invalidate_all" | "invalidate_incompatible" | "keep_compatible" ;

(* Cache Migration Strategy *)
cache_migration_strategy = "eager" | "lazy" ;

(* Event Log Purposes *)
event_log_purpose       = "intent_ingestion" | "event_sourcing" | "audit" | "dead_letter" ;

(* Entity Store Purposes *)
entity_store_purpose    = "authoritative_state" | "derived_state" | "cache" | "session" ;

(* Object Store Purposes *)
object_store_purpose    = "documents" | "media" | "assets" | "archives" ;

(* Access Patterns *)
access_pattern          = "key_value" | "range_scan" | "full_scan" ;

(* Retention Strategy *)
retention_strategy      = "bounded" | "unbounded" ;

(* Replication Consistency *)
replication_consistency = "strong" | "eventual" ;

(* HTTP Methods *)
http_method             = "GET" | "POST" | "PUT" | "PATCH" | "DELETE" ;
```

---

# 2. DOCUMENT STRUCTURE

## 2.1 Document and Declarations

A specification document consists of a single `system_decl` followed by any number of collection declarations. Each collection uses a **plural key** containing a YAML list of items. This ensures unique keys at the document root level and valid YAML syntax.

```ebnf
document      = system_decl , { collection_decl } ;

collection_decl = datatype_collection
                | model_collection
                | datastate_collection
                | constraint_collection
                | evolution_collection
                | materialization_collection
                | transition_collection
                | relationship_collection
                | authority_collection
                | view_collection
                | presentation_view_collection
                | presentation_binding_collection
                | presentation_composition_collection
                | experience_collection
                | input_intent_collection
                | sms_flow_collection
                | workers_collection
                | work_unit_contract_collection
                | policy_collection
                | signal_collection
                | realm_collection
                | search_intent_collection
                | webhook_receiver_collection
                | external_dependency_collection
                | graph_query_collection
                | data_governance_collection
                | consent_record_collection
                | erasure_request_collection
                | legal_hold_collection
                | notification_channel_collection
                | scheduled_trigger_collection
                | durable_workflow_collection
                | asset_collection
                | delegation_collection
                | conversational_experience_collection
                | device_capability_collection
                | device_telemetry_collection
                | actuator_command_collection
                | edge_device_collection
                (* NEW in v1.6 *)
                | storage_binding_collection
                | cache_layer_collection
                | resource_requirements_decl ;
```

## 2.2 Collection Syntax

All multi-instance declarations use plural keys with YAML list syntax. Each collection key is the plural form of its item type. Items within collections use list item syntax (`- `).

```ebnf
(* Collection Pattern — applies to all multi-instance types *)
(* Each collection uses a plural key containing a YAML list of item bodies *)

datatype_collection               = "dataTypes:" , NEWLINE , INDENT , { "- " , datatype_body } , DEDENT ;
model_collection                  = "models:" , NEWLINE , INDENT , { "- " , model_body } , DEDENT ;
datastate_collection              = "dataStates:" , NEWLINE , INDENT , { "- " , datastate_body } , DEDENT ;
constraint_collection             = "constraints:" , NEWLINE , INDENT , { "- " , constraint_body } , DEDENT ;
evolution_collection              = "dataEvolutions:" , NEWLINE , INDENT , { "- " , evolution_body } , DEDENT ;
materialization_collection        = "materializations:" , NEWLINE , INDENT , { "- " , materialization_body } , DEDENT ;
transition_collection             = "transitions:" , NEWLINE , INDENT , { "- " , transition_body } , DEDENT ;
relationship_collection           = "relationships:" , NEWLINE , INDENT , { "- " , relationship_body } , DEDENT ;
authority_collection              = "authorities:" , NEWLINE , INDENT , { "- " , authority_body } , DEDENT ;
view_collection                   = "views:" , NEWLINE , INDENT , { "- " , view_body } , DEDENT ;
presentation_view_collection      = "presentationViews:" , NEWLINE , INDENT , { "- " , presentation_view_body } , DEDENT ;
presentation_binding_collection   = "presentationBindings:" , NEWLINE , INDENT , { "- " , presentation_binding_body } , DEDENT ;
presentation_composition_collection = "presentationCompositions:" , NEWLINE , INDENT , { "- " , presentation_composition_body } , DEDENT ;
experience_collection             = "experiences:" , NEWLINE , INDENT , { "- " , experience_body } , DEDENT ;
input_intent_collection           = "inputIntents:" , NEWLINE , INDENT , { "- " , input_intent_body } , DEDENT ;
sms_flow_collection               = "smsFlows:" , NEWLINE , INDENT , { "- " , flow_body } , DEDENT ;
workers_collection                = "workers:" , NEWLINE , INDENT , { "- " , worker_body } , DEDENT ;
work_unit_contract_collection     = "workUnitContracts:" , NEWLINE , INDENT , { "- " , work_unit_contract_body } , DEDENT ;
policy_collection                 = "policies:" , NEWLINE , INDENT , { "- " , policy_body } , DEDENT ;
signal_collection                 = "signals:" , NEWLINE , INDENT , { "- " , signal_body } , DEDENT ;
realm_collection                  = "realms:" , NEWLINE , INDENT , { "- " , realm_body } , DEDENT ;
search_intent_collection          = "searchIntents:" , NEWLINE , INDENT , { "- " , search_intent_body } , DEDENT ;
webhook_receiver_collection       = "webhookReceivers:" , NEWLINE , INDENT , { "- " , webhook_receiver_body } , DEDENT ;
external_dependency_collection    = "externalDependencies:" , NEWLINE , INDENT , { "- " , external_dependency_body } , DEDENT ;
graph_query_collection            = "graphQueries:" , NEWLINE , INDENT , { "- " , graph_query_body } , DEDENT ;
data_governance_collection        = "dataGovernances:" , NEWLINE , INDENT , { "- " , data_governance_body } , DEDENT ;
consent_record_collection         = "consentRecords:" , NEWLINE , INDENT , { "- " , consent_record_body } , DEDENT ;
erasure_request_collection        = "erasureRequests:" , NEWLINE , INDENT , { "- " , erasure_request_body } , DEDENT ;
legal_hold_collection             = "legalHolds:" , NEWLINE , INDENT , { "- " , legal_hold_body } , DEDENT ;
notification_channel_collection   = "notificationChannels:" , NEWLINE , INDENT , { "- " , notification_channel_body } , DEDENT ;
scheduled_trigger_collection      = "scheduledTriggers:" , NEWLINE , INDENT , { "- " , scheduled_trigger_body } , DEDENT ;
durable_workflow_collection       = "durableWorkflows:" , NEWLINE , INDENT , { "- " , durable_workflow_body } , DEDENT ;
asset_collection                  = "assets:" , NEWLINE , INDENT , { "- " , asset_body } , DEDENT ;
delegation_collection             = "delegations:" , NEWLINE , INDENT , { "- " , delegation_body } , DEDENT ;
conversational_experience_collection = "conversationalExperiences:" , NEWLINE , INDENT , { "- " , conversational_experience_body } , DEDENT ;
device_capability_collection      = "deviceCapabilities:" , NEWLINE , INDENT , { "- " , device_capability_body } , DEDENT ;
device_telemetry_collection       = "deviceTelemetries:" , NEWLINE , INDENT , { "- " , device_telemetry_body } , DEDENT ;
actuator_command_collection       = "actuatorCommands:" , NEWLINE , INDENT , { "- " , actuator_command_body } , DEDENT ;
edge_device_collection            = "edgeDevices:" , NEWLINE , INDENT , { "- " , edge_device_body } , DEDENT ;
(* NEW in v1.6 *)
storage_binding_collection        = "storageBindings:" , NEWLINE , INDENT , { "- " , storage_binding_body } , DEDENT ;
cache_layer_collection            = "cacheLayers:" , NEWLINE , INDENT , { "- " , cache_layer_body } , DEDENT ;
```

**Singular Declarations** (only one per specification):
- `system:` — System declaration (exactly one per spec)
- `resourceRequirements:` — Infrastructure requirements (at most one per spec)

**YAML Example:**
```yaml
system:
  name: MySystem
  version: v1

dataTypes:
  - name: Account
    version: v1
    fields:
      id: uuid
      name: string

  - name: Transaction
    version: v1
    fields:
      id: uuid
      amount: decimal

models:
  - name: Account
    mutability:
      scope: entity

storageBindings:
  - name: AccountStore
    version: v1
    type: kv
    kv:
      bucket: "SMS_ACCOUNTS"
```

## 2.3 System Declaration

```ebnf
system_decl   = "system" , ":" , system_body ;
system_body   = "{" , { system_property } , "}" ;
system_property = "name" , ":" , string
                | "version" , ":" , version
                | "description" , ":" , string
                | "realm" , ":" , identifier ;
```

---

# 3. CORE GRAMMAR

## 3.1 Data Grammar

### 3.1.1 DataType

```ebnf
(* DataType item body — used within dataTypes: collection *)
datatype_body = "name" , ":" , identifier , "," ,
                "version" , ":" , version , "," ,
                "fields" , ":" , "{" , field_list , "}" ;

field_list    = field_def , { "," , field_def } ;
field_def     = identifier , ":" , type_ref , [ field_extensions ] ;

type_ref      = primitive_type
              | enum_type
              | array_type
              | reference_type ;

field_extensions = [ asset_spec ] , [ search_spec ] , [ inference_spec ] , 
                   [ structured_content_spec ] , [ point_spec ] ;

primitive_type = "string" | "integer" | "decimal" | "boolean" | "uuid" | 
                 "timestamp" | "duration" | "asset" | "point" | "structured_content" ;

enum_type     = "enum" , "[" , enum_values , "]" ;
enum_values   = identifier , { "," , identifier } ;

array_type    = "array" , "<" , type_ref , ">" ;
reference_type = identifier ;
```

### 3.1.2 Constraint

```ebnf
(* Constraint item body — used within constraints: collection *)
constraint_body = "name" , ":" , identifier , "," ,
                  "appliesTo" , ":" , identifier , "," ,
                  "rules" , ":" , "{" , rule_list , "}" ;

rule_list     = rule_def , { "," , rule_def } ;
rule_def      = version , ":" , expression ;
```

### 3.1.3 DataState

```ebnf
(* DataState item body — used within dataStates: collection *)
datastate_body = "name" , ":" , identifier , "," ,
                 "type" , ":" , identifier , "," ,
                 "lifecycle" , ":" , lifecycle_type , "," ,
                 [ "constraintPolicy" , ":" , constraint_policy , "," ] ,
                 [ "evolutionPolicy" , ":" , evolution_policy , "," ] ,
                 [ "governance" , ":" , governance_spec , "," ] ,
                 [ "storage" , ":" , storage_reference , "," ] ;  (* NEW in v1.6 *)

(* NEW in v1.6: Storage Reference *)
storage_reference = "{" ,
                      "binding" , ":" , identifier ,  (* References storageBinding *)
                    "}" ;

lifecycle_type = "intermediate" | "persistent" | "materialized" ;

constraint_policy = "{" ,
                      "enforcement" , ":" , enforcement_level ,
                    "}" ;
enforcement_level = "none" | "best_effort" | "strict" | "deferred" ;

evolution_policy = "{" ,
                     "allowed" , ":" , "[" , change_list , "]" , "," ,
                     "forbidden" , ":" , "[" , change_list , "]" ,
                   "}" ;
change_list   = change_type , { "," , change_type } ;
change_type   = "additive" | "constraint_refinement" | "destructive" | "semantic_break" ;
```

### 3.1.4 DataEvolution

```ebnf
(* DataEvolution item body — used within dataEvolutions: collection *)
evolution_body = "name" , ":" , identifier , "," ,
                 "from" , ":" , version_ref , "," ,
                 "to" , ":" , version_ref , "," ,
                 "changes" , ":" , "[" , change_ops , "]" , "," ,
                 "compatibility" , ":" , compatibility_spec ;

version_ref   = "{" , "type" , ":" , identifier , "," , "version" , ":" , version , "}" ;

change_ops    = change_op , { "," , change_op } ;
change_op     = add_field_op | remove_field_op | rename_field_op | 
                change_type_op | extend_enum_op | refine_constraint_op ;

add_field_op  = "addField" , ":" , "{" , "name" , ":" , identifier , "," , 
                "default" , ":" , value , "}" ;
remove_field_op = "removeField" , ":" , identifier ;
extend_enum_op = "extendEnum" , ":" , "{" , "field" , ":" , identifier , "," , 
                 "values" , ":" , "[" , enum_values , "]" , "}" ;

compatibility_spec = "{" ,
                       "read" , ":" , compatibility_mode , "," ,
                       "write" , ":" , compatibility_mode ,
                     "}" ;
compatibility_mode = "backward" | "forward" | "none" ;
```

### 3.1.5 Materialization

```ebnf
(* Materialization item body — used within materializations: collection *)
materialization_body = "name" , ":" , identifier , "," ,
                       "source" , ":" , source_spec , "," ,
                       "targetState" , ":" , identifier , "," ,
                       "freshness" , ":" , freshness_spec , "," ,
                       "evolution" , ":" , mat_evolution_spec ,
                       [ "," , "retrieval" , ":" , mat_retrieval_spec ] ,
                       [ "," , "temporal" , ":" , mat_temporal_spec ] ,
                       [ "," , "search_index" , ":" , search_index_spec ] ,
                       [ "," , "external_source" , ":" , external_source_spec ] ,
                       [ "," , "cache" , ":" , mat_cache_reference ] ;  (* NEW in v1.6 *)

(* NEW in v1.6: Cache Reference *)
mat_cache_reference = "{" ,
                        "layer" , ":" , identifier , "," ,  (* References cacheLayer *)
                        "key" , ":" , string ,              (* Cache key pattern *)
                      "}" ;

source_spec   = identifier | ( "[" , identifier , { "," , identifier } , "]" ) ;
freshness_spec = "{" , "maxStaleness" , ":" , duration , "}" ;

mat_evolution_spec = "{" ,
                       "strategy" , ":" , mat_strategy , "," ,
                       "blocking" , ":" , boolean ,
                     "}" ;
mat_strategy  = "rebuild" | "incremental" ;

view_authority_spec = [ "authority_agnostic" , ":" , boolean , "," ] ,
                      [ "epoch_tolerant" , ":" , boolean ] ;
```

### 3.1.6 Transition

```ebnf
(* Transition item body — used within transitions: collection *)
transition_body = "name" , ":" , identifier , "," ,
                  "from" , ":" , identifier , "," ,
                  "to" , ":" , identifier , "," ,
                  "triggeredBy" , ":" , trigger_source , "," ,
                  [ "guarantees" , ":" , "[" , identifier_list , "]" , "," ] ,
                  [ "idempotency" , ":" , idempotency_spec ] ;

trigger_source = "inputIntent" | "smsFlow" | identifier ;

idempotency_spec = "{" ,
                     "key" , ":" , key_expr , "," ,
                     "scope" , ":" , idempotency_scope ,
                   "}" ;
key_expr      = expression | ( "[" , expression_list , "]" ) ;
idempotency_scope = "global" | "per_subject" | "per_realm" ;
```

## 3.2 Authority Grammar

### 3.2.1 Model and Mutability Declaration

```ebnf
(* Model item body — used within models: collection *)
model_body = "name" , ":" , identifier , "," ,
             "mutability" , ":" , mutability_spec , "," ,
             [ "authority" , ":" , authority_body ] ;

mutability_spec = "{" ,
                    "scope" , ":" , mutability_scope , "," ,
                    "exclusive" , ":" , boolean , "," ,
                    [ "authority" , ":" , identifier ] ,
                  "}" ;

mutability_scope = "entity" | "append-only" ;
```

### 3.2.2 Authority Declaration

```ebnf
(* Authority item body — used within authorities: collection *)
authority_body = authority_scope , "," ,
                 authority_resolution , "," ,
                 [ authority_migration ] ;

authority_scope = "scope" , ":" , ( "entity" | "model" ) ;
authority_resolution = "resolution" , ":" , expression ;

authority_migration = "migration" , ":" , "{" ,
                        "allowed" , ":" , boolean , "," ,
                        [ "strategy" , ":" , migration_strategy , "," ] ,
                        [ "triggers" , ":" , "[" , trigger_list , "]" , "," ] ,
                        [ "constraints" , ":" , "[" , migration_constraint_list , "]" , "," ] ,
                        [ "governed_by" , ":" , identifier ] ,
                      "}" ;

migration_strategy = "pause-and-cutover" ;

trigger_list = migration_trigger , { "," , migration_trigger } ;
migration_trigger = "{" ,
                      "type" , ":" , trigger_type , "," ,
                      "condition" , ":" , expression ,
                    "}" ;
trigger_type = "load" | "latency" | "manual" | "time" ;

migration_constraint_list = migration_constraint , { "," , migration_constraint } ;
migration_constraint = "{" ,
                         "type" , ":" , constraint_type , "," ,
                         ( "allowed" , ":" , "[" , identifier_list , "]"
                         | "rule" , ":" , expression ) ,
                       "}" ;
constraint_type = "region" | "governance" ;
```

### 3.2.3 Authority State

```ebnf
authority_state = "authorityState" , ":" , "{" ,
                    "entity_id" , ":" , string , "," ,
                    "authority_region" , ":" , identifier , "," ,
                    "authority_epoch" , ":" , integer , "," ,
                    "authority_lease" , ":" , string , "," ,
                    "authority_status" , ":" , authority_status_type , "," ,
                    "entity_version" , ":" , integer ,
                  "}" ;

authority_status_type = "ACTIVE" | "TRANSITIONING" ;
```

### 3.2.4 Storage Role

```ebnf
storage_role_decl = "storage" , ":" , "{" ,
                      "role" , ":" , storage_role_type , "," ,
                      "rebuildable" , ":" , boolean ,
                    "}" ;

storage_role_type = "data" | "control" ;
```

### 3.2.5 CAS Token

```ebnf
cas_decl = "cas" , ":" , "{" ,
             "token_fields" , ":" , "[" , identifier_list , "]" ,
           "}" ;
```

## 3.3 Relationship Grammar

### 3.3.1 Relationship Declaration

```ebnf
(* Relationship item body — used within relationships: collection *)
relationship_body = "name" , ":" , identifier , "," ,
                    "from" , ":" , identifier , "," ,
                    "to" , ":" , identifier , "," ,
                    [ "type" , ":" , identifier , "," ] ,
                    [ "cardinality" , ":" , cardinality_type , "," ] ,
                    [ "semantics" , ":" , relationship_semantics ] ;

cardinality_type = "one-to-one" | "one-to-many" | "many-to-one" | "many-to-many" ;

relationship_semantics = "{" ,
                           [ "causal" , ":" , boolean , "," ] ,
                           [ "ordered" , ":" , boolean ] ,
                         "}" ;
```

### 3.3.2 Delegation Semantics

```ebnf
delegation_spec = "delegation" , ":" , "{" ,
                    "scope" , ":" , "[" , permission_list , "]" , "," ,
                    [ "constraints" , ":" , delegation_constraints , "," ] ,
                    [ "requires" , ":" , delegation_requires , "," ] ,
                    [ "validity" , ":" , delegation_validity , "," ] ,
                    [ "revocation" , ":" , delegation_revocation ] ,
                  "}" ;

permission_list = identifier , { "," , identifier } ;

delegation_constraints = "{" ,
                           [ "transitive" , ":" , boolean , "," ] ,
                           [ "max_depth" , ":" , integer , "," ] ,
                           [ "same_realm" , ":" , boolean ] ,
                         "}" ;

delegation_requires = "{" , "approval" , ":" , approval_type , "}" ;
approval_type = "grantor" | "grantee" | "both" | "none" ;

delegation_validity = "{" ,
                        "type" , ":" , validity_type , "," ,
                        [ "duration" , ":" , duration ] ,
                      "}" ;
validity_type = "temporal" | "permanent" | "conditional" ;

delegation_revocation = "{" , "by" , ":" , revocation_actor , "}" ;
revocation_actor = "grantor" | "grantee" | "either" | "admin" ;
```

## 3.4 Invariant Grammar

### 3.4.1 Invariant Policy Declaration

```ebnf
invariant_policy_decl = "invariants" , ":" , "{" ,
                          "enforced_by" , ":" , invariant_enforcer , "," ,
                          [ "description" , ":" , string ] ,
                        "}" ;

invariant_enforcer = "view" | "policy" ;
```

### 3.4.2 View Invariant Extension

```ebnf
view_invariant = "invariant" , ":" , "{" ,
                   "expression" , ":" , expression , "," ,
                   [ "on_violation" , ":" , violation_action ] ,
                 "}" ;

violation_action = "reject" | "flag" | "log" ;
```

## 3.5 Availability Grammar

```ebnf
availability_decl = "availability" , ":" , "{" ,
                      "write_scope" , ":" , availability_scope , "," ,
                      "pause_allowed" , ":" , boolean , "," ,
                      "read_continuity" , ":" , continuity_level ,
                    "}" ;

availability_scope = "entity" | "model" | "region" ;
continuity_level = "best-effort" | "guaranteed" ;

write_failure_decl = "write_failure" , ":" , "{" ,
                       "strategy" , ":" , failure_strategy ,
                     "}" ;

failure_strategy = "reject" | "client_queue" | "regional_buffer" ;
```

---

# 4. PRESENTATION GRAMMAR

## 4.1 PresentationView

```ebnf
(* PresentationView item body — used within presentationViews: collection *)
presentation_view_body = "name" , ":" , identifier , "," ,
                         "version" , ":" , version , "," ,
                         [ "description" , ":" , string , "," ] ,
                         "consumes" , ":" , consumes_spec , "," ,
                         [ "guarantees" , ":" , "[" , string_list , "]" , "," ] ,
                         [ "interactionModel" , ":" , interaction_model , "," ] ,
                         [ "fieldPermissions" , ":" , "[" , field_permission_list , "]" , "," ] ,
                         [ "triggers" , ":" , view_triggers ] ;

consumes_spec = "{" ,
                  "dataState" , ":" , identifier , "," ,
                  "version" , ":" , version ,
                "}" ;

interaction_model = "{" , { interaction_element } , "}" ;
interaction_element = "badges" , ":" , "[" , badge_list , "]"
                    | "actions" , ":" , "[" , action_list , "]"
                    | "filters" , ":" , "[" , filter_list , "]"
                    | "sorts" , ":" , "[" , sort_list , "]" ;

badge_list    = badge , { "," , badge } ;
badge         = "{" ,
                  "if" , ":" , expression , "," ,
                  "label" , ":" , string , "," ,
                  [ "severity" , ":" , severity_level ] ,
                "}" ;
severity_level = "info" | "warning" | "error" | "success" | "critical" ;

action_list   = action , { "," , action } ;
action        = "{" ,
                  "if" , ":" , expression , "," ,
                  "action" , ":" , identifier , "," ,
                  "label" , ":" , string , "," ,
                  [ "intent" , ":" , identifier ] ,
                "}" ;

filter_list   = filter , { "," , filter } ;
filter        = "{" ,
                  "field" , ":" , identifier , "," ,
                  "type" , ":" , filter_type ,
                "}" ;
filter_type   = "search" | "date_range" | "enum" | "range" ;

sort_list     = sort , { "," , sort } ;
sort          = "{" ,
                  "field" , ":" , identifier , "," ,
                  [ "default" , ":" , sort_direction ] ,
                "}" ;
sort_direction = "asc" | "desc" ;
```

## 4.2 Field Permissions

```ebnf
field_permission_list = field_permission , { "," , field_permission } ;

field_permission = "{" ,
                     "field" , ":" , identifier , "," ,
                     "visible" , ":" , visibility_spec ,
                     [ "," , "mask" , ":" , mask_spec ] ,
                   "}" ;

visibility_spec = boolean 
                | "{" , "policy" , ":" , identifier , "}" ;

mask_spec     = "{" ,
                  "unless" , ":" , expression , "," ,
                  "type" , ":" , mask_type ,
                  [ "," , "reveal" , ":" , reveal_type ] ,
                "}" ;

mask_type     = "full" | "partial" | "hash" ;
reveal_type   = "last4" | "first3" | "none" ;
```

## 4.3 View Triggers

```ebnf
view_triggers = "{" ,
                  "intent" , ":" , identifier , "," ,
                  "binding" , ":" , binding_type , "," ,
                  [ "field_mapping" , ":" , "[" , field_mapping_list , "]" , "," ] ,
                  [ "confirmation" , ":" , confirmation_spec , "," ] ,
                  "on_success" , ":" , success_action_spec , "," ,
                  "on_error" , ":" , error_action_spec ,
                "}" ;

binding_type  = "form" | "action" | "link" ;

field_mapping_list = field_mapping_entry , { "," , field_mapping_entry } ;
field_mapping_entry = "{" ,
                        "view_field" , ":" , identifier , "," ,
                        "supplies" , ":" , identifier , "," ,
                        "source" , ":" , field_source_type , "," ,
                        "required" , ":" , boolean ,
                        [ "," , "default" , ":" , value ] ,
                        [ "," , "validation" , ":" , field_validation ] ,
                      "}" ;

field_source_type = "context" | "input" | "computed" ;

field_validation = "{" ,
                     "constraint" , ":" , expression , "," ,
                     "message" , ":" , string ,
                   "}" ;

confirmation_spec = "{" ,
                      "required" , ":" , boolean , "," ,
                      "message" , ":" , string ,
                    "}" ;

success_action_spec = "{" ,
                        "action" , ":" , success_action_type ,
                        [ "," , "target" , ":" , identifier ] ,
                        [ "," , "params" , ":" , "{" , param_map , "}" ] ,
                        [ "," , "notification" , ":" , notification_spec ] ,
                      "}" ;
success_action_type = "navigate_back" | "navigate_to" | "stay" | "refresh" | "close" ;

error_action_spec = "{" ,
                      "action" , ":" , error_action_type ,
                      [ "," , "show_field_errors" , ":" , boolean ] ,
                      [ "," , "notification" , ":" , notification_spec ] ,
                    "}" ;
error_action_type = "display_inline" | "display_modal" | "retry" ;

notification_spec = "{" ,
                      "type" , ":" , notification_type , "," ,
                      "message" , ":" , string ,
                    "}" ;
notification_type = "toast" | "inline" | "modal" | "none" ;

param_map     = param_entry , { "," , param_entry } ;
param_entry   = identifier , ":" , expression ;
```

## 4.4 PresentationBinding

```ebnf
(* PresentationBinding item body — used within presentationBindings: collection *)
presentation_binding_body = "name" , ":" , identifier , "," ,
                            "rules" , ":" , "[" , binding_rules , "]" ;

binding_rules = binding_rule , { "," , binding_rule } ;
binding_rule  = "{" ,
                  "if" , ":" , expression , "," ,
                  "use" , ":" , view_ref ,
                "}"
              | "{" , "else" , ":" , "{" , "use" , ":" , view_ref , "}" , "}" ;

view_ref      = identifier , [ version ] ;
```

## 4.5 Presentation Composition

```ebnf
(* PresentationComposition item body — used within presentationCompositions: collection *)
presentation_composition_body = "name" , ":" , identifier , "," ,
                                "version" , ":" , version , "," ,
                                [ "description" , ":" , string , "," ] ,
                                "primary" , ":" , identifier , "," ,
                                [ "navigation" , ":" , composition_navigation , "," ] ,
                                [ "related" , ":" , "[" , related_view_list , "]" , "," ] ,
                                [ "fetch" , ":" , composition_fetch_spec ] ;
```

### 4.5.1 Composition Navigation

```ebnf
composition_navigation = "{" ,
                           "label" , ":" , string , "," ,
                           [ "purpose" , ":" , purpose_type , "," ] ,
                           [ "parent" , ":" , identifier , "," ] ,
                           [ "param" , ":" , identifier , "," ] ,
                           [ "policy_set" , ":" , identifier , "," ] ,
                           [ "indicator" , ":" , indicator_spec ] ,
                         "}" ;

purpose_type = "home" | "accounts" | "transactions" | "customers" | 
               "alerts" | "compliance" | "audit" | "reports" | 
               "settings" | "help" | "authentication" | "action" ;

indicator_spec = "{" ,
                   "when" , ":" , expression , "," ,
                   "severity" , ":" , severity_level ,
                   [ "," , "name" , ":" , identifier ] ,
                   [ "," , "type" , ":" , indicator_type ] ,
                   [ "," , "source" , ":" , indicator_source ] ,
                   [ "," , "update" , ":" , indicator_update ] ,
                   [ "," , "display" , ":" , indicator_display ] ,
                 "}" ;

indicator_type = "count" | "boolean" | "value" ;

indicator_source = "{" ,
                     "view" , ":" , identifier ,
                     [ "," , "field" , ":" , identifier ] ,
                     [ "," , "scope" , ":" , indicator_scope ] ,
                     [ "," , "filter" , ":" , expression ] ,
                     [ "," , "aggregation" , ":" , agg_function ] ,
                   "}" ;

indicator_scope = "{" ,
                    [ "expression" , ":" , expression , "," ] ,
                    "type" , ":" , scope_type ,
                  "}" ;
scope_type    = "user" | "entity" | "global" ;

indicator_update = "{" ,
                     "trigger" , ":" , update_trigger ,
                     [ "," , "interval" , ":" , duration ] ,
                     [ "," , "throttle" , ":" , duration ] ,
                     [ "," , "debounce" , ":" , duration ] ,
                   "}" ;
update_trigger = "view_change" | "interval" | "manual" | "event" ;

indicator_display = "{" ,
                      "format" , ":" , display_format ,
                      [ "," , "max_value" , ":" , integer ] ,
                      [ "," , "hide_when_zero" , ":" , boolean ] ,
                    "}" ;
display_format = "number" | "badge" | "dot" ;
```

### 4.5.2 Related Views

```ebnf
related_view_list = related_view , { "," , related_view } ;

related_view = "{" ,
                 "view" , ":" , identifier , "," ,
                 "relationship" , ":" , view_relationship_type , "," ,
                 [ "cardinality" , ":" , view_cardinality , "," ] ,
                 [ "required" , ":" , boolean , "," ] ,
                 [ "basis" , ":" , relationship_basis , "," ] ,
                 [ "alias" , ":" , string , "," ] ,
                 [ "trigger" , ":" , identifier , "," ] ,
                 [ "policy" , ":" , identifier , "," ] ,
                 [ "policy_set" , ":" , identifier , "," ] ,
                 [ "data_scope" , ":" , data_scope_spec , "," ] ,
                 [ "navigation" , ":" , related_view_navigation ] ,
               "}" ;

view_relationship_type = "derived" | "detail" | "contextual" | "action" | 
                         "supplementary" | "historical" ;

view_cardinality = "one" | "many" ;

relationship_basis = identifier 
                   | "[" , identifier , { "," , identifier } , "]" ;

data_scope_spec = "{" , { scope_field } , "}" ;
scope_field = identifier , ":" , expression , [ "," ] ;

related_view_navigation = "{" ,
                            "label" , ":" , string , "," ,
                            [ "scope" , ":" , navigation_scope ] ,
                          "}" ;
navigation_scope = "within_composition" | "always_visible" | "on_action" ;
```

### 4.5.3 Composition Fetch Semantics

```ebnf
composition_fetch_spec = "{" ,
                           "order" , ":" , fetch_order , "," ,
                           [ "primary" , ":" , primary_fetch_config , "," ] ,
                           [ "related" , ":" , related_fetch_config , "," ] ,
                           [ "on_partial_failure" , ":" , partial_failure_spec , "," ] ,
                           [ "retry" , ":" , fetch_retry_spec , "," ] ,
                           [ "cache" , ":" , fetch_cache_spec ] ,
                         "}" ;

fetch_order   = "primary_first" | "parallel_all" | "sequential" ;

primary_fetch_config = "{" ,
                         "required" , ":" , boolean , "," ,
                         "timeout" , ":" , duration ,
                       "}" ;

related_fetch_config = "{" ,
                         "strategy" , ":" , fetch_strategy , "," ,
                         "timeout" , ":" , duration ,
                         [ "," , "max_concurrent" , ":" , integer ] ,
                       "}" ;
fetch_strategy = "parallel" | "sequential" | "lazy" ;

partial_failure_spec = "{" ,
                         "primary" , ":" , primary_failure_action , "," ,
                         "related" , ":" , related_failure_action ,
                       "}" ;
primary_failure_action = "fail_composition" | "degrade" ;
related_failure_action = "degrade_gracefully" | "omit" | "fail_composition" ;

fetch_retry_spec = "{" ,
                     "enabled" , ":" , boolean ,
                     [ "," , "max_attempts" , ":" , integer ] ,
                     [ "," , "scope" , ":" , retry_scope ] ,
                   "}" ;
retry_scope   = "failed_only" | "all" ;

fetch_cache_spec = "{" ,
                     "strategy" , ":" , cache_strategy , "," ,
                     "ttl" , ":" , duration ,
                     [ "," , "scope" , ":" , cache_scope ] ,
                   "}" ;
cache_strategy = "none" | "cache_first" | "stale_while_revalidate" | "network_first" ;
cache_scope   = "composition" | "view" ;
```

## 4.6 Experience Grammar

```ebnf
(* Experience item body — used within experiences: collection *)
experience_body = "name" , ":" , identifier , "," ,
                  "version" , ":" , version , "," ,
                  [ "description" , ":" , string , "," ] ,
                  "entry_point" , ":" , identifier , "," ,
                  "includes" , ":" , "[" , identifier_list , "]" , "," ,
                  [ "policy_set" , ":" , identifier , "," ] ,
                  [ "unauthenticated" , ":" , unauthenticated_spec , "," ] ,
                  [ "on_unauthorized" , ":" , unauthorized_action , "," ] ,
                  [ "session" , ":" , session_spec , "," ] ,
                  [ "devices" , ":" , devices_spec , "," ] ,
                  [ "offline" , ":" , offline_spec , "," ] ,
                  [ "navigation" , ":" , experience_navigation , "," ] ,
                  [ "clientCache" , ":" , client_cache_config , "," ] ;  (* NEW in v1.6 *)

(* Experience Navigation *)
experience_navigation = "{" ,
                          [ "root" , ":" , identifier , "," ] ,
                          [ "tree" , ":" , "[" , nav_tree_list , "]" , "," ] ,
                        "}" ;

nav_tree_list = nav_tree_item , { "," , nav_tree_item } ;
nav_tree_item = "{" ,
                  "name" , ":" , string , "," ,
                  "view" , ":" , identifier ,
                  [ "," , "children" , ":" , "[" , nav_tree_list , "]" ] ,
                "}" ;

unauthenticated_spec = "{" , "redirect_to" , ":" , identifier , "}" ;
unauthorized_action = "conceal" | "indicate" | "redirect" | "deny" ;
```

## 4.7 Navigation Grammar

```ebnf
navigation_decl = "navigation" , ":" , "{" ,
                    [ "purposes" , ":" , "[" , purpose_def_list , "]" , "," ] ,
                    [ "scopes" , ":" , "[" , scope_def_list , "]" ] ,
                  "}" ;

purpose_def_list = purpose_def , { "," , purpose_def } ;
purpose_def = "{" ,
                "name" , ":" , identifier , "," ,
                "description" , ":" , string ,
              "}" ;

scope_def_list = scope_def , { "," , scope_def } ;
scope_def = "{" ,
              "name" , ":" , identifier , "," ,
              "description" , ":" , string ,
            "}" ;
```

---

# 5. EXECUTION GRAMMAR

## 5.1 Input (Form) Grammar

### 5.1.1 InputIntent

```ebnf
(* InputIntent item body — used within inputIntents: collection *)
input_intent_body = "name" , ":" , identifier , "," ,
                    "version" , ":" , version , "," ,
                    "proposes" , ":" , proposes_spec , "," ,
                    "supplies" , ":" , supplies_spec , "," ,
                    "constraints" , ":" , input_constraints , "," ,
                    "transitions" , ":" , transition_target ,
                    [ "," , "delivery" , ":" , intent_delivery ] ,
                    [ "," , "response" , ":" , intent_response ] ,
                    [ "," , "routing" , ":" , intent_routing ] ;

proposes_spec = "{" ,
                  "dataState" , ":" , identifier , "," ,
                  "version" , ":" , version ,
                "}" ;

supplies_spec = "{" ,
                  "required" , ":" , "[" , identifier_list , "]" , "," ,
                  "optional" , ":" , "[" , identifier_list , "]" ,
                "}" ;

input_constraints = "{" ,
                      "guarantees" , ":" , "[" , expression_list , "]" , "," ,
                      "defers" , ":" , "[" , identifier_list , "]" ,
                    "}" ;

transition_target = "{" , "to" , ":" , identifier , "}" ;
```

### 5.1.2 Intent Delivery Contract

```ebnf
intent_delivery = "{" ,
                    "guarantee" , ":" , delivery_guarantee , "," ,
                    "acknowledgment" , ":" , ack_mode , "," ,
                    "timeout" , ":" , duration ,
                    [ "," , "retry" , ":" , delivery_retry_config ] ,
                  "}" ;

delivery_guarantee = "at_least_once" | "at_most_once" | "exactly_once" ;
ack_mode      = "required" | "optional" | "fire_and_forget" ;

delivery_retry_config = "{" ,
                          "allowed" , ":" , boolean ,
                          [ "," , "max_attempts" , ":" , integer ] ,
                          [ "," , "backoff" , ":" , backoff_strategy ] ,
                        "}" ;

intent_response = "{" ,
                    "on_success" , ":" , response_success_spec , "," ,
                    "on_error" , ":" , response_error_spec ,
                  "}" ;

response_success_spec = "{" ,
                          "includes" , ":" , "[" , identifier_list , "]" ,
                          [ "," , "guarantees" , ":" , "[" , identifier_list , "]" ] ,
                        "}" ;

response_error_spec = "{" ,
                        "includes" , ":" , "[" , identifier_list , "]" ,
                        [ "," , "field_errors" , ":" , field_errors_config ] ,
                        [ "," , "categories" , ":" , "[" , error_category_list , "]" ] ,
                      "}" ;

field_errors_config = "{" ,
                        "enabled" , ":" , boolean ,
                        [ "," , "format" , ":" , field_error_format ] ,
                      "}" ;
field_error_format = "map" | "list" ;

error_category_list = error_category , { "," , error_category } ;
error_category = "validation" | "authorization" | "conflict" | "unavailable" | "internal" ;
```

## 5.2 SMS (Flow) Grammar

### 5.2.1 Flow Declaration

```ebnf
(* Flow item body — used within smsFlows: collection *)
flow_body     = "name" , ":" , identifier , "," ,
                [ "purpose" , ":" , string , "," ] ,
                "triggeredBy" , ":" , trigger_spec , "," ,
                "steps" , ":" , "[" , step_list , "]" , "," ,
                [ "authorization" , ":" , auth_spec , "," ] ,
                [ "onPolicyChange" , ":" , policy_change_action , "," ] ,
                [ "failure" , ":" , failure_spec ] ;

trigger_spec  = "{" , "inputIntent" , ":" , identifier , "}" ;

step_list     = step_def , { "," , step_def } ;
step_def      = simple_step | atomic_group | parallel_group | conditional_step ;

simple_step   = "{" , "work_unit" , ":" , identifier , "," , [ "boundary" , ":" , identifier ] , "}"
              | identifier , ":" , expression ;

auth_spec     = "{" , "policySet" , ":" , identifier , "}" ;
policy_change_action = "revalidate" | "drain" | "fail" ;
```

### 5.2.2 Atomic Group

```ebnf
atomic_group  = "{" ,
                  "atomic" , identifier , ":" , "{" ,
                    "boundary" , ":" , identifier , "," ,
                    "steps" , ":" , "[" , atomic_step_list , "]" ,
                  "}" ,
                "}" ;

atomic_step_list = atomic_step , { "," , atomic_step } ;
atomic_step   = "{" , "work_unit" , ":" , identifier , "}" ;
```

### 5.2.3 Failure & Compensation

```ebnf
failure_spec  = "{" ,
                  "policy" , ":" , failure_policy , "," ,
                  [ "max_attempts" , ":" , integer , "," ] ,
                  [ "compensations" , ":" , "[" , compensation_list , "]" ] ,
                "}" ;

failure_policy = "abort" | "retry" | "compensate" | "continue" ;

compensation_list = compensation , { "," , compensation } ;
compensation  = "{" ,
                  "on" , ":" , identifier , "," ,
                  "do" , ":" , identifier ,
                "}" ;
```

## 5.3 Work Unit Contract Grammar

```ebnf
(* WorkUnitContract item body — used within workUnitContracts: collection *)
work_unit_contract_body = "name" , ":" , identifier , "," ,
                          "version" , ":" , version , "," ,
                          [ "description" , ":" , string , "," ] ,
                          "input" , ":" , wuc_input_spec , "," ,
                          "output" , ":" , wuc_output_spec , "," ,
                          [ "sideEffects" , ":" , "[" , side_effect_list , "]" , "," ] ,
                          [ "preconditions" , ":" , "[" , expression_list , "]" , "," ] ,
                          [ "postconditions" , ":" , "[" , expression_list , "]" , "," ] ,
                          [ "errors" , ":" , "[" , error_def_list , "]" , "," ] ,
                          [ "dependencies" , ":" , dependency_spec , "," ] ,
                          [ "timeout" , ":" , duration , "," ] ,
                          [ "idempotency" , ":" , wuc_idempotency_spec ] ;

wuc_input_spec = "{" ,
                   "schema" , ":" , inline_schema , "," ,
                   [ "from" , ":" , identifier , "," ] ,
                   [ "validation" , ":" , wuc_validation_spec ] ,
                 "}" ;

inline_schema = "{" , field_list , "}" ;

wuc_validation_spec = "{" ,
                        "timing" , ":" , validation_timing , "," ,
                        [ "on_failure" , ":" , validation_failure_action ] ,
                      "}" ;
validation_timing = "immediate" | "deferred" | "lazy" ;
validation_failure_action = "reject" | "warn" | "log" ;

wuc_output_spec = "{" ,
                    "schema" , ":" , inline_schema , "," ,
                    [ "to" , ":" , identifier , "," ] ,
                    [ "cardinality" , ":" , output_cardinality ] ,
                  "}" ;
output_cardinality = "one" | "zero_or_one" | "many" | "stream" ;

side_effect_list = side_effect , { "," , side_effect } ;
side_effect = "{" ,
                "type" , ":" , side_effect_type , "," ,
                "target" , ":" , string , "," ,
                [ "description" , ":" , string , "," ] ,
                [ "idempotency" , ":" , string , "," ] ,
                [ "reversible" , ":" , boolean ] ,
              "}" ;
side_effect_type = "database_write" | "database_read" | "event_publish" | 
                   "event_consume" | "api_call" | "cache_write" | "cache_read" | 
                   "cache_invalidate" | "file_write" | "file_read" | 
                   "message_send" | "notification" | "audit_log" ;

error_def_list = error_def , { "," , error_def } ;
error_def = "{" ,
              "code" , ":" , identifier , "," ,
              "retryable" , ":" , boolean , "," ,
              [ "message" , ":" , string , "," ] ,
              [ "httpStatus" , ":" , integer , "," ] ,
              [ "backoff" , ":" , backoff_spec , "," ] ,
              [ "maxRetries" , ":" , integer , "," ] ,
              [ "category" , ":" , error_category ] ,
            "}" ;

backoff_spec = backoff_type | "{" ,
                                "strategy" , ":" , backoff_type , "," ,
                                [ "initial" , ":" , duration , "," ] ,
                                [ "multiplier" , ":" , number , "," ] ,
                                [ "max" , ":" , duration ] ,
                              "}" ;
backoff_type = "none" | "linear" | "exponential" | "fixed" | "jittered" ;
backoff_strategy = backoff_type ;

dependency_spec = "{" ,
                    [ "databases" , ":" , "[" , dependency_ref_list , "]" , "," ] ,
                    [ "services" , ":" , "[" , dependency_ref_list , "]" , "," ] ,
                    [ "caches" , ":" , "[" , dependency_ref_list , "]" , "," ] ,
                    [ "queues" , ":" , "[" , dependency_ref_list , "]" , "," ] ,
                    [ "externalApis" , ":" , "[" , dependency_ref_list , "]" ] ,
                  "}" ;

dependency_ref_list = dependency_ref , { "," , dependency_ref } ;
dependency_ref = identifier | "{" ,
                               "name" , ":" , identifier , "," ,
                               [ "required" , ":" , boolean , "," ] ,
                               [ "timeout" , ":" , duration , "," ] ,
                               [ "fallback" , ":" , fallback_behavior ] ,
                             "}" ;
fallback_behavior = "error" | "default" | "cache" | "skip" ;

wuc_idempotency_spec = "{" ,
                         "strategy" , ":" , idempotency_strategy , "," ,
                         [ "key" , ":" , key_expr , "," ] ,
                         [ "scope" , ":" , idempotency_scope , "," ] ,
                         [ "ttl" , ":" , duration ] ,
                       "}" ;
idempotency_strategy = "deterministic_key" | "timestamp" | "content_hash" ;

contract_extension = "extends" , ":" , "[" , contract_ref_list , "]" ;
contract_ref_list = contract_ref , { "," , contract_ref } ;
contract_ref = identifier , [ version ] ;
```

## 5.4 Workers Grammar

### 5.4.1 Topology Declaration

```ebnf
workers_decl  = "topology" , ":" , "{" , topology_body , "}" 
              | "workers" , ":" , "{" , workers_body , "}" ;

topology_body = { topology_element } ;
topology_element = location_decl | worker_class_decl | scheduler_decl | 
                   placement_decl | scaling_policy_decl | failure_domain_decl ;

workers_body  = "workers" , ":" , "[" , worker_list , "]" , "," ,
                [ "routing" , ":" , routing_spec ] ;
```

### 5.4.2 Location

```ebnf
location_decl = "location" , identifier , ":" , "{" , location_body , "}" ;

location_body = "scope" , ":" , scope_type , "," ,
                [ "capabilities" , ":" , "[" , identifier_list , "]" , "," ] ,
                [ "latency_to_user" , ":" , latency_class , "," ] ,
                [ "provisioner" , ":" , identifier , "," ] ,
                [ "provisionable" , ":" , boolean ] ;

latency_class = "minimal" | "low" | "medium" | "high" ;
```

### 5.4.3 Worker Class

```ebnf
worker_class_decl = "worker_class" , identifier , ":" , "{" , worker_class_body , "}" ;

worker_class_body = "runtime" , ":" , identifier , "," ,
                    "supports" , ":" , "[" , identifier_list , "]" , "," ,
                    [ "transport" , ":" , "[" , transport_list , "]" , "," ] ,
                    [ "resources" , ":" , resources_spec , "," ] ,
                    [ "device" , ":" , device_spec ] ;

transport_list = transport_type , { "," , transport_type } ;
transport_type = "nats" | "http" | "grpc" | "unix_socket" | "in_process" | "websocket" ;

resources_spec = "{" , { resource_property } , "}" ;
resource_property = identifier , ":" , ( number | string ) , [ "," ] ;
```

### 5.4.4 Placement, Scaling, Worker, Routing

```ebnf
placement_decl = "placement" , identifier , ":" , "{" ,
                   "worker_class" , ":" , identifier , "," ,
                   "locations" , ":" , "[" , identifier_list , "]" , "," ,
                   "min" , ":" , integer , "," ,
                   "max" , ":" , integer , "," ,
                   [ "preferred" , ":" , identifier ] ,
                 "}" ;

scaling_policy_decl = "scaling_policy" , identifier , ":" , "{" ,
                        "target" , ":" , identifier , "," ,
                        "signals" , ":" , "{" , signal_rules , "}" , "," ,
                        [ "cooldown" , ":" , duration ] ,
                      "}" ;

signal_rules  = signal_rule , { "," , signal_rule } ;
signal_rule   = expression , "=>" , scale_action , [ "," ] ;
scale_action  = "scale_up" | "scale_down" | "hold" ;

(* Worker item body — used within workers: collection *)
worker_body   = "name" , ":" , identifier , "," ,
                "acceptsInputs" , ":" , "[" , input_refs , "]" , "," ,
                "produces" , ":" , "[" , identifier_list , "]" , "," ,
                "compatibleVersions" , ":" , "[" , version_list , "]" ;

input_refs    = input_ref , { "," , input_ref } ;
input_ref     = identifier , [ version ] ;
version_list  = version , { "," , version } ;

routing_spec  = "{" ,
                  "strategy" , ":" , routing_strategy , "," ,
                  "rules" , ":" , "[" , routing_rules , "]" ,
                "}" ;
routing_strategy = "versioned" | "capability" | "load" ;

routing_rules = routing_rule , { "," , routing_rule } ;
routing_rule  = "{" , "if" , ":" , expression , "," , "routeTo" , ":" , identifier , "}"
              | "{" , "else" , ":" , "{" , "routeTo" , ":" , identifier , "}" , "}" ;

scheduler_decl = "scheduler" , identifier , ":" , "{" ,
                   "scope" , ":" , scope_type , "," ,
                   "authority" , ":" , boolean , "," ,
                   "observes" , ":" , "[" , identifier_list , "]" ,
                 "}" ;

failure_domain_decl = "failure_domain" , identifier , ":" , "{" ,
                        "scope" , ":" , scope_type , "," ,
                        "affected_locations" , ":" , "[" , identifier_list , "]" ,
                      "}" ;
```

## 5.5 Policy & Authorization Grammar

```ebnf
(* Policy item body — used within policies: collection *)
(* Each item in the collection can be a policy, policySet, dataPolicy, or policyArtifact *)
policy_body   = policy_def_body | policy_set_body | data_policy_body | policy_artifact_body ;

policy_def_body = "name" , ":" , identifier , "," ,
                  "version" , ":" , version , "," ,
                  "appliesTo" , ":" , applies_to_spec , "," ,
                  "effect" , ":" , effect_type , "," ,
                  "when" , ":" , expression ;

applies_to_spec = "{" ,
                    "type" , ":" , policy_target_type , "," ,
                    "name" , ":" , identifier ,
                  "}" ;
policy_target_type = "inputIntent" | "dataState" | "transition" | 
                     "materialization" | "presentationView" | "smsFlow" ;

effect_type   = "allow" | "deny" ;

policy_set_def = "policySet" , ":" , "{" ,
                   "name" , ":" , identifier , "," ,
                   "resolution" , ":" , resolution_strategy , "," ,
                   "policies" , ":" , "[" , policy_refs , "]" ,
                 "}" ;
resolution_strategy = "deny_overrides" | "allow_overrides" ;

policy_refs   = policy_ref , { "," , policy_ref } ;
policy_ref    = identifier , [ version ] ;

data_policy_def = "dataPolicy" , ":" , "{" ,
                    "name" , ":" , identifier , "," ,
                    "appliesTo" , ":" , data_target_spec , "," ,
                    "permissions" , ":" , permissions_spec ,
                  "}" ;

data_target_spec = "{" ,
                     "dataState" , ":" , identifier , "," ,
                     "version" , ":" , version ,
                   "}" ;

permissions_spec = "{" ,
                     "read" , ":" , expression , "," ,
                     "write" , ":" , expression , "," ,
                     "transition" , ":" , expression ,
                   "}" ;

policy_artifact_def = "policyArtifact" , ":" , "{" ,
                        "policy" , ":" , policy_content , "," ,
                        "lifecycle" , ":" , lifecycle_spec , "," ,
                        "scope" , ":" , scope_spec ,
                      "}" ;

policy_content = "{" ,
                   "name" , ":" , identifier , "," ,
                   "version" , ":" , version , "," ,
                   "definition" , ":" , ( policy_def | policy_set_def | data_policy_def ) ,
                 "}" ;

lifecycle_spec = "{" ,
                   "state" , ":" , lifecycle_state , "," ,
                   "supersedes" , ":" , ( version | "null" ) ,
                 "}" ;
lifecycle_state = "shadow" | "enforce" | "deprecated" | "revoked" ;

scope_spec    = "{" ,
                  "domain" , ":" , identifier , "," ,
                  "component" , ":" , component_ref ,
                "}" ;
component_ref = identifier | ( identifier , "." , identifier ) ;

subject_decl  = "subject" , ":" , "{" ,
                  "name" , ":" , identifier , "," ,
                  "attributes" , ":" , "{" , attribute_map , "}" ,
                "}" ;

role_decl     = "role" , ":" , "{" ,
                  "name" , ":" , identifier , "," ,
                  "grants" , ":" , "[" , identifier_list , "]" ,
                "}" ;

attribute_def = "attribute" , ":" , "{" ,
                  "name" , ":" , identifier , "," ,
                  "source" , ":" , attribute_source ,
                "}" ;
attribute_source = "subject" | "data" | "environment" | "runtime" ;

attribute_map = attribute_entry , { "," , attribute_entry } ;
attribute_entry = identifier , ":" , type_ref ;
```

## 5.6 Signals Grammar

```ebnf
(* Signal item body — used within signals: collection *)
signal_body   = "type" , ":" , signal_type , "," ,
                "source" , ":" , signal_source , "," ,
                [ "subject" , ":" , signal_subject , "," ] ,
                "metrics" , ":" , signal_metrics , "," ,
                "severity" , ":" , signal_severity , "," ,
                "timestamp" , ":" , string ;

(* Extended in v1.6 to include 'cache' type *)
signal_type   = "capacity" | "health" | "latency" | "saturation" | 
                "policy" | "backpressure" | "error" | "cache" ;

(* Extended in v1.6 to include 'cache' component and 'tier' field *)
signal_source = "{" ,
                  "component" , ":" , component_type , "," ,
                  "name" , ":" , identifier ,
                  [ "," , "region" , ":" , identifier ] ,
                  [ "," , "tier" , ":" , cache_tier ] ,  (* NEW in v1.6 *)
                "}" ;

component_type = "worker" | "sms" | "ui" | "infra" | "cache" ;  (* 'cache' NEW in v1.6 *)

(* Extended in v1.6 to include 'cache' reference *)
signal_subject = "{" ,
                   [ "intent" , ":" , identifier , "," ] ,
                   [ "dataState" , ":" , identifier , "," ] ,
                   [ "cache" , ":" , identifier , "," ] ,  (* NEW in v1.6 *)
                 "}" ;

signal_metrics = "{" , metric_def , { "," , metric_def } , "}" ;
metric_def    = identifier , ":" , ( number | string | duration ) ;

signal_severity = "info" | "warn" | "error" | "critical" ;

(* Cache-specific metrics for cache signals - v1.6 *)
cache_signal_metrics = "{" ,
                         [ "hit_rate" , ":" , float , "," ] ,
                         [ "miss_rate" , ":" , float , "," ] ,
                         [ "hit_count" , ":" , integer , "," ] ,
                         [ "miss_count" , ":" , integer , "," ] ,
                         [ "latency_p50" , ":" , duration , "," ] ,
                         [ "latency_p95" , ":" , duration , "," ] ,
                         [ "latency_p99" , ":" , duration , "," ] ,
                         [ "latency_avg" , ":" , duration , "," ] ,
                         [ "size" , ":" , integer , "," ] ,
                         [ "size_bytes" , ":" , integer , "," ] ,
                         [ "max_entries" , ":" , integer , "," ] ,
                         [ "max_bytes" , ":" , integer , "," ] ,
                         [ "evictions" , ":" , integer , "," ] ,
                         [ "eviction_rate" , ":" , float , "," ] ,
                         [ "pressure" , ":" , pressure_level , "," ] ,
                       "}" ;
```

## 5.7 Cross-Cutting Grammar

```ebnf
execution_context = "executionContext" , ":" , "{" ,
                      "trace_id" , ":" , string , "," ,
                      "causation_id" , ":" , string , "," ,
                      "actor" , ":" , actor_spec , "," ,
                      [ "metadata" , ":" , "{" , metadata_map , "}" ] ,
                    "}" ;

actor_spec    = "{" ,
                  "subject" , ":" , identifier , "," ,
                  "roles" , ":" , "[" , identifier_list , "]" ,
                "}" ;

metadata_map  = metadata_entry , { "," , metadata_entry } ;
metadata_entry = identifier , ":" , string ;

time_constraint = "timeConstraint" , ":" , "{" ,
                    "type" , ":" , time_type , "," ,
                    "value" , ":" , duration ,
                  "}" ;
time_type     = "ttl" | "deadline" | "staleness" ;

idempotency_config = "idempotency" , ":" , "{" ,
                       "strategy" , ":" , idempotency_strategy , "," ,
                       "key" , ":" , key_expression , "," ,
                       "scope" , ":" , idempotency_scope ,
                     "}" ;
key_expression = expression ;

execution_outcome = "executionOutcome" , ":" , "{" ,
                      "type" , ":" , outcome_type , "," ,
                      "retryable" , ":" , boolean , "," ,
                      "reason" , ":" , string ,
                    "}" ;
outcome_type  = "success" | "rejection" | "failure" ;

(* Realm item body — used within realms: collection *)
realm_body    = "name" , ":" , identifier , "," ,
                "isolation" , ":" , isolation_spec ;

isolation_spec = "{" ,
                   "data" , ":" , isolation_level , "," ,
                   "policy" , ":" , isolation_level , "," ,
                   "signals" , ":" , signal_isolation ,
                 "}" ;
isolation_level = "strict" | "shared" ;
signal_isolation = "shared" | "isolated" ;
```

---

# 6. v1.5 EXTENSIONS

## 6.1 Data Governance

### 6.1.1 Governance Specification

```ebnf
governance_spec = "{" ,
                    [ "classification" , ":" , data_classification , "," ] ,
                    [ "retention" , ":" , retention_policy_spec , "," ] ,
                    [ "residency" , ":" , residency_spec , "," ] ,
                    [ "audit" , ":" , audit_spec , "," ] ,
                    [ "encryption" , ":" , encryption_spec ] ,
                  "}" ;

data_classification = "pii" | "phi" | "pci" | "financial" | "public" | "internal" | "confidential" ;

retention_policy_spec = "{" ,
                          "policy" , ":" , retention_policy_type , "," ,
                          [ "duration" , ":" , duration , "," ] ,
                          [ "after_event" , ":" , identifier , "," ] ,
                          [ "grace_period" , ":" , duration , "," ] ,
                          [ "archive" , ":" , archive_spec , "," ] ,
                          [ "on_expiry" , ":" , expiry_action ] ,
                        "}" ;

retention_policy_type = "time_based" | "event_based" | "indefinite" | "legal_hold" ;
expiry_action = "delete" | "anonymize" | "archive" ;

archive_spec = "{" ,
                 "enabled" , ":" , boolean , "," ,
                 "after" , ":" , duration , "," ,
                 "tier" , ":" , storage_tier ,
               "}" ;
storage_tier = "warm" | "cold" | "glacier" ;

residency_spec = "{" ,
                   [ "allowed_regions" , ":" , "[" , identifier_list , "]" , "," ] ,
                   [ "forbidden_regions" , ":" , "[" , identifier_list , "]" , "," ] ,
                   [ "primary_region" , ":" , expression , "," ] ,
                   [ "transfer" , ":" , transfer_spec ] ,
                 "}" ;

transfer_spec = "{" ,
                  "allowed" , ":" , boolean , "," ,
                  [ "requires_consent" , ":" , boolean , "," ] ,
                  [ "mechanisms" , ":" , "[" , transfer_mechanism_list , "]" ] ,
                "}" ;

transfer_mechanism_list = transfer_mechanism , { "," , transfer_mechanism } ;
transfer_mechanism = "standard_clauses" | "adequacy" | "binding_rules" ;

audit_spec = "{" ,
               "access_logging" , ":" , boolean , "," ,
               "modification_logging" , ":" , boolean , "," ,
               [ "retention_for_logs" , ":" , duration , "," ] ,
               [ "immutable" , ":" , boolean ] ,
             "}" ;

encryption_spec = "{" ,
                    "at_rest" , ":" , encryption_level , "," ,
                    "in_transit" , ":" , encryption_level , "," ,
                    [ "key_rotation" , ":" , duration ] ,
                  "}" ;
encryption_level = "required" | "optional" | "none" ;
```

### 6.1.2 Consent Management

```ebnf
(* ConsentRecord item body — used within consentRecords: collection *)
consent_record_body = "name" , ":" , identifier , "," ,
                      "version" , ":" , version , "," ,
                      [ "description" , ":" , string , "," ] ,
                      "subject_field" , ":" , identifier , "," ,
                      "scopes" , ":" , "[" , consent_scope_list , "]" , "," ,
                      [ "purposes" , ":" , "[" , purpose_list , "]" , "," ] ,
                      [ "lifecycle" , ":" , consent_lifecycle , "," ] ,
                      [ "withdrawal" , ":" , consent_withdrawal ] ;

consent_scope_list = consent_scope , { "," , consent_scope } ;
consent_scope = "{" ,
                  "name" , ":" , identifier , "," ,
                  [ "description" , ":" , string , "," ] ,
                  [ "required" , ":" , boolean , "," ] ,
                  [ "default" , ":" , boolean , "," ] ,
                  [ "revocable" , ":" , boolean ] ,
                "}" ;

purpose_list = purpose_ref , { "," , purpose_ref } ;
purpose_ref = identifier ;

consent_lifecycle = "{" ,
                      "type" , ":" , consent_lifetime_type , "," ,
                      [ "duration" , ":" , duration , "," ] ,
                      [ "renewable" , ":" , boolean ] ,
                    "}" ;
consent_lifetime_type = "perpetual" | "temporary" | "session" ;

consent_withdrawal = "{" ,
                       "allowed" , ":" , boolean , "," ,
                       [ "grace_period" , ":" , duration , "," ] ,
                       [ "effect" , ":" , withdrawal_effect ] ,
                     "}" ;
withdrawal_effect = "immediate" | "end_of_session" | "scheduled" ;
```

### 6.1.3 Erasure Requests

```ebnf
(* ErasureRequest item body — used within erasureRequests: collection *)
erasure_request_body = "name" , ":" , identifier , "," ,
                       "version" , ":" , version , "," ,
                       [ "description" , ":" , string , "," ] ,
                       "scope" , ":" , erasure_scope , "," ,
                       "strategy" , ":" , erasure_strategy , "," ,
                       [ "cascade" , ":" , erasure_cascade , "," ] ,
                       [ "verification" , ":" , erasure_verification , "," ] ,
                       [ "processing" , ":" , erasure_processing ] ;

erasure_scope = "{" , "subject_field" , ":" , identifier , "}" ;
erasure_strategy = "delete" | "anonymize" | "hybrid" ;

erasure_cascade = "{" ,
                    "behavior" , ":" , cascade_behavior , "," ,
                    "entities" , ":" , "[" , identifier_list , "]" , "," ,
                    "relationship_handling" , ":" , relationship_handling ,
                  "}" ;
cascade_behavior = "automatic" | "manual_review" | "refuse" ;
relationship_handling = "delete" | "orphan" | "reassign" ;

erasure_verification = "{" ,
                         "required" , ":" , boolean , "," ,
                         "methods" , ":" , "[" , verification_method_list , "]" , "," ,
                         "timeout" , ":" , duration ,
                       "}" ;

verification_method_list = verification_method , { "," , verification_method } ;
verification_method = "email_confirmation" | "identity_verification" | "mfa" ;

erasure_processing = "{" ,
                       "sla" , ":" , duration , "," ,
                       [ "notification" , ":" , erasure_notification ] ,
                     "}" ;

erasure_notification = "{" ,
                         "on_complete" , ":" , boolean , "," ,
                         "on_partial" , ":" , boolean ,
                       "}" ;
```

### 6.1.4 Legal Hold

```ebnf
(* LegalHold item body — used within legalHolds: collection *)
legal_hold_body = "name" , ":" , identifier , "," ,
                  "version" , ":" , version , "," ,
                  "scope" , ":" , legal_hold_scope , "," ,
                  "reason" , ":" , string , "," ,
                  "reference" , ":" , string , "," ,
                  "custodian" , ":" , identifier , "," ,
                  "preserves" , ":" , "[" , preserve_spec_list , "]" , "," ,
                  "overrides" , ":" , legal_hold_overrides , "," ,
                  [ "lifecycle" , ":" , legal_hold_lifecycle ] ;

legal_hold_scope = "{" ,
                     "entities" , ":" , "[" , identifier_list , "]" , "," ,
                     "filter" , ":" , expression ,
                   "}" ;

preserve_spec_list = preserve_spec , { "," , preserve_spec } ;
preserve_spec = "{" ,
                  "dataState" , ":" , identifier , "," ,
                  "fields" , ":" , preserve_fields ,
                "}" ;
preserve_fields = "all" | "[" , identifier_list , "]" ;

legal_hold_overrides = "{" ,
                         "retention" , ":" , boolean , "," ,
                         "erasure" , ":" , boolean ,
                       "}" ;

legal_hold_lifecycle = "{" ,
                         "start_date" , ":" , string , "," ,
                         "review_interval" , ":" , duration ,
                       "}" ;
```

## 6.2 Session & Offline Support

### 6.2.1 Session Contract

```ebnf
session_spec  = "{" ,
                  "required" , ":" , boolean , "," ,
                  "subject_binding" , ":" , subject_binding_spec , "," ,
                  "contains" , ":" , session_contains_spec , "," ,
                  "lifetime" , ":" , session_lifetime_spec , "," ,
                  "on_expired" , ":" , expired_action_spec , "," ,
                  "on_invalid" , ":" , invalid_action_spec ,
                "}" ;

subject_binding_spec = "{" , "mode" , ":" , binding_mode , "}" ;
binding_mode  = "authenticated" | "anonymous" | "either" ;

session_contains_spec = "{" ,
                          "required" , ":" , "[" , identifier_list , "]" ,
                          [ "," , "optional" , ":" , "[" , identifier_list , "]" ] ,
                        "}" ;

session_lifetime_spec = "{" ,
                          "mode" , ":" , lifetime_mode ,
                          [ "," , "idle_timeout" , ":" , duration ] ,
                          [ "," , "max_duration" , ":" , duration ] ,
                        "}" ;
lifetime_mode = "bounded" | "sliding" | "permanent" ;

expired_action_spec = "{" ,
                        "action" , ":" , expired_action ,
                        [ "," , "preserve_location" , ":" , boolean ] ,
                      "}" ;
expired_action = "redirect_to_auth" | "prompt_reauth" | "degrade_to_anonymous" ;

invalid_action_spec = "{" , "action" , ":" , invalid_action , "}" ;
invalid_action = "redirect_to_auth" | "clear_and_retry" | "error" ;
```

### 6.2.2 Multi-Device Semantics

```ebnf
devices_spec  = "{" ,
                  "enabled" , ":" , boolean ,
                  [ "," , "max_active" , ":" , integer ] ,
                  [ "," , "identification" , ":" , device_identification ] ,
                  [ "," , "state_scope" , ":" , device_state_scope ] ,
                  [ "," , "handoff" , ":" , device_handoff ] ,
                  [ "," , "concurrent_limits" , ":" , concurrent_limits ] ,
                "}" ;

device_identification = "{" , "by" , ":" , identification_method , "}" ;
identification_method = "device_id" | "fingerprint" | "user_agent" ;

device_state_scope = "{" ,
                       [ "per_device" , ":" , "[" , field_path_list , "]" , "," ] ,
                       [ "shared" , ":" , "[" , field_path_list , "]" ] ,
                     "}" ;

field_path_list = field_path , { "," , field_path } ;
field_path    = identifier , { "." , identifier } ;

device_handoff = "{" ,
                   "enabled" , ":" , boolean ,
                   [ "," , "state_transfer" , ":" , transfer_mode ] ,
                   [ "," , "transferable" , ":" , "[" , field_path_list , "]" ] ,
                 "}" ;
transfer_mode = "immediate" | "on_request" | "manual" ;

concurrent_limits = "{" , "on_new_device" , ":" , new_device_action , "}" ;
new_device_action = "allow" | "logout_oldest" | "prompt" | "reject" ;
```

### 6.2.3 Offline Operation Semantics

```ebnf
offline_spec  = "{" ,
                  "enabled" , ":" , boolean ,
                  [ "," , "capabilities" , ":" , offline_capabilities ] ,
                  [ "," , "sync" , ":" , offline_sync ] ,
                  [ "," , "queue" , ":" , offline_queue ] ,
                  [ "," , "cache" , ":" , offline_cache ] ,
                "}" ;

offline_capabilities = "{" ,
                         [ "read" , ":" , boolean , "," ] ,
                         [ "write" , ":" , write_capability ] ,
                       "}" ;
write_capability = "queue" | "reject" ;

offline_sync  = "{" ,
                  [ "on_reconnect" , ":" , reconnect_strategy , "," ] ,
                  [ "conflict_resolution" , ":" , conflict_strategy , "," ] ,
                  [ "field_strategies" , ":" , "[" , field_strategy_list , "]" ] ,
                "}" ;

reconnect_strategy = "immediate" | "background" | "manual" ;
conflict_strategy = "client_wins" | "server_wins" | "merge" | "prompt" ;

field_strategy_list = field_strategy , { "," , field_strategy } ;
field_strategy = "{" ,
                   "field" , ":" , field_path , "," ,
                   "strategy" , ":" , field_conflict_strategy ,
                 "}" ;
field_conflict_strategy = "client_wins" | "server_wins" | "merge" | "union" | "max" | "min" ;

offline_queue = "{" ,
                  [ "max_size" , ":" , integer , "," ] ,
                  [ "max_age" , ":" , duration , "," ] ,
                  [ "on_overflow" , ":" , overflow_action ] ,
                "}" ;
overflow_action = "drop_oldest" | "reject_new" ;

offline_cache = "{" ,
                  [ "views" , ":" , "[" , view_ref_list , "]" , "," ] ,
                  [ "max_size" , ":" , bytes , "," ] ,
                  [ "ttl" , ":" , duration ] ,
                "}" ;

view_ref_list = view_ref , { "," , view_ref } ;
```

## 6.3 Asset Semantics

```ebnf
asset_spec    = "asset" , ":" , "{" ,
                  "category" , ":" , asset_category , "," ,
                  [ "constraints" , ":" , asset_constraints , "," ] ,
                  [ "lifecycle" , ":" , asset_lifecycle , "," ] ,
                  [ "reference" , ":" , asset_reference , "," ] ,
                  [ "access" , ":" , asset_access , "," ] ,
                  [ "variants" , ":" , "[" , asset_variant_list , "]" ] ,
                "}" ;

asset_category = "image" | "document" | "media" | "archive" | "generic" ;

asset_constraints = "{" ,
                      [ "max_size" , ":" , size_expression , "," ] ,
                      [ "mime_types" , ":" , "[" , mime_type_list , "]" , "," ] ,
                      [ "extensions" , ":" , "[" , extension_list , "]" ] ,
                    "}" ;

mime_type_list = string , { "," , string } ;
extension_list = string , { "," , string } ;

asset_lifecycle = "{" ,
                    "type" , ":" , asset_lifecycle_type ,
                    [ "," , "expires_after" , ":" , duration ] ,
                  "}" ;
asset_lifecycle_type = "permanent" | "temporary" | "expiring" ;

asset_reference = "{" , "style" , ":" , reference_style , "}" ;
reference_style = "by_reference" | "embedded" ;

asset_access  = "{" ,
                  "read" , ":" , access_mode ,
                  [ "," , "policy" , ":" , identifier ] ,
                  [ "," , "signed_url_ttl" , ":" , duration ] ,
                "}" ;
access_mode   = "public" | "signed" | "authenticated" | "policy" ;

asset_variant_list = asset_variant , { "," , asset_variant } ;
asset_variant = "{" ,
                  "name" , ":" , identifier , "," ,
                  "transform" , ":" , transform_type ,
                  [ "," , "params" , ":" , "{" , transform_params , "}" ] ,
                "}" ;

transform_type = "resize" | "thumbnail" | "compress" | "convert" | "extract_page" ;
transform_params = transform_param , { "," , transform_param } ;
transform_param = identifier , ":" , value ;
value         = string | number | boolean ;
```

## 6.4 Search Semantics

### 6.4.1 Search Field Specification

```ebnf
search_spec   = "search" , ":" , "{" ,
                  "indexed" , ":" , boolean , "," ,
                  "strategy" , ":" , search_strategy ,
                  [ "," , "full_text" , ":" , full_text_config ] ,
                  [ "," , "fuzzy" , ":" , fuzzy_config ] ,
                  [ "," , "weight" , ":" , decimal ] ,
                  [ "," , "boost_recent" , ":" , boolean ] ,
                  [ "," , "suggest" , ":" , suggest_config ] ,
                  [ "," , "facet" , ":" , facet_config ] ,
                  [ "," , "highlight" , ":" , highlight_config ] ,
                "}" ;

search_strategy = "full_text" | "exact" | "prefix" | "fuzzy" | "semantic" ;

full_text_config = "{" ,
                     "analyzer" , ":" , analyzer_type ,
                     [ "," , "language" , ":" , string ] ,
                     [ "," , "stemming" , ":" , boolean ] ,
                     [ "," , "stop_words" , ":" , boolean ] ,
                     [ "," , "synonyms" , ":" , identifier ] ,
                   "}" ;
analyzer_type = "standard" | "language" | "custom" ;

fuzzy_config  = "{" ,
                  "max_edits" , ":" , integer , "," ,
                  "prefix_length" , ":" , integer ,
                "}" ;

suggest_config = "{" ,
                   "enabled" , ":" , boolean ,
                   [ "," , "type" , ":" , suggest_type ] ,
                 "}" ;
suggest_type  = "completion" | "phrase" | "term" ;

facet_config  = "{" ,
                  "enabled" , ":" , boolean ,
                  [ "," , "type" , ":" , facet_type ] ,
                  [ "," , "ranges" , ":" , "[" , range_list , "]" ] ,
                "}" ;
facet_type    = "terms" | "range" | "date_histogram" | "nested" ;

range_list    = range_def , { "," , range_def } ;
range_def     = "{" , "from" , ":" , value , "," , "to" , ":" , value , "}" ;

highlight_config = "{" ,
                     "enabled" , ":" , boolean ,
                     [ "," , "fragment_size" , ":" , integer ] ,
                     [ "," , "pre_tag" , ":" , string ] ,
                     [ "," , "post_tag" , ":" , string ] ,
                   "}" ;
```

### 6.4.2 Search Intent Declaration

```ebnf
(* SearchIntent item body — used within searchIntents: collection *)
search_intent_body = "name" , ":" , identifier , "," ,
                     "version" , ":" , version , "," ,
                     "searches" , ":" , identifier , "," ,
                     "query" , ":" , search_query_spec , "," ,
                     [ "filters" , ":" , search_filter_spec , "," ] ,
                     [ "facets" , ":" , search_facet_spec , "," ] ,
                     [ "pagination" , ":" , search_pagination_spec , "," ] ,
                     [ "sorting" , ":" , search_sorting_spec ] ;

search_query_spec = "{" ,
                      "fields" , ":" , "[" , identifier_list , "]" , "," ,
                      "default_operator" , ":" , query_operator ,
                      [ "," , "phrase_slop" , ":" , integer ] ,
                    "}" ;
query_operator = "and" | "or" ;

search_filter_spec = "{" ,
                       "required" , ":" , "[" , identifier_list , "]" ,
                       [ "," , "optional" , ":" , "[" , identifier_list , "]" ] ,
                     "}" ;

search_facet_spec = "{" , "include" , ":" , "[" , identifier_list , "]" , "}" ;

search_pagination_spec = "{" ,
                           "default_size" , ":" , integer , "," ,
                           "max_size" , ":" , integer ,
                         "}" ;

search_sorting_spec = "{" ,
                        "default_field" , ":" , identifier ,
                        [ "," , "default_order" , ":" , order_direction ] ,
                        [ "," , "allowed_fields" , ":" , "[" , identifier_list , "]" ] ,
                      "}" ;
order_direction = "asc" | "desc" ;
```

### 6.4.3 Search Index Configuration

```ebnf
search_index_spec = "{" ,
                      "enabled" , ":" , boolean , "," ,
                      [ "document" , ":" , search_document_spec , "," ] ,
                      [ "freshness" , ":" , search_freshness_spec , "," ] ,
                      [ "rebuild" , ":" , search_rebuild_spec ] ,
                    "}" ;

search_document_spec = "{" ,
                         "root" , ":" , identifier ,
                         [ "," , "include_related" , ":" , "[" , related_entity_list , "]" ] ,
                       "}" ;

related_entity_list = related_entity_spec , { "," , related_entity_spec } ;
related_entity_spec = "{" ,
                        "entity" , ":" , identifier , "," ,
                        "relationship" , ":" , identifier , "," ,
                        "fields" , ":" , "[" , identifier_list , "]" ,
                      "}" ;

search_freshness_spec = "{" ,
                          "strategy" , ":" , search_freshness_strategy ,
                          [ "," , "max_lag" , ":" , duration ] ,
                          [ "," , "batch_interval" , ":" , duration ] ,
                        "}" ;
search_freshness_strategy = "real_time" | "near_real_time" | "batch" ;

search_rebuild_spec = "{" ,
                        "trigger" , ":" , rebuild_trigger , "," ,
                        "zero_downtime" , ":" , boolean ,
                      "}" ;
rebuild_trigger = "manual" | "schema_change" | "daily" ;
```

## 6.5 Materialization Extensions

### 6.5.1 Retrieval Contract

```ebnf
mat_retrieval_spec = "{" ,
                       "mode" , ":" , retrieval_mode , "," ,
                       [ "entity_key" , ":" , identifier , "," ] ,
                       [ "query_fields" , ":" , "[" , identifier_list , "]" , "," ] ,
                       [ "list" , ":" , list_config , "," ] ,
                       [ "freshness" , ":" , retrieval_freshness , "," ] ,
                       [ "on_unavailable" , ":" , unavailable_strategy ] ,
                     "}" ;

retrieval_mode = "by_entity" | "by_query" | "singleton" ;

list_config   = "{" ,
                  "max_items" , ":" , integer ,
                  [ "," , "default_order" , ":" , identifier ] ,
                  [ "," , "order_direction" , ":" , order_direction ] ,
                "}" ;

retrieval_freshness = "{" , "max_staleness" , ":" , duration , "}" ;

unavailable_strategy = "{" ,
                         "strategy" , ":" , unavail_action , "," ,
                         [ "cache_ttl" , ":" , duration , "," ] ,
                         [ "degrade_to" , ":" , identifier , "," ] ,
                         [ "wait_timeout" , ":" , duration ] ,
                       "}" ;
unavail_action = "use_cached" | "fail" | "degrade" | "wait" ;
```

### 6.5.2 Temporal Window Semantics

```ebnf
mat_temporal_spec = "{" ,
                      "window" , ":" , temporal_window , "," ,
                      [ "aggregation" , ":" , "[" , temporal_agg_list , "]" , "," ] ,
                      [ "timestamp_field" , ":" , identifier , "," ] ,
                      [ "watermark" , ":" , duration , "," ] ,
                      [ "overflow" , ":" , overflow_spec , "," ] ,
                      [ "emit" , ":" , emit_spec ] ,
                    "}" ;

temporal_window = "{" ,
                    "type" , ":" , window_type , "," ,
                    "size" , ":" , duration ,
                    [ "," , "slide" , ":" , duration ] ,
                    [ "," , "gap" , ":" , duration ] ,
                  "}" ;
window_type   = "tumbling" | "sliding" | "session" | "hopping" ;

temporal_agg_list = temporal_agg , { "," , temporal_agg } ;
temporal_agg  = "{" ,
                  "field" , ":" , identifier , "," ,
                  "function" , ":" , agg_function , "," ,
                  "as" , ":" , identifier ,
                "}" ;
agg_function  = "sum" | "avg" | "min" | "max" | "count" | "first" | "last" | "stddev" ;

overflow_spec = "{" ,
                  "strategy" , ":" , overflow_strategy ,
                  [ "," , "buffer_size" , ":" , integer ] ,
                  [ "," , "sample_rate" , ":" , decimal ] ,
                "}" ;
overflow_strategy = "drop_oldest" | "pause_producer" | "sample" | "buffer" ;

emit_spec     = "{" ,
                  "trigger" , ":" , emit_trigger ,
                  [ "," , "periodic_interval" , ":" , duration ] ,
                  [ "," , "include_partial" , ":" , boolean ] ,
                "}" ;
emit_trigger  = "on_window_close" | "on_each_event" | "periodic" ;
```

### 6.5.3 External Source for Materialization

```ebnf
external_source_spec = "{" ,
                         "dependency" , ":" , identifier , "," ,
                         "capability" , ":" , identifier , "," ,
                         "refresh" , ":" , external_refresh_spec ,
                         [ "," , "transform" , ":" , identifier ] ,
                       "}" ;

external_refresh_spec = "{" ,
                          "strategy" , ":" , refresh_strategy ,
                          [ "," , "interval" , ":" , duration ] ,
                          [ "," , "webhook" , ":" , identifier ] ,
                        "}" ;
refresh_strategy = "polling" | "webhook" | "manual" | "hybrid" ;
```

## 6.6 Inference Fields

```ebnf
inference_spec = "inference" , ":" , "{" ,
                   "enabled" , ":" , boolean , "," ,
                   "source" , ":" , inference_source , "," ,
                   "model" , ":" , inference_model , "," ,
                   "output" , ":" , inference_output , "," ,
                   "freshness" , ":" , inference_freshness , "," ,
                   "fallback" , ":" , inference_fallback ,
                 "}" ;

inference_source = "{" ,
                     "fields" , ":" , "[" , identifier_list , "]" ,
                   "}" ;

inference_model = "{" ,
                    "name" , ":" , identifier , "," ,
                    "version" , ":" , version ,
                  "}" ;

inference_output = "{" ,
                     "type" , ":" , type_ref ,
                     [ "," , "range" , ":" , "{" , "min" , ":" , number , "," , "max" , ":" , number , "}" ] ,
                   "}" ;

inference_freshness = "{" , "strategy" , ":" , inference_strategy , "}" ;
inference_strategy = "on_write" | "on_read" | "scheduled" | "manual" ;

inference_fallback = "{" , "on_unavailable" , ":" , fallback_action , "}" ;
fallback_action = "queue" | "fail" | "degrade" | "cache" ;
```

## 6.7 Structured Content

```ebnf
structured_content_spec = "structured_content" , ":" , "{" ,
                            "schema" , ":" , identifier , "," ,
                            "blocks" , ":" , content_blocks_spec , "," ,
                            "marks" , ":" , content_marks_spec ,
                            [ "," , "constraints" , ":" , content_constraints ] ,
                          "}" ;

content_blocks_spec = "{" , "allowed" , ":" , "[" , block_type_list , "]" , "}" ;

block_type_list = block_type , { "," , block_type } ;
block_type = "paragraph" | "heading" | "list" | "blockquote" | "table" | 
             "image" | "video" | "poll" | "code" | "divider" | "embed" ;

content_marks_spec = "{" , "allowed" , ":" , "[" , mark_type_list , "]" , "}" ;

mark_type_list = mark_type , { "," , mark_type } ;
mark_type = "bold" | "italic" | "underline" | "strikethrough" | "code" | 
            "link" | "mention" | "hashtag" | "highlight" ;

content_constraints = "{" , [ "max_length" , ":" , integer ] , "}" ;
```

## 6.8 Spatial Types

```ebnf
point_spec = "point" , ":" , "{" ,
               "coordinate_system" , ":" , coordinate_system , "," ,
               [ "precision" , ":" , integer ] ,
             "}" ;

coordinate_system = "wgs84" | "web_mercator" | "utm" | "custom" ;

spatial_query_spec = "{" ,
                       "indexed" , ":" , boolean , "," ,
                       [ "index_type" , ":" , spatial_index_type ] ,
                     "}" ;
spatial_index_type = "geohash" | "quadtree" | "rtree" ;
```

## 6.9 Graph Query Grammar

```ebnf
(* GraphQuery item body — used within graphQueries: collection *)
graph_query_body = "name" , ":" , identifier , "," ,
                   "version" , ":" , version , "," ,
                   [ "description" , ":" , string , "," ] ,
                   "traverses" , ":" , identifier , "," ,
                   "start" , ":" , graph_start_spec , "," ,
                   [ "pattern" , ":" , graph_pattern_spec , "," ] ,
                   [ "filters" , ":" , "[" , graph_filter_list , "]" , "," ] ,
                   [ "limits" , ":" , graph_limits_spec ] ;

graph_start_spec = "{" ,
                     "from" , ":" , identifier , "," ,
                     "where" , ":" , expression ,
                   "}" ;

graph_pattern_spec = "{" ,
                       "type" , ":" , graph_pattern_type , "," ,
                       [ "max_depth" , ":" , integer , "," ] ,
                       [ "direction" , ":" , traversal_direction ] ,
                     "}" ;
graph_pattern_type = "shortest_path" | "all_paths" | "reachable" | "neighbors" | "subgraph" ;
traversal_direction = "forward" | "backward" | "both" ;

graph_filter_list = graph_filter , { "," , graph_filter } ;
graph_filter = "{" ,
                 "level" , ":" , filter_level , "," ,
                 "condition" , ":" , expression ,
               "}" ;
filter_level = "node" | "edge" | "path" ;

graph_limits_spec = "{" ,
                      [ "max_results" , ":" , integer , "," ] ,
                      [ "max_depth" , ":" , integer , "," ] ,
                      [ "timeout" , ":" , duration ] ,
                    "}" ;
```

## 6.10 External Integration Grammar

### 6.10.1 External Dependency Declaration

```ebnf
(* ExternalDependency item body — used within externalDependencies: collection *)
external_dependency_body = "name" , ":" , identifier , "," ,
                           "version" , ":" , version , "," ,
                           "description" , ":" , string , "," ,
                           "type" , ":" , external_type , "," ,
                           "capabilities" , ":" , "[" , identifier_list , "]" , "," ,
                           "contract" , ":" , "{" , capability_contract_map , "}" , "," ,
                           [ "sla" , ":" , sla_spec , "," ] ,
                           [ "resilience" , ":" , resilience_spec , "," ] ,
                           [ "fallback" , ":" , fallback_spec , "," ] ,
                           [ "rate_limit" , ":" , rate_limit_spec , "," ] ,
                           [ "credentials" , ":" , credentials_spec , "," ] ,
                           [ "health_check" , ":" , health_check_spec ] ;

external_type = "api" | "data_feed" | "message_queue" | "storage" ;

capability_contract_map = capability_contract , { "," , capability_contract } ;
capability_contract = identifier , ":" , "{" ,
                        "input" , ":" , contract_schema , "," ,
                        "output" , ":" , contract_schema ,
                        [ "," , "errors" , ":" , "[" , error_spec_list , "]" ] ,
                      "}" ;

contract_schema = "{" , "schema" , ":" , "{" , schema_fields , "}" , "}" ;
schema_fields = schema_field , { "," , schema_field } ;
schema_field  = identifier , ":" , "{" , "type" , ":" , type_ref , "," , "required" , ":" , boolean , "}" ;

error_spec_list = error_spec , { "," , error_spec } ;
error_spec    = "{" , "code" , ":" , identifier , "," , "retryable" , ":" , boolean , "}" ;

sla_spec      = "{" ,
                  "availability" , ":" , decimal , "," ,
                  "latency_p99" , ":" , duration ,
                "}" ;

resilience_spec = "{" ,
                    "timeout" , ":" , duration ,
                    [ "," , "retry" , ":" , external_retry_spec ] ,
                    [ "," , "circuit_breaker" , ":" , circuit_breaker_spec ] ,
                    [ "," , "bulkhead" , ":" , bulkhead_spec ] ,
                  "}" ;

external_retry_spec = "{" ,
                        "enabled" , ":" , boolean , "," ,
                        "max_attempts" , ":" , integer , "," ,
                        "backoff" , ":" , backoff_config ,
                      "}" ;

backoff_config = "{" ,
                   "strategy" , ":" , backoff_strategy , "," ,
                   "initial" , ":" , duration , "," ,
                   "max" , ":" , duration ,
                   [ "," , "jitter" , ":" , boolean ] ,
                 "}" ;

circuit_breaker_spec = "{" ,
                         "enabled" , ":" , boolean , "," ,
                         "threshold" , ":" , integer , "," ,
                         "reset_after" , ":" , duration ,
                         [ "," , "half_open_requests" , ":" , integer ] ,
                       "}" ;

bulkhead_spec = "{" ,
                  "enabled" , ":" , boolean , "," ,
                  "max_concurrent" , ":" , integer ,
                  [ "," , "queue_size" , ":" , integer ] ,
                "}" ;

fallback_spec = "{" ,
                  "on_unavailable" , ":" , fallback_action ,
                  [ "," , "queue_ttl" , ":" , duration ] ,
                  [ "," , "cache_ttl" , ":" , duration ] ,
                  [ "," , "degrade_to" , ":" , identifier ] ,
                "}" ;

rate_limit_spec = "{" ,
                    [ "requests_per_second" , ":" , integer , "," ] ,
                    [ "requests_per_minute" , ":" , integer , "," ] ,
                    "on_exceeded" , ":" , rate_limit_action ,
                  "}" ;
rate_limit_action = "queue" | "reject" | "throttle" ;

credentials_spec = "{" ,
                     "type" , ":" , credential_type , "," ,
                     "secret_ref" , ":" , identifier ,
                     [ "," , "oauth2" , ":" , oauth2_config ] ,
                   "}" ;
credential_type = "api_key" | "oauth2" | "basic" | "mtls" | "custom" ;

oauth2_config = "{" ,
                  "grant_type" , ":" , grant_type , "," ,
                  "token_endpoint" , ":" , string ,
                  [ "," , "scopes" , ":" , "[" , string_list , "]" ] ,
                "}" ;
grant_type    = "client_credentials" | "authorization_code" ;

health_check_spec = "{" ,
                      "enabled" , ":" , boolean , "," ,
                      "endpoint" , ":" , identifier , "," ,
                      "interval" , ":" , duration , "," ,
                      "timeout" , ":" , duration ,
                    "}" ;
```

### 6.10.2 Webhook Receiver Declaration

```ebnf
(* WebhookReceiver item body — used within webhookReceivers: collection *)
webhook_receiver_body = "name" , ":" , identifier , "," ,
                        "version" , ":" , version , "," ,
                        "description" , ":" , string , "," ,
                        "source" , ":" , identifier , "," ,
                        "maps_to" , ":" , "{" , "inputIntent" , ":" , identifier , "}" , "," ,
                          "authentication" , ":" , webhook_auth_spec , "," ,
                        "event_mapping" , ":" , "[" , event_mapping_list , "]" ,
                        [ "," , "idempotency" , ":" , webhook_idempotency ] ,
                        [ "," , "replay" , ":" , webhook_replay ] ;

webhook_auth_spec = "{" ,
                      "type" , ":" , webhook_auth_type ,
                      [ "," , "signature" , ":" , signature_config ] ,
                      [ "," , "bearer" , ":" , bearer_config ] ,
                      [ "," , "ip_whitelist" , ":" , ip_whitelist_config ] ,
                    "}" ;
webhook_auth_type = "signature" | "bearer" | "basic" | "ip_whitelist" | "none" ;

signature_config = "{" ,
                     "algorithm" , ":" , signature_algorithm , "," ,
                     "header" , ":" , string , "," ,
                     "secret_ref" , ":" , identifier , "," ,
                     "encoding" , ":" , signature_encoding ,
                     [ "," , "include_body" , ":" , boolean ] ,
                     [ "," , "timestamp_tolerance" , ":" , duration ] ,
                   "}" ;
signature_algorithm = "hmac_sha256" | "hmac_sha512" | "rsa_sha256" ;
signature_encoding = "hex" | "base64" ;

bearer_config = "{" , "secret_ref" , ":" , identifier , "}" ;
ip_whitelist_config = "{" , "allowed" , ":" , "[" , cidr_list , "]" , "}" ;
cidr_list     = string , { "," , string } ;

event_mapping_list = event_mapping , { "," , event_mapping } ;
event_mapping = "{" ,
                  "external_event" , ":" , string , "," ,
                  "intent_type" , ":" , identifier , "," ,
                  "field_mapping" , ":" , "{" , intent_field_mapping , "}" ,
                "}" ;

intent_field_mapping = field_map_entry , { "," , field_map_entry } ;
field_map_entry = identifier , ":" , string ;

webhook_idempotency = "{" ,
                        "key_path" , ":" , string , "," ,
                        "ttl" , ":" , duration ,
                      "}" ;

webhook_replay = "{" ,
                   "enabled" , ":" , boolean ,
                   [ "," , "window" , ":" , duration ] ,
                 "}" ;
```

## 6.11 Edge Device & IoT Grammar

### 6.11.1 Device Specification for Worker Class

```ebnf
device_spec = "{" ,
                "type" , ":" , device_type , "," ,
                "capabilities" , ":" , "[" , identifier_list , "]" , "," ,
                [ "hardware" , ":" , device_hardware , "," ] ,
                [ "constraints" , ":" , device_constraints , "," ] ,
                [ "health" , ":" , device_health ] ,
              "}" ;

device_type = "sensor" | "actuator" | "gateway" | "hybrid" ;

device_hardware = "{" ,
                    "category" , ":" , hardware_category , "," ,
                    [ "certifications" , ":" , "[" , identifier_list , "]" ] ,
                  "}" ;
hardware_category = "industrial" | "consumer" | "medical" | "automotive" ;

device_constraints = "{" ,
                       [ "power" , ":" , power_constraints , "," ] ,
                       [ "connectivity" , ":" , connectivity_constraints , "," ] ,
                       [ "processing" , ":" , processing_constraints ] ,
                     "}" ;

power_constraints = "{" ,
                      "source" , ":" , power_source , "," ,
                      [ "battery_capacity" , ":" , integer , "," ] ,
                      [ "sleep_supported" , ":" , boolean ] ,
                    "}" ;
power_source = "battery" | "mains" | "solar" | "poe" ;

connectivity_constraints = "{" ,
                             "type" , ":" , connectivity_type , "," ,
                             "protocols" , ":" , "[" , protocol_list , "]" ,
                           "}" ;
connectivity_type = "always_on" | "intermittent" | "scheduled" ;

protocol_list = protocol , { "," , protocol } ;
protocol = "wifi" | "ble" | "lora" | "nbiot" | "zigbee" | "thread" | "ethernet" ;

processing_constraints = "{" , "level" , ":" , processing_level , "}" ;
processing_level = "minimal" | "standard" | "edge_compute" ;

device_health = "{" ,
                  "heartbeat_interval" , ":" , duration , "," ,
                  "offline_threshold" , ":" , duration , "," ,
                  [ "battery_low_threshold" , ":" , integer ] ,
                "}" ;
```

### 6.11.2 Device Capability Declaration

```ebnf
(* DeviceCapability item body — used within deviceCapabilities: collection *)
device_capability_body = "name" , ":" , identifier , "," ,
                         "version" , ":" , version , "," ,
                         "type" , ":" , capability_type , "," ,
                         ( sensor_config | actuator_config ) ,
                         [ "," , "safety" , ":" , capability_safety ] ;

capability_type = "sensor" | "actuator" | "diagnostic" ;

sensor_config = "sensor" , ":" , "{" ,
                  "measures" , ":" , measurement_type , "," ,
                  "output" , ":" , sensor_output , "," ,
                  [ "sampling" , ":" , sensor_sampling ] ,
                "}" ;

measurement_type = "temperature" | "humidity" | "pressure" | "motion" | "light" | 
                   "proximity" | "acceleration" | "gps" | "current" | "voltage" ;

sensor_output = "{" ,
                  "type" , ":" , type_ref , "," ,
                  "unit" , ":" , string , "," ,
                  [ "precision" , ":" , integer , "," ] ,
                  [ "range" , ":" , "{" , "min" , ":" , number , "," , "max" , ":" , number , "}" ] ,
                "}" ;

sensor_sampling = "{" ,
                    "min_interval" , ":" , duration , "," ,
                    "max_interval" , ":" , duration ,
                  "}" ;

actuator_config = "actuator" , ":" , "{" ,
                    "controls" , ":" , control_type , "," ,
                    "input" , ":" , actuator_input , "," ,
                    [ "feedback" , ":" , actuator_feedback ] ,
                  "}" ;

control_type = "switch" | "dimmer" | "motor" | "valve" | "lock" | "hvac" ;

actuator_input = "{" ,
                   "type" , ":" , type_ref , "," ,
                   [ "range" , ":" , "{" , "min" , ":" , number , "," , "max" , ":" , number , "}" ] ,
                 "}" ;

actuator_feedback = "{" ,
                      "confirmation" , ":" , confirmation_mode , "," ,
                      "timeout" , ":" , duration ,
                    "}" ;
confirmation_mode = "required" | "optional" | "none" ;

capability_safety = "{" ,
                      "constraints" , ":" , "[" , safety_constraint_list , "]" , "," ,
                      [ "clamp_to" , ":" , "{" , "min" , ":" , number , "," , "max" , ":" , number , "}" ] ,
                    "}" ;

safety_constraint_list = safety_constraint , { "," , safety_constraint } ;
safety_constraint = "{" ,
                      "expression" , ":" , expression , "," ,
                      "on_violation" , ":" , safety_action ,
                    "}" ;
safety_action = "reject" | "clamp" | "alert" | "emergency_stop" ;
```

### 6.11.3 Device Telemetry Declaration

```ebnf
(* DeviceTelemetry item body — used within deviceTelemetries: collection *)
device_telemetry_body = "name" , ":" , identifier , "," ,
                        "version" , ":" , version , "," ,
                        "source" , ":" , telemetry_source , "," ,
                        "collection" , ":" , telemetry_collection , "," ,
                        [ "buffering" , ":" , telemetry_buffering , "," ] ,
                        [ "delivery" , ":" , telemetry_delivery ] ;

telemetry_source = "{" ,
                     "device_type" , ":" , identifier , "," ,
                     "capability" , ":" , identifier ,
                   "}" ;

telemetry_collection = "{" ,
                         "mode" , ":" , collection_mode , "," ,
                         ( continuous_config | on_change_config | threshold_config | scheduled_config ) ,
                       "}" ;
collection_mode = "continuous" | "on_change" | "threshold" | "scheduled" ;

continuous_config = "continuous" , ":" , "{" , "interval" , ":" , duration , "}" ;
on_change_config = "on_change" , ":" , "{" , "tolerance" , ":" , number , "}" ;
threshold_config = "threshold" , ":" , "{" ,
                     [ "above" , ":" , number , "," ] ,
                     [ "below" , ":" , number ] ,
                   "}" ;
scheduled_config = "scheduled" , ":" , "{" , "cron" , ":" , string , "}" ;

telemetry_buffering = "{" ,
                        "on_offline" , ":" , buffer_strategy , "," ,
                        "max_buffer_size" , ":" , integer , "," ,
                        "max_buffer_duration" , ":" , duration , "," ,
                        "sync_on_reconnect" , ":" , sync_strategy ,
                      "}" ;
buffer_strategy = "buffer_local" | "discard" | "aggregate" ;
sync_strategy = "immediate" | "batched" | "background" ;

telemetry_delivery = "{" ,
                       "guarantee" , ":" , delivery_guarantee , "," ,
                       [ "compression" , ":" , compression_type , "," ] ,
                       [ "batching" , ":" , batching_config ] ,
                     "}" ;
compression_type = "none" | "gzip" | "lz4" ;

batching_config = "{" ,
                    "enabled" , ":" , boolean , "," ,
                    "max_size" , ":" , integer , "," ,
                    "max_delay" , ":" , duration ,
                  "}" ;
```

### 6.11.4 Actuator Command Declaration

```ebnf
(* ActuatorCommand item body — used within actuatorCommands: collection *)
actuator_command_body = "name" , ":" , identifier , "," ,
                        "version" , ":" , version , "," ,
                        "target" , ":" , command_target , "," ,
                        "command" , ":" , command_spec , "," ,
                        [ "safety" , ":" , command_safety , "," ] ,
                        [ "delivery" , ":" , command_delivery ] ;

command_target = "{" ,
                   "device_type" , ":" , identifier , "," ,
                   "capability" , ":" , identifier , "," ,
                   "selector" , ":" , expression ,
                 "}" ;

command_spec = "{" ,
                 "type" , ":" , command_type , "," ,
                 ( set_state_config | toggle_config | pulse_config ) ,
               "}" ;
command_type = "set_state" | "toggle" | "pulse" ;

set_state_config = "set_state" , ":" , "{" , "value" , ":" , expression , "}" ;
toggle_config = "toggle" , ":" , "{" , [ "duration" , ":" , duration ] , "}" ;
pulse_config = "pulse" , ":" , "{" , "duration" , ":" , duration , "}" ;

command_safety = "{" ,
                   "constraints" , ":" , "[" , expression_list , "]" , "," ,
                   "on_violation" , ":" , safety_action ,
                 "}" ;

command_delivery = "{" ,
                     "guarantee" , ":" , delivery_guarantee , "," ,
                     "timeout" , ":" , duration , "," ,
                     "retry_on_offline" , ":" , offline_retry_action , "," ,
                     [ "queue_ttl" , ":" , duration ] ,
                   "}" ;
offline_retry_action = "queue" | "reject" ;
```

---

# 7. V1.6 EXTENSIONS

## 7.1 Storage Binding Grammar

Storage Binding provides declarative configuration for data persistence.

```ebnf
(* StorageBinding item body — used within storageBindings: collection *)
storage_binding_body = "name:" , identifier , NEWLINE ,
                       "version:" , version , NEWLINE ,
                       "type:" , storage_type , NEWLINE ,
                       ( kv_config | stream_config | object_store_storage_config ) ,
                       [ access_config ] ;
```

### 7.1.1 Key-Value Storage Configuration

```ebnf
kv_config     = "kv:" , NEWLINE , INDENT ,
                  "bucket:" , string , NEWLINE ,
                  [ "key_pattern:" , string , NEWLINE ] ,
                  [ "serialization:" , serialization_type , NEWLINE ] ,
                  [ "history:" , integer , NEWLINE ] ,
                  [ "ttl:" , duration , NEWLINE ] ,
                  [ "replicas:" , integer , NEWLINE ] ,
                  DEDENT ;
```

### 7.1.2 Stream Storage Configuration

```ebnf
stream_config = "stream:" , NEWLINE , INDENT ,
                  "name:" , string , NEWLINE ,
                  "subjects:" , subject_list , NEWLINE ,
                  [ "retention:" , v16_retention_policy , NEWLINE ] ,
                  [ "max_msgs:" , integer , NEWLINE ] ,
                  [ "max_bytes:" , size_expression , NEWLINE ] ,
                  [ "max_age:" , duration , NEWLINE ] ,
                  [ "storage:" , storage_medium , NEWLINE ] ,
                  [ "replicas:" , integer , NEWLINE ] ,
                  DEDENT ;

subject_list  = "[" , subject_pattern , { "," , subject_pattern } , "]" ;
subject_pattern = string ;
```

### 7.1.3 Object Store Configuration

```ebnf
object_store_storage_config = "object_store:" , NEWLINE , INDENT ,
                                "bucket:" , string , NEWLINE ,
                                [ "description:" , string , NEWLINE ] ,
                                [ "max_object_size:" , size_expression , NEWLINE ] ,
                                [ "replicas:" , integer , NEWLINE ] ,
                                DEDENT ;
```

### 7.1.4 Access Configuration

```ebnf
access_config = "access:" , NEWLINE , INDENT ,
                  [ "read_policy:" , read_policy , NEWLINE ] ,
                  [ "write_consistency:" , write_consistency , NEWLINE ] ,
                  DEDENT ;
```

### 7.1.5 Key Pattern Grammar

```ebnf
key_pattern_expr = key_segment , { "." , key_segment } ;

key_segment   = static_key_segment
              | dynamic_key_segment ;

static_key_segment = identifier ;

dynamic_key_segment = "{{" , field_ref , "}}" ;

field_ref     = identifier , { "." , identifier } ;
```

---

## 7.2 Cache Layer Grammar

Unified Caching provides declarative configuration for multi-tier caching.

```ebnf
(* CacheLayer item body — used within cacheLayers: collection *)
cache_layer_body = "name:" , identifier , NEWLINE ,
                   [ "version:" , version , NEWLINE ] ,
                   [ "description:" , string , NEWLINE ] ,
                   topology_config ,
                   [ invalidation_config ] ,
                   [ warmup_config ] ,
                   [ cache_observability_config ] ;
```

### 7.2.1 Topology Configuration

```ebnf
topology_config = "topology:" , NEWLINE , INDENT ,
                    "tier:" , cache_tier , NEWLINE ,
                    [ local_cache_config ] ,
                    [ distributed_cache_config ] ,
                    [ hybrid_cache_config ] ,
                    DEDENT ;

local_cache_config = "local:" , NEWLINE , INDENT ,
                       [ "max_entries:" , integer , NEWLINE ] ,
                       [ "ttl:" , duration , NEWLINE ] ,
                       [ "eviction:" , eviction_policy , NEWLINE ] ,
                       [ "max_size:" , size_expression , NEWLINE ] ,
                       DEDENT ;

distributed_cache_config = "distributed:" , NEWLINE , INDENT ,
                             "resource:" , identifier , NEWLINE ,
                             [ "ttl:" , duration , NEWLINE ] ,
                             DEDENT ;

hybrid_cache_config = "hybrid:" , NEWLINE , INDENT ,
                        [ "promotion:" , promotion_policy , NEWLINE ] ,
                        DEDENT ;
```

### 7.2.2 Invalidation Configuration

```ebnf
invalidation_config = "invalidation:" , NEWLINE , INDENT ,
                        "strategy:" , invalidation_strategy , NEWLINE ,
                        [ "ttl:" , duration , NEWLINE ] ,
                        [ "watch_keys:" , key_list , NEWLINE ] ,
                        [ "propagate:" , boolean , NEWLINE ] ,
                        DEDENT ;

key_list      = "[" , cache_key_pattern , { "," , cache_key_pattern } , "]" ;
cache_key_pattern = string ;
```

### 7.2.3 Warmup Configuration

```ebnf
warmup_config = "warmup:" , NEWLINE , INDENT ,
                  [ "enabled:" , boolean , NEWLINE ] ,
                  [ "strategy:" , warmup_strategy , NEWLINE ] ,
                  [ "on_startup:" , boolean , NEWLINE ] ,
                  [ "queries:" , query_list , NEWLINE ] ,
                  DEDENT ;

query_list    = "[" , query_expression , { "," , query_expression } , "]" ;
query_expression = string ;
```

### 7.2.4 Cache Observability Configuration

```ebnf
cache_observability_config = "observability:" , NEWLINE , INDENT ,
                               [ "emit_signals:" , boolean , NEWLINE ] ,
                               [ "metrics:" , cache_metric_list , NEWLINE ] ,
                               [ "interval:" , duration , NEWLINE ] ,
                               DEDENT ;

cache_metric_list = "[" , cache_metric , { "," , cache_metric } , "]" ;
```

---

## 7.3 Intent Routing Grammar

Intent Routing provides declarative configuration for InputIntent transport.

```ebnf
(* Extension to inputIntent declaration *)
intent_routing = "routing:" , NEWLINE , INDENT ,
                   "transport:" , v16_transport_type , NEWLINE ,
                   transport_config ,
                   [ response_config ] ,
                   DEDENT ;

transport_config = nats_transport_config
                 | http_transport_config
                 | grpc_transport_config ;
```

### 7.3.1 NATS Transport Configuration

```ebnf
nats_transport_config = "nats:" , NEWLINE , INDENT ,
                          "subject_pattern:" , string , NEWLINE ,
                          [ "delivery:" , delivery_mode , NEWLINE ] ,
                          [ "stream:" , identifier , NEWLINE ] ,
                          [ "queue:" , identifier , NEWLINE ] ,
                          [ "dead_letter:" , string , NEWLINE ] ,
                          DEDENT ;
```

### 7.3.2 HTTP Transport Configuration

```ebnf
http_transport_config = "http:" , NEWLINE , INDENT ,
                          "path:" , string , NEWLINE ,
                          [ "method:" , http_method , NEWLINE ] ,
                          [ "timeout:" , duration , NEWLINE ] ,
                          DEDENT ;
```

### 7.3.3 gRPC Transport Configuration

```ebnf
grpc_transport_config = "grpc:" , NEWLINE , INDENT ,
                          "service:" , identifier , NEWLINE ,
                          "method:" , identifier , NEWLINE ,
                          [ "streaming:" , boolean , NEWLINE ] ,
                          DEDENT ;
```

### 7.3.4 Response Configuration

```ebnf
response_config = "response:" , NEWLINE , INDENT ,
                    "mode:" , response_mode , NEWLINE ,
                    [ "timeout:" , duration , NEWLINE ] ,
                    [ "callback:" , string , NEWLINE ] ,
                    DEDENT ;
```

### 7.3.5 Subject Pattern Grammar

```ebnf
routing_subject_pattern = subject_segment , { "." , subject_segment } ;

subject_segment = static_subject_segment
                | dynamic_subject_segment
                | wildcard_segment ;

static_subject_segment = identifier ;

dynamic_subject_segment = "{{" , variable_name , "}}" ;

variable_name = "domain"
              | "intent_name"
              | "realm"
              | "version"
              | "entity_id"
              | "correlation_id"
              | identifier ;

wildcard_segment = "*"       (* single token wildcard *)
                 | ">" ;     (* multi-token wildcard *)
```

---

## 7.4 Resource Requirements Grammar

Resource Requirements provides abstract infrastructure specification.

```ebnf
resource_requirements_decl = "resourceRequirements:" , NEWLINE , INDENT ,
                               "name:" , identifier , NEWLINE ,
                               [ "version:" , version , NEWLINE ] ,
                               [ "description:" , string , NEWLINE ] ,
                               [ event_logs_config ] ,
                               [ entity_stores_config ] ,
                               [ object_stores_rr_config ] ,
                               [ provisioning_config ] ,
                               DEDENT ;
```

### 7.4.1 Event Logs Configuration

```ebnf
event_logs_config = "eventLogs:" , NEWLINE , INDENT ,
                      { event_log_decl } ,
                      DEDENT ;

event_log_decl = "- name:" , identifier , NEWLINE , INDENT ,
                   [ "purpose:" , event_log_purpose , NEWLINE ] ,
                   [ "ordering:" , ordering_guarantee , NEWLINE ] ,
                   [ "delivery:" , delivery_semantic , NEWLINE ] ,
                   [ "durability:" , durability_type , NEWLINE ] ,
                   [ retention_rr_config ] ,
                   [ replication_config ] ,
                   DEDENT ;

retention_rr_config = "retention:" , NEWLINE , INDENT ,
                        [ "strategy:" , retention_strategy , NEWLINE ] ,
                        [ "max_age:" , duration , NEWLINE ] ,
                        [ "max_size:" , size_expression , NEWLINE ] ,
                        [ "max_count:" , integer , NEWLINE ] ,
                        DEDENT ;

replication_config = "replication:" , NEWLINE , INDENT ,
                       [ "min_replicas:" , integer , NEWLINE ] ,
                       [ "consistency:" , replication_consistency , NEWLINE ] ,
                       DEDENT ;
```

### 7.4.2 Entity Stores Configuration

```ebnf
entity_stores_config = "entityStores:" , NEWLINE , INDENT ,
                         { entity_store_decl } ,
                         DEDENT ;

entity_store_decl = "- name:" , identifier , NEWLINE , INDENT ,
                      [ "purpose:" , entity_store_purpose , NEWLINE ] ,
                      [ "access_pattern:" , access_pattern , NEWLINE ] ,
                      [ "consistency:" , store_consistency , NEWLINE ] ,
                      [ "durability:" , durability_type , NEWLINE ] ,
                      [ history_config ] ,
                      [ expiration_config ] ,
                      [ replication_config ] ,
                      [ capacity_config ] ,
                      DEDENT ;

history_config = "history:" , NEWLINE , INDENT ,
                   [ "enabled:" , boolean , NEWLINE ] ,
                   [ "depth:" , integer , NEWLINE ] ,
                   DEDENT ;

expiration_config = "expiration:" , NEWLINE , INDENT ,
                      [ "enabled:" , boolean , NEWLINE ] ,
                      [ "ttl:" , duration , NEWLINE ] ,
                      DEDENT ;

capacity_config = "capacity:" , NEWLINE , INDENT ,
                    [ "max_entry_size:" , size_expression , NEWLINE ] ,
                    [ "max_entries:" , integer , NEWLINE ] ,
                    DEDENT ;
```

### 7.4.3 Object Stores Configuration

```ebnf
object_stores_rr_config = "objectStores:" , NEWLINE , INDENT ,
                            { object_store_rr_decl } ,
                            DEDENT ;

object_store_rr_decl = "- name:" , identifier , NEWLINE , INDENT ,
                         [ "purpose:" , object_store_purpose , NEWLINE ] ,
                         [ "durability:" , durability_type , NEWLINE ] ,
                         [ replication_config ] ,
                         [ object_capacity_config ] ,
                         DEDENT ;

object_capacity_config = "capacity:" , NEWLINE , INDENT ,
                           [ "max_object_size:" , size_expression , NEWLINE ] ,
                           DEDENT ;
```

### 7.4.4 Provisioning Configuration

```ebnf
provisioning_config = "provisioning:" , NEWLINE , INDENT ,
                        [ "mode:" , provisioning_mode , NEWLINE ] ,
                        [ migrations_config ] ,
                        DEDENT ;

migrations_config = "migrations:" , NEWLINE , INDENT ,
                      [ "enabled:" , boolean , NEWLINE ] ,
                      [ "strategy:" , v16_migration_strategy , NEWLINE ] ,
                      DEDENT ;
```

---

## 7.5 Client Cache Grammar

Client Cache Contract provides client-side caching configuration for Experiences.

```ebnf
client_cache_config = "clientCache:" , NEWLINE , INDENT ,
                        "enabled:" , boolean , NEWLINE ,
                        client_storage_config ,
                        [ client_strategies_config ] ,
                        [ client_prefetch_config ] ,
                        [ client_offline_config ] ,
                        [ version_policy_config ] ,
                        DEDENT ;
```

### 7.5.1 Client Storage Configuration

```ebnf
client_storage_config = "storage:" , NEWLINE , INDENT ,
                          "type:" , client_storage_type , NEWLINE ,
                          [ "max_size:" , size_expression , NEWLINE ] ,
                          [ "encryption:" , boolean , NEWLINE ] ,
                          DEDENT ;
```

### 7.5.2 Client Strategies Configuration

```ebnf
client_strategies_config = "strategies:" , NEWLINE , INDENT ,
                             [ primary_strategy_config ] ,
                             [ related_strategy_config ] ,
                             DEDENT ;

primary_strategy_config = "primary:" , NEWLINE , INDENT ,
                            "strategy:" , v16_cache_strategy , NEWLINE ,
                            [ "ttl:" , duration , NEWLINE ] ,
                            DEDENT ;

related_strategy_config = "related:" , NEWLINE , INDENT ,
                            "strategy:" , v16_cache_strategy , NEWLINE ,
                            [ "ttl:" , duration , NEWLINE ] ,
                            [ "lazy_cache:" , boolean , NEWLINE ] ,
                            DEDENT ;
```

### 7.5.3 Client Prefetch Configuration

```ebnf
client_prefetch_config = "prefetch:" , NEWLINE , INDENT ,
                           [ "enabled:" , boolean , NEWLINE ] ,
                           [ "views:" , view_list , NEWLINE ] ,
                           [ "trigger:" , prefetch_trigger , NEWLINE ] ,
                           DEDENT ;

view_list     = "[" , view_ref , { "," , view_ref } , "]" ;
view_ref      = identifier ;
```

### 7.5.4 Client Offline Configuration

```ebnf
client_offline_config = "offline:" , NEWLINE , INDENT ,
                          [ "enabled:" , boolean , NEWLINE ] ,
                          [ "required_views:" , view_list , NEWLINE ] ,
                          [ "sync_on_reconnect:" , boolean , NEWLINE ] ,
                          [ "conflict_resolution:" , v16_conflict_strategy , NEWLINE ] ,
                          [ intent_sync_config ] ,
                          [ governance_sync_config ] ,
                          DEDENT ;

intent_sync_config = "intent_sync:" , NEWLINE , INDENT ,
                       [ "on_cas_failure:" , cas_failure_action , NEWLINE ] ,
                       [ "retry_with_refresh:" , boolean , NEWLINE ] ,
                       DEDENT ;

governance_sync_config = "governance:" , NEWLINE , INDENT ,
                           [ "validate_on_sync:" , boolean , NEWLINE ] ,
                           [ "on_violation:" , v16_violation_action , NEWLINE ] ,
                           DEDENT ;
```

### 7.5.5 Version Policy Configuration

```ebnf
version_policy_config = "version_policy:" , NEWLINE , INDENT ,
                          [ "on_upgrade:" , upgrade_action , NEWLINE ] ,
                          [ "grace_period:" , duration , NEWLINE ] ,
                          [ "migration:" , cache_migration_strategy , NEWLINE ] ,
                          DEDENT ;
```

---

# 8. VALIDATION RULES

These semantic rules MUST be enforced by conformant parsers/validators.

## 8.1 Structural Rules

```ebnf
valid_flow             = flow_has_steps 
                       ∧ no_duplicate_work_units_in_atomic_group
                       ∧ atomic_group_has_boundary ;

valid_atomic_group     = atomic_group_single_boundary
                       ∧ atomic_group_no_cross_boundary_steps ;
```

## 8.2 Policy Rules

```ebnf
valid_policy           = policy_has_effect
                       ∧ policy_target_exists
                       ∧ policy_expression_valid ;

valid_auth             = one_authority_per_scope
                       ∧ deny_overrides_allow ;
```

## 8.3 Evolution Rules

```ebnf
valid_evolution        = no_version_skip
                       ∧ compatible_changes_only ;
```

## 8.4 Idempotency Rules

```ebnf
valid_idempotency      = retry_implies_idempotent
                       ∧ key_deterministic ;
```

## 8.5 Reference Rules

```ebnf
valid_references       = all_work_units_defined
                       ∧ all_data_states_defined
                       ∧ all_policies_defined
                       ∧ all_boundaries_defined ;
```

## 8.6 Work Unit Contract Rules

```ebnf
valid_work_unit_contract = contract_has_name
                         ∧ contract_has_version
                         ∧ contract_has_input_schema
                         ∧ contract_has_output_schema
                         ∧ input_from_references_valid_datastate
                         ∧ output_to_references_valid_datastate
                         ∧ retryable_errors_have_backoff
                         ∧ side_effects_have_targets
                         ∧ database_writes_have_idempotency ;

valid_contract_coverage = flow_work_units_have_contracts
                        ∧ contract_versions_match_references ;
```

## 8.7 Authority Rules

```ebnf
valid_authority        = one_authority_per_entity
                       ∧ authority_epoch_monotonic
                       ∧ no_overlapping_authority
                       ∧ cas_token_valid ;
```

## 8.8 Relationship Rules

```ebnf
valid_relationship     = relationship_from_exists
                       ∧ relationship_to_exists
                       ∧ cardinality_constraint_satisfied
                       ∧ no_synchronous_coordination_implied ;
```

## 8.9 Invariant Rules

```ebnf
valid_invariant        = invariants_enforced_by_view_or_policy
                       ∧ cross_entity_invariants_in_views
                       ∧ views_authority_agnostic ;
```

## 8.10 Storage Role Rules

```ebnf
valid_storage_role     = control_storage_strongly_consistent
                       ∧ control_storage_designated_writers
                       ∧ data_storage_rebuildable ;
```

## 8.11 Presentation Composition Rules

```ebnf
valid_composition      = composition_has_name
                       ∧ composition_has_version
                       ∧ composition_has_primary_view
                       ∧ composition_primary_view_exists
                       ∧ composition_related_views_exist
                       ∧ composition_navigation_hierarchy_acyclic ;
```

## 8.12 Experience Rules

```ebnf
valid_experience       = experience_has_name
                       ∧ experience_has_entry_point
                       ∧ experience_entry_point_in_includes
                       ∧ experience_includes_compositions_exist
                       ∧ experience_policy_set_exists ;
```

## 8.13 Field Permission Rules

```ebnf
valid_field_permission = field_permission_field_exists
                       ∧ field_permission_policy_exists
                       ∧ field_permission_mask_valid ;
```

## 8.14 Navigation Rules

```ebnf
valid_navigation       = navigation_parent_exists
                       ∧ navigation_hierarchy_acyclic
                       ∧ navigation_purpose_valid ;
```

## 8.15 v1.5 Extension Rules

### View Triggers Rules

```ebnf
valid_view_triggers    = triggers_intent_exists
                       ∧ triggers_required_fields_mapped
                       ∧ triggers_validation_consistent
                       ∧ triggers_context_fields_available ;
```

### Intent Delivery Rules

```ebnf
valid_intent_delivery  = delivery_timeout_positive
                       ∧ retry_implies_idempotent
                       ∧ response_includes_non_empty ;
```

### Materialization Retrieval Rules

```ebnf
valid_retrieval        = by_entity_has_key
                       ∧ by_query_has_fields
                       ∧ singleton_no_key_or_fields ;
```

### Session Rules

```ebnf
valid_session          = session_contains_required_valid
                       ∧ lifetime_timeouts_consistent
                       ∧ devices_max_active_positive ;
```

### Indicator Source Rules

```ebnf
valid_indicator_source = indicator_view_exists
                       ∧ indicator_scope_expression_valid
                       ∧ indicator_update_trigger_consistent ;
```

### Composition Fetch Rules

```ebnf
valid_composition_fetch = fetch_timeouts_positive
                        ∧ fetch_max_concurrent_positive
                        ∧ fetch_retry_scope_valid ;
```

### Asset Rules

```ebnf
valid_asset            = asset_max_size_positive
                       ∧ asset_mime_types_valid
                       ∧ asset_lifecycle_expiry_consistent
                       ∧ asset_variants_unique ;
```

### Search Rules

```ebnf
valid_search           = search_indexed_fields_exist
                       ∧ search_facet_ranges_valid
                       ∧ search_intent_fields_exist ;
```

### External Dependency Rules

```ebnf
valid_external_dep     = external_capabilities_non_empty
                       ∧ external_contract_schemas_valid
                       ∧ external_credentials_secret_exists
                       ∧ external_circuit_breaker_threshold_positive ;
```

### Webhook Rules

```ebnf
valid_webhook          = webhook_source_exists
                       ∧ webhook_maps_to_intent_exists
                       ∧ webhook_event_mapping_valid
                       ∧ webhook_auth_secret_exists ;
```

### Governance Rules

```ebnf
valid_governance       = retention_duration_positive
                       ∧ residency_regions_valid
                       ∧ encryption_consistent
                       ∧ audit_retention_reasonable ;
```

### Consent Rules

```ebnf
valid_consent          = consent_scopes_non_empty
                       ∧ consent_subject_field_exists
                       ∧ consent_withdrawal_consistent ;
```

### Erasure Rules

```ebnf
valid_erasure          = erasure_scope_valid
                       ∧ erasure_sla_reasonable
                       ∧ erasure_cascade_entities_exist ;
```

### Legal Hold Rules

```ebnf
valid_legal_hold       = hold_entities_exist
                       ∧ hold_overrides_valid
                       ∧ hold_custodian_exists ;
```

### Graph Query Rules

```ebnf
valid_graph_query      = graph_relationship_exists
                       ∧ graph_start_entity_valid
                       ∧ graph_max_depth_reasonable ;
```

### Device Rules

```ebnf
valid_device           = device_capabilities_non_empty
                       ∧ device_heartbeat_reasonable
                       ∧ sensor_range_consistent
                       ∧ actuator_safety_defined ;
```

### Telemetry Rules

```ebnf
valid_telemetry        = telemetry_source_exists
                       ∧ collection_mode_consistent
                       ∧ buffer_size_reasonable ;
```

### Actuator Command Rules

```ebnf
valid_actuator_cmd     = actuator_target_exists
                       ∧ command_timeout_positive
                       ∧ safety_constraints_valid ;
```

### Inference Rules

```ebnf
valid_inference        = inference_model_exists
                       ∧ inference_source_fields_exist
                       ∧ inference_output_type_valid ;
```

### Spatial Rules

```ebnf
valid_spatial          = coordinate_system_valid
                       ∧ spatial_precision_reasonable ;
```

### Structured Content Rules

```ebnf
valid_structured_content = content_schema_exists
                         ∧ blocks_allowed_non_empty
                         ∧ marks_allowed_non_empty ;
```

## 8.16 v1.6 Validation Rules

### Storage Binding Rules

```ebnf
(* Storage binding must specify exactly one storage type *)
storage_binding_single_type = has_kv_config XOR has_stream_config XOR has_object_store_config ;

(* Key pattern must be valid with proper variable references *)
valid_key_pattern = key_pattern_syntax_valid
                  ∧ key_pattern_variables_resolvable ;

(* Storage references must point to existing bindings *)
storage_reference_valid = referenced_binding_exists
                        ∧ storage_type_compatible_with_lifecycle ;
```

### Cache Layer Rules

```ebnf
(* Cache topology must be valid *)
valid_cache_topology = tier_config_complete
                     ∧ tier_compatible_with_invalidation ;

(* Cache invalidation strategy must be consistent *)
valid_invalidation = invalidation_strategy_valid
                   ∧ watch_keys_exist_if_watch_strategy
                   ∧ ttl_specified_if_ttl_strategy ;

(* Cache warmup queries must be valid *)
valid_warmup = warmup_queries_parseable
             ∧ warmup_queries_reference_valid_states ;

(* Cache references in materialization must point to existing layers *)
cache_reference_valid = referenced_layer_exists
                      ∧ cache_key_pattern_valid ;
```

### Resource Requirements Rules

```ebnf
(* Event log configuration must be complete *)
valid_event_log = ordering_consistent_with_delivery
                ∧ retention_strategy_has_limits_if_bounded
                ∧ replication_min_replicas_positive ;

(* Entity store configuration must be valid *)
valid_entity_store = access_pattern_valid
                   ∧ consistency_compatible_with_durability
                   ∧ history_depth_positive_if_enabled ;

(* Provisioning mode must be consistent *)
valid_provisioning = migration_strategy_valid
                   ∧ mode_consistent_with_environment ;
```

### Intent Routing Rules

```ebnf
(* Transport configuration must be complete *)
valid_transport = transport_type_has_config
                ∧ transport_config_complete ;

(* NATS transport validation *)
valid_nats_transport = subject_pattern_valid
                     ∧ stream_exists_if_jetstream
                     ∧ queue_name_valid ;

(* HTTP transport validation *)
valid_http_transport = path_valid
                     ∧ method_supported
                     ∧ timeout_positive ;

(* gRPC transport validation *)
valid_grpc_transport = service_name_valid
                     ∧ method_name_valid ;

(* Response mode must be compatible with transport *)
valid_response = response_mode_compatible_with_transport
               ∧ timeout_reasonable
               ∧ callback_valid_if_async ;
```

### Client Cache Rules

```ebnf
(* Client cache configuration must be valid *)
valid_client_cache = storage_type_valid
                   ∧ max_size_positive
                   ∧ strategies_valid ;

(* Strategy configuration must be complete *)
valid_strategy = strategy_has_ttl
               ∧ ttl_positive
               ∧ lazy_cache_only_for_related ;

(* Prefetch configuration must be valid *)
valid_prefetch = prefetch_views_exist
               ∧ trigger_valid ;

(* Offline configuration must be consistent *)
valid_offline = required_views_exist
              ∧ sync_on_reconnect_if_offline_enabled
              ∧ conflict_resolution_strategy_valid ;

(* Version policy must be complete *)
valid_version_policy = upgrade_action_valid
                     ∧ grace_period_positive
                     ∧ migration_strategy_valid ;
```

### Cross-Reference Rules (v1.6)

```ebnf
(* All storage references must resolve to items in the storageBindings collection *)
storage_references_valid = ∀ ds ∈ dataStates where ds.storage exists
                          : ds.storage.binding ∈ { sb.name | sb ∈ storageBindings } ;

(* All cache references must resolve to items in the cacheLayers collection *)
cache_references_valid = ∀ m ∈ materializations where m.cache exists
                        : m.cache.layer ∈ { cl.name | cl ∈ cacheLayers } ;

(* All resource requirements must be satisfiable *)
resource_requirements_satisfiable = ∀ resourceRequirements r
                                   : r.eventLogs ⊆ available_event_logs
                                   ∧ r.entityStores ⊆ available_entity_stores
                                   ∧ r.objectStores ⊆ available_object_stores ;
```

---

# 9. FORMAL SEMANTICS

## 9.1 Execution Semantics

```ebnf
(* Flow Execution *)
execute(flow, input) = 
  if ¬authorized(flow, input.context) then
    Rejection
  else
    let state = input in
    for step in flow.steps do
      state := execute_step(step, state)
      if state = Failure then
        return handle_failure(flow.failure_spec, state)
    done
    return Success(state)
  fi

(* Atomic Group Execution *)
execute_atomic(group, input) =
  let location = select_boundary(group.boundary) in
  let state = input in
  for step in group.steps do
    state := execute_at(location, step, state)
    if state = Failure then
      return Failure  (* Fail entire group *)
  done
  return state

(* Policy Evaluation *)
evaluate_policy(policy, context, target) =
  if policy.lifecycle ≠ enforce then
    return Shadow(evaluate_condition(policy.when, context))
  else
    let result = evaluate_condition(policy.when, context) in
    return if policy.effect = allow then result else ¬result
  fi
```

---

# 10. CONFORMANCE

## 10.1 Parser Conformance

A conformant parser MUST:

1. Accept all valid specifications
2. Reject all invalid specifications
3. Report errors with location and reason
4. Support all grammar elements
5. Enforce all semantic rules

## 10.2 Runtime Conformance

A conformant runtime MUST:

1. Execute flows as defined
2. Enforce atomic groups correctly
3. Evaluate policies locally
4. Respect version compatibility
5. Implement idempotency
6. Propagate correlation context
7. Emit required signals

## 10.3 Tooling Conformance

Conformant tools MUST:

1. Parse specifications correctly
2. Validate against semantic rules
3. Generate conformant code
4. Preserve intent in transformations

## 10.4 Parser Implementation Guidance

### Recommended Parsing Strategy

1. **Lexical Analysis**
   - Tokenize based on lexical grammar
   - Handle comments and whitespace
   - Validate identifiers and literals

2. **Syntax Analysis**
   - Build AST from token stream
   - Validate structure against grammar
   - Report syntax errors with location

3. **Semantic Analysis**
   - Validate cross-references
   - Check version consistency
   - Verify constraint expressions
   - Validate policy targets

4. **Validation**
   - Apply semantic rules
   - Check for anti-patterns
   - Verify completeness
   - Generate warnings for potential issues

### Error Reporting

Good error messages include:

- Location (file, line, column)
- Rule violated
- Expected vs actual
- Suggestion for fix

Example:

```
Error: Atomic group spans multiple boundaries
  at flow "CheckoutFlow" line 23
  Rule: atomic_group_single_boundary
  Found: boundaries [inventory, billing]
  Fix: Split into separate steps or use same boundary
```

---

# 11. VERSION COMPATIBILITY

## 11.1 Grammar Versioning

| Version | Description |
|---------|-------------|
| v1.0 | Initial release |
| v1.1 | Added signals, scheduling, policy lifecycle |
| v1.2 | Added WorkUnitContract grammar |
| v1.3 | Added authority, mutability, relationships, invariants, storage roles |
| v1.4 | Added presentation composition, experiences, field permissions, navigation |
| v1.5 | Added form binding, session management, assets, search, external integration, streaming, offline, multi-device, IoT |
| v1.6 | Added storage binding, cache layer, resource requirements, intent routing, client cache, cache signals |

## 11.2 Compatibility Guarantee

- Minor versions (1.x) are backward compatible
- All v1.0 documents valid in v1.1 through v1.6
- Tools SHOULD support multiple versions
- Runtimes MUST declare supported versions
- v1.6 is purely additive—no breaking changes from v1.5

---

# 12. GRAMMAR COMPLETENESS

## 12.1 What This Grammar Defines

| Layer | Constructs |
|-------|------------|
| **Data Layer** | DataType, DataState, DataEvolution, Materialization, Transition, Constraint |
| **Interface Layer** | InputIntent, PresentationView, PresentationBinding |
| **Execution Layer** | SMS Flows, Work Units, Work Unit Contracts, Atomic Groups, Failure/Compensation |
| **Topology Layer** | Locations, Workers, Placement, Scaling |
| **Policy Layer** | Policies, Policy Sets, Data Policies, Lifecycle |
| **Operational Layer** | Signals, Schedulers, Failure Domains, Realms |
| **Cross-Cutting** | Versioning, Correlation, Time constraints, Idempotency |
| **Authority Layer** | Mutability, Authority, Authority State, CAS Tokens, Storage Roles |
| **Relationship Layer** | Relationships, Cardinality, Semantics (causal, ordered), Delegation |
| **Invariant Layer** | Invariant Policies, View Invariants, Availability Semantics |
| **Presentation Composition Layer** | Compositions, Experiences, Field Permissions, Navigation |
| **Application Development Layer** | Form Intent Binding, View Materialization Contracts, Intent Delivery Contracts, Session Management, Multi-Device Semantics, Offline Operation, Indicator Source Binding, Composition Fetch Semantics, Asset Management, Search and Discovery, External Integration, Webhook Receivers, Data Governance, Consent Management, Erasure Rights, Legal Hold, Graph Queries, Inference Fields, Structured Content, Spatial Types, Edge Devices, Device Capabilities, Device Telemetry, Actuator Commands |

## 12.2 What This Grammar Does NOT Define

| Category | Examples |
|----------|----------|
| **Implementation Details** | Storage engines, Serialization formats, Network protocols, Concurrency models, UI frameworks |
| **Infrastructure** | Container orchestration, Network topology, Hardware specs, Cloud providers |

These are intentionally left to runtime implementations.

---

# 13. SUMMARY

This formal grammar provides:

1. **Complete lexical specification** for tokenization
2. **Structural grammar** for all major concepts
3. **Semantic rules** for validation
4. **Formal semantics** for execution model
5. **Conformance criteria** for implementations
6. **Parser guidance** for tool builders
7. **Storage and caching primitives** for data persistence (v1.6)
8. **Intent routing** for message transport (v1.6)
9. **Resource requirements** for infrastructure (v1.6)

The grammar is:

- **Complete**: Covers all specification elements
- **Unambiguous**: One interpretation only
- **Machine-readable**: Suitable for parser generation
- **Versioned**: Supports evolution
- **Validated**: Tested against examples
- **Application-Complete**: Enables end-to-end development (v1.5)
- **Infrastructure-Ready**: Defines storage, caching, and transport primitives (v1.6)
- **Deduplicated**: Each production rule appears exactly once
- **Collection-Based**: All multi-instance types use plural key collections for valid YAML

---

**End of SMS v1.6 EBNF Grammar**

**Document Status**: Complete and Normative  
**Version**: 1.6  
**Last Updated**: 2026-02-04
