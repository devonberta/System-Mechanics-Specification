# System Mechanics Specification - Formal EBNF Grammar v1.5

## Overview

This document provides the complete formal grammar in Extended Backus-Naur Form (EBNF) suitable for parser generation, validation tooling, and formal analysis. This v1.5 grammar extends v1.4 with additional constructs for complete end-to-end application development.

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
              | authority_decl            (* NEW in v1.3 *)
              | presentation_composition_decl  (* NEW in v1.4 *)
              | experience_decl           (* NEW in v1.4 *)
              | search_intent_decl        (* NEW in v1.5 *)
              | webhook_receiver_decl     (* NEW in v1.5 *)
              | external_dependency_decl  (* NEW in v1.5 *)
              | graph_query_decl          (* NEW in v1.5 *)
              | consent_record_decl       (* NEW in v1.5 *)
              | erasure_request_decl      (* NEW in v1.5 *)
              | legal_hold_decl           (* NEW in v1.5 *)
              | device_capability_decl    (* NEW in v1.5 *)
              | device_telemetry_decl     (* NEW in v1.5 *)
              | actuator_command_decl ; (* NEW in v1.5 *)
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

field_def     = identifier , ":" , type_ref , [ field_extensions ] ;  (* NEW in v1.5: field extensions *)

type_ref      = primitive_type
              | enum_type
              | array_type
              | reference_type ;

(* NEW in v1.5: Field extensions for asset, search, inference, structured content, spatial *)
field_extensions = [ asset_spec ] , [ search_spec ] , [ inference_spec ] , [ structured_content_spec ] , [ point_spec ] ;

primitive_type = "string" | "integer" | "decimal" | "boolean" | "uuid" | "timestamp" | "duration" | "asset" | "point" | "structured_content" ; (* NEW in v1.5: asset, point, structured_content types *)

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
                   [ "evolutionPolicy" , ":" , evolution_policy , "," ] ,
                   [ "governance" , ":" , governance_spec ] ,             (* NEW in v1.5 *)
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

### Data Governance Semantics (NEW in v1.5)

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

### Asset Field Semantics (NEW in v1.5)

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

size_expression = integer , ( "KB" | "MB" | "GB" ) ;

mime_type_list = string , { "," , string } ;

extension_list = string , { "," , string } ;

asset_lifecycle = "{" ,
                    "type" , ":" , lifecycle_type ,
                    [ "," , "expires_after" , ":" , duration ] ,
                  "}" ;

lifecycle_type = "permanent" | "temporary" | "expiring" ;

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
```

### Search Field Semantics (NEW in v1.5)

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

### Inference Field Semantics (NEW in v1.5)

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

inference_freshness = "{" ,
                        "strategy" , ":" , inference_strategy ,
                      "}" ;

inference_strategy = "on_write" | "on_read" | "scheduled" | "manual" ;

inference_fallback = "{" ,
                       "on_unavailable" , ":" , fallback_action ,
                     "}" ;
```

### Structured Content Field Semantics (NEW in v1.5)

```ebnf
structured_content_spec = "structured_content" , ":" , "{" ,
                            "schema" , ":" , identifier , "," ,
                            "blocks" , ":" , content_blocks_spec , "," ,
                            "marks" , ":" , content_marks_spec ,
                            [ "," , "constraints" , ":" , content_constraints ] ,
                          "}" ;

content_blocks_spec = "{" ,
                        "allowed" , ":" , "[" , block_type_list , "]" ,
                      "}" ;

block_type_list = block_type , { "," , block_type } ;

block_type = "paragraph" | "heading" | "list" | "blockquote" | "table" | 
             "image" | "video" | "poll" | "code" | "divider" | "embed" ;

content_marks_spec = "{" ,
                       "allowed" , ":" , "[" , mark_type_list , "]" ,
                     "}" ;

mark_type_list = mark_type , { "," , mark_type } ;

mark_type = "bold" | "italic" | "underline" | "strikethrough" | "code" | 
            "link" | "mention" | "hashtag" | "highlight" ;

content_constraints = "{" ,
                        [ "max_length" , ":" , integer ] ,
                      "}" ;
```

### Spatial Field Semantics (NEW in v1.5)

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

### Materialization

```ebnf
materialization_decl = "materialization" , ":" , "{" ,
                         "name" , ":" , identifier , "," ,
                         "source" , ":" , source_spec , "," ,
                         "targetState" , ":" , identifier , "," ,
                         "freshness" , ":" , freshness_spec , "," ,
                         "evolution" , ":" , mat_evolution_spec ,
                         [ "," , "retrieval" , ":" , mat_retrieval_spec ] ,     (* NEW in v1.5 *)
                         [ "," , "temporal" , ":" , mat_temporal_spec ] ,       (* NEW in v1.5 *)
                         [ "," , "search_index" , ":" , search_index_spec ] ,   (* NEW in v1.5 *)
                         [ "," , "external_source" , ":" , external_source_spec ] , (* NEW in v1.5 *)
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

### Materialization Retrieval Contract (NEW in v1.5)

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

order_direction = "asc" | "desc" ;

retrieval_freshness = "{" , "max_staleness" , ":" , duration , "}" ;

unavailable_strategy = "{" ,
                         "strategy" , ":" , unavail_action , "," ,
                         [ "cache_ttl" , ":" , duration , "," ] ,
                         [ "degrade_to" , ":" , identifier , "," ] ,
                         [ "wait_timeout" , ":" , duration ] ,
                       "}" ;

unavail_action = "use_cached" | "fail" | "degrade" | "wait" ;
```

### Temporal Window Semantics (NEW in v1.5)

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

### Search Index Configuration (NEW in v1.5)

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

### External Source for Materialization (NEW in v1.5)

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

### Delegation Semantics (NEW in v1.5)

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

delegation_requires = "{" ,
                        "approval" , ":" , approval_type ,
                      "}" ;

approval_type = "grantor" | "grantee" | "both" | "none" ;

delegation_validity = "{" ,
                        "type" , ":" , validity_type , "," ,
                        [ "duration" , ":" , duration ] ,
                      "}" ;

validity_type = "temporal" | "permanent" | "conditional" ;

delegation_revocation = "{" ,
                          "by" , ":" , revocation_actor ,
                        "}" ;

revocation_actor = "grantor" | "grantee" | "either" | "admin" ;
```

---

## GRAPH QUERY GRAMMAR (NEW in v1.5)

### Graph Query Declaration

```ebnf
graph_query_decl = "graphQuery" , ":" , "{" ,
                     "name" , ":" , identifier , "," ,
                     "version" , ":" , version , "," ,
                     [ "description" , ":" , string , "," ] ,
                     "traverses" , ":" , identifier , "," ,
                     "start" , ":" , graph_start_spec , "," ,
                     [ "pattern" , ":" , graph_pattern_spec , "," ] ,
                     [ "filters" , ":" , "[" , graph_filter_list , "]" , "," ] ,
                     [ "limits" , ":" , graph_limits_spec ] ,
                   "}" ;

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
                           [ "description" , ":" , string , "," ] ,
                           "consumes" , ":" , consumes_spec , "," ,
                           [ "guarantees" , ":" , "[" , string_list , "]" , "," ] ,
                           [ "interactionModel" , ":" , interaction_model , "," ] ,
                           [ "fieldPermissions" , ":" , "[" , field_permission_list , "]" , "," ] ,  (* NEW in v1.4 *)
                           [ "triggers" , ":" , view_triggers ] ,  (* NEW in v1.5 *)
                         "}" ;

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

### Field Permissions (NEW in v1.4)

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

### View Triggers (NEW in v1.5)

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
                      [ "," , "delivery" , ":" , intent_delivery ] ,     (* NEW in v1.5 *)
                      [ "," , "response" , ":" , intent_response ] ,     (* NEW in v1.5 *)
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

### Intent Delivery Contract (NEW in v1.5)

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
                    [ "resources" , ":" , resources_spec , "," ] ,
                    [ "device" , ":" , device_spec ] ;            (* NEW in v1.5 *)

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

## DEVICE & IOT GRAMMAR (NEW in v1.5)

### Device Specification for Worker Class

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

### Device Capability Declaration

```ebnf
device_capability_decl = "deviceCapability" , ":" , "{" ,
                           "name" , ":" , identifier , "," ,
                           "version" , ":" , version , "," ,
                           "type" , ":" , capability_type , "," ,
                           ( sensor_config | actuator_config ) ,
                           [ "," , "safety" , ":" , capability_safety ] ,
                         "}" ;

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

### Device Telemetry Declaration

```ebnf
device_telemetry_decl = "deviceTelemetry" , ":" , "{" ,
                          "name" , ":" , identifier , "," ,
                          "version" , ":" , version , "," ,
                          "source" , ":" , telemetry_source , "," ,
                          "collection" , ":" , telemetry_collection , "," ,
                          [ "buffering" , ":" , telemetry_buffering , "," ] ,
                          [ "delivery" , ":" , telemetry_delivery ] ,
                        "}" ;

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

### Actuator Command Declaration

```ebnf
actuator_command_decl = "actuatorCommand" , ":" , "{" ,
                          "name" , ":" , identifier , "," ,
                          "version" , ":" , version , "," ,
                          "target" , ":" , command_target , "," ,
                          "command" , ":" , command_spec , "," ,
                          [ "safety" , ":" , command_safety , "," ] ,
                          [ "delivery" , ":" , command_delivery ] ,
                        "}" ;

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

## DATA GOVERNANCE & CONSENT GRAMMAR (NEW in v1.5)

### Consent Record Declaration

```ebnf
consent_record_decl = "consentRecord" , ":" , "{" ,
                        "name" , ":" , identifier , "," ,
                        "version" , ":" , version , "," ,
                        [ "description" , ":" , string , "," ] ,
                        "subject_field" , ":" , identifier , "," ,
                        "scopes" , ":" , "[" , consent_scope_list , "]" , "," ,
                        [ "purposes" , ":" , "[" , purpose_list , "]" , "," ] ,
                        [ "lifecycle" , ":" , consent_lifecycle , "," ] ,
                        [ "withdrawal" , ":" , consent_withdrawal ] ,
                      "}" ;

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

### Erasure Request Declaration

```ebnf
erasure_request_decl = "erasureRequest" , ":" , "{" ,
                         "name" , ":" , identifier , "," ,
                         "version" , ":" , version , "," ,
                         [ "description" , ":" , string , "," ] ,
                         "scope" , ":" , erasure_scope , "," ,
                         "strategy" , ":" , erasure_strategy , "," ,
                         [ "cascade" , ":" , erasure_cascade , "," ] ,
                         [ "verification" , ":" , erasure_verification , "," ] ,
                         [ "processing" , ":" , erasure_processing ] ,
                       "}" ;

erasure_scope = "{" ,
                  "subject_field" , ":" , identifier ,
                "}" ;

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

### Legal Hold Declaration

```ebnf
legal_hold_decl = "legalHold" , ":" , "{" ,
                    "name" , ":" , identifier , "," ,
                    "version" , ":" , version , "," ,
                    "scope" , ":" , legal_hold_scope , "," ,
                    "reason" , ":" , string , "," ,
                    "reference" , ":" , string , "," ,
                    "custodian" , ":" , identifier , "," ,
                    "preserves" , ":" , "[" , preserve_spec_list , "]" , "," ,
                    "overrides" , ":" , legal_hold_overrides , "," ,
                    [ "lifecycle" , ":" , legal_hold_lifecycle ] ,
                  "}" ;

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

## PRESENTATION COMPOSITION GRAMMAR (NEW in v1.4)

### PresentationComposition Declaration

```ebnf
presentation_composition_decl = "presentationComposition" , ":" , "{" ,
                                  "name" , ":" , identifier , "," ,
                                  "version" , ":" , version , "," ,
                                  [ "description" , ":" , string , "," ] ,
                                  "primary" , ":" , identifier , "," ,
                                  [ "navigation" , ":" , composition_navigation , "," ] ,
                                  [ "related" , ":" , "[" , related_view_list , "]" , "," ] ,
                                  [ "fetch" , ":" , composition_fetch_spec ] ,   (* NEW in v1.5 *)
                                "}" ;
```

### Composition Navigation

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
                   [ "," , "name" , ":" , identifier ] ,            (* NEW in v1.5 *)
                   [ "," , "type" , ":" , indicator_type ] ,        (* NEW in v1.5 *)
                   [ "," , "source" , ":" , indicator_source ] ,    (* NEW in v1.5 *)
                   [ "," , "update" , ":" , indicator_update ] ,    (* NEW in v1.5 *)
                   [ "," , "display" , ":" , indicator_display ] ,  (* NEW in v1.5 *)
                 "}" ;

(* NEW in v1.5: Enhanced indicator grammar *)
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

### Related Views

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

### Composition Fetch Semantics (NEW in v1.5)

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

---

## EXPERIENCE GRAMMAR (NEW in v1.4)

### Experience Declaration

```ebnf
experience_decl = "experience" , ":" , "{" ,
                    "name" , ":" , identifier , "," ,
                    "version" , ":" , version , "," ,
                    [ "description" , ":" , string , "," ] ,
                    "entry_point" , ":" , identifier , "," ,
                    "includes" , ":" , "[" , identifier_list , "]" , "," ,
                    [ "policy_set" , ":" , identifier , "," ] ,
                    [ "unauthenticated" , ":" , unauthenticated_spec , "," ] ,
                    [ "on_unauthorized" , ":" , unauthorized_action , "," ] ,
                    [ "session" , ":" , session_spec , "," ] ,        (* NEW in v1.5 *)
                    [ "devices" , ":" , devices_spec , "," ] ,        (* NEW in v1.5 *)
                    [ "offline" , ":" , offline_spec ] ,              (* NEW in v1.5 *)
                  "}" ;

unauthenticated_spec = "{" ,
                         "redirect_to" , ":" , identifier ,
                       "}" ;

unauthorized_action = "conceal" | "indicate" | "redirect" | "deny" ;
```

### Session Contract (NEW in v1.5)

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

### Multi-Device Semantics (NEW in v1.5)

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

### Offline Operation Semantics (NEW in v1.5)

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

bytes         = integer , [ "KB" | "MB" | "GB" ] ;
```

---

## NAVIGATION GRAMMAR (NEW in v1.4)

### Navigation Definitions

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

## SEARCH GRAMMAR (NEW in v1.5)

### Search Intent Declaration

```ebnf
search_intent_decl = "searchIntent" , ":" , "{" ,
                       "name" , ":" , identifier , "," ,
                       "version" , ":" , version , "," ,
                       "searches" , ":" , identifier , "," ,
                       "query" , ":" , search_query_spec , "," ,
                       [ "filters" , ":" , search_filter_spec , "," ] ,
                       [ "facets" , ":" , search_facet_spec , "," ] ,
                       [ "pagination" , ":" , search_pagination_spec , "," ] ,
                       [ "sorting" , ":" , search_sorting_spec ] ,
                     "}" ;

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
```

---

## EXTERNAL INTEGRATION GRAMMAR (NEW in v1.5)

### External Dependency Declaration

```ebnf
external_dependency_decl = "externalDependency" , ":" , "{" ,
                             "name" , ":" , identifier , "," ,
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
                             [ "health_check" , ":" , health_check_spec ] ,
                           "}" ;

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

fallback_action = "queue" | "fail" | "degrade" | "cache" ;

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

### Webhook Receiver Declaration

```ebnf
webhook_receiver_decl = "webhookReceiver" , ":" , "{" ,
                          "name" , ":" , identifier , "," ,
                          "version" , ":" , version , "," ,
                          "description" , ":" , string , "," ,
                          "source" , ":" , identifier , "," ,
                          "maps_to" , ":" , "{" , "inputIntent" , ":" , identifier , "}" , "," ,
                          "authentication" , ":" , webhook_auth_spec , "," ,
                          "event_mapping" , ":" , "[" , event_mapping_list , "]" ,
                          [ "," , "idempotency" , ":" , webhook_idempotency ] ,
                          [ "," , "replay" , ":" , webhook_replay ] ,
                        "}" ;

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

field_map_entry = identifier , ":" , string ;  (* identifier: json_path *)

webhook_idempotency = "{" ,
                        "key_path" , ":" , string , "," ,
                        "ttl" , ":" , duration ,
                      "}" ;

webhook_replay = "{" ,
                   "enabled" , ":" , boolean ,
                   [ "," , "window" , ":" , duration ] ,
                 "}" ;
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

(* Presentation Composition Rules - NEW in v1.4 *)
valid_composition      = composition_has_name
                       ∧ composition_has_version
                       ∧ composition_has_primary_view
                       ∧ composition_primary_view_exists
                       ∧ composition_related_views_exist
                       ∧ composition_navigation_hierarchy_acyclic ;

(* Experience Rules - NEW in v1.4 *)
valid_experience       = experience_has_name
                       ∧ experience_has_entry_point
                       ∧ experience_entry_point_in_includes
                       ∧ experience_includes_compositions_exist
                       ∧ experience_policy_set_exists ;

(* Field Permission Rules - NEW in v1.4 *)
valid_field_permission = field_permission_field_exists
                       ∧ field_permission_policy_exists
                       ∧ field_permission_mask_valid ;

(* Navigation Rules - NEW in v1.4 *)
valid_navigation       = navigation_parent_exists
                       ∧ navigation_hierarchy_acyclic
                       ∧ navigation_purpose_valid ;

(* View Triggers Rules - NEW in v1.5 *)
valid_view_triggers    = triggers_intent_exists
                       ∧ triggers_required_fields_mapped
                       ∧ triggers_validation_consistent
                       ∧ triggers_context_fields_available ;

(* Intent Delivery Rules - NEW in v1.5 *)
valid_intent_delivery  = delivery_timeout_positive
                       ∧ retry_implies_idempotent
                       ∧ response_includes_non_empty ;

(* Materialization Retrieval Rules - NEW in v1.5 *)
valid_retrieval        = by_entity_has_key
                       ∧ by_query_has_fields
                       ∧ singleton_no_key_or_fields ;

(* Session Rules - NEW in v1.5 *)
valid_session          = session_contains_required_valid
                       ∧ lifetime_timeouts_consistent
                       ∧ devices_max_active_positive ;

(* Indicator Source Rules - NEW in v1.5 *)
valid_indicator_source = indicator_view_exists
                       ∧ indicator_scope_expression_valid
                       ∧ indicator_update_trigger_consistent ;

(* Composition Fetch Rules - NEW in v1.5 *)
valid_composition_fetch = fetch_timeouts_positive
                        ∧ fetch_max_concurrent_positive
                        ∧ fetch_retry_scope_valid ;

(* Asset Rules - NEW in v1.5 *)
valid_asset            = asset_max_size_positive
                       ∧ asset_mime_types_valid
                       ∧ asset_lifecycle_expiry_consistent
                       ∧ asset_variants_unique ;

(* Search Rules - NEW in v1.5 *)
valid_search           = search_indexed_fields_exist
                       ∧ search_facet_ranges_valid
                       ∧ search_intent_fields_exist ;

(* External Dependency Rules - NEW in v1.5 *)
valid_external_dep     = external_capabilities_non_empty
                       ∧ external_contract_schemas_valid
                       ∧ external_credentials_secret_exists
                       ∧ external_circuit_breaker_threshold_positive ;

(* Webhook Rules - NEW in v1.5 *)
valid_webhook          = webhook_source_exists
                       ∧ webhook_maps_to_intent_exists
                       ∧ webhook_event_mapping_valid
                       ∧ webhook_auth_secret_exists ;

(* Governance Rules - NEW in v1.5 *)
valid_governance       = retention_duration_positive
                       ∧ residency_regions_valid
                       ∧ encryption_consistent
                       ∧ audit_retention_reasonable ;

(* Consent Rules - NEW in v1.5 *)
valid_consent          = consent_scopes_non_empty
                       ∧ consent_subject_field_exists
                       ∧ consent_withdrawal_consistent ;

(* Erasure Rules - NEW in v1.5 *)
valid_erasure          = erasure_scope_valid
                       ∧ erasure_sla_reasonable
                       ∧ erasure_cascade_entities_exist ;

(* Legal Hold Rules - NEW in v1.5 *)
valid_legal_hold       = hold_entities_exist
                       ∧ hold_overrides_valid
                       ∧ hold_custodian_exists ;

(* Graph Query Rules - NEW in v1.5 *)
valid_graph_query      = graph_relationship_exists
                       ∧ graph_start_entity_valid
                       ∧ graph_max_depth_reasonable ;

(* Device Rules - NEW in v1.5 *)
valid_device           = device_capabilities_non_empty
                       ∧ device_heartbeat_reasonable
                       ∧ sensor_range_consistent
                       ∧ actuator_safety_defined ;

(* Telemetry Rules - NEW in v1.5 *)
valid_telemetry        = telemetry_source_exists
                       ∧ collection_mode_consistent
                       ∧ buffer_size_reasonable ;

(* Actuator Command Rules - NEW in v1.5 *)
valid_actuator_cmd     = actuator_target_exists
                       ∧ command_timeout_positive
                       ∧ safety_constraints_valid ;

(* Inference Rules - NEW in v1.5 *)
valid_inference        = inference_model_exists
                       ∧ inference_source_fields_exist
                       ∧ inference_output_type_valid ;

(* Spatial Rules - NEW in v1.5 *)
valid_spatial          = coordinate_system_valid
                       ∧ spatial_precision_reasonable ;

(* Structured Content Rules - NEW in v1.5 *)
valid_structured_content = content_schema_exists
                         ∧ blocks_allowed_non_empty
                         ∧ marks_allowed_non_empty ;
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

✅ **Presentation Composition Layer (NEW in v1.4)**
- Presentation Compositions
- Experiences
- Field Permissions
- Navigation

✅ **Application Development Layer (NEW in v1.5)**
- Form Intent Binding
- View Materialization Contracts
- Intent Delivery Contracts
- Session Management
- Multi-Device Semantics
- Offline Operation
- Indicator Source Binding
- Composition Fetch Semantics
- Asset Management
- Search and Discovery
- External Integration
- Webhook Receivers
- Data Governance (Retention, Residency, Audit)
- Consent Management (GDPR Compliance)
- Erasure Rights (Right-to-Deletion)
- Legal Hold (Litigation Preservation)
- Graph Queries (Traversal, Path Finding)
- Delegation Semantics (Authority Delegation)
- Inference Fields (ML-Derived Values)
- Structured Content (Rich Text)
- Spatial Types (Geographic Data)
- Edge Devices (IoT Integration)
- Device Capabilities (Sensors, Actuators)
- Device Telemetry (Sensor Streaming)
- Actuator Commands (Device Control)

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
- v1.4: Added presentation composition, experiences, field permissions, navigation
- v1.5: Added form binding, session management, assets, search, external integration, streaming, offline, multi-device

### v1.5 Additions

The following grammar elements were added in v1.5:

1. **Form Intent Binding**: `triggers` block in presentationView for form submission semantics
2. **View Materialization Contract**: `retrieval` and `temporal` blocks for data access patterns
3. **Intent Delivery Contract**: `delivery` and `response` blocks for intent routing
4. **Session Contract**: `session`, `devices`, and `offline` blocks in experience
5. **Indicator Source Binding**: Enhanced indicator with `source`, `update`, and `display`
6. **Composition Fetch Semantics**: `fetch` block for data loading strategies
7. **Asset Semantics**: `asset` field type with lifecycle and access control
8. **Search Semantics**: `search` field annotations and `searchIntent` declaration
9. **External Integration**: `externalDependency` and `webhookReceiver` declarations
10. **Temporal Window Semantics**: Tumbling, sliding, session, and hopping windows
11. **Multi-Device Support**: Device identification, state scoping, and handoff
12. **Offline Operation**: Queue, sync, conflict resolution, and cache semantics
13. **Real-Time Streaming**: Aggregations, watermarks, overflow handling, emit triggers
14. **Data Governance**: `governance` block for retention, residency, audit, encryption
15. **Consent Management**: `consentRecord` declaration for GDPR compliance
16. **Erasure Rights**: `erasureRequest` declaration for right-to-deletion
17. **Legal Hold**: `legalHold` declaration for litigation preservation
18. **Graph Queries**: `graphQuery` declaration for traversal and path queries
19. **Delegation Semantics**: `delegation` block in relationships for authority delegation
20. **Inference Fields**: `inference` field extension for ML-derived values
21. **Structured Content**: `structured_content` field type for rich text
22. **Spatial Types**: `point` field type for geographic data
23. **Edge Devices**: `device` spec in worker_class for IoT semantics
24. **Device Capabilities**: `deviceCapability` declaration for sensors and actuators
25. **Device Telemetry**: `deviceTelemetry` declaration for sensor data streaming
26. **Actuator Commands**: `actuatorCommand` declaration for device control

These additions enable:
- Complete end-to-end application development
- Form-based user interactions
- Multi-device and offline-first applications
- Asset upload and management
- Search and discovery features
- External API integration
- Real-time data processing
- Progressive web app capabilities

### v1.4 Additions

The following grammar elements were added in v1.4:

1. **Presentation Composition Grammar**: Semantic grouping of views into workspaces
2. **Experience Grammar**: Application-level navigation and user type grouping
3. **Field Permissions Grammar**: View-level field visibility and masking
4. **Navigation Grammar**: Purposes, scopes, and indicators
5. **Related View Relationships**: Derived, detail, contextual, action, supplementary, historical
6. **Navigation Scopes**: Within_composition, always_visible, on_action
7. **On_unauthorized Behavior**: Conceal, indicate, redirect, deny

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
- All v1.0 documents valid in v1.1, v1.2, v1.3, v1.4, and v1.5
- Tools SHOULD support multiple versions
- Runtimes MUST declare supported versions
- v1.5 is purely additive - no breaking changes from v1.4

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
- **Application-Complete**: Enables end-to-end development (v1.5)

This is the authoritative grammar for System Mechanics Specification v1.5.

### New in v1.5: Complete Application Development Support

The v1.5 grammar additions enable complete end-to-end application development:

1. **User Interaction Layer**
   - Form intent binding with field mapping and validation
   - Success/error handling with navigation actions
   - Confirmation dialogs and notifications

2. **Data Access Layer**
   - View retrieval contracts (by_entity, by_query, singleton)
   - Temporal window semantics (tumbling, sliding, session, hopping)
   - Real-time aggregations and stream processing
   - Search index configuration and query semantics

3. **Session & Identity Layer**
   - Session management with lifetime semantics
   - Multi-device support with state scoping
   - Offline operation with sync and conflict resolution
   - Device handoff and continuity

4. **Integration Layer**
   - External dependency contracts with resilience patterns
   - Webhook receivers with authentication
   - Circuit breakers, bulkheads, and rate limiting
   - Inbound event mapping to intents

5. **Asset Management**
   - Binary asset field type with lifecycle
   - Size constraints and mime type validation
   - Signed URL access patterns
   - Asset variants and transformations

6. **Search & Discovery**
   - Searchable field annotations
   - Full-text, fuzzy, prefix, semantic strategies
   - Faceting and aggregation
   - Relevance tuning and highlighting

7. **Enhanced Observability**
   - Indicator source binding to materialized views
   - Update triggers and throttling
   - Scope filtering (user, entity, global)

These additions maintain the intent-first philosophy while providing complete specification coverage for:
- Progressive web applications
- Multi-device mobile apps
- Real-time dashboards
- Search-driven experiences
- External API integration
- Offline-first applications

### New in v1.4: UI Composition, Experiences, and Navigation

The v1.4 grammar additions address intent-driven UI development:

1. **PresentationComposition**: Semantic grouping of views into workspaces
2. **Experience**: Application-level navigation and user journey grouping
3. **Field Permissions**: Declarative field visibility and masking
4. **Navigation Grammar**: Purposes, scopes, and dynamic indicators
5. **View Relationships**: Derived, detail, contextual, action, supplementary, historical
6. **On_unauthorized Behaviors**: Conceal, indicate, redirect, deny

These additions enable:
- Intent-driven UI composition without prescribing layout
- Role-based field masking with semantic masking types
- Hierarchical navigation inferred from parent relationships
- Experience-level policy enforcement
- Dynamic navigation indicators for state signaling

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

### New in v1.2: WorkUnitContract

The WorkUnitContract grammar extension addresses the critical gap where work units referenced 
in SMS flows were named but not formally specified. This enables:

1. **Type-safe code generation** from contract definitions
2. **Compile-time validation** of flow steps against contracts
3. **Contract coverage reporting** for work units in flows
4. **Explicit side effect documentation** for operational awareness
5. **Formal error handling** with retry strategies
