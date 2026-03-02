# System Mechanics Specification - Formal EBNF Grammar v1.3

## Overview

This document provides the complete formal grammar in Extended Backus-Naur Form (EBNF) suitable for parser generation, validation tooling, and formal analysis.

---

## LEXICAL GRAMMAR

### Basic Tokens

```ebnf
(* Characters *)
letter        = "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m" |
                "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z" |
                "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J" | "K" | "L" | "M" |
                "N" | "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X" | "Y" | "Z" ;

digit         = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;

underscore    = "_" ;

(* Identifiers *)
identifier    = letter , { letter | digit | underscore } ;

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

(* Versions *)
version       = "v" , integer ;

(* Comments *)
comment       = "//" , { ? any character ? } , newline ;
newline       = "\n" | "\r\n" ;

(* Whitespace *)
whitespace    = " " | "\t" | newline ;
```

---

## TOP-LEVEL GRAMMAR

### Document Structure

```ebnf
document      = { declaration } ;

declaration   = system_decl
              | data_decl
              | ui_decl
              | input_decl
              | sms_decl
              | wts_decl
              | policy_decl
              | signal_decl
              | realm_decl
              | work_unit_contract_decl   (* NEW in v1.2 *)
              | relationship_decl         (* NEW in v1.3 *)
              | authority_decl ;          (* NEW in v1.3 *)
```

### System Declaration

```ebnf
system_decl   = "system" , ":" , system_body ;

system_body   = "{" , { system_property } , "}" ;

system_property = "name" , ":" , string
                | "version" , ":" , version
                | "description" , ":" , string
                | "realm" , ":" , identifier ;
```

---

## DATA GRAMMAR

### DataType

```ebnf
data_decl     = datatype_decl
              | constraint_decl
              | datastate_decl
              | evolution_decl
              | materialization_decl
              | transition_decl ;

datatype_decl = "dataType" , ":" , "{" ,
                  "name" , ":" , identifier , "," ,
                  "version" , ":" , version , "," ,
                  "fields" , ":" , "{" , field_list , "}" ,
                "}" ;

field_list    = field_def , { "," , field_def } ;

field_def     = identifier , ":" , type_ref ;

type_ref      = primitive_type
              | enum_type
              | array_type
              | reference_type ;

primitive_type = "string" | "integer" | "decimal" | "boolean" | "uuid" | "timestamp" | "duration" ;

enum_type     = "enum" , "[" , enum_values , "]" ;
enum_values   = identifier , { "," , identifier } ;

array_type    = "array" , "<" , type_ref , ">" ;

reference_type = identifier ;  (* References another DataType *)
```

### Constraint

```ebnf
constraint_decl = "constraint" , ":" , "{" ,
                    "name" , ":" , identifier , "," ,
                    "appliesTo" , ":" , identifier , "," ,
                    "rules" , ":" , "{" , rule_list , "}" ,
                  "}" ;

rule_list     = rule_def , { "," , rule_def } ;

rule_def      = version , ":" , expression ;

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
```

### DataState

```ebnf
datastate_decl = "dataState" , ":" , "{" ,
                   "name" , ":" , identifier , "," ,
                   "type" , ":" , identifier , "," ,
                   "lifecycle" , ":" , lifecycle_type , "," ,
                   [ "constraintPolicy" , ":" , constraint_policy , "," ] ,
                   [ "evolutionPolicy" , ":" , evolution_policy ] ,
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

### DataEvolution

```ebnf
evolution_decl = "dataEvolution" , ":" , "{" ,
                   "name" , ":" , identifier , "," ,
                   "from" , ":" , version_ref , "," ,
                   "to" , ":" , version_ref , "," ,
                   "changes" , ":" , "[" , change_ops , "]" , "," ,
                   "compatibility" , ":" , compatibility_spec ,
                 "}" ;

version_ref   = "{" , "type" , ":" , identifier , "," , "version" , ":" , version , "}" ;

change_ops    = change_op , { "," , change_op } ;

change_op     = add_field_op
              | remove_field_op
              | rename_field_op
              | change_type_op
              | extend_enum_op
              | refine_constraint_op ;

add_field_op  = "addField" , ":" , "{" , "name" , ":" , identifier , "," , "default" , ":" , value , "}" ;

remove_field_op = "removeField" , ":" , identifier ;

extend_enum_op = "extendEnum" , ":" , "{" , "field" , ":" , identifier , "," , "values" , ":" , "[" , enum_values , "]" , "}" ;

compatibility_spec = "{" ,
                       "read" , ":" , compatibility_mode , "," ,
                       "write" , ":" , compatibility_mode ,
                     "}" ;

compatibility_mode = "backward" | "forward" | "none" ;
```

### Materialization

```ebnf
materialization_decl = "materialization" , ":" , "{" ,
                         "name" , ":" , identifier , "," ,
                         "source" , ":" , source_spec , "," ,
                         "targetState" , ":" , identifier , "," ,
                         "freshness" , ":" , freshness_spec , "," ,
                         "evolution" , ":" , mat_evolution_spec ,
                       "}" ;

source_spec   = identifier | ( "[" , identifier , { "," , identifier } , "]" ) ;

freshness_spec = "{" , "maxStaleness" , ":" , duration , "}" ;

mat_evolution_spec = "{" ,
                       "strategy" , ":" , mat_strategy , "," ,
                       "blocking" , ":" , boolean ,
                     "}" ;

mat_strategy  = "rebuild" | "incremental" ;

(* Authority properties for views - NEW in v1.3 *)
view_authority_spec = [ "authority_agnostic" , ":" , boolean , "," ] ,
                      [ "epoch_tolerant" , ":" , boolean ] ;
```

### Transition

```ebnf
transition_decl = "transition" , ":" , "{" ,
                    "name" , ":" , identifier , "," ,
                    "from" , ":" , identifier , "," ,
                    "to" , ":" , identifier , "," ,
                    "triggeredBy" , ":" , trigger_source , "," ,
                    [ "guarantees" , ":" , "[" , identifier_list , "]" , "," ] ,
                    [ "idempotency" , ":" , idempotency_spec ] ,
                  "}" ;

trigger_source = "inputIntent" | "smsFlow" | identifier ;

idempotency_spec = "{" ,
                     "key" , ":" , key_expr , "," ,
                     "scope" , ":" , idempotency_scope ,
                   "}" ;

key_expr      = expression | ( "[" , expression_list , "]" ) ;

expression_list = expression , { "," , expression } ;

idempotency_scope = "global" | "per_subject" | "per_realm" ;
```

---

## AUTHORITY & MUTABILITY GRAMMAR (NEW in v1.3)

### Mutability Declaration

```ebnf
mutability_decl = "mutability" , ":" , "{" ,
                    "scope" , ":" , mutability_scope , "," ,
                    "exclusive" , ":" , boolean , "," ,
                    [ "authority" , ":" , identifier ] ,
                  "}" ;

mutability_scope = "entity" | "append-only" ;
```

### Authority Declaration

```ebnf
authority_decl = "authority" , ":" , "{" ,
                   authority_scope , "," ,
                   authority_resolution , "," ,
                   [ authority_migration ] ,
                 "}" ;

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

### Authority State

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

### Storage Role

```ebnf
storage_role_decl = "storage" , ":" , "{" ,
                      "role" , ":" , storage_role_type , "," ,
                      "rebuildable" , ":" , boolean ,
                    "}" ;

storage_role_type = "data" | "control" ;
```

### CAS Token

```ebnf
cas_decl = "cas" , ":" , "{" ,
             "token_fields" , ":" , "[" , identifier_list , "]" ,
           "}" ;
```

---

## RELATIONSHIP GRAMMAR (NEW in v1.3)

### Relationship Declaration

```ebnf
relationship_decl = "relationship" , ":" , "{" ,
                      "name" , ":" , identifier , "," ,
                      "from" , ":" , identifier , "," ,
                      "to" , ":" , identifier , "," ,
                      [ "type" , ":" , identifier , "," ] ,
                      [ "cardinality" , ":" , cardinality_type , "," ] ,
                      [ "semantics" , ":" , relationship_semantics ] ,
                    "}" ;

cardinality_type = "one-to-one" | "one-to-many" | "many-to-one" | "many-to-many" ;

relationship_semantics = "{" ,
                           [ "causal" , ":" , boolean , "," ] ,
                           [ "ordered" , ":" , boolean ] ,
                         "}" ;
```

---

## INVARIANT GRAMMAR (NEW in v1.3)

### Invariant Policy Declaration

```ebnf
invariant_policy_decl = "invariants" , ":" , "{" ,
                          "enforced_by" , ":" , invariant_enforcer , "," ,
                          [ "description" , ":" , string ] ,
                        "}" ;

invariant_enforcer = "view" | "policy" ;
```

### View Invariant Extension

```ebnf
view_invariant = "invariant" , ":" , "{" ,
                   "expression" , ":" , expression , "," ,
                   [ "on_violation" , ":" , violation_action ] ,
                 "}" ;

violation_action = "reject" | "flag" | "log" ;
```

---

## AVAILABILITY GRAMMAR (NEW in v1.3)

### Availability Declaration

```ebnf
availability_decl = "availability" , ":" , "{" ,
                      "write_scope" , ":" , availability_scope , "," ,
                      "pause_allowed" , ":" , boolean , "," ,
                      "read_continuity" , ":" , continuity_level ,
                    "}" ;

availability_scope = "entity" | "model" | "region" ;

continuity_level = "best-effort" | "guaranteed" ;
```

### Write Failure Strategy

```ebnf
write_failure_decl = "write_failure" , ":" , "{" ,
                       "strategy" , ":" , failure_strategy ,
                     "}" ;

failure_strategy = "reject" | "client_queue" | "regional_buffer" ;
```

---

## PRESENTATION (UI) GRAMMAR

### PresentationView

```ebnf
ui_decl       = presentation_view_decl
              | presentation_binding_decl ;

presentation_view_decl = "presentationView" , ":" , "{" ,
                           "name" , ":" , identifier , "," ,
                           "version" , ":" , version , "," ,
                           "consumes" , ":" , consumes_spec , "," ,
                           [ "guarantees" , ":" , "[" , string_list , "]" , "," ] ,
                           [ "interactionModel" , ":" , interaction_model ] ,
                         "}" ;

consumes_spec = "{" ,
                  "dataState" , ":" , identifier , "," ,
                  "version" , ":" , version ,
                "}" ;

interaction_model = "{" , { interaction_element } , "}" ;

interaction_element = "badges" , ":" , "[" , badge_list , "]"
                    | "actions" , ":" , "[" , action_list , "]" ;

badge_list    = badge , { "," , badge } ;

badge         = "{" ,
                  "if" , ":" , expression , "," ,
                  "label" , ":" , string , "," ,
                  [ "color" , ":" , string ] ,
                "}" ;

action_list   = action , { "," , action } ;

action        = "{" ,
                  "if" , ":" , expression , "," ,
                  "action" , ":" , identifier , "," ,
                  "label" , ":" , string ,
                "}" ;
```

### PresentationBinding

```ebnf
presentation_binding_decl = "presentationBinding" , ":" , "{" ,
                              "name" , ":" , identifier , "," ,
                              "rules" , ":" , "[" , binding_rules , "]" ,
                            "}" ;

binding_rules = binding_rule , { "," , binding_rule } ;

binding_rule  = "{" ,
                  "if" , ":" , expression , "," ,
                  "use" , ":" , view_ref ,
                "}"
              | "{" , "else" , ":" , "{" , "use" , ":" , view_ref , "}" , "}" ;

view_ref      = identifier , [ version ] ;
```

---

## INPUT (FORM) GRAMMAR

### InputIntent

```ebnf
input_decl    = input_intent_decl ;

input_intent_decl = "inputIntent" , ":" , "{" ,
                      "name" , ":" , identifier , "," ,
                      "version" , ":" , version , "," ,
                      "proposes" , ":" , proposes_spec , "," ,
                      "supplies" , ":" , supplies_spec , "," ,
                      "constraints" , ":" , input_constraints , "," ,
                      "transitions" , ":" , transition_target ,
                    "}" ;

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

identifier_list = identifier , { "," , identifier } ;
```

---

## SMS (FLOW) GRAMMAR

### Flow Declaration

```ebnf
sms_decl      = "sms" , ":" , "{" ,
                  "flows" , ":" , "[" , flow_list , "]" ,
                "}" ;

flow_list     = flow_def , { "," , flow_def } ;

flow_def      = "{" ,
                  "name" , ":" , identifier , "," ,
                  [ "purpose" , ":" , string , "," ] ,
                  "triggeredBy" , ":" , trigger_spec , "," ,
                  "steps" , ":" , "[" , step_list , "]" , "," ,
                  [ "authorization" , ":" , auth_spec , "," ] ,
                  [ "onPolicyChange" , ":" , policy_change_action , "," ] ,
                  [ "failure" , ":" , failure_spec ] ,
                "}" ;

trigger_spec  = "{" , "inputIntent" , ":" , identifier , "}" ;

step_list     = step_def , { "," , step_def } ;

step_def      = simple_step
              | atomic_group
              | parallel_group
              | conditional_step ;

simple_step   = "{" ,
                  "work_unit" , ":" , identifier , "," ,
                  [ "boundary" , ":" , identifier ] ,
                "}"
              | identifier , ":" , expression ;  (* Operation like validate/enrich/persist *)

auth_spec     = "{" , "policySet" , ":" , identifier , "}" ;

policy_change_action = "revalidate" | "drain" | "fail" ;
```

### Atomic Group

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

### Failure & Compensation

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

---

## WORK UNIT CONTRACT GRAMMAR (NEW in v1.2)

### Overview

Work Unit Contracts define typed contracts for work units, providing explicit input/output schemas, 
side effects, pre/postconditions, error definitions, and dependency specifications. This addresses 
the gap where work units are named in SMS flows but not formally specified.

### WorkUnitContract Declaration

```ebnf
work_unit_contract_decl = "workUnitContract" , ":" , "{" ,
                            "name" , ":" , identifier , "," ,
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
                            [ "idempotency" , ":" , wuc_idempotency_spec ] ,
                          "}" ;
```

### Input Specification

```ebnf
wuc_input_spec = "{" ,
                   "schema" , ":" , inline_schema , "," ,
                   [ "from" , ":" , identifier , "," ] ,  (* Optional DataState reference *)
                   [ "validation" , ":" , wuc_validation_spec ] ,
                 "}" ;

inline_schema = "{" , field_list , "}" ;

wuc_validation_spec = "{" ,
                        "timing" , ":" , validation_timing , "," ,
                        [ "on_failure" , ":" , validation_failure_action ] ,
                      "}" ;

validation_timing = "immediate" | "deferred" | "lazy" ;

validation_failure_action = "reject" | "warn" | "log" ;
```

### Output Specification

```ebnf
wuc_output_spec = "{" ,
                    "schema" , ":" , inline_schema , "," ,
                    [ "to" , ":" , identifier , "," ] ,  (* Optional DataState reference *)
                    [ "cardinality" , ":" , output_cardinality ] ,
                  "}" ;

output_cardinality = "one" | "zero_or_one" | "many" | "stream" ;
```

### Side Effects

```ebnf
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
```

### Error Definitions

```ebnf
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

error_category = "validation" | "authorization" | "not_found" | "conflict" |
                 "timeout" | "unavailable" | "resource_exhausted" | 
                 "internal" | "external_dependency" ;
```

### Dependencies

```ebnf
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
```

### Idempotency for Work Unit Contracts

```ebnf
wuc_idempotency_spec = "{" ,
                         "strategy" , ":" , idempotency_strategy , "," ,
                         [ "key" , ":" , key_expr , "," ] ,
                         [ "scope" , ":" , idempotency_scope , "," ] ,
                         [ "ttl" , ":" , duration ] ,
                       "}" ;
```

### Contract Composition

```ebnf
(* Contracts can extend or compose with other contracts *)
contract_extension = "extends" , ":" , "[" , contract_ref_list , "]" ;

contract_ref_list = contract_ref , { "," , contract_ref } ;

contract_ref = identifier , [ version ] ;
```

---

## WTS (WORKER TOPOLOGY) GRAMMAR

### Topology Declaration

```ebnf
wts_decl      = "topology" , ":" , "{" , topology_body , "}" 
              | "wts" , ":" , "{" , wts_body , "}" ;

topology_body = { topology_element } ;

topology_element = location_decl
                 | worker_class_decl
                 | scheduler_decl
                 | placement_decl
                 | scaling_policy_decl
                 | failure_domain_decl ;

wts_body      = "workers" , ":" , "[" , worker_list , "]" , "," ,
                [ "routing" , ":" , routing_spec ] ;
```

### Location

```ebnf
location_decl = "location" , identifier , ":" , "{" , location_body , "}" ;

location_body = "scope" , ":" , scope_type , "," ,
                [ "capabilities" , ":" , "[" , identifier_list , "]" , "," ] ,
                [ "latency_to_user" , ":" , latency_class , "," ] ,
                [ "provisioner" , ":" , identifier , "," ] ,
                [ "provisionable" , ":" , boolean ] ;

scope_type    = "global" | "region" | "zone" | "cluster" | "runtime" ;

latency_class = "minimal" | "low" | "medium" | "high" ;
```

### Worker Class

```ebnf
worker_class_decl = "worker_class" , identifier , ":" , "{" , worker_class_body , "}" ;

worker_class_body = "runtime" , ":" , identifier , "," ,
                    "supports" , ":" , "[" , identifier_list , "]" , "," ,
                    [ "transport" , ":" , "[" , transport_list , "]" , "," ] ,
                    [ "resources" , ":" , resources_spec ] ;

transport_list = transport_type , { "," , transport_type } ;

transport_type = "nats" | "http" | "grpc" | "unix_socket" | "in_process" | "websocket" ;

resources_spec = "{" , { resource_property } , "}" ;

resource_property = identifier , ":" , ( number | string ) , [ "," ] ;
```

### Placement

```ebnf
placement_decl = "placement" , identifier , ":" , "{" ,
                   "worker_class" , ":" , identifier , "," ,
                   "locations" , ":" , "[" , identifier_list , "]" , "," ,
                   "min" , ":" , integer , "," ,
                   "max" , ":" , integer , "," ,
                   [ "preferred" , ":" , identifier ] ,
                 "}" ;
```

### Scaling Policy

```ebnf
scaling_policy_decl = "scaling_policy" , identifier , ":" , "{" ,
                        "target" , ":" , identifier , "," ,
                        "signals" , ":" , "{" , signal_rules , "}" , "," ,
                        [ "cooldown" , ":" , duration ] ,
                      "}" ;

signal_rules  = signal_rule , { "," , signal_rule } ;

signal_rule   = expression , "=>" , scale_action , [ "," ] ;

scale_action  = "scale_up" | "scale_down" | "hold" ;
```

### Worker & Routing

```ebnf
worker_list   = worker_decl , { "," , worker_decl } ;

worker_decl   = "{" ,
                  "name" , ":" , identifier , "," ,
                  "acceptsInputs" , ":" , "[" , input_refs , "]" , "," ,
                  "produces" , ":" , "[" , identifier_list , "]" , "," ,
                  "compatibleVersions" , ":" , "[" , version_list , "]" ,
                "}" ;

input_refs    = input_ref , { "," , input_ref } ;

input_ref     = identifier , [ version ] ;

version_list  = version , { "," , version } ;

routing_spec  = "{" ,
                  "strategy" , ":" , routing_strategy , "," ,
                  "rules" , ":" , "[" , routing_rules , "]" ,
                "}" ;

routing_strategy = "versioned" | "capability" | "load" ;

routing_rules = routing_rule , { "," , routing_rule } ;

routing_rule  = "{" ,
                  "if" , ":" , expression , "," ,
                  "routeTo" , ":" , identifier ,
                "}"
              | "{" , "else" , ":" , "{" , "routeTo" , ":" , identifier , "}" , "}" ;
```

### Scheduler

```ebnf
scheduler_decl = "scheduler" , identifier , ":" , "{" ,
                   "scope" , ":" , scope_type , "," ,
                   "authority" , ":" , boolean , "," ,
                   "observes" , ":" , "[" , identifier_list , "]" ,
                 "}" ;
```

### Failure Domain

```ebnf
failure_domain_decl = "failure_domain" , identifier , ":" , "{" ,
                        "scope" , ":" , scope_type , "," ,
                        "affected_locations" , ":" , "[" , identifier_list , "]" ,
                      "}" ;
```

---

## POLICY & AUTHORIZATION GRAMMAR

### Policy Declaration

```ebnf
policy_decl   = policy_def
              | policy_set_def
              | data_policy_def
              | policy_artifact_def ;

policy_def    = "policy" , ":" , "{" ,
                  "name" , ":" , identifier , "," ,
                  "version" , ":" , version , "," ,
                  "appliesTo" , ":" , applies_to_spec , "," ,
                  "effect" , ":" , effect_type , "," ,
                  "when" , ":" , expression ,
                "}" ;

applies_to_spec = "{" ,
                    "type" , ":" , policy_target_type , "," ,
                    "name" , ":" , identifier ,
                  "}" ;

policy_target_type = "inputIntent" | "dataState" | "transition" | 
                     "materialization" | "presentationView" | "smsFlow" ;

effect_type   = "allow" | "deny" ;
```

### Policy Set

```ebnf
policy_set_def = "policySet" , ":" , "{" ,
                   "name" , ":" , identifier , "," ,
                   "resolution" , ":" , resolution_strategy , "," ,
                   "policies" , ":" , "[" , policy_refs , "]" ,
                 "}" ;

resolution_strategy = "deny_overrides" | "allow_overrides" ;

policy_refs   = policy_ref , { "," , policy_ref } ;

policy_ref    = identifier , [ version ] ;
```

### Data Policy

```ebnf
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
```

### Policy Artifact

```ebnf
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
```

### Subject, Role, Attribute

```ebnf
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

---

## SIGNALS & SCHEDULING GRAMMAR

### Signal Definition

```ebnf
signal_decl   = "signal" , ":" , "{" ,
                  "type" , ":" , signal_type , "," ,
                  "source" , ":" , signal_source , "," ,
                  [ "subject" , ":" , signal_subject , "," ] ,
                  "metrics" , ":" , "{" , metric_map , "}" , "," ,
                  "severity" , ":" , severity_level , "," ,
                  "timestamp" , ":" , string ,
                "}" ;

signal_type   = "capacity" | "health" | "latency" | "saturation" | 
                "policy" | "backpressure" | "error" ;

signal_source = "{" ,
                  "component" , ":" , component_type , "," ,
                  "name" , ":" , identifier , "," ,
                  [ "region" , ":" , identifier ] ,
                "}" ;

component_type = "worker" | "sms" | "ui" | "infra" ;

signal_subject = "{" ,
                   [ "intent" , ":" , identifier , "," ] ,
                   [ "dataState" , ":" , identifier ] ,
                 "}" ;

metric_map    = metric_entry , { "," , metric_entry } ;

metric_entry  = identifier , ":" , ( number | string ) ;

severity_level = "info" | "warn" | "critical" ;
```

---

## CROSS-CUTTING GRAMMAR

### Execution Context

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
```

### Time Constraint

```ebnf
time_constraint = "timeConstraint" , ":" , "{" ,
                    "type" , ":" , time_type , "," ,
                    "value" , ":" , duration ,
                  "}" ;

time_type     = "ttl" | "deadline" | "staleness" ;
```

### Idempotency

```ebnf
idempotency_config = "idempotency" , ":" , "{" ,
                       "strategy" , ":" , idempotency_strategy , "," ,
                       "key" , ":" , key_expression , "," ,
                       "scope" , ":" , idempotency_scope ,
                     "}" ;

idempotency_strategy = "deterministic_key" | "timestamp" | "content_hash" ;
```

### Execution Outcome

```ebnf
execution_outcome = "executionOutcome" , ":" , "{" ,
                      "type" , ":" , outcome_type , "," ,
                      "retryable" , ":" , boolean , "," ,
                      "reason" , ":" , string ,
                    "}" ;

outcome_type  = "success" | "rejection" | "failure" ;
```

### Realm

```ebnf
realm_decl    = "realm" , ":" , "{" ,
                  "name" , ":" , identifier , "," ,
                  "isolation" , ":" , isolation_spec ,
                "}" ;

isolation_spec = "{" ,
                   "data" , ":" , isolation_level , "," ,
                   "policy" , ":" , isolation_level , "," ,
                   "signals" , ":" , signal_isolation ,
                 "}" ;

isolation_level = "strict" | "shared" ;

signal_isolation = "shared" | "isolated" ;
```

---

## VALIDATION RULES (Semantic Constraints)

### Must-Pass Rules

These rules MUST be enforced by conformant parsers/validators:

```ebnf
(* Structural Rules *)
valid_flow             = flow_has_steps 
                       ∧ no_duplicate_work_units_in_atomic_group
                       ∧ atomic_group_has_boundary ;

(* Boundary Rules *)
valid_atomic_group     = atomic_group_single_boundary
                       ∧ atomic_group_no_cross_boundary_steps ;

(* Policy Rules *)
valid_policy           = policy_has_effect
                       ∧ policy_target_exists
                       ∧ policy_expression_valid ;

(* Evolution Rules *)
valid_evolution        = no_version_skip
                       ∧ compatible_changes_only ;

(* Authorization Rules *)
valid_auth             = one_authority_per_scope
                       ∧ deny_overrides_allow ;

(* Idempotency Rules *)
valid_idempotency      = retry_implies_idempotent
                       ∧ key_deterministic ;

(* Reference Rules *)
valid_references       = all_work_units_defined
                       ∧ all_data_states_defined
                       ∧ all_policies_defined
                       ∧ all_boundaries_defined ;

(* Work Unit Contract Rules - NEW in v1.2 *)
valid_work_unit_contract = contract_has_name
                         ∧ contract_has_version
                         ∧ contract_has_input_schema
                         ∧ contract_has_output_schema
                         ∧ input_from_references_valid_datastate
                         ∧ output_to_references_valid_datastate
                         ∧ retryable_errors_have_backoff
                         ∧ side_effects_have_targets
                         ∧ database_writes_have_idempotency ;

(* Contract Coverage Rules *)
valid_contract_coverage = flow_work_units_have_contracts
                        ∧ contract_versions_match_references ;

(* Authority Rules - NEW in v1.3 *)
valid_authority        = one_authority_per_entity
                       ∧ authority_epoch_monotonic
                       ∧ no_overlapping_authority
                       ∧ cas_token_valid ;

(* Relationship Rules - NEW in v1.3 *)
valid_relationship     = relationship_from_exists
                       ∧ relationship_to_exists
                       ∧ cardinality_constraint_satisfied
                       ∧ no_synchronous_coordination_implied ;

(* Invariant Rules - NEW in v1.3 *)
valid_invariant        = invariants_enforced_by_view_or_policy
                       ∧ cross_entity_invariants_in_views
                       ∧ views_authority_agnostic ;

(* Storage Role Rules - NEW in v1.3 *)
valid_storage_role     = control_storage_strongly_consistent
                       ∧ control_storage_designated_writers
                       ∧ data_storage_rebuildable ;
```

---

## FORMAL SEMANTICS

### Execution Semantics

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

## GRAMMAR COMPLETENESS

### What This Grammar Defines

✅ **Data Layer**
- Structure (DataType)
- Lifecycle (DataState)
- Evolution (DataEvolution)
- Derivation (Materialization)
- Transitions (Transition)
- Constraints (Constraint)

✅ **Interface Layer**
- Input (InputIntent)
- Output (PresentationView)
- Binding (PresentationBinding)

✅ **Execution Layer**
- Flows (SMS)
- Work Units
- Work Unit Contracts (NEW in v1.2)
- Atomic Groups
- Failure/Compensation

✅ **Topology Layer**
- Locations
- Workers
- Placement
- Scaling

✅ **Policy Layer**
- Policies
- Policy Sets
- Data Policies
- Lifecycle

✅ **Operational Layer**
- Signals
- Schedulers
- Failure Domains
- Realms

✅ **Cross-Cutting**
- Versioning
- Correlation
- Time constraints
- Idempotency

✅ **Authority Layer (NEW in v1.3)**
- Mutability
- Authority
- Authority State
- Authority Transitions
- CAS Tokens
- Storage Roles

✅ **Relationship Layer (NEW in v1.3)**
- Relationships
- Cardinality
- Semantics (causal, ordered)

✅ **Invariant Layer (NEW in v1.3)**
- Invariant Policies
- View Invariants
- Availability Semantics

### What This Grammar Does NOT Define

❌ **Implementation Details**
- Storage engines
- Serialization formats
- Network protocols
- Concurrency models
- UI frameworks

❌ **Infrastructure**
- Container orchestration
- Network topology
- Hardware specs
- Cloud providers

These are intentionally left to runtime implementations.

---

## PARSER IMPLEMENTATION GUIDANCE

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

## CONFORMANCE

### Parser Conformance

A conformant parser MUST:
1. Accept all valid specifications
2. Reject all invalid specifications
3. Report errors with location and reason
4. Support all grammar elements
5. Enforce all semantic rules

### Runtime Conformance

A conformant runtime MUST:
1. Execute flows as defined
2. Enforce atomic groups correctly
3. Evaluate policies locally
4. Respect version compatibility
5. Implement idempotency
6. Propagate correlation context
7. Emit required signals

### Tooling Conformance

Conformant tools MUST:
1. Parse specifications correctly
2. Validate against semantic rules
3. Generate conformant code
4. Preserve intent in transformations

---

## VERSION COMPATIBILITY

### Grammar Versioning

- v1.0: Initial release
- v1.1: Added signals, scheduling, policy lifecycle
- v1.2: Added WorkUnitContract grammar for typed work unit specifications
- v1.3: Added authority, mutability, relationships, invariants, storage roles

### v1.3 Additions

The following grammar elements were added in v1.3:

1. **Authority Grammar**: Entity-scoped authority with migration support
2. **Mutability Grammar**: Exclusive write semantics declaration
3. **Relationship Grammar**: Semantic relationships with cardinality
4. **Invariant Grammar**: View-level invariant enforcement
5. **Availability Grammar**: Entity-scoped availability semantics
6. **Storage Role Grammar**: Control vs data storage distinction
7. **CAS Token Grammar**: Compare-and-set token structure

### Compatibility Guarantee

- Minor versions (1.x) are backward compatible
- All v1.0 documents valid in v1.1, v1.2, and v1.3
- Tools SHOULD support multiple versions
- Runtimes MUST declare supported versions

---

## SUMMARY

This formal grammar provides:

1. **Complete lexical specification** for tokenization
2. **Structural grammar** for all major concepts
3. **Semantic rules** for validation
4. **Formal semantics** for execution model
5. **Conformance criteria** for implementations
6. **Parser guidance** for tool builders

The grammar is:
- **Complete**: Covers all specification elements
- **Unambiguous**: One interpretation only
- **Machine-readable**: Suitable for parser generation
- **Versioned**: Supports evolution
- **Validated**: Tested against examples

This is the authoritative grammar for System Mechanics Specification v1.3.

### New in v1.2: WorkUnitContract

The WorkUnitContract grammar extension addresses the critical gap where work units referenced 
in SMS flows were named but not formally specified. This enables:

1. **Type-safe code generation** from contract definitions
2. **Compile-time validation** of flow steps against contracts
3. **Contract coverage reporting** for work units in flows
4. **Explicit side effect documentation** for operational awareness
5. **Formal error handling** with retry strategies

### New in v1.3: Authority, Relationships, and Invariants

The v1.3 grammar additions address multi-region survivability and invariant-oriented modeling:

1. **Entity-Scoped Authority**: Write authority resolved per entity, not model
2. **Authority Transitions**: Safe, observable transfer of authority between regions
3. **Mutability Semantics**: Exclusive write authority declaration
4. **Storage Roles**: Distinguish control state from data state
5. **Relationship Grammar**: Semantic linkage with cardinality and causality
6. **Invariant Enforcement**: View-level invariant validation
7. **Availability Semantics**: Entity-scoped write availability
8. **CAS Tokens**: Linearizable writes across authority migrations

These additions enable:
- Multi-region deployments with formal failure semantics
- Invariant-oriented data modeling without global transactions
- Safe authority mobility for follow-the-sun workloads
- Formal separation of control and data planes

