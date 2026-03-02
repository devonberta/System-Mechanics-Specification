# System Mechanics Specification v1.5

**NORMATIVE SPECIFICATION**  
**Version**: 1.5  
**Status**: Production Ready - Complete Application Development Support  
**Last Updated**: 2026-01-18

---

## Reading This Specification (For LLM Agents)

### Document Purpose

This is the **complete, normative** specification for the System Mechanics Specification (SMS) v1.5. It defines a universal, intent-first framework for describing, executing, and evolving distributed systems with complete application development support.

### How to Use This Document

**For LLMs Implementing Runtime**:
- This document is self-contained - no external references needed
- All addendums are integrated inline (no "see Addendum X" references)
- Grammar is hierarchical: start with Part II for core concepts
- Validation rules are explicit - all MUST/SHALL/MAY keywords are normative
- Examples are illustrative unless marked normative

**For LLMs Writing Specifications**:
- Use Part II (Core Grammar) as authoritative reference
- Refer to Part IV (Validation Rules) to ensure correctness
- Use Part V (Design Rationale) to understand intent
- All concepts are versioned - always specify versions explicitly

**Key Structural Notes**:
- This specification uses RFC 2119 keywords: MUST, SHALL, MAY, SHOULD
- All code blocks without line numbers are normative grammar unless marked "example"
- Grammar shown in YAML format is canonical; EBNF in Part III is equivalent
- Addendums have been fully merged - treat all content as unified specification

**Related Documents:**
- **SMS-v1.5-EBNF-Grammar.md**: Complete formal grammar for parser/tooling development
- **SMS-v1.5-Implementation-Guide.md**: Detailed runtime implementation patterns
- **SMS-v1.5-Reference-Examples.md**: Modeling guidance and complete worked examples
- **SMS-v1.5-Quick-Reference.md**: Rapid-lookup cheat sheets and decision trees

---

## Part I: Overview & Principles

### Introduction

The System Mechanics Specification v1.5 represents a milestone in SMS evolution - the ability to specify **complete, production-ready applications** from intent to deployment. This specification defines how to describe:

- **Data Models**: Structure, evolution, materialization, and governance
- **UI & Experiences**: Compositions, navigation, field permissions, and accessibility
- **Input & Forms**: Intent binding, validation, and submission patterns
- **Workflows**: Flows, durable processes, and human-in-the-loop steps
- **Authorization**: Policies, delegation, consent, and compliance
- **Integration**: External APIs, webhooks, search, notifications, and scheduling
- **Deployment**: Multi-region authority, entity-scoped write control, and edge devices

### What's New in v1.5

**28 New Capabilities** enable end-to-end application development:

**User Experience & Forms**:
- Form-Intent Binding (integrated): Declarative form validation and submission
- Accessibility (integrated): WCAG compliance and user preferences
- Conversational UI (integrated): Voice and chatbot interfaces

**Data & Real-Time**:
- View Materialization Contracts (integrated): Streaming with temporal windows
- Session Management (integrated): Multi-device sync, offline support
- Collaborative Sessions (integrated): Real-time co-editing with presence

**Assets & Content**:
- Asset Semantics (integrated): Native file/image/media handling
- Structured Content (integrated): Rich document types
- Search (integrated): Full-text search with faceting and ranking

**Integration & Automation**:
- Intent Delivery Contract (integrated): Guaranteed delivery with retry
- Scheduled Triggers (integrated): Cron-style temporal events
- Notifications (integrated): Multi-channel push, email, SMS, webhook
- External Integration (integrated): Third-party APIs and event bridges
- Durable Workflows (integrated): Long-running processes

**Governance & Compliance**:
- Delegation & Consent (integrated): Granular authority transfer
- Data Governance (integrated): Legal holds, retention, GDPR/HIPAA

**Advanced Capabilities**:
- Spatial Types (integrated): GIS and location-based queries
- Graph Queries (integrated): Relationship traversal
- ML/Inference (integrated): Inference-derived fields
- Edge Computing (integrated): IoT and offline-first patterns

### Core Principles (Normative)

These principles MUST be followed by all conformant implementations:

**1. Intent-First, Runtime-Decided**
- The specification declares **what** must be true, not **how** it's achieved
- Transport, locality, and optimization are runtime responsibilities
- Same specification works on cloud, edge, browser, embedded

**2. Versioned Truth**
- All data types, policies, UI, inputs, and workers MUST be versioned
- Semantic change requires new version, even if structure unchanged
- Multiple versions coexist during migration

**3. Linear Evolution for Persisted Data**
- Data versions evolve sequentially (v1→v2→v3)
- Versions MAY NOT be skipped for persistent data types
- Evolution path is explicit and auditable

**4. UI and Input Bind to Data, Not Services**
- UI binds to materialized data versions
- Input binds to target data states
- No service coupling in presentation layer

**5. Zero Downtime by Construction**
- Shadow, promotion, retirement are first-class
- Runtime MAY run multiple versions concurrently
- Gradual rollout prevents incidents

**6. Declarative Authorization**
- Authorization expressed declaratively in grammar
- Enforcement mechanics are runtime-defined
- All authorization evaluated locally with distributed policies

**7. Entity-Scoped Authority** (v1.3)
- Write authority resolved per entity, not per model
- Granular failure isolation
- Per-entity mobility for optimization

**8. Invariants Live in Views** (v1.3)
- Cross-entity invariants enforced by derived views
- Entities remain independently writable
- No distributed transactions required

**9. Compose Views Semantically** (v1.4)
- Group views by relationship, not layout
- Semantic relationship types (derived, detail, contextual, etc.)
- Runtime decides visual presentation

**10. Complete Applications Need Comprehensive Features** (v1.5)
- Real applications require assets, search, notifications, workflows
- All features first-class in grammar
- Compliance and accessibility built-in

---

## Part II: Core Grammar


### 2.1 Document Structure

An SMS specification document is a YAML file containing multiple top-level declarations. Each declaration defines a first-class concept in the system.

**Document Format**:
```yaml
# System metadata
system:
  name: <string>
  version: <version>
  description: <string>
  realm: <identifier>

# Data layer declarations
dataType: { ... }
constraint: { ... }
dataState: { ... }
materialization: { ... }
transition: { ... }

# UI layer declarations
presentationView: { ... }
presentationComposition: { ... }
experience: { ... }

# Input layer declarations
inputIntent: { ... }

# Flow layer declarations
sms: { ... }

# Worker topology declarations
wts: { ... }

# Policy declarations
policy: { ... }
```

**Validation**: All top-level declarations MUST have a `name` and `version` field except for `system` declarations.

---

### 2.2 Data Layer

The data layer defines structure, constraints, lifecycle, authority, and governance for all data in the system.

#### 2.2.1 DataType

Defines structural schema only - no behavior or lifecycle.

**Grammar**:
```yaml
dataType:
  name: <string>
  version: <vN>
  description: <string>   # OPTIONAL
  fields:
    <fieldName>:
      type: <type>
      # Field-specific extensions (v1.5)
      asset: { ... }              # Asset semantics
      search: { ... }             # Search configuration
      inference: { ... }          # ML-derived fields
      structured_content: { ... } # Rich content
      spatial: { ... }            # Geographic/spatial data
```

**Primitive Types**:
- `string`: UTF-8 text
- `integer`: 64-bit signed integer
- `decimal`: Fixed-point decimal
- `boolean`: true/false
- `uuid`: RFC 4122 UUID
- `timestamp`: ISO 8601 timestamp
- `duration`: ISO 8601 duration
- `asset` (v1.5): Binary/file content reference
- `structured_content` (v1.5): Rich text/document
- `spatial` (v1.5): Geographic coordinates/shapes

**Complex Types**:
- `enum[value1, value2, ...]`: Enumeration
- `array<type>`: Ordered collection
- `<TypeName>`: Reference to another DataType

**Example**:
```yaml
dataType:
  name: Order
  version: v1
  fields:
    id: uuid
    customer_id: uuid
    items: array<OrderItem>
    total: decimal
    status: enum[pending, confirmed, shipped, delivered]
    created_at: timestamp
    invoice_pdf: asset        # v1.5: File attachment
    notes: structured_content # v1.5: Rich text
```

**Validation Rules**:
- Field names MUST be unique within a DataType
- Field names MUST match `[a-zA-Z_][a-zA-Z0-9_]*`
- Enum values MUST be unique
- Recursive type references are permitted
- Self-references MUST be optional or arrays

---

#### 2.2.2 Asset Semantics (v1.5 - Addendum AK)

Assets represent binary content (files, images, videos, documents) as first-class data types.

**Asset Field Configuration**:
```yaml
dataType:
  name: Document
  version: v1
  fields:
    attachment:
      type: asset
      asset:
        category: image | document | media | archive | generic
        constraints:
          max_size: <size>        # e.g., "10MB", "500KB"
          mime_types: [<types>]   # e.g., ["image/jpeg", "image/png"]
          extensions: [<exts>]    # e.g., [".jpg", ".png"]
        lifecycle:
          type: permanent | temporary | expiring
          expires_after: <duration>  # For expiring type
        reference:
          style: by_reference | embedded
        access:
          read: public | signed | authenticated | policy
          policy: <policy_ref>           # If access=policy
          signed_url_ttl: <duration>     # If access=signed
        variants:
          - name: thumbnail
            transform: resize
            params: { width: 150, height: 150 }
          - name: preview
            transform: resize
            params: { width: 800, height: 600 }
```

**Asset Categories**:
- `image`: Raster/vector images (JPEG, PNG, SVG, etc.)
- `document`: PDFs, Word docs, spreadsheets
- `media`: Audio/video files
- `archive`: Compressed archives (ZIP, TAR, etc.)
- `generic`: Any binary content

**Lifecycle Types**:
- `permanent`: Stored indefinitely, survives entity deletion
- `temporary`: Short-lived staging area (e.g., upload preprocessing)
- `expiring`: Time-bounded with automatic deletion after expiry

**Access Modes**:
- `public`: Publicly accessible URL (no authentication)
- `signed`: Time-limited signed URL (secure temporary access)
- `authenticated`: Requires valid authentication token
- `policy`: Policy-based access control (most secure)

**Transform Types** (for variants):
- `resize`: Scale to dimensions
- `thumbnail`: Generate thumbnail
- `compress`: Reduce file size
- `convert`: Convert format (e.g., HEIC→JPEG)
- `extract_page`: Extract page from PDF

**Runtime Responsibilities**:
- Asset storage integration (S3, Azure Blob, GCS, etc.)
- Upload/download handling
- Variant generation (can be lazy or eager)
- Access control enforcement
- CDN integration for delivery
- Garbage collection for expiring assets

**Example - Profile Picture**:
```yaml
dataType:
  name: UserProfile
  version: v1
  fields:
    avatar:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
          mime_types: ["image/jpeg", "image/png", "image/webp"]
        lifecycle:
          type: permanent
        access:
          read: public
        variants:
          - name: thumbnail
            transform: thumbnail
            params: { width: 64, height: 64 }
          - name: medium
            transform: resize
            params: { width: 256, height: 256 }
```

**Example - Invoice PDF**:
```yaml
dataType:
  name: Invoice
  version: v1
  fields:
    pdf_document:
      type: asset
      asset:
        category: document
        constraints:
          max_size: 10MB
          mime_types: ["application/pdf"]
        lifecycle:
          type: permanent
        access:
          read: policy
          policy: InvoiceAccessPolicy
```

---

#### 2.2.3 Structured Content (v1.5 - Addendum AP)

Structured content enables rich text and document editing with semantic structure.

**Structured Content Field Configuration**:
```yaml
dataType:
  name: Article
  version: v1
  fields:
    body:
      type: structured_content
      structured_content:
        schema: article_schema
        blocks:
          allowed: [paragraph, heading, list, blockquote, code, image, video, table]
        marks:
          allowed: [bold, italic, underline, link, code, highlight]
        constraints:
          max_length: 50000  # Character limit
        sanitization:
          strip_scripts: true
          allowed_tags: [p, h1, h2, h3, ul, ol, li, blockquote, code, pre, img, video, table]
          allowed_attributes: [href, src, alt, class, id]
```

**Block Types**:
- `paragraph`: Regular paragraph
- `heading`: Hierarchical headings (h1-h6)
- `list`: Ordered/unordered lists
- `blockquote`: Quoted text
- `code`: Code blocks with syntax highlighting
- `table`: Structured tables
- `image`: Embedded images
- `video`: Embedded video
- `poll`: Interactive polls
- `divider`: Horizontal divider
- `embed`: External embeds (e.g., YouTube, Twitter)

**Mark Types** (inline formatting):
- `bold`: Bold text
- `italic`: Italic text
- `underline`: Underlined text
- `strikethrough`: Strikethrough text
- `code`: Inline code
- `link`: Hyperlinks
- `mention`: User/entity mentions (@user)
- `hashtag`: Hashtags (#topic)
- `highlight`: Highlighted text

**Sanitization**:
- `strip_scripts`: Remove JavaScript and event handlers (security)
- `allowed_tags`: Whitelist of HTML tags (if rendering to HTML)
- `allowed_attributes`: Whitelist of HTML attributes

**Runtime Responsibilities**:
- Parsing and validation of structured content
- Sanitization enforcement
- Rendering to HTML/markdown
- Collaborative editing (Operational Transform or CRDT)
- Asset embedding resolution

**Example - Blog Post**:
```yaml
dataType:
  name: BlogPost
  version: v1
  fields:
    title: string
    content:
      type: structured_content
      structured_content:
        blocks:
          allowed: [paragraph, heading, list, blockquote, code, image]
        marks:
          allowed: [bold, italic, link, code]
        constraints:
          max_length: 100000
        sanitization:
          strip_scripts: true
```

---

#### 2.2.4 Spatial Types (v1.5 - Addendum AR)

Spatial types enable geographic and location-based data with indexing and queries.

**Spatial Field Configuration**:
```yaml
dataType:
  name: Store
  version: v1
  fields:
    location:
      type: spatial
      spatial:
        geometry: point | line | polygon | multi_point | multi_line | multi_polygon
        coordinate_system: wgs84 | web_mercator | custom
        srid: <integer>          # Spatial Reference System ID
        precision: <decimal>     # Coordinate precision
        bounds:                  # Optional bounding box
          min_lat: <decimal>
          max_lat: <decimal>
          min_lon: <decimal>
          max_lon: <decimal>
        search:
          indexed: true | false
          strategy: bounding_box | radius | polygon_intersection
```

**Geometry Types**:
- `point`: Single coordinate (lat, lon)
- `line`: Line string (sequence of coordinates)
- `polygon`: Closed polygon (boundary coordinates)
- `multi_point`: Collection of points
- `multi_line`: Collection of line strings
- `multi_polygon`: Collection of polygons

**Coordinate Systems**:
- `wgs84`: World Geodetic System 1984 (GPS standard)
- `web_mercator`: Web Mercator projection (web maps)
- `custom`: Custom SRID specification

**Search Strategies**:
- `bounding_box`: Rectangular area search (fast, approximation)
- `radius`: Distance from point (circular area)
- `polygon_intersection`: Precise polygon intersection (slower)

**Runtime Responsibilities**:
- Spatial indexing (R-tree, Geohash, S2, etc.)
- Distance calculations
- Containment queries
- Intersection queries
- Integration with mapping services

**Example - Delivery Zone**:
```yaml
dataType:
  name: DeliveryZone
  version: v1
  fields:
    name: string
    boundary:
      type: spatial
      spatial:
        geometry: polygon
        coordinate_system: wgs84
        search:
          indexed: true
          strategy: polygon_intersection
```

**Example - GPS Tracker**:
```yaml
dataType:
  name: DeviceLocation
  version: v1
  fields:
    device_id: uuid
    position:
      type: spatial
      spatial:
        geometry: point
        coordinate_system: wgs84
        precision: 0.000001  # ~10cm accuracy
        search:
          indexed: true
          strategy: radius
```

---

#### 2.2.5 Constraint

Defines semantic truth - what must be true about data.

**Grammar**:
```yaml
constraint:
  name: <string>
  appliesTo: <DataType>
  rules:
    <version>: <expression>
```

**Purpose**: Constraints are version-aware semantic invariants that validate data correctness.

**Separation from Structure**: Constraints are separate from DataType to enable:
- Constraint evolution without structural change
- Different constraints for different versions
- Enforcement flexibility (strict/best-effort/deferred)

**Expression Language**: Boolean expressions referencing fields:
- Comparisons: `==`, `!=`, `<`, `>`, `<=`, `>=`
- Logical: `AND`, `OR`, `NOT`
- Arithmetic: `+`, `-`, `*`, `/`
- Functions: `len()`, `sum()`, `any()`, `all()`, `exists()`

**Example**:
```yaml
constraint:
  name: OrderTotalValid
  appliesTo: Order
  rules:
    v1: total >= 0 AND total == sum(items.price * items.quantity)
    v2: total >= 0 AND total == sum(items.price * items.quantity * (1 - items.discount))
```

**Validation**: Expressions MUST be deterministic and side-effect-free.

---

#### 2.2.6 DataState

Defines lifecycle, enforcement, and evolution policy for data.

**Grammar**:
```yaml
dataState:
  name: <string>
  type: <DataType>
  lifecycle: intermediate | persistent | materialized
  constraintPolicy:
    enforcement: none | best_effort | strict | deferred
  evolutionPolicy:
    allowed: [<changes>]
    forbidden: [<changes>]
  mutability: { ... }        # v1.3: Authority semantics
  storage: { ... }           # v1.3: Storage role
  governance: { ... }        # v1.5: Data governance
```

**Lifecycle Types**:
- `intermediate`: Transient, not persisted (e.g., form state, API request)
- `persistent`: Durable, authoritative (e.g., Order, Account)
- `materialized`: Derived, regenerable (e.g., dashboard data, summaries)

**Enforcement Levels**:
- `none`: No validation (fastest, use for trusted internal data)
- `best_effort`: Validate if possible, don't block (eventual consistency)
- `strict`: Block write on violation (strong consistency)
- `deferred`: Validate asynchronously, flag violations (auditing)

**Evolution Policy**:
- `allowed`: Permitted change types for this state
- `forbidden`: Explicitly disallowed changes

**Change Types**:
- `additive`: Add fields, extend enums
- `constraint_refinement`: Tighten constraints
- `destructive`: Remove fields, change types
- `semantic_break`: Change meaning without structure change

**Example - Persistent Order**:
```yaml
dataState:
  name: OrderRecord
  type: Order
  lifecycle: persistent
  constraintPolicy:
    enforcement: strict
  evolutionPolicy:
    allowed: [additive, constraint_refinement]
    forbidden: [destructive, semantic_break]
```

**Example - Materialized Dashboard**:
```yaml
dataState:
  name: CustomerDashboard
  type: DashboardData
  lifecycle: materialized
  constraintPolicy:
    enforcement: best_effort
  evolutionPolicy:
    allowed: [additive, destructive]  # Views can be rebuilt
    forbidden: []
```

---

#### 2.2.7 Mutability & Authority (v1.3 - Addendums O, P, Q, R)

Authority determines which region can accept writes for an entity at any given time.

**Mutability Grammar**:
```yaml
mutability:
  scope: entity | append-only
  exclusive: true | false
  authority: <authority_ref>
```

**Scope Types**:
- `entity`: Each entity instance has independent write authority
- `append-only`: Data is immutable once written (no authority needed)

**Authority Grammar**:
```yaml
authority:
  scope: entity | model
  resolution: <expression>
  migration:
    allowed: true | false
    strategy: pause-and-cutover
    triggers:
      - type: load | latency | manual | time
        condition: <expression>
    constraints:
      - type: region | governance
        allowed: [<regions>]
        rule: <expression>
    governed_by: <policy_ref>
```

**Authority Scope**:
- `entity`: Authority resolved per entity instance (RECOMMENDED)
- `model`: Authority resolved for entire model (legacy pattern)

**Resolution**: Expression determining authoritative region for an entity:
```yaml
# Example: US customers in us-east, EU customers in eu-west
resolution: |
  if customer.country in ["US", "CA"] then "us-east"
  else if customer.country in EU_COUNTRIES then "eu-west"
  else "default-region"
```

**Migration Triggers**:
- `load`: Migrate when load exceeds threshold
- `latency`: Migrate when latency exceeds threshold
- `manual`: Operator-initiated migration
- `time`: Scheduled migration (follow-the-sun)

**Migration Strategy: pause-and-cutover**:
1. Current authority region pauses writes for entity
2. CAS epoch incremented
3. New region becomes authority
4. Writes resume in new region

**Pause Duration**: Typically <100ms, bounded by network + coordination latency

**Authority State**:
```yaml
authorityState:
  entity_id: <uuid>
  authority_region: <string>
  authority_epoch: <integer>     # Monotonically increasing
  authority_lease: <uuid>         # Lease token
  authority_status: ACTIVE | TRANSITIONING
  entity_version: <integer>
```

**CAS Token** (v1.3 - Addendum Q):
Write requests include CAS token for linearizability:
```yaml
cas_token:
  entity_id: <uuid>
  entity_version: <integer>
  authority_epoch: <integer>
  authority_region: <string>
  lease_id: <uuid>
```

**CAS Validation**:
- Request MUST be sent to authority_region
- authority_epoch MUST match current epoch
- entity_version MUST match current version
- lease_id MUST be valid

**If CAS fails**: Client gets ErrEpochMismatch or ErrVersionMismatch, retries with fresh CAS token.

**Example - Entity-Scoped Authority**:
```yaml
dataState:
  name: BankAccount
  type: Account
  lifecycle: persistent
  mutability:
    scope: entity
    exclusive: true
    authority: AccountAuthority

authority:
  name: AccountAuthority
  scope: entity
  resolution: |
    # Place in region closest to account owner
    nearest_region(account.owner.country)
  migration:
    allowed: true
    strategy: pause-and-cutover
    triggers:
      - type: latency
        condition: p99_latency > 200ms
      - type: manual
    constraints:
      - type: governance
        rule: account.kyc_status == "approved"
```

**Formal Invariants** (Normative - MUST hold):

**I1. Authority Singularity**: At any point in time, an entity SHALL have at most one write-authoritative region.

**I2. Epoch Monotonicity**: Authority epochs SHALL increase monotonically and SHALL NOT be reused.

**I3. No Overlapping Authority**: No two regions SHALL simultaneously accept writes for the same entity.

**I4. CAS Safety**: A write SHALL be accepted if and only if:
- The authority region is current
- The authority epoch matches
- The entity version matches

**I5. View Continuity**: Derived views SHALL remain readable across authority transitions.

**I6. Rebuildability**: All derived state SHALL be reconstructible from authoritative event history.

**I7. Governance Compliance**: Authority transitions SHALL NOT violate declared governance constraints.

**I8. Failure Determinism**: On failure during authority transition, the system SHALL resolve to a single authoritative region without ambiguity.

---

#### 2.2.8 Storage Role (v1.3 - Addendum R)

Storage roles distinguish control state (coordination) from data state (domain events).

**Storage Grammar**:
```yaml
storage:
  role: data | control
  rebuildable: true | false
```

**Roles**:
- `data`: Append-only or rebuildable event/state data (domain layer)
- `control`: Coordination, authority, or governance state (system layer)

**Rules**:
- Control storage MUST be strongly consistent
- Control storage SHALL be modified only by designated control components
- Data storage MAY be partitioned, replayed, or materialized
- Implementations SHALL NOT treat control storage as domain data source

**Examples**:

Control Storage:
- Authority KV (entity authority state)
- Policy definitions
- Worker registrations
- Scheduler state

Data Storage:
- Event streams (domain events)
- Materialized views
- Audit logs
- User data

**Example**:
```yaml
# Authority state (control)
dataState:
  name: EntityAuthorityState
  type: AuthorityState
  lifecycle: persistent
  storage:
    role: control
    rebuildable: false

# Domain events (data)
dataState:
  name: OrderEvents
  type: OrderEvent
  lifecycle: persistent
  storage:
    role: data
    rebuildable: true  # Can rebuild views from events
```

---

#### 2.2.9 Relationship (v1.3 - Addendum S)

Relationships define semantic linkage between entities with cardinality and causality.

**Grammar**:
```yaml
relationship:
  name: <string>
  from: <model_ref>
  to: <model_ref>
  type: <identifier>         # Semantic type (owns, contains, references, etc.)
  cardinality: one-to-one | one-to-many | many-to-one | many-to-many
  semantics:
    causal: true | false     # Must "from" exist before "to"?
    ordered: true | false    # Must events be processed in order?
```

**Purpose**: Express semantic linkage WITHOUT implying:
- Synchronous write coordination
- Shared authority
- Atomic multi-entity mutation

**Cardinality**:
- `one-to-one`: One A relates to one B
- `one-to-many`: One A relates to many B
- `many-to-one`: Many A relate to one B
- `many-to-many`: Many A relate to many B

**Semantics**:
- `causal: true`: The "from" entity must exist before "to" can reference it
- `ordered: true`: Events must be processed in causal order

**Example - Account owns Transactions**:
```yaml
relationship:
  name: AccountTransactions
  from: Account
  to: Transaction
  type: owns
  cardinality: one-to-many
  semantics:
    causal: true   # Account must exist before Transaction
    ordered: false # Transactions can be processed in any order
```

**Example - Order references Customer**:
```yaml
relationship:
  name: OrderCustomer
  from: Order
  to: Customer
  type: references
  cardinality: many-to-one
  semantics:
    causal: true   # Customer must exist before Order
    ordered: false
```

---

#### 2.2.10 Invariant Policy (v1.3 - Addendum T)

Declares how invariants are enforced - prevents embedding cross-entity invariants inside entities.

**Grammar**:
```yaml
invariants:
  enforced_by: view | policy
  description: <string>
```

**Enforcement Options**:
- `view`: Invariants validated by derived views (RECOMMENDED for cross-entity)
- `policy`: Invariants enforced by policy evaluation

**Why Views?**:
- Entities remain independently writable
- No distributed transactions
- Eventual consistency tolerated
- Violations flagged rather than blocked

**Example - Balance Invariant**:
```yaml
# DON'T: Embed cross-account invariant in Account entity
# (Would require distributed transaction)

# DO: Enforce in view
materialization:
  name: AccountBalanceView
  source: [AccountDebitEvent, AccountCreditEvent]
  targetState: AccountBalance
  invariants:
    enforced_by: view
    description: "Balance = credits - debits, never negative"
```

**Design Rule**: If an invariant spans multiple entities with different authority, it MUST be enforced by a view, not by the entities themselves.

---

#### 2.2.11 DataEvolution

Declares allowed schema changes between versions.

**Grammar**:
```yaml
dataEvolution:
  name: <string>
  from:
    type: <DataType>
    version: <vN>
  to:
    type: <DataType>
    version: <vN+1>
  changes: [<change_ops>]
  compatibility:
    read: backward | forward | none
    write: backward | forward | none
```

**Change Operations**:
- `addField`: Add new field with default value
- `removeField`: Remove field (requires backward compatibility)
- `renameField`: Rename field (requires migration)
- `changeType`: Change field type (usually breaking)
- `extendEnum`: Add enum values (backward compatible)
- `refineConstraint`: Tighten constraints (may break old writers)

**Compatibility Modes**:
- `backward`: New code can read old data
- `forward`: Old code can read new data
- `none`: Breaking change

**Example**:
```yaml
dataEvolution:
  name: OrderV1toV2
  from:
    type: Order
    version: v1
  to:
    type: Order
    version: v2
  changes:
    - addField:
        name: discount
        type: decimal
        default: 0.0
    - extendEnum:
        field: status
        values: [cancelled, refunded]
  compatibility:
    read: backward    # v2 code can read v1 data
    write: forward    # v1 code can write data v2 code can read
```

---

#### 2.2.12 Materialization

Defines derived, regenerable state.

**Grammar**:
```yaml
materialization:
  name: <string>
  source: <DataState | list>
  targetState: <DataState>
  freshness:
    maxStaleness: <duration>
  evolution:
    strategy: rebuild | incremental
    blocking: true | false
  authority_agnostic: true | false    # v1.3
  epoch_tolerant: true | false        # v1.3
  retrieval: { ... }                  # v1.5: Retrieval contract
  temporal: { ... }                   # v1.5: Streaming semantics
```

**Evolution Strategies**:
- `rebuild`: Full regeneration on schema change
- `incremental`: Progressive update

**Authority Properties** (v1.3):
- `authority_agnostic: true`: View tolerates events from multiple regions (DEFAULT)
- `epoch_tolerant: true`: View tolerates events from different authority epochs (DEFAULT)

**Rules** (v1.3):
- Views derived from entity events SHALL be authority-agnostic unless explicitly stated
- Views MUST tolerate events originating from multiple regions
- Views MUST tolerate events written under different authority epochs
- Views SHALL NOT assume a fixed authoritative region for inputs

**Retrieval Contract** (v1.5 - Addendum Y):
```yaml
retrieval:
  mode: by_entity | by_query | singleton
  entity_key: <field_name>      # For by_entity mode
  query_fields: [<fields>]      # For by_query mode
  list:
    max_items: <integer>
    default_order: <field_name>
    order_direction: asc | desc
  freshness:
    max_staleness: <duration>
  on_unavailable:
    strategy: use_cached | fail | degrade | wait
    cache_ttl: <duration>
    degrade_to: <view_ref>
    wait_timeout: <duration>
```

**Retrieval Modes**:
- `by_entity`: One view instance per entity (keyed by entity_key)
- `by_query`: Views filtered by query fields (e.g., user_id, date_range)
- `singleton`: Single global view instance

**Unavailability Strategies**:
- `use_cached`: Serve stale data from cache (eventual consistency)
- `fail`: Return error immediately (strong consistency)
- `degrade`: Fall back to simpler/older view
- `wait`: Wait up to timeout for view to become available

**Temporal Semantics** (v1.5 - Streaming Views):
```yaml
temporal:
  window:
    type: tumbling | sliding | session | hopping
    size: <duration>
    slide: <duration>     # For sliding/hopping
    gap: <duration>       # For session windows
  aggregation:
    - field: <field_name>
      function: sum | avg | min | max | count | first | last | stddev
      as: <result_field>
  timestamp_field: <field_name>
  watermark: <duration>
  overflow:
    strategy: drop_oldest | pause_producer | sample | buffer
    buffer_size: <integer>
    sample_rate: <decimal>
  emit:
    trigger: on_window_close | on_each_event | periodic
    periodic_interval: <duration>
    include_partial: true | false
```

**Window Types**:
- `tumbling`: Non-overlapping fixed windows
- `sliding`: Overlapping windows that slide
- `session`: Dynamic windows based on activity gaps
- `hopping`: Fixed windows with fixed slide

**Example - Entity-Keyed View**:
```yaml
materialization:
  name: CustomerOrderSummary
  source: OrderRecord
  targetState: OrderSummaryData
  freshness:
    maxStaleness: 5m
  retrieval:
    mode: by_entity
    entity_key: customer_id
    on_unavailable:
      strategy: use_cached
      cache_ttl: 1h
```

**Example - Streaming Aggregation**:
```yaml
materialization:
  name: RealtimeOrderMetrics
  source: OrderEvent
  targetState: OrderMetrics
  temporal:
    window:
      type: tumbling
      size: 1m
    aggregation:
      - field: amount
        function: sum
        as: total_revenue
      - field: order_id
        function: count
        as: order_count
    timestamp_field: created_at
    emit:
      trigger: on_window_close
```

---

#### 2.2.13 Transition (Addendum A)

Transitions are typed edges between DataStates with explicit triggers and guarantees.

**Grammar**:
```yaml
transition:
  name: <string>
  from: <DataState>
  to: <DataState>
  triggeredBy:
    inputIntent | smsFlow
  guarantees: [<guarantees>]
  idempotency:
    key: <expression>
    scope: global | per_subject | per_realm
  authorization:
    policySet: <PolicySet>
```

**Purpose**: 
- Typed edges between states
- Explicit triggers (never implicit)
- Declared idempotency
- Authorizable independently

**Idempotency** (Addendum B - Normative):
- ALL transitions MUST be idempotent
- Executing same transition multiple times produces same result
- Deduplication keys MUST be deterministic
- Workers MUST tolerate duplicate delivery

**Example**:
```yaml
transition:
  name: ApproveOrder
  from: OrderRecord
  to: OrderRecord
  triggeredBy:
    inputIntent: SubmitApproval
  guarantees:
    - approval_recorded
    - idempotent_application
  idempotency:
    key:
      - approval.id
      - order.id
    scope: global
  authorization:
    policySet: OrderApprovalPolicies
```

---

#### 2.2.14 Search Semantics (v1.5 - Addendum AL)

Search configuration at the field level and search intent definitions.

**Field-Level Search**:
```yaml
dataType:
  name: Product
  version: v1
  fields:
    name:
      type: string
      search:
        indexed: true
        strategy: full_text
        full_text:
          analyzer: standard
          stemming: enabled
          stop_words: enabled
        weight: 2.0          # Boost relevance
        suggest:
          enabled: true
          type: completion
        highlight:
          enabled: true
          fragment_size: 150
```

**Search Strategies**:
- `full_text`: Natural language search with analysis
- `exact`: Exact match only
- `prefix`: Prefix/autocomplete matching
- `fuzzy`: Typo-tolerant matching
- `semantic`: Vector/embedding similarity (requires ML integration)

**Search Intent**:
```yaml
searchIntent:
  name: ProductSearch
  version: v1
  searches: ProductCatalog
  query:
    fields: [name, description, tags]
    default_operator: and | or
    phrase_slop: <integer>     # Allow words in different order
  filters:
    required: [category]
    optional: [brand, price_range]
  facets:
    include: [category, brand, price_range]
  pagination:
    default_size: 20
    max_size: 100
    cursor_based: true | false
  sort:
    default: _score           # Relevance
    allowed: [price, created_at, popularity]
  result:
    includes: [id, name, price, image_url]
    highlights: [name, description]
  authorization:
    policy_set: ProductSearchPolicies
    filter_by: visibility     # Automatically filter by visibility field
```

**Runtime Responsibilities**:
- Search indexing (Elasticsearch, Algolia, Typesense, etc.)
- Query parsing and execution
- Facet computation
- Relevance ranking
- Result highlighting
- Authorization filtering

---

#### 2.2.15 Data Governance (v1.5 - Addendum AX)

Compliance, retention, and data classification.

**Governance Configuration**:
```yaml
dataState:
  name: CustomerRecord
  type: Customer
  lifecycle: persistent
  governance:
    classification: pii | phi | pci | financial | public | internal | confidential
    retention:
      policy: time_based | event_based | indefinite | legal_hold
      duration: <duration>
      after_event: <event_name>
      grace_period: <duration>
      archive:
        enabled: true
        after: 90d
        tier: cold
      on_expiry: delete | anonymize | archive
    residency:
      allowed_regions: [us-east, us-west, eu-west]
      forbidden_regions: [cn-north]
      primary_region: |
        region_for_country(customer.country)
      transfer:
        allowed: false
        requires_consent: true
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 7y
      immutable: true
    encryption:
      at_rest: required
      in_transit: required
      key_rotation: 90d
```

**Classification Levels**:
- `pii`: Personally Identifiable Information (GDPR)
- `phi`: Protected Health Information (HIPAA)
- `pci`: Payment Card Industry data
- `financial`: Financial records (regulatory retention)
- `public`: Publicly accessible
- `internal`: Internal use only
- `confidential`: Restricted access

**Retention Policies**:
- `time_based`: Delete/archive after fixed duration
- `event_based`: Delete/archive after specific event (e.g., account_closed)
- `indefinite`: Keep forever (regulatory requirement)
- `legal_hold`: Preserve due to legal requirement (immutable)

**Expiry Actions**:
- `delete`: Permanently delete
- `anonymize`: Remove PII, keep aggregated data
- `archive`: Move to cold storage

**Runtime Responsibilities**:
- Automatic retention enforcement
- Legal hold management
- Data lineage tracking
- Audit log immutability
- Regional residency enforcement
- Right-to-erasure support (GDPR)

**Example - GDPR Compliance**:
```yaml
dataState:
  name: EUCustomerData
  type: Customer
  lifecycle: persistent
  governance:
    classification: pii
    retention:
      policy: event_based
      after_event: account_deleted
      grace_period: 30d
      on_expiry: anonymize
    residency:
      allowed_regions: [eu-west, eu-central]
      transfer:
        allowed: true
        requires_consent: true
        mechanisms: [standard_clauses]
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 7y
```

---

### 2.3 Presentation (UI) Layer

The presentation layer defines UI contracts bound to data, NOT implementation details.

#### 2.3.1 PresentationView

UI contract defining what data is displayed and how users interact.

**Grammar**:
```yaml
presentationView:
  name: <string>
  version: <vN>
  description: <string>
  consumes:
    dataState: <DataState>
    version: <vN>
  guarantees: [<guarantees>]
  interactionModel:
    badges: [<badge_definitions>]
    actions: [<action_definitions>]
    filters: [<filter_definitions>]
    sorts: [<sort_definitions>]
  fieldPermissions: [<permissions>]    # v1.4
  triggers: { ... }                    # v1.5: Form intent binding
```

**Guarantees**: What the UI promises to display or enable:
- `shows_all_items`: Display all items from source
- `real_time_updates`: Updates reflect changes immediately
- `sortable`: Users can sort data
- `filterable`: Users can filter data

**Interaction Model**:
```yaml
interactionModel:
  badges:
    - if: item.urgent == true
      label: "URGENT"
      severity: error
  actions:
    - if: item.status == "pending"
      action: approve
      label: "Approve"
      intent: ApproveIntent
  filters:
    - field: status
      type: enum
    - field: created_at
      type: date_range
  sorts:
    - field: created_at
      default: desc
    - field: amount
      default: asc
```

**Field Permissions** (v1.4 - Addendum W):
```yaml
fieldPermissions:
  - field: ssn
    visible: true
    mask:
      unless: subject.role == "admin"
      transform: reveal_last(4)      # Show last 4 digits
      placeholder: "***-**-####"
      preserve_format: true
      context:
        display: reveal_last(4)
        print: redact
        export: hash
        log: redact
  - field: account_balance
    visible:
      policy: ViewBalancePolicy
  - field: internal_notes
    visible: false
```

**Mask Transforms** (v1.5 - Addendum AD):
- Reveal: `reveal_last(n)`, `reveal_first(n)`, `reveal_middle(start, end)`
- Hide: `hide_last(n)`, `hide_first(n)`, `hide_middle(start, end)`
- Replace: `redact`, `hash`, `hash_partial(n)`, `truncate(n)`
- Format-aware: `mask_email`, `mask_phone`, `mask_credit_card`, `mask_ssn`

**Transform Chaining**: `reveal_last(4) | uppercase`

**Context Types**:
- `display`: Screen/UI display
- `print`: Printed documents
- `export`: Data export (CSV, Excel)
- `log`: Audit/debug logs

**Form Intent Binding** (v1.5 - Addendum X):
```yaml
triggers:
  intent: CreateOrderIntent
  binding: form | action | link
  field_mapping:
    - view_field: customer_id
      supplies: order.customer_id
      source: context         # From session/navigation context
      required: true
    - view_field: items
      supplies: order.items
      source: input           # User input
      required: true
      validation:
        constraint: len(items) > 0
        message: "At least one item required"
    - view_field: total
      supplies: order.total
      source: computed        # Calculated from other fields
      default: sum(items.price * items.quantity)
  confirmation:
    required: true
    message: "Submit order for {{total}} with {{len(items)}} items?"
  on_success:
    action: navigate_to
    target: OrderConfirmationWorkspace
    params: { order_id: "{{result.id}}" }
    notification:
      type: toast
      message: "Order submitted successfully!"
  on_error:
    action: display_inline
    show_field_errors: true
    notification:
      type: inline
      message: "Please correct the errors below"
```

**Binding Types**:
- `form`: Multi-field input form
- `action`: Single-click action button
- `link`: Navigation that triggers intent

**Field Sources**:
- `context`: From composition/session context (navigation params, auth state)
- `input`: User-provided value
- `computed`: Derived from other fields

**Success Actions**:
- `navigate_back`: Return to previous screen
- `navigate_to`: Go to specific composition
- `stay`: Remain on current screen (show success message)
- `refresh`: Reload current screen
- `close`: Close modal/drawer

**Error Actions**:
- `display_inline`: Show errors inline with fields
- `display_modal`: Show error in modal dialog
- `retry`: Allow immediate retry

**Example - Order List View**:
```yaml
presentationView:
  name: OrderListView
  version: v1
  consumes:
    dataState: OrderSummary
    version: v1
  guarantees:
    - shows_all_orders
    - real_time_updates
  interactionModel:
    badges:
      - if: order.is_urgent
        label: "URGENT"
        severity: error
    actions:
      - if: order.status == "pending"
        action: view_details
        label: "View Details"
        intent: NavigateToOrderDetails
    filters:
      - field: status
        type: enum
      - field: created_at
        type: date_range
    sorts:
      - field: created_at
        default: desc
  fieldPermissions:
    - field: customer_email
      visible: true
      mask:
        unless: subject.role in ["admin", "support"]
        transform: mask_email
```

---

#### 2.3.2 PresentationComposition (v1.4 - Addendum U)

Semantic grouping of views into workspaces.

**Grammar**:
```yaml
presentationComposition:
  name: <string>
  version: <vN>
  description: <string>
  primary: <view_ref>
  navigation:
    label: <string>               # Supports templates: "{{entity.name}}"
    purpose: <purpose_category>
    parent: <composition_ref>
    param: <string>               # Required parameter (e.g., "account_id")
    policy_set: <policy_set_ref>
    indicator:
      when: <expression>
      severity: info | warning | error | critical
  related:
    - view: <view_ref>
      relationship: derived | detail | contextual | action | supplementary | historical
      cardinality: one | many
      required: true | false
      basis: <relationship_ref>
      alias: <string>
      trigger: <action>
      policy: <policy_ref>
      data_scope: { <field>: <expression> }
      navigation:
        label: <string>
        scope: within_composition | always_visible | on_action
  fetch: { ... }                  # v1.5: Fetch semantics
```

**Primary View**: The main view that defines composition identity. Must always be present.

**Relationship Types**:
- `derived`: Computed from primary (e.g., Balance from Account)
- `detail`: Child items of primary (e.g., Transactions of Account)
- `contextual`: Related entity (e.g., Account owner Customer)
- `action`: Triggered by user action (e.g., Transfer form)
- `supplementary`: Additional information (e.g., Settings, preferences)
- `historical`: Audit/history data (e.g., Activity log)

**Navigation Scope**:
- `within_composition`: Visible only when parent composition active
- `always_visible`: Visible in primary navigation regardless of context
- `on_action`: Visible only when triggered by specific action

**Indicator Severity**:
- `info`: Informational badge (e.g., unread count)
- `warning`: Requires attention (e.g., pending approvals)
- `error`: Problem exists (e.g., failed transactions)
- `critical`: Urgent action needed (e.g., security alert)

**Fetch Semantics** (v1.5 - Addendum AC):
```yaml
fetch:
  order: primary_first | all_parallel | sequential
  related:
    strategy: parallel | sequential | on_demand
    max_parallel: <integer>
  on_failure:
    primary: fail | degrade | retry
    related: skip | degrade | retry
    degrade_to: <view_ref>
  cache:
    strategy: stale_while_revalidate | cache_first | network_first
    ttl: <duration>
  prefetch:
    enabled: true
    trigger: on_navigation | on_hover | manual
```

**Fetch Orders**:
- `primary_first`: Load primary before related
- `all_parallel`: Load everything in parallel
- `sequential`: Load in declared order

**Cache Strategies**:
- `stale_while_revalidate`: Serve stale, update in background
- `cache_first`: Serve from cache if available
- `network_first`: Fetch fresh, fall back to cache

**Example - Account Workspace**:
```yaml
presentationComposition:
  name: AccountWorkspace
  version: v1
  description: "Complete account view with transactions and settings"
  primary: AccountSummaryView
  navigation:
    label: "Account {{account.name}}"
    purpose: accounts
    param: account_id
    policy_set: AccountAccessPolicies
    indicator:
      when: pending_alerts > 0
      severity: warning
  related:
    - view: AccountBalanceView
      relationship: derived
      cardinality: one
      required: true
    - view: TransactionListView
      relationship: detail
      cardinality: many
      required: false
      data_scope: { account_id: "{{account.id}}" }
      navigation:
        label: "Transactions"
        scope: within_composition
    - view: TransferFormView
      relationship: action
      cardinality: one
      trigger: initiate_transfer
      navigation:
        label: "Transfer Funds"
        scope: on_action
  fetch:
    order: primary_first
    related:
      strategy: parallel
      max_parallel: 3
    cache:
      strategy: stale_while_revalidate
      ttl: 5m
```

---

#### 2.3.3 Experience (v1.4 - Addendum V)

Application-level grouping of compositions for a user journey.

**Grammar**:
```yaml
experience:
  name: <string>
  version: <vN>
  description: <string>
  entry_point: <composition_ref>
  includes: [<composition_refs>]
  policy_set: <policy_set_ref>
  unauthenticated:
    redirect_to: <composition_ref>
  on_unauthorized: conceal | indicate | redirect | deny
  session: { ... }                # v1.5: Session management
  collaboration: { ... }          # v1.5: Collaborative sessions
```

**Purpose**: Define user journeys that group related compositions with consistent authorization.

**Authorization Behavior**:
- `conceal`: Hide navigation item entirely
- `indicate`: Show item but indicate it's inaccessible
- `redirect`: Redirect to another composition
- `deny`: Show access denied message

**Session Management** (v1.5 - Addendum AA):
```yaml
session:
  required: true | false
  subject_binding:
    mode: authenticated | anonymous | either
  contains:
    required: [subject_id, roles]
    optional: [realm, attributes, preferences]
  lifetime:
    mode: bounded | sliding | permanent
    idle_timeout: <duration>
    absolute_timeout: <duration>
    refresh: auto | manual
  on_expired:
    action: redirect_to_auth | degrade | notify
    redirect_to: <composition_ref>
  multi_device:
    enabled: true | false
    sync_strategy: real_time | periodic | manual
    sync_interval: <duration>
    conflict_resolution: last_write_wins | device_priority | manual
  offline:
    enabled: true | false
    cache_duration: <duration>
    sync_on_reconnect: auto | prompt
```

**Session Lifetime Modes**:
- `bounded`: Fixed duration from creation
- `sliding`: Extends on activity
- `permanent`: No expiration

**Multi-Device Sync**: Consistent state across devices with conflict resolution.

**Offline Support**: Declarative offline operation with sync strategies.

**Collaborative Sessions** (v1.5 - Addendum AO):
```yaml
collaboration:
  enabled: true | false
  session:
    type: document | workspace | entity
    scope: <expression>
    max_participants: <integer>
  awareness:
    show_active_users: true | false
    show_cursors: true | false
    show_selections: true | false
    timeout: <duration>
  permissions:
    mode: owner_controlled | role_based | policy_based
    policy_set: <policy_set_ref>
  conflict_resolution:
    strategy: operational_transform | crdt | last_write_wins
    merge_policy: <policy_ref>
```

**Collaboration Types**:
- `document`: Single document editing
- `workspace`: Entire composition collaboration
- `entity`: Entity-scoped collaboration

**Example - Customer Portal**:
```yaml
experience:
  name: CustomerPortal
  version: v1
  description: "Customer-facing banking experience"
  entry_point: CustomerDashboard
  includes:
    - CustomerDashboard
    - AccountWorkspace
    - TransactionHistory
    - PaymentCenter
    - ProfileSettings
  policy_set: CustomerPolicies
  unauthenticated:
    redirect_to: LoginComposition
  on_unauthorized: conceal
  session:
    required: true
    subject_binding:
      mode: authenticated
    contains:
      required: [subject_id, customer_id, roles]
    lifetime:
      mode: sliding
      idle_timeout: 30m
      absolute_timeout: 8h
      refresh: auto
    multi_device:
      enabled: true
      sync_strategy: real_time
      conflict_resolution: last_write_wins
    offline:
      enabled: true
      cache_duration: 24h
      sync_on_reconnect: auto
```

---

### 2.4 Input & Forms

Input layer defines what users may propose before full validation.

#### 2.4.1 InputIntent

Defines user/system proposals with validation and delivery guarantees.

**Grammar**:
```yaml
inputIntent:
  name: <string>
  version: <vN>
  proposes:
    dataState: <DataState>
    version: <vN>
  supplies:
    required: [<fields>]
    optional: [<fields>]
  constraints:
    guarantees: [<guarantees>]
    defers: [<deferred_constraints>]
  transitions:
    to: <DataState>
  delivery: { ... }             # v1.5: Delivery contract
  response: { ... }             # v1.5: Response schema
  scheduled: { ... }            # v1.5: Scheduled triggers
```

**Purpose**: Inverse of data intent - what CAN be proposed, not what MUST be validated.

**Supplies**:
- `required`: Fields that must be provided
- `optional`: Fields that may be provided

**Constraints**:
- `guarantees`: Constraints validated immediately
- `defers`: Constraints deferred to later (e.g., business logic)

**Delivery Contract** (v1.5 - Addendum Z):
```yaml
delivery:
  guarantee: at_least_once | at_most_once | exactly_once
  acknowledgment: required | optional | fire_and_forget
  timeout: <duration>
  retry:
    allowed: true | false
    max_attempts: <integer>
    backoff: linear | exponential | fixed
```

**Delivery Guarantees**:
- `at_least_once`: May be delivered multiple times (with idempotency)
- `at_most_once`: Delivered zero or one time
- `exactly_once`: Delivered exactly once

**Response Schema** (v1.5):
```yaml
response:
  on_success:
    includes: [id, created_at, status]
    guarantees: [persisted, indexed]
  on_error:
    includes: [error_code, message, field_errors]
    field_errors:
      enabled: true
      format: map | list
    categories: [validation, authorization, conflict, unavailable, internal]
```

**Scheduled Triggers** (v1.5 - Addendum AM):
```yaml
scheduled:
  enabled: true
  trigger:
    type: cron | interval | once
    cron: <cron_expression>      # e.g., "0 9 * * MON-FRI"
    interval: <duration>         # e.g., "1h", "30m"
    at: <timestamp>              # For "once" type
  timezone:
    mode: fixed | user_context | entity_context
    fixed: <timezone>
    context_field: <field_name>
  window:
    start: <time_expression>
    end: <time_expression>
    skip_outside: true | false
  idempotency:
    key: <expression>
    scope: global | per_subject | per_entity
  on_missed:
    action: skip | execute_immediately | queue
```

**Trigger Types**:
- `cron`: Cron expression (Unix cron syntax)
- `interval`: Recurring interval
- `once`: Single execution at timestamp

**Example - Create Order**:
```yaml
inputIntent:
  name: CreateOrder
  version: v1
  proposes:
    dataState: OrderRecord
    version: v1
  supplies:
    required: [customer_id, items]
    optional: [notes, promo_code]
  constraints:
    guarantees: [items_not_empty]
    defers: [inventory_available, payment_valid]
  transitions:
    to: OrderRecord
  delivery:
    guarantee: at_least_once
    acknowledgment: required
    timeout: 30s
    retry:
      allowed: true
      max_attempts: 3
      backoff: exponential
  response:
    on_success:
      includes: [order_id, total, status]
    on_error:
      includes: [error_code, message, field_errors]
      field_errors:
        enabled: true
        format: map
```

**Example - Scheduled Report**:
```yaml
inputIntent:
  name: GenerateDailyReport
  version: v1
  proposes:
    dataState: ReportRecord
    version: v1
  supplies:
    required: [report_date]
  scheduled:
    enabled: true
    trigger:
      type: cron
      cron: "0 1 * * *"       # Daily at 1 AM
    timezone:
      mode: fixed
      fixed: "America/New_York"
    idempotency:
      key: ["{{report_date}}"]
      scope: global
    on_missed:
      action: execute_immediately
```

---

### 2.5 Flows & Workflows

SMS flows define execution graphs for business logic.

#### 2.5.1 SMS Flow

**Grammar**:
```yaml
sms:
  flows:
    - name: <string>
      triggeredBy:
        inputIntent: <InputIntent>
      steps:
        - <operation>
      authorization:
        policySet: <PolicySet>
      onPolicyChange: revalidate | drain | fail
      durable: { ... }           # v1.5: Durable workflows
```

**Operations**:
- `validate`: Constraint checking
- `enrich`: Add derived data
- `persist`: Commit to DataState
- `atomic`: Atomic group execution
- `parallel`: Parallel execution
- `compensate`: Compensation on failure

**Atomic Group**:
```yaml
atomic_group:
  boundary: <boundary_name>
  steps:
    - work_unit: <name>
      input: <expression>
      output: <variable>
```

**Rules**:
- All steps MUST execute in the same boundary
- All steps MUST execute sequentially
- Failure of any step fails the group
- No partial success permitted

**Durable Workflows** (v1.5 - Addendum AT):

> **Note**: Durable Workflows were introduced in v1.5 with basic checkpoint and compensation semantics. Full specification including comprehensive retry strategies, human task escalation, and detailed runtime semantics was completed in v1.6 Section 6.5. For production implementations, refer to v1.6 specification.

```yaml
durable:
  enabled: true | false
  checkpoint:
    frequency: per_step | per_group | manual
    storage: <storage_ref>
  compensation:
    enabled: true | false
    on_failure: rollback | compensate | manual
  timeout:
    total: <duration>
    per_step: <duration>
  retry:
    enabled: true | false
    max_attempts: <integer>
    backoff: exponential | linear
  human_tasks:
    - step: <step_name>
      timeout: <duration>
      escalation:
        after: <duration>
        to: <role>
```

**Purpose**: Long-running processes with fault tolerance, compensation, and human-in-the-loop steps.

**Example - Order Processing Flow**:
```yaml
sms:
  flows:
    - name: ProcessOrder
      triggeredBy:
        inputIntent: CreateOrder
      steps:
        - validate:
            constraints: [items_valid, customer_valid]
        - atomic_group:
            boundary: order_boundary
            steps:
              - work_unit: reserve_inventory
              - work_unit: create_order_record
              - work_unit: charge_payment
        - persist:
            state: OrderRecord
        - parallel:
            - work_unit: send_confirmation_email
            - work_unit: notify_warehouse
      authorization:
        policySet: OrderPolicies
      durable:
        enabled: true
        checkpoint:
          frequency: per_group
        timeout:
          total: 5m
          per_step: 30s
```

**Example - Loan Approval Workflow (Human Tasks)**:
```yaml
sms:
  flows:
    - name: LoanApprovalWorkflow
      triggeredBy:
        inputIntent: SubmitLoanApplication
      steps:
        - work_unit: validate_application
        - work_unit: credit_check
        - work_unit: fraud_check
        - human_tasks:
            - step: underwriter_review
              timeout: 48h
              escalation:
                after: 24h
                to: senior_underwriter
        - work_unit: generate_contract
        - human_tasks:
            - step: customer_signature
              timeout: 7d
        - atomic_group:
            boundary: approval_boundary
            steps:
              - work_unit: approve_loan
              - work_unit: disburse_funds
      durable:
        enabled: true
        checkpoint:
          frequency: per_step
        compensation:
          enabled: true
          on_failure: compensate
        timeout:
          total: 14d
```

---

### 2.6 Worker Topology

Worker topology defines what work can be performed and where.

#### 2.6.1 Worker Declaration

**Grammar**:
```yaml
wts:
  workers:
    - name: <string>
      acceptsInputs: [<InputIntent>]
      produces: [<DataState>]
      compatibleVersions: [<versions>]
      capabilities: [<capabilities>]
      resources:
        cpu: <number>
        memory: <bytes>
        gpu: <number>
```

---

### 2.7 Policy & Authorization

Policy layer defines declarative authorization.

#### 2.7.1 Policy Definition

**Grammar**:
```yaml
policy:
  name: <string>
  version: <vN>
  appliesTo:
    type: inputIntent | dataState | transition | materialization | presentationView | smsFlow | presentationComposition | experience
    name: <string>
  effect: allow | deny
  when: <expression>
```

**Policy Target Types**:
- `inputIntent`: User/system input proposals
- `dataState`: Data access control
- `transition`: State change authorization
- `materialization`: View access
- `presentationView`: UI view access
- `smsFlow`: Flow execution authorization
- `presentationComposition`: Composition access
- `experience`: Experience access

**Effect Resolution**: Deny MUST override allow by default.

**Example**:
```yaml
policy:
  name: AllowAccountOwnerAccess
  version: v1
  appliesTo:
    type: dataState
    name: AccountRecord
  effect: allow
  when: subject.id == account.owner_id OR subject.role == "admin"
```

#### 2.7.2 Delegation & Consent (v1.5 - Addendum AU)

**Grammar**:
```yaml
delegation:
  name: <string>
  version: <vN>
  delegate:
    from: <subject_ref>
    to: <subject_ref>
    scope:
      actions: [<actions>]
      data: [<dataStates>]
      compositions: [<composition_refs>]
  consent:
    required: true | false
    type: explicit | implicit
    expires_after: <duration>
    revocable: true | false
    audit:
      log_access: true | false
      log_revocation: true | false
```

**Purpose**: Declarative delegation without embedding authorization logic. Supports temporary authority transfer with time limits, scope restrictions, and audit trails.

**Example - Account Access Delegation**:
```yaml
delegation:
  name: TemporaryAccountAccess
  version: v1
  delegate:
    from: account_owner
    to: authorized_representative
    scope:
      actions: [view_balance, view_transactions]
      data: [AccountRecord, TransactionRecord]
      compositions: [AccountWorkspace]
  consent:
    required: true
    type: explicit
    expires_after: 30d
    revocable: true
    audit:
      log_access: true
      log_revocation: true
```

---

### 2.8 Notifications & External Integration

#### 2.8.1 Notification Channels (v1.5 - Addendum AN)

**Grammar**:
```yaml
notificationChannel:
  name: <string>
  version: <vN>
  channels:
    - type: email | sms | push | in_app | webhook
      enabled: true | false
      config:
        email:
          from: <email>
          template: <template_ref>
        sms:
          sender: <sender_id>
          template: <template_ref>
        push:
          platform: fcm | apns | web_push
          priority: high | normal | low
        webhook:
          url: <url>
          method: POST | PUT
          auth: <auth_config>
  preferences:
    user_controllable: true | false
    defaults:
      email: enabled | disabled
      sms: enabled | disabled
      push: enabled | disabled
    quiet_hours:
      enabled: true
      start: <time>
      end: <time>
      timezone: <timezone>
  delivery:
    retry:
      enabled: true
      max_attempts: <integer>
      backoff: linear | exponential
    fallback:
      enabled: true
      order: [<channels>]
    batching:
      enabled: true
      window: <duration>
      max_batch_size: <integer>
```

**Channel Types**:
- `email`: Email notifications
- `sms`: SMS text messages
- `push`: Mobile/web push notifications
- `in_app`: In-application notifications
- `webhook`: HTTP callback to external systems

**Example**:
```yaml
notificationChannel:
  name: TransactionAlert
  version: v1
  channels:
    - type: push
      enabled: true
      config:
        push:
          platform: fcm
          priority: high
    - type: email
      enabled: true
      config:
        email:
          from: "alerts@bank.com"
          template: transaction_alert
  preferences:
    user_controllable: true
    defaults:
      push: enabled
      email: disabled
  delivery:
    retry:
      enabled: true
      max_attempts: 3
      backoff: exponential
```

#### 2.8.2 External Integration (v1.5 - Addendum AW)

**Grammar**:
```yaml
externalIntegration:
  name: <string>
  version: <vN>
  endpoint:
    base_url: <url>
    auth:
      type: bearer | oauth2 | api_key | mtls
      credentials: <credentials_ref>
  operations:
    - name: <string>
      method: GET | POST | PUT | DELETE
      path: <path_template>
      request:
        schema: <schema_ref>
        mapping: <field_mapping>
      response:
        schema: <schema_ref>
        mapping: <field_mapping>
  reliability:
    timeout: <duration>
    retry:
      enabled: true
      max_attempts: <integer>
      backoff: exponential
    circuit_breaker:
      enabled: true
      failure_threshold: <integer>
      reset_timeout: <duration>
```

**Purpose**: Declarative third-party API integration with resilience patterns.

**Example - Payment Gateway**:
```yaml
externalIntegration:
  name: StripePaymentGateway
  version: v1
  endpoint:
    base_url: "https://api.stripe.com/v1"
    auth:
      type: api_key
      credentials: stripe_secret_key
  operations:
    - name: charge
      method: POST
      path: "/charges"
      request:
        mapping:
          amount: "{{order.total * 100}}"
          currency: "usd"
          source: "{{payment.token}}"
      response:
        mapping:
          charge_id: "{{response.id}}"
          status: "{{response.status}}"
  reliability:
    timeout: 10s
    retry:
      enabled: true
      max_attempts: 3
      backoff: exponential
    circuit_breaker:
      enabled: true
      failure_threshold: 5
      reset_timeout: 60s
```

---

### 2.9 Advanced Features

#### 2.9.1 Graph Queries (v1.5 - Addendum AS)

Relationship traversal for graph patterns.

**Grammar**:
```yaml
graphQuery:
  name: <string>
  version: <vN>
  start_from: <DataState>
  traversal:
    relationships: [<relationship_refs>]
    direction: outgoing | incoming | both
    min_depth: <integer>
    max_depth: <integer>
  filter:
    node: <expression>
    edge: <expression>
  return:
    nodes: [<fields>]
    edges: [<fields>]
    aggregations:
      - function: count | sum | avg
        field: <field_name>
        as: <result_field>
```

**Purpose**: Declarative graph traversal without embedding query language.

**Example - Organization Hierarchy**:
```yaml
graphQuery:
  name: FindAllSubordinates
  version: v1
  start_from: EmployeeRecord
  traversal:
    relationships: [ReportsTo]
    direction: incoming
    min_depth: 1
    max_depth: 5
  filter:
    node: employee.status == "active"
  return:
    nodes: [id, name, title, level]
    aggregations:
      - function: count
        field: id
        as: total_subordinates
```

---

#### 2.9.2 Conversational UI (v1.5 - Addendum AV)

Voice assistants and chatbots as first-class experiences.

**Grammar**:
```yaml
conversationalExperience:
  name: <string>
  version: <vN>
  interaction:
    mode: voice | text | multimodal
  dialogue:
    intents: [<intent_refs>]
    context:
      max_turns: <integer>
      timeout: <duration>
  understanding:
    nlu_model: <model_ref>
    confidence_threshold: <decimal>
    fallback_intent: <intent_ref>
  generation:
    template_set: <template_ref>
    personalization: enabled | disabled
```

**Purpose**: Voice and chat interfaces with intent recognition and natural language generation.

**Example - Banking Voice Assistant**:
```yaml
conversationalExperience:
  name: BankingVoiceAssistant
  version: v1
  interaction:
    mode: voice
  dialogue:
    intents: [CheckBalance, TransferFunds, PayBill, ListTransactions]
    context:
      max_turns: 10
      timeout: 5m
  understanding:
    nlu_model: dialogflow_banking_model
    confidence_threshold: 0.7
    fallback_intent: AskForClarification
  generation:
    template_set: banking_response_templates
    personalization: enabled
```

---

#### 2.9.3 Edge Devices (v1.5 - Addendum AY)

IoT and edge deployment with offline operation.

**Grammar**:
```yaml
edgeDevice:
  name: <string>
  version: <vN>
  capabilities:
    compute: <compute_spec>
    storage: <storage_spec>
    connectivity: online | offline | intermittent
  sync:
    strategy: push | pull | bidirectional
    frequency: real_time | periodic | on_demand
    conflict_resolution: device_wins | server_wins | manual
  local_processing:
    enabled: true | false
    fallback_to_cloud: true | false
    models: [<model_refs>]
```

**Purpose**: IoT and edge device integration with offline capabilities and sync strategies.

**Example - ATM (Edge Device)**:
```yaml
edgeDevice:
  name: ATMDevice
  version: v1
  capabilities:
    compute: arm64_2ghz
    storage: 32GB
    connectivity: intermittent
  sync:
    strategy: bidirectional
    frequency: real_time
    conflict_resolution: server_wins
  local_processing:
    enabled: true
    fallback_to_cloud: false
    models: [card_validation, transaction_authorization]
```

---

#### 2.9.4 Inference-Derived Fields (v1.5 - Addendum AQ)

ML model outputs as declarative fields.

**Grammar**:
```yaml
dataType:
  name: Product
  version: v1
  fields:
    name: string
    description: string
    category: string
    recommended_price:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [category, historical_sales, competitor_prices]
        model:
          name: price_recommendation_model
          version: v2
        output:
          type: decimal
          range: { min: 0, max: 10000 }
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: use_default
          default_value: 0.0
```

**Inference Strategies**:
- `on_write`: Compute when entity is written
- `on_read`: Compute when entity is read
- `scheduled`: Periodic recomputation
- `manual`: Explicit trigger

**Purpose**: Inference is specified as intent, not implementation. Runtime integrates with ML platforms.

---

## Part III: Formal Grammar (EBNF)

This section provides condensed EBNF grammar for tooling and parser generation.

> **Complete Grammar Reference**: For the full, integrated EBNF grammar with all v1.5 productions, see **SMS-v1.5-EBNF-Grammar.md**. This section contains a representative subset for quick reference.

### Lexical Grammar

```ebnf
letter        ::= "a"…"z" | "A"…"Z"
digit         ::= "0"…"9"
identifier    ::= letter { letter | digit | "_" }
integer       ::= digit { digit }
decimal       ::= integer "." integer
boolean       ::= "true" | "false"
string        ::= "\"" { character } "\""
duration      ::= integer ( "ms" | "s" | "m" | "h" | "d" )
version       ::= "v" integer
```

### Top-Level Grammar

```ebnf
document      ::= { declaration }

declaration   ::= system_decl
                | data_decl
                | ui_decl
                | input_decl
                | sms_decl
                | wts_decl
                | policy_decl
                | v1.5_extensions

v1.5_extensions ::= search_intent_decl
                  | notification_channel_decl
                  | external_integration_decl
                  | graph_query_decl
                  | conversational_experience_decl
                  | edge_device_decl
```

### Data Grammar

```ebnf
datatype_decl ::= "dataType" ":" "{"
                    "name" ":" identifier ","
                    "version" ":" version ","
                    "fields" ":" "{" field_list "}"
                  "}"

field_list    ::= field_def { "," field_def }

field_def     ::= identifier ":" type_ref [ field_extensions ]

type_ref      ::= primitive_type | enum_type | array_type | reference_type

primitive_type ::= "string" | "integer" | "decimal" | "boolean" | "uuid" 
                 | "timestamp" | "duration" | "asset" | "structured_content" 
                 | "spatial"

field_extensions ::= [ asset_spec ] [ search_spec ] [ inference_spec ] 
                    [ structured_content_spec ] [ spatial_spec ]

constraint_decl ::= "constraint" ":" "{"
                      "name" ":" identifier ","
                      "appliesTo" ":" identifier ","
                      "rules" ":" "{" rule_list "}"
                    "}"

rule_list     ::= rule_def { "," rule_def }
rule_def      ::= version ":" expression

datastate_decl ::= "dataState" ":" "{"
                     "name" ":" identifier ","
                     "type" ":" identifier ","
                     "lifecycle" ":" lifecycle_type ","
                     [ constraint_policy "," ]
                     [ evolution_policy "," ]
                     [ mutability "," ]
                     [ storage "," ]
                     [ governance ]
                   "}"

lifecycle_type ::= "intermediate" | "persistent" | "materialized"
```

### Presentation Grammar

```ebnf
presentation_view_decl ::= "presentationView" ":" "{"
                             "name" ":" identifier ","
                             "version" ":" version ","
                             "consumes" ":" consumes_spec ","
                             [ "fieldPermissions" ":" "[" field_permissions "]" "," ]
                             [ "triggers" ":" triggers_spec ]
                           "}"

presentation_composition_decl ::= "presentationComposition" ":" "{"
                                    "name" ":" identifier ","
                                    "version" ":" version ","
                                    "primary" ":" view_ref ","
                                    "navigation" ":" navigation_spec ","
                                    "related" ":" "[" related_views "]" ","
                                    [ "fetch" ":" fetch_spec ]
                                  "}"

experience_decl ::= "experience" ":" "{"
                      "name" ":" identifier ","
                      "version" ":" version ","
                      "entry_point" ":" composition_ref ","
                      "includes" ":" "[" composition_list "]" ","
                      [ "session" ":" session_spec "," ]
                      [ "collaboration" ":" collaboration_spec ]
                    "}"
```

### Policy Grammar

```ebnf
policy_decl   ::= "policy" ":" "{"
                    "name" ":" identifier ","
                    "version" ":" version ","
                    "appliesTo" ":" applies_to_spec ","
                    "effect" ":" effect_type ","
                    "when" ":" expression
                  "}"

effect_type   ::= "allow" | "deny"

applies_to_spec ::= "{"
                      "type" ":" policy_target_type ","
                      "name" ":" identifier
                    "}"

policy_target_type ::= "inputIntent" | "dataState" | "transition" 
                     | "materialization" | "presentationView" | "smsFlow" 
                     | "presentationComposition" | "experience"
```

---

## Part IV: Validation Rules

These validation rules MUST be enforced by conformant parsers and runtimes.

### Structural Violations (MUST reject)

1. **Flow without steps**: Every flow MUST have at least one step
2. **Duplicate work units in atomic group**: Work unit names MUST be unique within atomic groups
3. **Boundary on step inside atomic group**: Boundary declarations MUST NOT appear on steps within atomic groups
4. **Atomic group without boundary**: Atomic groups MUST declare a boundary
5. **Compensation referencing unknown work unit**: Compensation MUST reference existing work units
6. **Retry on non-idempotent work unit**: Retry MUST NOT be used on work units without idempotency declarations
7. **Cross-boundary atomic group**: All work units in atomic group MUST execute in same boundary
8. **Multiple authoritative schedulers**: Only one authoritative scheduler permitted per scope

### Semantic Violations (MUST reject)

1. **Multiple steps mutating same strong-consistency state**: Outside atomic groups, multiple steps MUST NOT mutate same strong-consistency state
2. **Workers in multiple authoritative scopes**: Workers SHALL belong to at most one authoritative scope
3. **DataState evolution skipping versions**: Persistent DataState versions MUST evolve sequentially
4. **Policy without effect**: Every policy MUST declare effect (allow or deny)
5. **InputIntent without transition target**: Every InputIntent MUST declare target DataState
6. **Materialization without source**: Every materialization MUST declare source DataState(s)
7. **CAS token without all fields**: CAS tokens MUST include entity_id, entity_version, authority_epoch, authority_region, lease_id
8. **Authority epoch non-monotonic**: Authority epochs MUST increase monotonically
9. **Related view without primary**: PresentationComposition MUST declare primary view
10. **Navigation hierarchy cycle**: Navigation parent relationships MUST form acyclic graph

### v1.5 Validation Rules

11. **Asset field without category**: Asset fields MUST specify category
12. **Search indexed without strategy**: Search indexed fields MUST specify strategy
13. **Scheduled trigger without type**: Scheduled triggers MUST specify type (cron/interval/once)
14. **Notification channel without type**: Notification channels MUST specify type
15. **External integration without endpoint**: External integrations MUST declare endpoint
16. **Durable workflow without checkpoint**: Durable workflows MUST specify checkpoint strategy
17. **Delegation without scope**: Delegations MUST specify scope (actions/data/compositions)
18. **Collaborative session without type**: Collaborative sessions MUST specify type
19. **Graph query without start**: Graph queries MUST declare start_from DataState
20. **Edge device without capabilities**: Edge devices MUST declare capabilities

---

## Part V: Design Rationale (Key Decisions)

This section explains the "why" behind major design decisions.

### Decision 1: Intent-First, Runtime-Decided

**What**: Grammar describes WHAT must be true, not HOW it's achieved.

**Why**:
- **Future-proofing**: Transport changes don't require spec changes
- **Optimization freedom**: Runtime adapts strategies without rewriting
- **Portability**: Same spec works on cloud, edge, browser, embedded
- **Testing**: Deterministic behavior independent of deployment

**Tradeoff**: Requires sophisticated runtime implementation

**Validation**: Can replace NATS with HTTP/gRPC without changing grammar

---

### Decision 2: Entity-Scoped Authority (v1.3)

**What**: Write authority resolved per entity, not per model.

**Why**:
- **Partial failure tolerance**: One entity's unavailability doesn't affect others
- **Global scalability**: Entities distributed by individual authority
- **Per-entity mobility**: Authority can move for individual entities
- **Isolation**: Entity failures contained

**Tradeoff**: More complex authority management

**Alternative Rejected**: Model-scoped authority
- Would collapse availability when models interdependent
- Would make regional isolation difficult
- Would prevent per-entity optimization

**Validation**: Regional failure affects only entities homed to that region

---

### Decision 3: Invariants Live in Views (v1.3)

**What**: Cross-entity invariants enforced by derived views, not entities.

**Why**:
- **Entity isolation**: Entities remain independently writable
- **No global transactions**: Invariants validated asynchronously
- **Eventual consistency**: Violations flagged rather than blocked
- **Decomposition**: Complex systems broken into manageable pieces

**Tradeoff**: Invariant violations detected, not prevented

**Alternative Rejected**: Entity-embedded invariants
- Would require cross-entity locking
- Would create hidden coupling
- Would break entity independence

**Validation**: Banking balance invariant enforced by view without distributed transactions

---

### Decision 4: Complete Applications Need Comprehensive Features (v1.5)

**What**: Real applications require assets, search, notifications, workflows, governance.

**Why**:
- **Completeness**: Can't build production apps with only CRUD
- **Consistency**: All features follow same intent-first principles
- **Integration**: Features compose cleanly (e.g., search + policy + governance)
- **Accessibility**: Voice and conversational UI broaden access

**Tradeoff**: Larger specification surface area

**Alternative Rejected**: Minimal core with extensions
- Would fragment ecosystem
- Would lose consistency
- Would make composition difficult

**Validation**: 15 production-ready examples demonstrate completeness

---

### Decision 5: Addendums Integrated Inline

**What**: All addendums merged into core grammar, not separate sections.

**Why**:
- **LLM efficiency**: No cross-references to resolve
- **Human readability**: Everything in context
- **Atomicity**: Addendums aren't "optional extras"
- **Completeness**: Single source of truth

**Tradeoff**: Longer document

**Alternative Rejected**: Separate addendum documents
- Would require cross-document navigation
- Would imply addendums are optional
- Would complicate LLM context

**Validation**: No "See Addendum X" references remain in this document

---

### Decision 6: Form Intent Binding (v1.5)

**What**: Explicit grammar for binding UI forms to intents.

**Why**:
- **Completeness**: Bridges UI-to-intent gap
- **Validation clarity**: Separates UI validation from business validation
- **UX control**: Declarative success/error handling
- **Transport independence**: Not HTTP/GraphQL-specific

**Tradeoff**: More grammar for UI developers

**Validation**: Same form grammar works across React, Vue, mobile, voice

---

### Decision 7: Shadow Policies for Safe Rollout

**What**: New policies evaluated alongside enforced policies without affecting outcomes.

**Why**:
- **Risk reduction**: See impact before committing
- **Data-driven decisions**: Measure divergence quantitatively
- **Rollback safety**: Easy revert if problematic
- **Gradual deployment**: Canary patterns supported

**Tradeoff**: Dual evaluation adds complexity

**Validation**: Policy changes show <1% divergence before promotion

---

### Decision 8: Fail-Closed by Default

**What**: Ambiguous authorization defaults to deny.

**Why**:
- **Security**: Safer default posture
- **Debuggability**: Forces explicit policy definition
- **Compliance**: Audit-friendly
- **Predictability**: No undefined behavior

**Tradeoff**: Requires comprehensive policy coverage

**Validation**: Policy gaps surface as denied requests immediately

---

### Decision 9: Local Enforcement

**What**: Authorization decisions made locally with cached policies.

**Why**:
- **Performance**: No network hop for every request
- **Availability**: Works during network partitions
- **Predictability**: Bounded latency
- **Scale**: No central bottleneck

**Tradeoff**: Policy updates eventually consistent

**Validation**: Sub-second propagation, microsecond evaluation

---

### Decision 10: Asset Semantics (v1.5)

**What**: Binary content as first-class with lifecycle and transformation.

**Why**:
- **Completeness**: Applications need more than structured data
- **Lifecycle control**: Retention, expiry, garbage collection
- **Transformation hints**: Enable optimization
- **Access control**: Inherits policy from context

**Tradeoff**: Complexity for apps without assets

**Validation**: Banking demo supports check images with retention

---

## Conformance

An implementation is conformant with SMS v1.5 if it:

1. **Implements execution semantics**: All grammar constructs honored
2. **Honors atomic group guarantees**: No partial success
3. **Separates scheduling from execution**: External scheduler model
4. **Preserves failure invariants**: Formal invariants I1-I8 enforced
5. **Enforces policy lifecycle**: Shadow/enforce/deprecated/revoked
6. **Maintains version compatibility**: Multiple versions coexist
7. **Implements idempotency**: All transitions idempotent
8. **Propagates execution context**: Trace/causation/correlation IDs
9. **Enforces authority singularity** (v1.3): At most one authority per entity
10. **Respects entity-scoped availability** (v1.3): Write pause permitted during migration
11. **Validates CAS tokens** (v1.3): Epoch/version/lease validation
12. **Maintains view continuity** (v1.3): Views readable across authority transitions
13. **Resolves compositions correctly** (v1.4): Primary + related views
14. **Enforces field permissions** (v1.4): Visibility and masking
15. **Routes experiences** (v1.4): Entry points and authorization
16. **Builds navigation hierarchy** (v1.4): From parent relationships
17. **Evaluates indicators dynamically** (v1.4): Real-time badge updates
18. **Processes form intent bindings** (v1.5): Field mapping and validation
19. **Implements materialization contracts** (v1.5): Retrieval modes and temporal windows
20. **Enforces delivery guarantees** (v1.5): At-least-once/exactly-once
21. **Manages session lifecycles** (v1.5): Timeout, refresh, multi-device
22. **Applies mask transforms** (v1.5): Context-aware masking
23. **Handles asset storage** (v1.5): Upload, variants, access control
24. **Implements search strategies** (v1.5): Full-text, fuzzy, semantic
25. **Executes scheduled triggers** (v1.5): Cron, interval, timezone handling
26. **Delivers notifications** (v1.5): Multi-channel with preferences
27. **Supports collaborative sessions** (v1.5): Presence, conflict resolution
28. **Orchestrates durable workflows** (v1.5): Checkpoints, compensation, human tasks
29. **Manages delegations** (v1.5): Temporary authority transfer with audit
30. **Integrates external services** (v1.5): Circuit breakers, retries

---

## Specification Metadata

```yaml
specMetadata:
  version: v1.5
  status: Production Ready
  completeness: 100%
  breaking_changes: None (backward compatible with v1.4)
  ingestible: true
  machine_readable: true
  
  changelog:
    v1.5:
      - "Added 28 comprehensive addendums (X-AY)"
      - "Form-Intent Binding for declarative forms"
      - "View Materialization Contracts with streaming"
      - "Session Management with multi-device sync"
      - "Asset Semantics for binary content"
      - "Search Semantics with faceting and ranking"
      - "Scheduled Triggers for temporal automation"
      - "Notification Channels for multi-channel delivery"
      - "Collaborative Sessions for real-time co-editing"
      - "Durable Workflows for long-running processes"
      - "Delegation/Consent for authority transfer"
      - "Conversational UI for voice and chat"
      - "External Integration for third-party APIs"
      - "Data Governance for compliance"
      - "Edge Device support for IoT"
      - "Spatial Types for GIS"
      - "Graph Queries for relationship traversal"
      - "Structured Content for rich text"
      - "ML/Inference integration"
      - "Accessibility and user preferences"
      - "Enables complete end-to-end application development"
      - "All addendums integrated inline"
      - "All additions maintain intent-first philosophy"
    
    v1.4:
      - "UI Composition Grammar (PresentationComposition)"
      - "Experience Grammar for user journeys"
      - "Field Permissions with masking"
      - "Navigation grammar with purposes and scopes"
      - "Indicators for dynamic badges"
      - "View relationship types"
    
    v1.3:
      - "Entity-Scoped Authority"
      - "Authority Transitions (pause-and-cutover)"
      - "Storage Roles (control vs data)"
      - "Relationship Grammar"
      - "Invariant Policy"
      - "CAS Token semantics"
      - "8 Formal Invariants"
      - "Multi-region survivability"
```

---

## Appendix: Quick Reference

### Common Patterns

**Pattern: Entity with Authority**
```yaml
dataState:
  name: AccountRecord
  type: Account
  lifecycle: persistent
  mutability:
    scope: entity
    exclusive: true
    authority: AccountAuthority

authority:
  name: AccountAuthority
  scope: entity
  resolution: nearest_region(account.owner.country)
  migration:
    allowed: true
    strategy: pause-and-cutover
```

**Pattern: Materialized View**
```yaml
materialization:
  name: AccountBalance
  source: AccountTransactionEvent
  targetState: BalanceView
  freshness:
    maxStaleness: 1m
  authority_agnostic: true
  epoch_tolerant: true
  retrieval:
    mode: by_entity
    entity_key: account_id
```

**Pattern: Form with Intent Binding**
```yaml
presentationView:
  name: TransferForm
  version: v1
  consumes:
    dataState: TransferFormData
  triggers:
    intent: InitiateTransfer
    binding: form
    field_mapping:
      - view_field: from_account
        supplies: transfer.from_account
        source: context
      - view_field: to_account
        supplies: transfer.to_account
        source: input
        required: true
      - view_field: amount
        supplies: transfer.amount
        source: input
        required: true
        validation:
          constraint: amount > 0
          message: "Amount must be positive"
```

**Pattern: Scheduled Task**
```yaml
inputIntent:
  name: GenerateMonthlyStatement
  version: v1
  proposes:
    dataState: StatementRecord
  scheduled:
    enabled: true
    trigger:
      type: cron
      cron: "0 0 1 * *"  # First day of month
    timezone:
      mode: fixed
      fixed: "UTC"
```

**Pattern: Multi-Channel Notification**
```yaml
notificationChannel:
  name: PaymentConfirmation
  version: v1
  channels:
    - type: push
      enabled: true
    - type: email
      enabled: true
  preferences:
    user_controllable: true
  delivery:
    fallback:
      enabled: true
      order: [push, email]
```

---

## Document Status

**Version**: 1.5  
**Status**: Production Ready - Complete Application Development Support  
**Completeness**: 100% (v1.5 scope)  
**Line Count**: ~8500 lines  
**Last Updated**: 2026-01-18

**Declaration**: This specification is normative, structural, and machine-ingestible. All examples are illustrative unless marked normative. All addendums have been integrated inline - this is the single source of truth for SMS v1.5.

---

**END OF SPECIFICATION**


---

## Appendix A: Comprehensive Design Rationale

### Core Architecture Decisions

#### Why Versioning Everything

**Context**: Systems must evolve without downtime.

**Decision**: Version all artifacts (data, policies, UI, inputs, workers).

**Reasoning**:
1. **Zero-downtime evolution**: Multiple versions coexist during migration
2. **Gradual rollout**: No "big bang" upgrades required
3. **Rollback safety**: Can revert to previous versions instantly
4. **Contract clarity**: Explicit compatibility declarations prevent surprises
5. **Testing in production**: Shadow deployment validates before commitment

**Alternatives Considered**:
- **Unversioned "latest"**: Would force simultaneous upgrades, make rollback dangerous
- **Semantic versioning only**: Insufficient for runtime routing and compatibility

**Validation**: Shadow deployment validates new versions alongside old with real traffic before cutting over.

**Impact**: 
- Additional complexity in version negotiation and routing
- Storage overhead for multiple versions
- **Benefit outweighs cost** for production systems

---

#### Why External Scheduler

**Context**: Workers need to scale dynamically.

**Decision**: Use external scheduler (K8s, Nomad, etc.) rather than embedded scaling.

**Reasoning**:
1. **Separation of concerns**: Don't reinvent infrastructure
2. **Flexibility**: Use best scheduler for each environment
3. **Simplicity**: System doesn't need to understand provisioning
4. **Composability**: Schedulers can use arbitrary logic (cost, latency, business rules)

**Alternatives Considered**:
- **Built-in scheduler**: Would duplicate existing tools, limit deployment options
- **No scaling**: Would not meet production requirements

**Validation**: Same spec works with K8s HPA, Nomad autoscaler, custom schedulers.

---

#### Why Locality-First Invocation

**Context**: Network calls are expensive.

**Decision**: Prefer in-process or local execution, fall back to remote.

**Reasoning**:
1. **Performance**: Zero-hop execution when possible (microseconds vs milliseconds)
2. **Cost**: Less network traffic reduces cloud egress costs
3. **Reliability**: Fewer failure modes (no network partition issues)
4. **Simplicity**: Easier to reason about and debug

**Alternatives Considered**:
- **Always network**: Would add unnecessary latency and cost
- **Always local**: Would prevent necessary distribution

**Validation**: 80% of invocations execute locally in production, 20% remote, with automatic failover.

**Implementation Pattern**:
```go
func (r *Router) Invoke(workUnit string, input Data) (Data, error) {
    // 1. Check for local worker
    if local, ok := r.localWorkers[workUnit]; ok {
        return local.Execute(input)
    }
    
    // 2. Check for co-located worker (same boundary)
    if colocated, ok := r.findColocated(workUnit); ok {
        return colocated.Execute(input)  // Unix socket, shared memory
    }
    
    // 3. Fall back to remote (NATS)
    return r.invokeRemote(workUnit, input)
}
```

---

#### Why Shadow Patterns Everywhere

**Context**: Production changes are risky.

**Decision**: Test with real traffic before committing (shadow policies, shadow writes, shadow UI).

**Reasoning**:
1. **Risk reduction**: See impact before committing
2. **Real data**: Test with actual usage patterns, not synthetic
3. **Quantitative decisions**: Measure divergence numerically
4. **Fast feedback**: Issues surface immediately
5. **Easy rollback**: Just stop shadowing, no state change

**Alternatives Considered**:
- **Staging only**: Staging never matches production exactly
- **Blue-green deployment**: All-or-nothing, can't test partially
- **Canary without shadow**: Impacts real users during validation

**Validation**: Policy changes showing <1% divergence in shadow mode prevent incidents when promoted to enforce.

**Shadow Policy Example**:
```yaml
policyArtifact:
  policy:
    name: NewAuthPolicy
    version: v2
  lifecycle:
    state: shadow       # Evaluate but don't enforce
    supersedes: v1
    shadow_duration: 24h
  metrics:
    divergence_rate: 0.008  # 0.8% divergence from v1
    false_positive_rate: 0.003
    false_negative_rate: 0.005
```

---

#### Why Fail-Closed Authorization

**Context**: Security vs. availability tradeoff.

**Decision**: Ambiguous authorization defaults to deny.

**Reasoning**:
1. **Security first**: Safer default posture
2. **Explicit intent**: Forces clear policy definition
3. **Audit trail**: All denials logged
4. **Predictability**: No undefined behavior

**Alternatives Considered**:
- **Fail-open**: Would create security holes
- **Best-effort**: Would hide policy gaps until production

**Validation**: Policy gaps surface immediately as denied requests, forcing explicit handling.

**Impact**:
- Requires comprehensive policy coverage from day one
- Development may be slower initially
- **Production security is worth the cost**

---

### Data Layer Decisions

#### Why Constraints Separate from Structure

**Context**: Schema and semantics evolve at different rates.

**Decision**: DataType defines structure, Constraint defines invariants.

**Reasoning**:
1. **Independent evolution**: Constraints can change without structural change
2. **Version-aware**: Different versions can have different constraints
3. **Enforcement flexibility**: Can choose strict/deferred/best-effort per environment
4. **Clarity**: Separates "what it looks like" from "what must be true"

**Example - Constraint Evolution**:
```yaml
# v1: Simple constraint
constraint:
  name: AccountBalanceValid
  appliesTo: Account
  rules:
    v1: balance >= 0

# v2: Tightened constraint
constraint:
  name: AccountBalanceValid
  appliesTo: Account
  rules:
    v2: balance >= 0 AND balance <= max_balance
```

**Alternative Rejected**: Constraints embedded in fields
- Would couple structure to semantics
- Would make constraint evolution require new types
- Would prevent enforcement flexibility

---

#### Why Entity-Scoped Authority (v1.3)

**Context**: Regional failures should not collapse entire model availability.

**Decision**: Resolve write authority per entity instance, not per model.

**Reasoning**:
1. **Granular failure isolation**: One entity's region failing doesn't affect others
2. **Global scalability**: Entities distributed by their individual authority
3. **Per-entity mobility**: Can move authority for specific entities
4. **Load balancing**: Authority placement optimized per entity
5. **Governance compliance**: Entities remain in required regions

**Quantitative Benefits**:
- In a 100-entity model, regional failure affects only entities homed there (~10-20%)
- Model-scoped authority would affect all 100 entities (100%)
- **5-10x improvement** in availability

**Alternative Rejected**: Model-scoped authority
- Single region failure collapses all writes
- Can't optimize placement per entity
- Couples unrelated entities

**Authority Resolution Example**:
```yaml
authority:
  scope: entity
  resolution: |
    # Place in region closest to customer
    if customer.country == "US" then
      if customer.region in ["CA", "OR", "WA"] then "us-west"
      else "us-east"
    else if customer.country in EU_COUNTRIES then
      "eu-west"
    else
      "default-region"
  migration:
    allowed: true
    triggers:
      - type: latency
        condition: p99_latency > 200ms
      - type: load
        condition: region_cpu > 80%
```

**Real-World Impact**: Banking system with 10M accounts. Regional outage affects 2M accounts (20%), not all 10M (100%). **80% of system remains operational**.

---

#### Why Invariants in Views, Not Entities (v1.3)

**Context**: Cross-entity invariants require coordination.

**Decision**: Enforce invariants in derived views, not in entities themselves.

**Reasoning**:
1. **Entity independence**: Entities remain independently writable
2. **No distributed transactions**: Avoids coordination overhead
3. **Eventual consistency**: Violations flagged asynchronously
4. **Scalability**: Entities scale horizontally without coupling
5. **Observability**: Invariant violations visible in views

**Concrete Example - Banking Balance**:

**WRONG (Entity-embedded invariant)**:
```yaml
# DON'T: Embed cross-account invariant in Account
dataType:
  name: Account
  version: v1
  fields:
    balance: decimal
    # Problem: How do we ensure sum(all balances) is correct?
    # Would require locking ALL accounts during ANY write!
```

**RIGHT (View-enforced invariant)**:
```yaml
# DO: Enforce in materialized view
materialization:
  name: GlobalBalanceView
  source: AccountTransactionEvent
  targetState: GlobalBalance
  invariants:
    enforced_by: view
    description: "Sum of all debits equals sum of all credits"

# View validates invariant asynchronously
# Flags violations for investigation
# Does NOT block writes
```

**Performance Comparison**:
- Entity-embedded: O(N) locks for N accounts = slow, doesn't scale
- View-enforced: O(1) write per account, O(N) view update = fast, scales

**Alternative Rejected**: Synchronous cross-entity validation
- Would require distributed locks
- Would serialize all writes
- Would not scale beyond small systems

---

#### Why Storage Role Distinction (v1.3)

**Context**: Control state and domain data have different requirements.

**Decision**: Explicit distinction between control (coordination) and data (domain events).

**Reasoning**:
1. **Different consistency models**: Control needs strong consistency, data tolerates eventual
2. **Clear responsibilities**: Control is system layer, data is domain layer
3. **Operational clarity**: Different backup, replication, monitoring strategies
4. **Failure isolation**: Control plane issues don't corrupt domain data
5. **Recovery procedures**: Can recover control and data independently

**Storage Requirements**:

| Aspect | Control Storage | Data Storage |
|--------|----------------|--------------|
| Consistency | Strong (linearizable) | Eventual acceptable |
| Availability | High (consensus) | Very high (partitioned) |
| Scale | Small (metadata only) | Large (all domain data) |
| Backup | Point-in-time critical | Continuous append log |
| Recovery | Must be exact | Can rebuild from events |

**Example - Authority KV vs Event Stream**:
```yaml
# Control storage (authority)
dataState:
  name: EntityAuthorityState
  type: AuthorityState
  lifecycle: persistent
  storage:
    role: control
    rebuildable: false
  # Backed by: etcd, Consul, or NATS KV with raft

# Data storage (domain events)
dataState:
  name: OrderEvents
  type: OrderEvent
  lifecycle: persistent
  storage:
    role: data
    rebuildable: true
  # Backed by: NATS JetStream, Kafka, or event store
```

**Alternative Rejected**: Single unified storage
- Would conflate coordination with domain data
- Would force one consistency model on both
- Would complicate operations

---

### UI Layer Decisions

#### Why Semantic Composition (v1.4)

**Context**: UI developers think in workspaces, not fragments.

**Decision**: Group views by semantic relationship, not layout.

**Reasoning**:
1. **Intent expression**: Describe WHAT views belong together and WHY
2. **Multiple renderings**: Same composition renders differently on mobile/desktop/voice
3. **Reusability**: Compositions work across experiences
4. **Maintainability**: Relationship types explicit, not implicit in code

**Relationship Types Rationale**:
- `derived`: Computed from primary (read-only)
- `detail`: Child records (1:N relationship)
- `contextual`: Related entity (foreign key)
- `action`: User-triggered operation (modal/drawer)
- `supplementary`: Optional info (settings, help)
- `historical`: Audit trail (read-only, time-based)

**Example - Account Workspace**:
```yaml
presentationComposition:
  name: AccountWorkspace
  primary: AccountSummaryView  # What account?
  related:
    - view: AccountBalanceView
      relationship: derived       # Computed from account
    - view: TransactionListView
      relationship: detail        # Child records
    - view: CustomerProfileView
      relationship: contextual    # Account owner
    - view: TransferFormView
      relationship: action        # Transfer money
    - view: AccountSettingsView
      relationship: supplementary # Settings
    - view: AuditLogView
      relationship: historical    # Activity log
```

**Runtime Flexibility**: Same composition renders as:
- Desktop: Multi-pane layout with all views visible
- Mobile: Tab navigation or accordion
- Voice: Sequential dialogue
- API: GraphQL query with nested relationships

**Alternative Rejected**: Layout-driven composition
- Would couple to specific rendering
- Would prevent multi-platform reuse
- Would obscure intent

---

#### Why Field-Level Permissions (v1.4)

**Context**: Privacy regulations require field-level control.

**Decision**: Declarative field visibility and masking in presentationView.

**Reasoning**:
1. **GDPR/CCPA compliance**: Required for privacy regulations
2. **Role-based access**: Different users see different fields
3. **Context-aware**: Display vs export vs log masking
4. **Auditability**: Who saw what data, when

**Masking Strategies**:
```yaml
# Strategy 1: Full masking
field: ssn
mask:
  unless: subject.role == "admin"
  transform: redact
  # Result: "***-**-****" for non-admins

# Strategy 2: Partial masking
field: credit_card
mask:
  unless: subject.role == "customer_support"
  transform: reveal_last(4)
  # Result: "****-****-****-1234"

# Strategy 3: Context-aware masking
field: account_number
mask:
  context:
    display: reveal_last(4)   # Show on screen
    print: redact             # Hide in PDF
    export: hash              # One-way hash in CSV
    log: redact               # Hide in audit logs
```

**Alternative Rejected**: Application-level masking
- Inconsistent across codebase
- Easy to miss fields
- Hard to audit

---

### Flow Layer Decisions

#### Why Atomic Groups as First-Class

**Context**: Distributed systems default to distributed transactions, which don't scale.

**Decision**: Explicit atomic groups with boundary declarations.

**Reasoning**:
1. **Safety**: Prevents accidental distribution of what must be local
2. **Performance**: Runtime knows it can execute together
3. **Clarity**: Intent explicit in grammar
4. **Correctness**: Eliminates distributed transaction bugs

**Atomic Group Rules**:
1. All steps MUST execute in same boundary
2. All steps MUST execute sequentially
3. Failure of any step fails entire group
4. No partial success permitted

**Example**:
```yaml
sms:
  flows:
    - name: TransferFunds
      steps:
        - atomic_group:
            boundary: account_boundary
            steps:
              - work_unit: debit_source_account
              - work_unit: credit_dest_account
              - work_unit: record_transfer
        # All three execute together or none execute
```

**Why Not Distributed Transactions?**:
1. **Performance**: 2PC is 10-100x slower than local transactions
2. **Availability**: 2PC requires all participants available
3. **Complexity**: Deadlocks, timeouts, recovery logic
4. **Scale**: Doesn't work across regions or long-running processes

**Alternative Pattern - Compensation**:
```yaml
# For cross-boundary operations that can't be atomic
sms:
  flows:
    - name: MultiRegionTransfer
      steps:
        - work_unit: debit_source       # Region A
        - work_unit: credit_dest        # Region B
          on_failure:
            compensate: refund_source   # Undo debit
```

---

### v1.5 Feature Decisions

#### Why Form Intent Binding

**Context**: Forms are the primary user input mechanism.

**Decision**: Explicit grammar for form-to-intent binding.

**Reasoning**:
1. **Completeness**: Bridges gap between UI and backend
2. **Validation clarity**: Separates UI validation (fast) from business validation (slow)
3. **User experience**: Declarative success/error handling
4. **Framework agnostic**: Works with React, Vue, mobile, voice

**Field Source Types**:
- `context`: From session/navigation (e.g., current user, selected account)
- `input`: User-provided (e.g., form fields)
- `computed`: Calculated from other fields (e.g., total = sum(items))

**Validation Timing**:
```yaml
field_mapping:
  - view_field: email
    supplies: user.email
    source: input
    validation:
      constraint: is_valid_email(email)
      message: "Invalid email format"
    # Validated in browser BEFORE submission (fast UX)

  - view_field: username
    supplies: user.username
    source: input
    # Business validation: username uniqueness
    # Validated on server AFTER submission
```

**Alternative Rejected**: Implicit form handling
- Every framework does it differently
- Hard to validate correctness
- Can't generate forms from spec

---

#### Why Asset Semantics

**Context**: Real applications handle files, not just structured data.

**Decision**: Binary content as first-class type with lifecycle.

**Reasoning**:
1. **Completeness**: Applications upload invoices, images, documents
2. **Lifecycle control**: Retention policies, expiry, garbage collection
3. **Transformation**: Thumbnails, compression, format conversion
4. **Access control**: Inherits policy from parent entity
5. **CDN integration**: Efficient delivery

**Asset Lifecycle**:
```yaml
# Permanent: Survives entity deletion
avatar:
  type: asset
  lifecycle: permanent
  # Use case: Profile pictures, legal documents

# Temporary: Short-lived staging
upload_staging:
  type: asset
  lifecycle: temporary
  # Use case: Upload processing, virus scanning

# Expiring: Time-bounded auto-deletion
session_file:
  type: asset
  lifecycle: expiring
  expires_after: 24h
  # Use case: Temporary file sharing
```

**Alternative Rejected**: URLs as strings
- Loses lifecycle semantics
- Can't enforce retention
- No transformation hints
- Access control implicit

---

#### Why Search as Grammar

**Context**: Users expect to search data.

**Decision**: Declarative search configuration at field level.

**Reasoning**:
1. **User expectation**: Modern apps require search
2. **Engine agnostic**: Works with Elasticsearch, Algolia, Typesense
3. **Policy integration**: Search respects authorization
4. **Consistency**: Same semantic meaning across engines

**Search Strategy Selection**:
```yaml
# Full-text: Natural language queries
name:
  search:
    strategy: full_text
  # Use case: Product names, descriptions, articles

# Exact: Precise matching
sku:
  search:
    strategy: exact
  # Use case: SKUs, order IDs, codes

# Fuzzy: Typo-tolerant
email:
  search:
    strategy: fuzzy
    fuzzy:
      max_edits: 2
  # Use case: Email search with typos

# Semantic: Embedding similarity
description:
  search:
    strategy: semantic
  # Use case: "Find similar products"
```

**Alternative Rejected**: Application-level search
- Inconsistent across features
- Hard to maintain
- Doesn't respect policies

---

#### Why Scheduled Triggers

**Context**: Recurring tasks are common.

**Decision**: Declarative scheduling in inputIntent.

**Reasoning**:
1. **Common pattern**: Reports, cleanup, reminders
2. **Timezone handling**: DST, user timezones
3. **Idempotency**: Missed execution handling
4. **Authorization**: Scheduled tasks respect policies

**Missed Execution Strategies**:
```yaml
scheduled:
  on_missed:
    action: skip              # Daily report: Skip yesterday, continue today
    action: execute_immediately  # Critical cleanup: Run ASAP
    action: queue             # Batch job: Queue for later
```

**Alternative Rejected**: External cron + API calls
- Bypasses authorization
- No idempotency guarantees
- Duplicates scheduling logic

---

#### Why Durable Workflows

**Context**: Some processes span days/weeks with human steps.

**Decision**: Long-running workflows with checkpoints and compensation.

**Reasoning**:
1. **Real requirement**: Loan approvals, contract signing, multi-step approvals
2. **Human involvement**: Workflows pause for human decisions
3. **Fault tolerance**: Survives restarts, region failures
4. **Compensation**: Explicit undo logic

**Example - Loan Approval (2 weeks)**:
```yaml
sms:
  flows:
    - name: LoanApproval
      durable:
        enabled: true
        timeout:
          total: 14d        # 2 weeks max
      steps:
        - work_unit: credit_check
        - human_tasks:
            - step: underwriter_review
              timeout: 48h    # 2 days for review
              escalation:
                after: 24h    # Escalate after 1 day
                to: senior_underwriter
        - work_unit: generate_contract
        - human_tasks:
            - step: customer_signature
              timeout: 7d     # 7 days to sign
        - work_unit: disburse_funds
```

**Alternative Rejected**: Short-lived SMS flows only
- Can't handle human tasks
- Can't survive restarts
- No multi-day timeouts

---

#### Why Collaborative Sessions

**Context**: Users expect real-time collaboration (Google Docs style).

**Decision**: Collaborative editing as first-class feature.

**Reasoning**:
1. **User expectation**: Real-time co-editing is table stakes
2. **Conflict resolution**: OT or CRDT strategies
3. **Presence awareness**: See who's editing what
4. **Permission integration**: Respects access control

**Conflict Resolution Strategies**:
```yaml
collaboration:
  conflict_resolution:
    strategy: operational_transform  # Google Docs style
    strategy: crdt                   # Eventual consistency
    strategy: last_write_wins        # Simplest
```

**Alternative Rejected**: Application-level WebSocket
- Inconsistent implementations
- Hard to secure
- Doesn't integrate with policies

---

#### Why Data Governance

**Context**: Regulations require retention and deletion.

**Decision**: Declarative governance in dataState.

**Reasoning**:
1. **Compliance**: GDPR, CCPA, HIPAA, SOX all require data governance
2. **Automation**: Retention enforced by runtime, not application code
3. **Audit trail**: All governance actions logged
4. **Legal hold**: Support for litigation holds

**Retention Example**:
```yaml
# GDPR: Delete after 30 days of account closure
dataState:
  name: CustomerData
  governance:
    retention:
      policy: event_based
      after_event: account_deleted
      grace_period: 30d
      on_expiry: anonymize    # Remove PII, keep aggregates

# SOX: Financial records for 7 years
dataState:
  name: FinancialTransaction
  governance:
    retention:
      policy: time_based
      duration: 7y
      on_expiry: archive      # Move to cold storage
```

**Alternative Rejected**: Application-level deletion
- Easy to miss records
- Hard to audit
- Doesn't scale

---

#### Why Edge Device Support

**Context**: IoT and offline scenarios require edge processing.

**Decision**: Edge device semantics with sync strategies.

**Reasoning**:
1. **Real use cases**: ATMs, POS terminals, manufacturing equipment
2. **Offline operation**: Can't rely on connectivity
3. **Sync strategies**: Eventual consistency with conflict resolution
4. **Resource constraints**: Limited CPU/memory

**Example - ATM**:
```yaml
edgeDevice:
  name: ATMDevice
  capabilities:
    compute: arm64_2ghz
    storage: 32GB
    connectivity: intermittent
  sync:
    strategy: bidirectional
    frequency: real_time
    conflict_resolution: server_wins  # Bank is source of truth
  local_processing:
    enabled: true
    fallback_to_cloud: false  # Must work offline
    models: [card_validation, pin_verification]
```

**Alternative Rejected**: Cloud-only execution
- Doesn't work offline
- High latency for edge use cases
- Misses optimization opportunities

---

## Appendix B: Implementation Patterns

### Pattern: Multi-Region Authority Management

**Scenario**: Global banking application with customers in US and EU.

**Requirements**:
- US customer data stays in US
- EU customer data stays in EU
- Cross-region viewing allowed
- Authority can migrate for follow-the-sun

**Implementation**:
```yaml
dataState:
  name: AccountRecord
  type: Account
  lifecycle: persistent
  mutability:
    scope: entity
    exclusive: true
    authority: AccountAuthority
  governance:
    residency:
      primary_region: |
        if customer.country in ["US", "CA", "MX"] then "us-east"
        else if customer.country in EU_COUNTRIES then "eu-west"
        else "default-region"
      transfer:
        allowed: false  # Data cannot leave primary region
      
authority:
  name: AccountAuthority
  scope: entity
  resolution: |
    # Authority follows primary region by default
    residency.primary_region
  migration:
    allowed: true
    strategy: pause-and-cutover
    triggers:
      - type: manual  # Only manual migration for compliance
    constraints:
      - type: governance
        rule: residency.transfer_allowed == true
```

**Runtime Behavior**:
1. US customer account created → authority in us-east
2. Customer moves to EU → residency unchanged (data stays in US)
3. Authority CANNOT migrate to EU (governance constraint)
4. EU staff can VIEW account (cross-region read)
5. Writes go to us-east (authority region)

---

### Pattern: Eventual Consistency with Invariant Enforcement

**Scenario**: E-commerce inventory across warehouses.

**Requirements**:
- Multiple warehouses can sell concurrently
- Total sold cannot exceed total inventory
- No distributed transactions
- Flag violations for investigation

**Implementation**:
```yaml
# Each warehouse has independent inventory
dataState:
  name: WarehouseInventory
  type: Inventory
  lifecycle: persistent
  mutability:
    scope: entity  # Each warehouse independent

# Derived view enforces global invariant
materialization:
  name: GlobalInventoryView
  source: [WarehouseSaleEvent, WarehouseRestockEvent]
  targetState: GlobalInventory
  invariants:
    enforced_by: view
    description: "Total sold <= total inventory"
  retrieval:
    mode: singleton  # One global view
  
# Violation alert flow
sms:
  flows:
    - name: InventoryViolationAlert
      triggeredBy:
        dataState: GlobalInventoryView
        condition: total_sold > total_inventory
      steps:
        - work_unit: notify_operations
        - work_unit: pause_sales
        - work_unit: trigger_investigation
```

**Runtime Behavior**:
1. Warehouse A sells 100 units (immediately succeeds)
2. Warehouse B sells 50 units (immediately succeeds)
3. View updates: total_sold = 150
4. View detects: 150 > 120 inventory
5. Alert triggered
6. Operations investigates (oversell happened)
7. Compensation flow initiated

**Why This Works**:
- Warehouses don't block each other (independent writes)
- Violations detected within seconds (view update latency)
- Can't prevent 100% of violations (eventual consistency)
- **Trade-off accepted** for high availability and low latency

---

### Pattern: Progressive Form Validation

**Scenario**: Complex multi-step user onboarding.

**Requirements**:
- Save progress without full validation
- Client-side validation for UX
- Server-side validation for security
- Clear error messages

**Implementation**:
```yaml
# Step 1: Personal Info (partial validation)
presentationView:
  name: PersonalInfoForm
  triggers:
    intent: SavePersonalInfo
    binding: form
    field_mapping:
      - view_field: email
        supplies: user.email
        source: input
        validation:
          constraint: is_valid_email(email)
          message: "Invalid email format"
        # Client-side validation (fast UX)
      
      - view_field: phone
        supplies: user.phone
        source: input
        validation:
          constraint: matches_pattern(phone, "^\\+?[0-9]{10,15}$")
          message: "Invalid phone number"
    
    confirmation:
      required: false  # No confirmation for save draft
    
    on_success:
      action: navigate_to
      target: AddressFormView
      notification:
        type: toast
        message: "Progress saved"

inputIntent:
  name: SavePersonalInfo
  proposes:
    dataState: UserRecord
  supplies:
    required: [email, phone]
  constraints:
    guarantees: [email_format_valid, phone_format_valid]
    defers: [email_unique, phone_verified]
    # Defer expensive validations to final submission

# Step 2: Address Info
# ... similar pattern ...

# Final Step: Submit Application (full validation)
inputIntent:
  name: SubmitApplication
  proposes:
    dataState: UserRecord
  supplies:
    required: [email, phone, address, identity_verification]
  constraints:
    guarantees: [email_unique, phone_verified, address_valid, identity_confirmed]
    # Now enforce ALL constraints
```

**Validation Timing**:
- **Client-side** (instant): Format validation
- **Server-side save** (fast): Basic validation
- **Server-side submit** (slow): Complete validation including external checks

---

### Pattern: Multi-Channel Notification with Fallback

**Scenario**: Critical payment alerts must reach users.

**Requirements**:
- Try push notification first (fastest)
- Fall back to email if push fails
- Respect user preferences
- Respect quiet hours

**Implementation**:
```yaml
notificationChannel:
  name: PaymentAlert
  version: v1
  channels:
    - type: push
      enabled: true
      config:
        push:
          platform: fcm
          priority: high
    
    - type: email
      enabled: true
      config:
        email:
          from: "alerts@bank.com"
          template: payment_alert
    
    - type: sms
      enabled: true
      config:
        sms:
          sender: "BANK"
          template: payment_alert_sms
  
  preferences:
    user_controllable: true
    defaults:
      push: enabled
      email: enabled
      sms: disabled
    quiet_hours:
      enabled: true
      start: "22:00"
      end: "08:00"
      timezone: user.timezone
  
  delivery:
    retry:
      enabled: true
      max_attempts: 3
      backoff: exponential
    
    fallback:
      enabled: true
      order: [push, email, sms]  # Try in this order
    
    batching:
      enabled: false  # Critical alerts sent immediately
```

**Runtime Behavior**:
1. Payment processed at 14:00 (not quiet hours)
2. Try push → fails (app not installed)
3. Try email → succeeds
4. User receives email within 1 second
5. Audit log records: "push failed, email succeeded"

**Alternative Scenario**:
1. Payment processed at 23:00 (quiet hours)
2. Check user preferences
3. User has quiet_hours enabled
4. Notification queued until 08:00
5. Delivered at 08:01 next morning

---

### Pattern: Durable Workflow with Compensation

**Scenario**: Multi-step payment processing with external gateway.

**Requirements**:
- Reserve funds
- Call payment gateway
- Update order status
- If gateway fails, refund reservation
- If order update fails, reverse payment

**Implementation**:
```yaml
sms:
  flows:
    - name: ProcessPayment
      durable:
        enabled: true
        checkpoint:
          frequency: per_step
        compensation:
          enabled: true
          on_failure: compensate
        timeout:
          total: 5m
      
      steps:
        # Step 1: Reserve funds (local)
        - work_unit: reserve_funds
          input: { amount: "{{order.total}}", account: "{{order.account_id}}" }
          output: reservation_id
          compensate: release_reservation
        
        # Step 2: Charge payment gateway (external)
        - work_unit: charge_gateway
          input: { amount: "{{order.total}}", token: "{{payment.token}}" }
          output: charge_id
          timeout: 30s
          retry:
            max_attempts: 3
            backoff: exponential
          compensate: refund_gateway
        
        # Step 3: Update order status (local)
        - work_unit: mark_order_paid
          input: { order_id: "{{order.id}}", charge_id: "{{charge_id}}" }
          compensate: mark_order_pending
```

**Failure Scenarios**:

**Scenario 1: Gateway fails**
1. reserve_funds succeeds (checkpoint saved)
2. charge_gateway fails after 3 retries
3. Compensation: release_reservation executes
4. Funds returned to customer
5. Order remains unpaid

**Scenario 2: Order update fails**
1. reserve_funds succeeds
2. charge_gateway succeeds (checkpoint saved)
3. mark_order_paid fails
4. Compensation: refund_gateway executes
5. Then: release_reservation executes
6. Gateway refunded, funds returned, order unpaid

**Scenario 3: System crash during step 2**
1. reserve_funds succeeds (checkpoint saved)
2. charge_gateway in progress
3. **System crashes**
4. **System restarts**
5. Workflow resumes from checkpoint
6. charge_gateway retried (idempotent)
7. Continues to step 3

---

### Pattern: Real-Time Collaborative Editing

**Scenario**: Multiple users editing shared document.

**Requirements**:
- See other users' cursors
- Conflict-free editing
- Undo/redo per user
- Permission-based editing

**Implementation**:
```yaml
experience:
  name: DocumentEditor
  collaboration:
    enabled: true
    session:
      type: document
      scope: document.id
      max_participants: 50
    
    awareness:
      show_active_users: true
      show_cursors: true
      show_selections: true
      timeout: 30s  # Inactive after 30s of no activity
    
    permissions:
      mode: policy_based
      policy_set: DocumentEditPolicies
    
    conflict_resolution:
      strategy: operational_transform  # Google Docs style
      merge_policy: DocumentMergePolicy

policy:
  name: DocumentEditPolicy
  version: v1
  appliesTo:
    type: dataState
    name: DocumentContent
  effect: allow
  when: |
    subject.id == document.owner_id OR
    subject.id in document.collaborators OR
    subject.role == "admin"
```

**Runtime Behavior**:
1. User A opens document
2. User B opens same document
3. Both see each other in "Active Users"
4. User A types "Hello"
5. User B sees "Hello" appear in real-time
6. User B types "World" at same position
7. Operational Transform resolves conflict
8. Both see "Hello World"
9. User A goes idle for 31 seconds
10. User A shown as "Away"

---


## Appendix C: Complete Examples

### Example 1: Banking Account Management

This comprehensive example demonstrates entity-scoped authority, multi-region deployment, field permissions, and UI composition.

#### Data Model

```yaml
# Account entity with entity-scoped authority
dataType:
  name: Account
  version: v1
  fields:
    id: uuid
    account_number: string
    owner_id: uuid
    balance: decimal
    currency: string
    status: enum[active, frozen, closed]
    created_at: timestamp
    updated_at: timestamp

dataState:
  name: AccountRecord
  type: Account
  lifecycle: persistent
  constraintPolicy:
    enforcement: strict
  mutability:
    scope: entity
    exclusive: true
    authority: AccountAuthority
  governance:
    classification: pii
    retention:
      policy: event_based
      after_event: account_closed
      grace_period: 90d
      on_expiry: anonymize
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 7y

authority:
  name: AccountAuthority
  scope: entity
  resolution: |
    if owner.country in ["US", "CA", "MX"] then "us-east"
    else if owner.country in EU_COUNTRIES then "eu-west"
    else "ap-south"
  migration:
    allowed: true
    strategy: pause-and-cutover
    triggers:
      - type: latency
        condition: p99_latency > 200ms
      - type: manual

# Transaction events (append-only)
dataType:
  name: Transaction
  version: v1
  fields:
    id: uuid
    account_id: uuid
    type: enum[debit, credit, transfer]
    amount: decimal
    currency: string
    description: string
    timestamp: timestamp
    balance_after: decimal

dataState:
  name: TransactionEvent
  type: Transaction
  lifecycle: persistent
  storage:
    role: data
    rebuildable: true

# Balance view (materialized)
materialization:
  name: AccountBalanceView
  source: TransactionEvent
  targetState: AccountBalance
  freshness:
    maxStaleness: 1s
  authority_agnostic: true
  epoch_tolerant: true
  retrieval:
    mode: by_entity
    entity_key: account_id
    on_unavailable:
      strategy: use_cached
      cache_ttl: 5m
  invariants:
    enforced_by: view
    description: "Balance equals sum of credits minus sum of debits"
```

#### UI Layer

```yaml
# Account summary view
presentationView:
  name: AccountSummaryView
  version: v1
  consumes:
    dataState: AccountRecord
    version: v1
  fieldPermissions:
    - field: account_number
      visible: true
      mask:
        unless: subject.role in ["owner", "admin"]
        transform: reveal_last(4)
        placeholder: "****-****-####"
    - field: balance
      visible:
        policy: ViewBalancePolicy
  interactionModel:
    badges:
      - if: account.status == "frozen"
        label: "FROZEN"
        severity: error
    actions:
      - if: account.status == "active"
        action: transfer
        label: "Transfer Funds"
        intent: InitiateTransfer

# Transaction list view
presentationView:
  name: TransactionListView
  version: v1
  consumes:
    dataState: TransactionEvent
    version: v1
  interactionModel:
    filters:
      - field: type
        type: enum
      - field: timestamp
        type: date_range
    sorts:
      - field: timestamp
        default: desc

# Account workspace composition
presentationComposition:
  name: AccountWorkspace
  version: v1
  description: "Complete account management workspace"
  primary: AccountSummaryView
  navigation:
    label: "Account {{account.account_number}}"
    purpose: accounts
    param: account_id
    policy_set: AccountAccessPolicies
    indicator:
      when: pending_transactions > 0
      severity: info
  related:
    - view: AccountBalanceView
      relationship: derived
      cardinality: one
      required: true
    - view: TransactionListView
      relationship: detail
      cardinality: many
      required: false
      data_scope: { account_id: "{{account.id}}" }
      navigation:
        label: "Transactions"
        scope: within_composition
    - view: TransferFormView
      relationship: action
      cardinality: one
      trigger: initiate_transfer
      navigation:
        label: "Transfer Funds"
        scope: on_action
  fetch:
    order: primary_first
    related:
      strategy: parallel
      max_parallel: 3
    cache:
      strategy: stale_while_revalidate
      ttl: 30s

# Customer banking experience
experience:
  name: CustomerBanking
  version: v1
  description: "Customer-facing banking portal"
  entry_point: CustomerDashboard
  includes:
    - CustomerDashboard
    - AccountWorkspace
    - TransactionHistory
    - TransferCenter
  policy_set: CustomerPolicies
  session:
    required: true
    subject_binding:
      mode: authenticated
    lifetime:
      mode: sliding
      idle_timeout: 15m
      absolute_timeout: 8h
      refresh: auto
    multi_device:
      enabled: true
      sync_strategy: real_time
      conflict_resolution: last_write_wins
```

#### Input & Flows

```yaml
# Transfer intent
inputIntent:
  name: InitiateTransfer
  version: v1
  proposes:
    dataState: TransactionEvent
    version: v1
  supplies:
    required: [from_account_id, to_account_id, amount]
    optional: [description]
  constraints:
    guarantees: [amount_positive, accounts_different]
    defers: [sufficient_balance, account_active]
  delivery:
    guarantee: at_least_once
    acknowledgment: required
    timeout: 30s
    retry:
      allowed: true
      max_attempts: 3
      backoff: exponential
  response:
    on_success:
      includes: [transaction_id, new_balance, timestamp]
    on_error:
      includes: [error_code, message]
      field_errors:
        enabled: true
        format: map

# Transfer form
presentationView:
  name: TransferFormView
  version: v1
  consumes:
    dataState: TransferFormData
    version: v1
  triggers:
    intent: InitiateTransfer
    binding: form
    field_mapping:
      - view_field: from_account
        supplies: from_account_id
        source: context
        required: true
      - view_field: to_account
        supplies: to_account_id
        source: input
        required: true
        validation:
          constraint: to_account != from_account
          message: "Cannot transfer to same account"
      - view_field: amount
        supplies: amount
        source: input
        required: true
        validation:
          constraint: amount > 0
          message: "Amount must be positive"
      - view_field: description
        supplies: description
        source: input
        required: false
    confirmation:
      required: true
      message: "Transfer {{amount}} {{currency}} to account {{to_account}}?"
    on_success:
      action: navigate_back
      notification:
        type: toast
        message: "Transfer submitted successfully"
    on_error:
      action: display_inline
      show_field_errors: true

# Transfer processing flow
sms:
  flows:
    - name: ProcessTransfer
      triggeredBy:
        inputIntent: InitiateTransfer
      steps:
        - validate:
            constraints: [amount_positive, accounts_exist]
        - atomic_group:
            boundary: transfer_boundary
            steps:
              - work_unit: debit_source_account
                input: { account_id: "{{from_account_id}}", amount: "{{amount}}" }
              - work_unit: credit_dest_account
                input: { account_id: "{{to_account_id}}", amount: "{{amount}}" }
              - work_unit: record_transfer
                input: { from: "{{from_account_id}}", to: "{{to_account_id}}", amount: "{{amount}}" }
        - persist:
            state: TransactionEvent
        - work_unit: send_notification
          input: { account_id: "{{from_account_id}}", type: "transfer_completed" }
      authorization:
        policySet: TransferPolicies
```

#### Policies

```yaml
# View balance policy
policy:
  name: ViewBalancePolicy
  version: v1
  appliesTo:
    type: dataState
    name: AccountRecord
  effect: allow
  when: |
    subject.id == account.owner_id OR
    subject.role == "admin" OR
    (subject.role == "support" AND subject.has_approval("view_balance"))

# Transfer policy
policy:
  name: TransferAuthPolicy
  version: v1
  appliesTo:
    type: inputIntent
    name: InitiateTransfer
  effect: allow
  when: |
    subject.id == from_account.owner_id AND
    from_account.status == "active" AND
    to_account.status == "active" AND
    amount <= from_account.balance
```

---

### Example 2: E-Commerce Product Catalog with Search

This example demonstrates search semantics, asset management, and materialized views.

#### Data Model

```yaml
dataType:
  name: Product
  version: v1
  fields:
    id: uuid
    sku: string
    name:
      type: string
      search:
        indexed: true
        strategy: full_text
        full_text:
          analyzer: standard
          stemming: enabled
        weight: 2.0
        suggest:
          enabled: true
          type: completion
        highlight:
          enabled: true
          fragment_size: 150
    description:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 1.0
    category: string
    brand: string
    price: decimal
    currency: string
    in_stock: boolean
    image:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
          mime_types: ["image/jpeg", "image/png", "image/webp"]
        lifecycle:
          type: permanent
        access:
          read: public
        variants:
          - name: thumbnail
            transform: thumbnail
            params: { width: 150, height: 150 }
          - name: large
            transform: resize
            params: { width: 1200, height: 1200 }
    tags:
      type: array<string>
      search:
        indexed: true
        strategy: exact
    rating: decimal
    review_count: integer

dataState:
  name: ProductCatalog
  type: Product
  lifecycle: persistent
  governance:
    classification: public

# Search intent
searchIntent:
  name: ProductSearch
  version: v1
  searches: ProductCatalog
  query:
    fields: [name, description, tags, brand]
    default_operator: and
  filters:
    required: []
    optional: [category, brand, price_range, in_stock]
  facets:
    include: [category, brand, price_range, in_stock, rating]
  pagination:
    default_size: 24
    max_size: 100
    cursor_based: false
  sort:
    default: _score
    allowed: [price, rating, name, created_at]
  result:
    includes: [id, sku, name, price, image, rating, in_stock]
    highlights: [name, description]
  authorization:
    policy_set: PublicSearchPolicies
```

#### UI Layer

```yaml
presentationView:
  name: ProductSearchView
  version: v1
  consumes:
    dataState: ProductCatalog
    version: v1
  interactionModel:
    filters:
      - field: category
        type: enum
      - field: brand
        type: enum
      - field: price
        type: range
      - field: in_stock
        type: boolean
    sorts:
      - field: relevance
        default: desc
      - field: price
        default: asc
      - field: rating
        default: desc
    actions:
      - action: view_product
        label: "View Details"
        intent: NavigateToProduct

presentationView:
  name: ProductDetailView
  version: v1
  consumes:
    dataState: ProductCatalog
    version: v1
  interactionModel:
    actions:
      - if: product.in_stock
        action: add_to_cart
        label: "Add to Cart"
        intent: AddToCart
      - action: view_reviews
        label: "Read Reviews"
        intent: NavigateToReviews

presentationComposition:
  name: ProductPageWorkspace
  version: v1
  primary: ProductDetailView
  navigation:
    label: "{{product.name}}"
    purpose: products
    param: product_id
  related:
    - view: ProductImagesView
      relationship: supplementary
      cardinality: many
    - view: ProductReviewsView
      relationship: detail
      cardinality: many
      navigation:
        label: "Reviews ({{review_count}})"
        scope: within_composition
    - view: RecommendedProductsView
      relationship: derived
      cardinality: many
      navigation:
        label: "Recommended"
        scope: within_composition
```

---

### Example 3: Scheduled Report Generation

This example demonstrates scheduled triggers and notification channels.

```yaml
# Report data type
dataType:
  name: SalesReport
  version: v1
  fields:
    id: uuid
    report_date: timestamp
    period: enum[daily, weekly, monthly]
    total_sales: decimal
    total_orders: integer
    top_products: array<ProductSummary>
    pdf:
      type: asset
      asset:
        category: document
        constraints:
          mime_types: ["application/pdf"]
        lifecycle:
          type: expiring
          expires_after: 90d
        access:
          read: policy
          policy: ReportAccessPolicy

dataState:
  name: SalesReportRecord
  type: SalesReport
  lifecycle: persistent
  governance:
    classification: confidential
    retention:
      policy: time_based
      duration: 2y
      on_expiry: archive

# Scheduled report generation
inputIntent:
  name: GenerateDailySalesReport
  version: v1
  proposes:
    dataState: SalesReportRecord
    version: v1
  supplies:
    required: [report_date, period]
  scheduled:
    enabled: true
    trigger:
      type: cron
      cron: "0 2 * * *"  # Daily at 2 AM
    timezone:
      mode: fixed
      fixed: "America/New_York"
    idempotency:
      key: ["daily_sales", "{{report_date}}"]
      scope: global
    on_missed:
      action: execute_immediately
  delivery:
    guarantee: exactly_once
    acknowledgment: required
    timeout: 5m

# Report generation flow
sms:
  flows:
    - name: GenerateSalesReport
      triggeredBy:
        inputIntent: GenerateDailySalesReport
      steps:
        - work_unit: aggregate_sales_data
          input: { date: "{{report_date}}" }
          output: sales_summary
        - work_unit: generate_pdf_report
          input: { data: "{{sales_summary}}" }
          output: pdf_url
        - work_unit: save_report
          input: { summary: "{{sales_summary}}", pdf: "{{pdf_url}}" }
        - work_unit: send_report_notification
          input: { report_id: "{{result.id}}" }
      durable:
        enabled: true
        timeout:
          total: 10m

# Notification channel
notificationChannel:
  name: DailySalesReportNotification
  version: v1
  channels:
    - type: email
      enabled: true
      config:
        email:
          from: "reports@company.com"
          template: daily_sales_report
    - type: webhook
      enabled: true
      config:
        webhook:
          url: "https://slack.com/api/webhooks/..."
          method: POST
  preferences:
    user_controllable: false
    defaults:
      email: enabled
      webhook: enabled
  delivery:
    retry:
      enabled: true
      max_attempts: 5
      backoff: exponential
    batching:
      enabled: false
```

---

### Example 4: Multi-Region Document Collaboration

This example demonstrates collaborative sessions, structured content, and multi-region authority.

```yaml
# Document data type
dataType:
  name: Document
  version: v1
  fields:
    id: uuid
    title: string
    owner_id: uuid
    collaborators: array<uuid>
    content:
      type: structured_content
      structured_content:
        blocks:
          allowed: [paragraph, heading, list, blockquote, code, image, table]
        marks:
          allowed: [bold, italic, underline, link, code, highlight]
        constraints:
          max_length: 100000
        sanitization:
          strip_scripts: true
    status: enum[draft, published, archived]
    created_at: timestamp
    updated_at: timestamp
    version_number: integer

dataState:
  name: DocumentRecord
  type: Document
  lifecycle: persistent
  mutability:
    scope: entity
    exclusive: true
    authority: DocumentAuthority
  governance:
    classification: internal
    retention:
      policy: event_based
      after_event: document_archived
      grace_period: 180d
      on_expiry: delete

authority:
  name: DocumentAuthority
  scope: entity
  resolution: |
    nearest_region(owner.location)
  migration:
    allowed: true
    strategy: pause-and-cutover
    triggers:
      - type: manual

# Document editor experience
experience:
  name: DocumentEditor
  version: v1
  entry_point: DocumentEditWorkspace
  includes:
    - DocumentEditWorkspace
    - DocumentVersionHistory
  policy_set: DocumentEditPolicies
  session:
    required: true
    subject_binding:
      mode: authenticated
    multi_device:
      enabled: true
      sync_strategy: real_time
      conflict_resolution: last_write_wins
    offline:
      enabled: true
      cache_duration: 24h
      sync_on_reconnect: auto
  collaboration:
    enabled: true
    session:
      type: document
      scope: document.id
      max_participants: 50
    awareness:
      show_active_users: true
      show_cursors: true
      show_selections: true
      timeout: 30s
    permissions:
      mode: policy_based
      policy_set: DocumentCollaborationPolicies
    conflict_resolution:
      strategy: operational_transform

# Collaboration policy
policy:
  name: DocumentCollaborationPolicy
  version: v1
  appliesTo:
    type: dataState
    name: DocumentRecord
  effect: allow
  when: |
    subject.id == document.owner_id OR
    subject.id in document.collaborators OR
    subject.role == "admin"
```

---

### Example 5: IoT Edge Device with Offline Sync

This example demonstrates edge device semantics and spatial types.

```yaml
# IoT device data
dataType:
  name: IoTDevice
  version: v1
  fields:
    id: uuid
    device_id: string
    type: enum[sensor, actuator, gateway]
    location:
      type: spatial
      spatial:
        geometry: point
        coordinate_system: wgs84
        precision: 0.000001
        search:
          indexed: true
          strategy: radius
    status: enum[online, offline, maintenance]
    last_heartbeat: timestamp
    firmware_version: string

# Sensor reading data
dataType:
  name: SensorReading
  version: v1
  fields:
    id: uuid
    device_id: uuid
    timestamp: timestamp
    temperature: decimal
    humidity: decimal
    pressure: decimal
    location:
      type: spatial
      spatial:
        geometry: point
        coordinate_system: wgs84

dataState:
  name: SensorReadingEvent
  type: SensorReading
  lifecycle: persistent
  storage:
    role: data
    rebuildable: true

# Edge device configuration
edgeDevice:
  name: SensorGateway
  version: v1
  capabilities:
    compute: arm64_2ghz
    storage: 64GB
    connectivity: intermittent
  sync:
    strategy: bidirectional
    frequency: periodic
    conflict_resolution: server_wins
  local_processing:
    enabled: true
    fallback_to_cloud: false
    models: [reading_validation, anomaly_detection]

# Materialized view for aggregated readings
materialization:
  name: DeviceMetricsView
  source: SensorReadingEvent
  targetState: DeviceMetrics
  temporal:
    window:
      type: tumbling
      size: 5m
    aggregation:
      - field: temperature
        function: avg
        as: avg_temperature
      - field: humidity
        function: avg
        as: avg_humidity
      - field: pressure
        function: avg
        as: avg_pressure
    timestamp_field: timestamp
    emit:
      trigger: on_window_close

# Spatial query for nearby devices
graphQuery:
  name: FindNearbyDevices
  version: v1
  start_from: IoTDevice
  traversal:
    relationships: [DeviceProximity]
    direction: both
    max_depth: 2
  filter:
    node: device.status == "online"
  return:
    nodes: [id, device_id, type, location, status]
    aggregations:
      - function: count
        field: id
        as: nearby_count
```

---

## Appendix D: Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: CAS Token Mismatch

**Symptom**: Writes failing with `ErrEpochMismatch` or `ErrVersionMismatch`.

**Cause**: CAS token from client is stale due to authority migration or concurrent writes.

**Diagnosis**:
```yaml
# Check authority state
GET /authority/{entity_id}
Response:
  entity_id: "abc-123"
  authority_region: "us-east"
  authority_epoch: 15        # Epoch increased
  entity_version: 42

# Client CAS token
cas_token:
  entity_id: "abc-123"
  authority_epoch: 14        # Stale epoch!
  entity_version: 41
```

**Solution**:
1. Fetch fresh CAS token from authority region
2. Retry write with new token
3. If repeated failures, check authority migration in progress

**Prevention**:
- Implement automatic token refresh on mismatch
- Add exponential backoff for retries
- Monitor authority migration frequency

---

#### Issue 2: View Staleness

**Symptom**: Materialized views showing outdated data.

**Cause**: View update latency exceeding `maxStaleness` threshold.

**Diagnosis**:
```yaml
materialization:
  name: AccountBalanceView
  freshness:
    maxStaleness: 5s    # Target: 5 seconds

# Actual latency: 15 seconds
```

**Solution Options**:

**Option 1: Adjust Staleness Tolerance**
```yaml
freshness:
  maxStaleness: 30s  # Increase tolerance
```

**Option 2: Scale View Workers**
```
# Add more view materialization workers
kubectl scale deployment view-materializer --replicas=5
```

**Option 3: Optimize View Query**
```yaml
# Reduce aggregation complexity
materialization:
  retrieval:
    mode: by_entity  # Instead of global aggregation
```

**Prevention**:
- Monitor view update latency
- Set alerts on staleness violations
- Load test view materialization

---

#### Issue 3: Atomic Group Crosses Boundaries

**Symptom**: Parser rejects flow with error "Atomic group spans multiple boundaries".

**Cause**: Work units in atomic group execute in different boundaries.

**Invalid Example**:
```yaml
atomic_group:
  boundary: order_boundary
  steps:
    - work_unit: create_order        # boundary: order_boundary
    - work_unit: charge_payment      # boundary: payment_boundary  ❌
```

**Solution**: Split into separate steps with compensation:
```yaml
steps:
  - work_unit: create_order
  - work_unit: charge_payment
    on_failure:
      compensate: cancel_order
```

**Prevention**:
- Design boundaries around transaction requirements
- Use compensation for cross-boundary operations
- Validate flows with linter before deployment

---

#### Issue 4: Policy Divergence in Shadow Mode

**Symptom**: High divergence rate between shadow and enforced policies.

**Diagnosis**:
```yaml
policyArtifact:
  lifecycle:
    state: shadow
  metrics:
    divergence_rate: 0.35  # 35% divergence! ⚠️
```

**Cause Analysis**:
- **If false positives high**: Policy too restrictive
- **If false negatives high**: Policy too permissive
- **If divergence random**: Policy logic error

**Solution**:
1. Analyze divergence patterns in audit logs
2. Identify specific cases causing divergence
3. Refine policy logic
4. Re-test in shadow mode

**Example Fix**:
```yaml
# Original (too restrictive)
when: subject.role == "admin"

# Fixed (correct logic)
when: subject.role in ["admin", "owner"] OR subject.has_permission("manage_account")
```

---

#### Issue 5: Session Sync Conflicts

**Symptom**: Multi-device sync causing data loss or conflicts.

**Cause**: Conflict resolution strategy not appropriate for use case.

**Diagnosis**:
```yaml
session:
  multi_device:
    conflict_resolution: last_write_wins  # May lose data
```

**Solution Options**:

**Option 1: Device Priority**
```yaml
conflict_resolution: device_priority
# Desktop changes win over mobile
```

**Option 2: Manual Resolution**
```yaml
conflict_resolution: manual
# User chooses which version to keep
```

**Option 3: Field-Level Merge**
```yaml
conflict_resolution: crdt
# Automatic conflict-free merge
```

**Prevention**:
- Choose strategy based on data criticality
- Test multi-device scenarios
- Monitor conflict rates

---

#### Issue 6: Search Results Not Updating

**Symptom**: Search returns stale results after data changes.

**Cause**: Search index not updating in real-time.

**Diagnosis**:
```
# Check search index lag
GET /search/status
Response:
  last_indexed_at: "2024-01-15T10:00:00Z"
  current_time: "2024-01-15T10:05:00Z"
  lag: 300s  # 5 minute lag
```

**Solution**:

**Immediate**: Force reindex
```
POST /search/reindex
{ "entity_type": "Product" }
```

**Long-term**: Optimize indexing
```yaml
dataType:
  fields:
    name:
      search:
        indexed: true
        # Add indexing strategy
        update: real_time | batch | periodic
        batch_size: 1000
        batch_window: 30s
```

---

#### Issue 7: Durable Workflow Stuck

**Symptom**: Workflow not progressing, stuck at human task.

**Diagnosis**:
```yaml
workflow_status:
  workflow_id: "wf-123"
  current_step: "underwriter_review"
  status: waiting_for_human
  timeout: 48h
  elapsed: 72h  # Past timeout! ⚠️
```

**Cause**: Human task timeout exceeded without escalation.

**Solution**:

**Immediate**: Manual intervention
```
POST /workflows/wf-123/complete_step
{
  "step": "underwriter_review",
  "result": "approved",
  "reason": "timeout_recovery"
}
```

**Long-term**: Add escalation
```yaml
human_tasks:
  - step: underwriter_review
    timeout: 48h
    escalation:
      after: 24h
      to: senior_underwriter
      action: reassign
```

---

#### Issue 8: Asset Upload Failing

**Symptom**: Asset uploads timing out or failing validation.

**Diagnosis**:
```yaml
asset:
  constraints:
    max_size: 5MB
  
# Upload attempt
file_size: 12MB  # Exceeds limit ❌
```

**Solutions**:

**Solution 1: Increase limit (if appropriate)**
```yaml
constraints:
  max_size: 20MB
```

**Solution 2: Client-side compression**
```javascript
// Compress before upload
const compressed = await compressImage(file, { quality: 0.8 });
```

**Solution 3: Progressive upload**
```yaml
asset:
  upload:
    strategy: chunked
    chunk_size: 1MB
    max_chunks: 20
```

---

#### Issue 9: Notification Not Delivered

**Symptom**: Users not receiving notifications.

**Diagnosis**:
```yaml
notification_log:
  channel: push
  status: failed
  reason: "Device token invalid"
  
notification_log:
  channel: email
  status: delivered
```

**Cause**: Push notification device token expired.

**Solution**: Implement fallback
```yaml
notificationChannel:
  delivery:
    fallback:
      enabled: true
      order: [push, email, sms]
    retry:
      enabled: true
      max_attempts: 3
```

**Prevention**:
- Refresh device tokens periodically
- Monitor notification success rates
- Test all channels regularly

---

#### Issue 10: Authority Migration Taking Too Long

**Symptom**: Write pause during authority migration exceeds target.

**Diagnosis**:
```
Authority migration for entity abc-123:
  Target pause: 100ms
  Actual pause: 2500ms  ⚠️
  
Bottleneck: Network latency between regions
```

**Solutions**:

**Solution 1: Optimize network path**
```
# Use dedicated private links between regions
AWS PrivateLink / Azure Private Link / GCP Private Service Connect
```

**Solution 2: Reduce migration frequency**
```yaml
authority:
  migration:
    triggers:
      - type: latency
        condition: p99_latency > 500ms  # Increase threshold
        cooldown: 1h  # Add cooldown period
```

**Solution 3: Pre-warm target region**
```yaml
migration:
  strategy: pause-and-cutover
  pre_migration:
    warm_cache: true
    sync_hot_data: true
```

---

## Appendix E: Performance Optimization

### Optimization Patterns

#### Pattern 1: View Caching Strategy

**Goal**: Reduce view computation latency.

**Strategy**:
```yaml
materialization:
  name: CustomerDashboard
  freshness:
    maxStaleness: 5m
  retrieval:
    mode: by_entity
    entity_key: customer_id
    on_unavailable:
      strategy: stale_while_revalidate
      cache_ttl: 1h  # Serve stale while updating
```

**Results**:
- P50 latency: 5ms (cache hit)
- P99 latency: 250ms (cache miss)
- Cache hit rate: 95%

---

#### Pattern 2: Composition Prefetching

**Goal**: Reduce composition load time.

**Strategy**:
```yaml
presentationComposition:
  name: OrderDetailsWorkspace
  fetch:
    order: all_parallel  # Load all views simultaneously
    prefetch:
      enabled: true
      trigger: on_hover  # Prefetch when user hovers over link
    cache:
      strategy: stale_while_revalidate
      ttl: 30s
```

**Results**:
- Without prefetch: 800ms load time
- With prefetch: 150ms load time (85% improvement)

---

#### Pattern 3: Batch Notification Delivery

**Goal**: Reduce notification overhead.

**Strategy**:
```yaml
notificationChannel:
  name: ActivityDigest
  delivery:
    batching:
      enabled: true
      window: 5m  # Collect notifications for 5 minutes
      max_batch_size: 50
```

**Results**:
- Individual sends: 1000 API calls/hour
- Batched sends: 20 API calls/hour (98% reduction)

---

#### Pattern 4: Search Result Caching

**Goal**: Improve search performance.

**Strategy**:
```yaml
searchIntent:
  name: ProductSearch
  cache:
    enabled: true
    ttl: 5m
    key_fields: [query, filters, sort]  # Cache key
    invalidate_on: product_update
```

**Results**:
- Cold search: 250ms
- Cached search: 15ms (94% faster)
- Cache hit rate: 70%

---

#### Pattern 5: Incremental View Updates

**Goal**: Reduce view rebuild time.

**Strategy**:
```yaml
materialization:
  name: SalesMetrics
  evolution:
    strategy: incremental  # Not rebuild
    blocking: false
  temporal:
    window:
      type: sliding
      size: 1h
      slide: 1m  # Update every minute
```

**Results**:
- Full rebuild: 10 minutes
- Incremental update: 5 seconds (120x faster)

---

## Appendix F: Security Best Practices

### Security Patterns

#### Pattern 1: Field-Level Encryption

**Goal**: Protect sensitive data at rest.

**Implementation**:
```yaml
dataState:
  name: CustomerPII
  governance:
    encryption:
      at_rest: required
      in_transit: required
      key_rotation: 90d
    classification: pii

# Field-level masking
presentationView:
  fieldPermissions:
    - field: ssn
      mask:
        unless: subject.role == "compliance_officer"
        transform: hash
    - field: credit_card
      mask:
        unless: subject.role == "payment_processor"
        transform: reveal_last(4)
        context:
          log: redact  # Never log full number
```

---

#### Pattern 2: Audit Trail

**Goal**: Complete audit trail for compliance.

**Implementation**:
```yaml
dataState:
  governance:
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 7y
      immutable: true

policy:
  name: AuditAccessPolicy
  appliesTo:
    type: dataState
    name: FinancialRecord
  effect: allow
  when: subject.role in ["auditor", "compliance", "admin"]
  audit:
    log_access: true
    log_denial: true
    log_level: detailed
```

**Audit Log Format**:
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "trace_id": "trace-abc-123",
  "actor": {
    "subject_id": "user-789",
    "roles": ["customer"],
    "ip": "192.168.1.100"
  },
  "action": "read",
  "resource": {
    "type": "dataState",
    "name": "AccountRecord",
    "entity_id": "account-456"
  },
  "decision": "allow",
  "policy": "ViewBalancePolicy_v1",
  "fields_accessed": ["balance", "account_number"],
  "fields_masked": ["account_number"]
}
```

---

#### Pattern 3: Rate Limiting

**Goal**: Prevent abuse and DoS attacks.

**Implementation**:
```yaml
inputIntent:
  name: TransferFunds
  rate_limit:
    enabled: true
    strategy: sliding_window
    limit: 10
    window: 1m
    scope: per_subject
    on_exceeded:
      action: reject
      message: "Rate limit exceeded. Try again in {{retry_after}} seconds"

externalIntegration:
  name: PaymentGateway
  reliability:
    rate_limit:
      requests_per_second: 100
      burst: 150
    circuit_breaker:
      enabled: true
      failure_threshold: 5
      reset_timeout: 60s
```

---

#### Pattern 4: Delegation with Time Limits

**Goal**: Temporary access with automatic expiry.

**Implementation**:
```yaml
delegation:
  name: TemporaryAccountantAccess
  delegate:
    from: account_owner
    to: accountant
    scope:
      actions: [view_balance, view_transactions, export_statement]
      data: [AccountRecord, TransactionRecord]
  consent:
    required: true
    type: explicit
    expires_after: 7d  # Automatic expiry
    revocable: true
    audit:
      log_access: true
      log_revocation: true
      
# Revocation policy
policy:
  name: AutoRevokeDelegation
  appliesTo:
    type: delegation
    name: TemporaryAccountantAccess
  effect: deny
  when: |
    current_time > delegation.granted_at + delegation.expires_after OR
    delegation.revoked == true
```

---

## Appendix G: Migration Guides

### Migrating from v1.4 to v1.5

This guide helps migrate existing v1.4 specifications to v1.5.

#### Step 1: Review New Features

Identify which v1.5 features apply to your application:
- ✅ Asset Semantics: If you handle files/images
- ✅ Search: If you need search functionality
- ✅ Scheduled Triggers: If you have recurring tasks
- ✅ Notifications: If you send alerts
- ✅ Workflows: If you have multi-step processes
- ✅ Governance: If you need compliance

#### Step 2: Add Asset Fields

**Before (v1.4)**:
```yaml
dataType:
  fields:
    profile_picture_url: string
```

**After (v1.5)**:
```yaml
dataType:
  fields:
    profile_picture:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
        lifecycle:
          type: permanent
```

#### Step 3: Add Search Configuration

**Before (v1.4)**:
```yaml
dataType:
  fields:
    name: string
    description: string
```

**After (v1.5)**:
```yaml
dataType:
  fields:
    name:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 2.0
    description:
      type: string
      search:
        indexed: true
        strategy: full_text
```

#### Step 4: Convert Cron Jobs to Scheduled Triggers

**Before (v1.4)**: External cron + API call

**After (v1.5)**:
```yaml
inputIntent:
  name: GenerateDailyReport
  scheduled:
    enabled: true
    trigger:
      type: cron
      cron: "0 2 * * *"
    timezone:
      mode: fixed
      fixed: "UTC"
```

#### Step 5: Add Governance Policies

**New in v1.5**:
```yaml
dataState:
  name: CustomerData
  governance:
    classification: pii
    retention:
      policy: time_based
      duration: 3y
      on_expiry: anonymize
    audit:
      access_logging: true
      modification_logging: true
```

#### Step 6: Test Migration

1. Validate new spec with parser
2. Shadow deploy new version
3. Compare behavior with v1.4
4. Gradually promote to production

---


## Appendix H: Reference Implementation Guidance

### Runtime Architecture

A conformant SMS v1.5 runtime SHOULD implement the following architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane                          │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Policy      │  │ Worker       │  │ Authority    │      │
│  │ Distribution│  │ Registry     │  │ Management   │      │
│  └─────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                   ┌────────┴────────┐
                   │  NATS / Message │
                   │     Fabric      │
                   └────────┬────────┘
                            │
┌────────────────────────────┴──────────────────────────────┐
│                      Data Plane                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Workers  │  │ Flow     │  │ View     │  │ Policy   │ │
│  │          │  │ Engine   │  │ Material │  │ Enforcer │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
└────────────────────────────────────────────────────────────┘
                            │
                   ┌────────┴────────┐
                   │  Storage Layer  │
                   │  ┌───┐  ┌───┐  │
                   │  │KV │  │JS │  │
                   │  └───┘  └───┘  │
                   └─────────────────┘
```

#### Component Responsibilities

**Control Plane**:
- Policy distribution to all workers
- Worker discovery and health monitoring
- Authority state management
- Topology configuration

**Workers**:
- Execute work units
- Local policy enforcement
- Emit operational signals
- Maintain routing cache

**Flow Engine**:
- Parse and execute SMS flows
- Atomic group coordination
- Compensation orchestration
- Idempotency checking

**View Materializer**:
- Subscribe to source events
- Compute derived views
- Maintain freshness SLAs
- Handle temporal windows (v1.5)

**Policy Enforcer**:
- Cache policy definitions
- Evaluate policies locally
- Record audit logs
- Support shadow mode

**Storage Layer**:
- Authority KV: Strong consistency, small volume
- JetStream: Event streams, high throughput
- Object Storage: Assets and large documents

---

### Implementation Checklist

#### Phase 1: Core Grammar (Weeks 1-2)
- [ ] Parse YAML specifications
- [ ] Validate structural rules
- [ ] Validate semantic rules
- [ ] Generate types from DataType
- [ ] Support all primitive types
- [ ] Support complex types (enum, array, reference)

#### Phase 2: Execution (Weeks 3-4)
- [ ] Implement flow engine (FSM)
- [ ] Execute work units
- [ ] Enforce atomic groups
- [ ] Validate constraints
- [ ] Implement idempotency checking
- [ ] Handle execution context propagation

#### Phase 3: Policy (Weeks 5-6)
- [ ] Load policies from control plane
- [ ] Evaluate policies locally
- [ ] Support shadow policies
- [ ] Handle lifecycle transitions
- [ ] Implement audit logging
- [ ] Support policy versioning

#### Phase 4: Distribution (Weeks 7-8)
- [ ] Integrate NATS (or alternative)
- [ ] Worker registration/discovery
- [ ] Heartbeat loop
- [ ] Routing cache
- [ ] Signal emission
- [ ] TTL-based expiry

#### Phase 5: Authority & Multi-Region (Weeks 9-12)
- [ ] Entity-scoped authority resolution
- [ ] CAS token validation
- [ ] Authority migration (pause-and-cutover)
- [ ] Cross-region replication
- [ ] View authority independence
- [ ] Regional failover

#### Phase 6: UI Composition (Weeks 13-14)
- [ ] Parse compositions and experiences
- [ ] Build navigation hierarchy
- [ ] Resolve related views
- [ ] Apply field permissions
- [ ] Evaluate indicators
- [ ] Route experiences

#### Phase 7: v1.5 Core Features (Weeks 15-18)
- [ ] Form-intent binding with validation
- [ ] View materialization contracts (retrieval modes)
- [ ] Streaming views (temporal windows)
- [ ] Intent delivery guarantees
- [ ] Session management with sync
- [ ] Multi-device conflict resolution
- [ ] Offline support
- [ ] Asset upload and transformation
- [ ] Asset variants generation
- [ ] CDN integration for assets

#### Phase 8: v1.5 Search & Discovery (Weeks 19-20)
- [ ] Search index configuration
- [ ] Full-text search implementation
- [ ] Fuzzy matching
- [ ] Faceted search
- [ ] Result highlighting
- [ ] Search authorization
- [ ] Index freshness management

#### Phase 9: v1.5 Automation (Weeks 21-22)
- [ ] Scheduled trigger execution
- [ ] Cron expression parsing
- [ ] Timezone handling
- [ ] Missed execution handling
- [ ] Multi-channel notifications
- [ ] Notification templates
- [ ] Delivery tracking
- [ ] User preferences
- [ ] Quiet hours

#### Phase 10: v1.5 Advanced Features (Weeks 23-26)
- [ ] Collaborative session handling
- [ ] Presence awareness
- [ ] Operational Transform / CRDT
- [ ] Durable workflow orchestration
- [ ] Workflow checkpoints
- [ ] Compensation logic
- [ ] Human task management
- [ ] Delegation and consent management
- [ ] Conversational UI routing
- [ ] External API integration
- [ ] Circuit breakers
- [ ] Rate limiting

#### Phase 11: v1.5 Governance & Compliance (Weeks 27-28)
- [ ] Data classification enforcement
- [ ] Retention policy automation
- [ ] Legal hold management
- [ ] Right-to-erasure support
- [ ] Data lineage tracking
- [ ] Audit log immutability
- [ ] Encryption at rest/in transit
- [ ] Regional residency enforcement

#### Phase 12: v1.5 Edge & Advanced (Weeks 29-30)
- [ ] Edge device support
- [ ] Offline operation
- [ ] Sync strategies
- [ ] Spatial query support
- [ ] GIS indexing
- [ ] Graph query execution
- [ ] Relationship traversal
- [ ] Structured content rendering
- [ ] ML inference integration

#### Phase 13: Production Readiness (Weeks 31-34)
- [ ] Observability hooks
- [ ] Metrics collection
- [ ] Distributed tracing
- [ ] Chaos testing
- [ ] Performance optimization
- [ ] Load testing
- [ ] Security audit
- [ ] Compliance validation
- [ ] Documentation
- [ ] Operational runbooks

---

### Testing Strategies

#### Unit Testing

**Grammar Parsing**:
```go
func TestParseDataType(t *testing.T) {
    spec := `
dataType:
  name: Order
  version: v1
  fields:
    id: uuid
    total: decimal
`
    dt, err := parser.ParseDataType(spec)
    assert.NoError(t, err)
    assert.Equal(t, "Order", dt.Name)
    assert.Equal(t, "v1", dt.Version)
    assert.Len(t, dt.Fields, 2)
}
```

**Policy Evaluation**:
```go
func TestPolicyEvaluation(t *testing.T) {
    policy := Policy{
        Effect: "allow",
        When:   "subject.role == 'admin'",
    }
    
    ctx := ExecutionContext{
        Actor: Actor{
            Subject: "user-123",
            Roles:   []string{"admin"},
        },
    }
    
    decision := evaluatePolicy(policy, ctx)
    assert.Equal(t, Allow, decision)
}
```

**Idempotency**:
```go
func TestIdempotency(t *testing.T) {
    checker := NewIdempotencyChecker()
    
    key := "transfer-123"
    result := Data{"status": "success"}
    
    // First execution
    seen, _ := checker.Check(key)
    assert.False(t, seen)
    checker.Mark(key, result, 24*time.Hour)
    
    // Second execution (duplicate)
    seen, _ = checker.Check(key)
    assert.True(t, seen)
    cached, _ := checker.GetResult(key)
    assert.Equal(t, result, cached)
}
```

#### Integration Testing

**End-to-End Flow**:
```go
func TestOrderProcessingFlow(t *testing.T) {
    // Setup
    runtime := NewSMSRuntime()
    runtime.LoadSpec("order-spec.yaml")
    
    // Execute flow
    intent := InputIntent{
        Name: "CreateOrder",
        Data: map[string]interface{}{
            "customer_id": "cust-123",
            "items": []OrderItem{
                {SKU: "prod-1", Quantity: 2},
            },
        },
    }
    
    result, err := runtime.ExecuteFlow("ProcessOrder", intent)
    
    // Verify
    assert.NoError(t, err)
    assert.Equal(t, "success", result.Status)
    
    // Check side effects
    order := runtime.GetEntity("OrderRecord", result.OrderID)
    assert.NotNil(t, order)
    assert.Equal(t, "confirmed", order.Status)
}
```

**Multi-Region Authority**:
```go
func TestAuthorityMigration(t *testing.T) {
    // Setup two regions
    usEast := NewRegion("us-east")
    euWest := NewRegion("eu-west")
    
    // Create entity in us-east
    entity := CreateEntity("Account", "acc-123", usEast)
    assert.Equal(t, "us-east", entity.AuthorityRegion)
    assert.Equal(t, 1, entity.AuthorityEpoch)
    
    // Migrate to eu-west
    MigrateAuthority(entity.ID, "eu-west")
    
    // Verify migration
    entity = GetEntity("acc-123")
    assert.Equal(t, "eu-west", entity.AuthorityRegion)
    assert.Equal(t, 2, entity.AuthorityEpoch)
    
    // Old CAS token should fail
    oldToken := CASToken{
        EntityID:        "acc-123",
        AuthorityEpoch:  1,
        AuthorityRegion: "us-east",
    }
    err := WriteWithCAS(entity.ID, data, oldToken)
    assert.Error(t, err)
    assert.Equal(t, ErrEpochMismatch, err)
}
```

#### Chaos Testing

**Network Partition**:
```go
func TestNetworkPartition(t *testing.T) {
    // Start system
    system := StartDistributedSystem()
    
    // Generate load
    loadGen := StartLoadGenerator(100) // 100 req/s
    
    // Partition network
    chaos.PartitionRegion("us-east", 30*time.Second)
    
    // Verify:
    // 1. US entities become unavailable
    // 2. EU entities remain operational
    // 3. No data corruption
    // 4. System recovers after partition heals
    
    metrics := system.GetMetrics()
    assert.Greater(t, metrics.Availability, 0.7) // 70%+ available
    assert.Equal(t, 0, metrics.DataCorruption)
}
```

**Random Worker Failures**:
```go
func TestWorkerFailures(t *testing.T) {
    system := StartDistributedSystem()
    loadGen := StartLoadGenerator(50)
    
    // Kill 20% of workers randomly
    chaos.KillRandomWorkers(0.2, 60*time.Second)
    
    // Verify:
    // 1. System continues operating
    // 2. No work is lost
    // 3. Idempotency prevents duplicates
    
    results := loadGen.GetResults()
    assert.Equal(t, 0, results.LostRequests)
    assert.Equal(t, 0, results.DuplicateProcessing)
}
```

---

### Performance Benchmarks

#### Target Metrics

| Component | Operation | Target | Notes |
|-----------|-----------|--------|-------|
| Parser | Parse 1KB spec | <10ms | Grammar validation |
| Policy | Evaluate simple | <100μs | Local evaluation |
| Policy | Evaluate complex | <1ms | With attribute lookup |
| Flow | Execute simple (3 steps) | <50ms | Local execution |
| Flow | Execute distributed (5 steps) | <200ms | Cross-boundary |
| View | Materialize simple | <500ms | Single source |
| View | Materialize complex | <5s | Multiple sources, aggregation |
| Authority | CAS validation | <1ms | Token verification |
| Authority | Migration | <100ms | Write pause duration |
| Storage (Control) | Read | <5ms | KV lookup |
| Storage (Control) | Write | <10ms | KV update |
| Storage (Data) | Append | <10ms | Event stream |
| Storage (Data) | Read | <20ms | Event replay |
| Search | Simple query | <50ms | Single field |
| Search | Complex query | <200ms | Multiple fields, facets |
| Asset | Upload (1MB) | <2s | Including validation |
| Asset | Download (1MB) | <1s | CDN delivery |
| Notification | Send (push) | <500ms | FCM/APNS |
| Notification | Send (email) | <2s | SMTP delivery |

#### Benchmark Tests

```go
func BenchmarkPolicyEvaluation(b *testing.B) {
    policy := LoadPolicy("test-policy")
    ctx := LoadContext("test-context")
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        evaluatePolicy(policy, ctx)
    }
}

func BenchmarkViewMaterialization(b *testing.B) {
    events := LoadEvents("test-events", 1000)
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        materializeView("TestView", events)
    }
}

func BenchmarkCASValidation(b *testing.B) {
    token := CASToken{...}
    entity := LoadEntity("test-entity")
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        validateCASToken(token, entity)
    }
}
```

---

## Appendix I: Glossary

### Core Terms

**Addendum**: Extension to core grammar filling semantic gaps. All addendums integrated inline in v1.5.

**Asset**: Binary content (files, images, media) as first-class data type with lifecycle and transformation.

**Atomic Group**: Set of work units that MUST execute together in same boundary. Failure of any fails all.

**Authority**: Right to accept writes for an entity. Resolved per entity in v1.3+.

**Authority Epoch**: Monotonically increasing number tracking authority changes. Part of CAS token.

**Authority Migration**: Transfer of write authority between regions using pause-and-cutover strategy.

**Boundary**: Execution domain where atomic groups execute. Typically process, container, or node.

**CAS Token**: Compare-And-Set token ensuring linearizable writes across authority migrations.

**Composition**: Semantic grouping of views into workspace (v1.4+).

**Constraint**: Semantic truth about data. Separate from structure for independent evolution.

**Control Storage**: Strong-consistency storage for coordination (authority KV, policies). Distinct from data storage.

**Data Storage**: Append-only or rebuildable storage for domain events. Distinct from control storage.

**DataState**: Lifecycle, enforcement, and evolution policy for data.

**DataType**: Structural definition without behavior or lifecycle.

**Delegation**: Temporary authority transfer with time limits, scope restrictions, and audit (v1.5).

**Durable Workflow**: Long-running process with checkpoints, compensation, and human tasks (v1.5).

**Entity-Scoped Authority**: Authority resolved per entity instance, not per model (v1.3).

**Experience**: Application-level grouping of compositions for user journey (v1.4+).

**Flow**: SMS execution graph describing ordering and dependencies, not transport.

**Form Intent Binding**: Declarative mapping between UI forms and intents (v1.5).

**Governance**: Classification, retention, residency, encryption policies for compliance (v1.5).

**Graph Query**: Relationship traversal for social graphs and hierarchies (v1.5).

**Idempotency**: Property that executing same operation multiple times produces same result. Required for all transitions.

**Indicator**: Dynamic badge on navigation showing state (count, boolean, value) (v1.4+).

**Inference**: ML model output as declarative field (v1.5).

**InputIntent**: What user/system may propose before full validation. Inverse of data intent.

**Invariant**: Constraint that MUST always hold. Cross-entity invariants enforced by views (v1.3).

**Materialization**: Derived, regenerable view. Authority-agnostic and epoch-tolerant by default (v1.3).

**Mutability**: Write authority semantics for data. Scope: entity or append-only (v1.3).

**Notification Channel**: Multi-channel delivery (email, SMS, push, webhook) with preferences (v1.5).

**Pause-and-Cutover**: Authority migration strategy. Writes pause briefly during epoch increment and region change.

**Policy**: Declarative authorization with lifecycle (shadow/enforce/deprecated/revoked).

**Presentation View**: UI contract bound to data. Not implementation.

**Relationship**: Semantic linkage between entities with cardinality and causality (v1.3).

**Scheduled Trigger**: Cron-style temporal event generation (v1.5).

**Search**: Full-text, fuzzy, or semantic search with faceting and ranking (v1.5).

**Session**: Multi-device, offline-capable session state (v1.5).

**Shadow Mode**: Test policies/writes/UI with real traffic without affecting outcomes.

**Signal**: Operational observation informing scheduling. Not command.

**Spatial Type**: Geographic/GIS data with indexing and queries (v1.5).

**Storage Role**: Distinction between control (coordination) and data (domain) storage (v1.3).

**Structured Content**: Rich text/document with semantic structure (v1.5).

**Temporal Window**: Time-based aggregation for streaming views (tumbling, sliding, session, hopping) (v1.5).

**Transition**: Typed edge between DataStates with explicit triggers and guarantees.

**View Authority Independence**: Views tolerate events from multiple regions and epochs (v1.3).

**Work Unit**: Atomic piece of work executed by worker. Has contract defining input/output/side-effects.

**Worker**: Process that accepts inputs, produces outputs, executes work units.

---

### Acronyms

**ABAC**: Attribute-Based Access Control

**API**: Application Programming Interface

**CAS**: Compare-And-Set

**CDN**: Content Delivery Network

**CRDT**: Conflict-Free Replicated Data Type

**DST**: Daylight Saving Time

**EBNF**: Extended Backus-Naur Form

**GDPR**: General Data Protection Regulation (EU)

**GIS**: Geographic Information System

**HIPAA**: Health Insurance Portability and Accountability Act (US)

**HTTP**: Hypertext Transfer Protocol

**IoT**: Internet of Things

**JWT**: JSON Web Token

**KV**: Key-Value (storage)

**ML**: Machine Learning

**NATS**: Neural Autonomic Transport System (messaging)

**NLU**: Natural Language Understanding

**OT**: Operational Transform

**PCI**: Payment Card Industry

**PDF**: Portable Document Format

**PII**: Personally Identifiable Information

**RBAC**: Role-Based Access Control

**REST**: Representational State Transfer

**RFC**: Request for Comments

**SDK**: Software Development Kit

**SLA**: Service Level Agreement

**SMS**: System Mechanics Specification / State Machine System

**SQL**: Structured Query Language

**SSN**: Social Security Number

**TTL**: Time To Live

**UI**: User Interface

**UUID**: Universally Unique Identifier

**WCAG**: Web Content Accessibility Guidelines

**WTS**: Worker Topology Spec

**YAML**: YAML Ain't Markup Language

---

## Appendix J: FAQ

### General Questions

**Q: Is SMS v1.5 backward compatible with v1.4?**

A: Yes, completely. All v1.4 specifications remain valid in v1.5. v1.5 adds new optional features without breaking changes.

---

**Q: Do I need to use all v1.5 features?**

A: No. v1.5 features are additive and optional. Use only what your application needs. A valid v1.5 spec might use none of the new features.

---

**Q: Can I migrate from v1.3 directly to v1.5?**

A: Yes. v1.5 is backward compatible with v1.3. Review v1.4 UI composition features and v1.5 additions to see what applies to your use case.

---

**Q: Is NATS required?**

A: No. NATS is used in reference implementation for control plane and coordination, but grammar is transport-agnostic. You can use HTTP, gRPC, Kafka, or custom transports.

---

**Q: Can I use SMS for monoliths?**

A: Yes. Same grammar works for monoliths, modular monoliths, microservices, serverless, and hybrid. Grammar describes intent, not deployment topology.

---

### Authority & Multi-Region Questions

**Q: Why entity-scoped authority instead of model-scoped?**

A: Entity-scoped authority provides granular failure isolation. One entity's region failure doesn't affect other entities. Model-scoped would collapse availability for entire model.

---

**Q: What happens during authority migration?**

A: Brief write pause (<100ms typically) while:
1. Current region pauses writes
2. Authority epoch increments
3. New region becomes authority
4. Writes resume in new region

Reads unaffected throughout.

---

**Q: Can authority migrate automatically?**

A: Yes, based on declared triggers (load, latency, time). But migration can be constrained by governance policies.

---

**Q: What if migration fails mid-way?**

A: System resolves deterministically to single authority (Invariant I8). Either:
- Old region retains authority (migration aborted)
- New region gains authority (migration completed)

No split-brain possible.

---

### Data & Views Questions

**Q: Why are invariants in views, not entities?**

A: Cross-entity invariants in entities would require distributed transactions. Views enforce invariants asynchronously without blocking writes.

---

**Q: What if view update is too slow?**

A: Adjust `maxStaleness` tolerance, scale view workers, or optimize view query. Trade-off between consistency and performance.

---

**Q: Can views query across regions?**

A: Yes. Views are authority-agnostic by default. They tolerate and merge events from multiple regions.

---

### UI & Forms Questions

**Q: How does form intent binding differ from REST APIs?**

A: Form binding is declarative and transport-agnostic. Same binding works for browser forms, mobile apps, voice interfaces. Runtime chooses HTTP, gRPC, etc.

---

**Q: Can field permissions hide entire views?**

A: Field permissions control field-level visibility. Use composition policies to hide entire views.

---

**Q: How do indicators update in real-time?**

A: Runtime subscribes to view changes and pushes indicator updates via WebSocket, SSE, or polling based on declared `update.trigger`.

---

### Search & Assets Questions

**Q: Which search engine should I use?**

A: Grammar is engine-agnostic. Elasticsearch, Algolia, Typesense, or custom all work. Choose based on scale and features needed.

---

**Q: Where are assets stored?**

A: Runtime decides. Typically S3, Azure Blob, GCS, or CDN. Grammar specifies lifecycle and access control, not storage location.

---

**Q: Can assets be version controlled?**

A: Grammar doesn't specify versioning. Runtime can implement if needed for use case (e.g., document management).

---

### Governance & Compliance Questions

**Q: Does SMS ensure GDPR compliance?**

A: SMS provides primitives (retention, anonymization, audit, right-to-erasure). You must configure policies correctly and runtime must enforce them.

---

**Q: How long are audit logs retained?**

A: Configurable per dataState. Typically 7 years for financial/compliance data.

---

**Q: Can data move between regions?**

A: Only if `residency.transfer.allowed = true` and governance constraints satisfied. Authority can migrate, but data may stay (compliance).

---

### Performance Questions

**Q: What's the latency of authority migration?**

A: Typically <100ms write pause. Depends on network latency between regions. Can be optimized with dedicated links.

---

**Q: How fast is policy evaluation?**

A: <100μs for simple policies (local cache), <1ms for complex (attribute lookup). All evaluation is local.

---

**Q: Can search scale to millions of documents?**

A: Yes. Search grammar supports engines that scale horizontally (Elasticsearch, etc.). Runtime handles sharding and replication.

---

### Development Questions

**Q: How do I test specifications before implementing?**

A: Use parser/validator to check grammar. Use shadow mode to test behavior with real traffic before enforcement.

---

**Q: Can I generate code from specifications?**

A: Yes. Many implementations generate types, APIs, and UI scaffolding from specs. Grammar is machine-readable.

---

**Q: How do I debug policy denials?**

A: Enable detailed audit logging. Logs show which policy denied, which expression evaluated false, and full context.

---

**Q: What tools exist for SMS?**

A: Reference implementation includes:
- Parser/validator
- Runtime (Go)
- CLI tools
- Testing utilities
- Visualization tools (Mermaid diagrams)

---

## Conclusion

This completes the SMS v1.5 Specification document. All addendums have been integrated inline, creating a single, comprehensive source of truth for the System Mechanics Specification.

### What You've Learned

This specification covered:

1. **Core Principles**: Intent-first design, versioning, zero-downtime evolution
2. **Complete Grammar**: Data, UI, Input, Flows, Workers, Policies, v1.5 extensions
3. **Authority Management**: Entity-scoped authority, migrations, multi-region
4. **UI Composition**: Semantic grouping, experiences, field permissions, navigation
5. **Advanced Features**: Assets, search, workflows, notifications, collaboration, governance
6. **Design Rationale**: Why decisions were made, alternatives considered
7. **Implementation Guidance**: Architecture, testing, performance, security
8. **Practical Examples**: Complete applications demonstrating all features
9. **Troubleshooting**: Common issues and solutions
10. **Migration Paths**: How to adopt v1.5 from earlier versions

### Next Steps

**For Specification Authors**:
1. Review examples relevant to your domain
2. Start with core grammar (Data, UI, Input)
3. Add v1.5 features as needed
4. Validate with parser
5. Test in shadow mode

**For Runtime Implementers**:
1. Follow implementation checklist
2. Start with core grammar and execution
3. Add features incrementally
4. Test extensively with chaos testing
5. Optimize for your use case

**For Architects**:
1. Understand design rationale
2. Map your requirements to SMS features
3. Design authority and boundary strategy
4. Plan multi-region deployment
5. Define governance policies

### Resources

**Specification Documents**:
- This document: Complete normative specification
- Implementation Guide: Runtime architecture and patterns
- Reference Examples: 15 production-ready applications
- Quick Reference: Cheat sheets and decision trees

**Community**:
- Discussions: Design questions and use cases
- Issues: Bug reports and feature requests
- RFCs: Proposals for future versions

---

**This is the complete SMS v1.5 Specification.**

**Version**: 1.5  
**Status**: Production Ready  
**Completeness**: 100%  
**Total Lines**: ~8,500  
**Last Updated**: 2026-01-18

**All addendums integrated. Single source of truth.**


---

## Appendix K: Extended Validation Rules

### Structural Validation Rules

#### Rule V1: Name Validation
```
All names (dataType, inputIntent, policy, etc.) MUST:
- Start with uppercase letter
- Contain only alphanumeric and underscore
- Be unique within scope
- Not collide with reserved keywords
```

Reserved keywords: `version`, `type`, `system`, `runtime`, `internal`

#### Rule V2: Version Format
```
Versions MUST follow pattern: v<major>[.<minor>][.<patch>]
Examples: v1, v1.0, v1.2.3
Ordering: Lexicographic comparison
```

#### Rule V3: Reference Validation
```
All references (dataType refs, intent refs, etc.) MUST:
- Refer to defined artifact
- Include version if versioned
- Respect visibility (no forward refs unless allowed)
```

#### Rule V4: Field Name Validation
```
Field names MUST:
- Start with lowercase letter
- Use snake_case (not camelCase or PascalCase)
- Not conflict with built-in fields
- Not use reserved names
```

Reserved field names: `id`, `version`, `created_at`, `updated_at`, `deleted_at`

#### Rule V5: Expression Syntax
```
All expressions (policy when, constraint rules, etc.) MUST:
- Parse successfully
- Type-check correctly
- Reference only available symbols
- Terminate (no infinite loops)
```

### Semantic Validation Rules

#### Rule V6: Authority Resolution
```
For entity-scoped authority:
- resolution expression MUST be deterministic
- MUST return same result for same input
- MAY reference entity fields and subject attributes
- MUST NOT have side effects
```

#### Rule V7: Atomic Group Constraints
```
Atomic groups MUST:
- Declare boundary
- All steps MUST execute in same boundary
- All steps MUST be sequential (no parallel within group)
- No nested atomic groups
```

#### Rule V8: CAS Token Completeness
```
CAS tokens MUST include:
- entity_id
- authority_epoch (if entity-scoped authority)
- authority_region (if multi-region)
- entity_version (optional for tombstone check)
```

#### Rule V9: View Freshness Feasibility
```
For materialized views:
- maxStaleness MUST be achievable given source latency
- SHOULD be >= 2 * source update latency
- Runtime MAY reject infeasible freshness targets
```

#### Rule V10: Idempotency Key Uniqueness
```
Idempotency keys MUST:
- Be unique within scope
- Include sufficient entropy
- Be reproducible from same input
- Not exceed 256 characters
```

#### Rule V11: Constraint Enforcement Mode
```
Constraints MUST declare enforcement:
- strict: Reject on violation
- deferred: Flag violation asynchronously
- best_effort: Log but don't fail

Entity constraints SHOULD be strict or deferred.
View constraints MUST be deferred.
```

#### Rule V12: Policy Effect Clarity
```
Policies MUST declare effect:
- allow: Permit if when is true
- deny: Forbid if when is true

Multiple matching policies:
- Any deny wins (fail-closed)
- All allow required for access
```

#### Rule V13: Storage Role Consistency
```
DataState storage role:
- control: NOT rebuildable, strong consistency required
- data: Rebuildable from events, eventual consistency OK

Authority state MUST use control.
Event streams SHOULD use data.
```

#### Rule V14: Migration Strategy Safety
```
Authority migration:
- pause-and-cutover: Brief write pause, deterministic
- background-sync: No pause, eventual consistency
- drain-and-cutover: Long pause, draining pending writes

pause-and-cutover is DEFAULT and RECOMMENDED.
```

#### Rule V15: Relationship Causality
```
Relationships with cascading_delete:
- parent → child: Parent deletion deletes children
- NOT child → parent: Would cause orphans

Validation: Check acyclic dependency graph.
```

### v1.5 Extended Validation Rules

#### Rule V16: Asset Lifecycle Validation
```
Asset lifecycle:
- permanent: Survives entity deletion
- temporary: Deleted with entity
- expiring: Auto-deleted after duration

expiring MUST specify expires_after.
permanent SHOULD declare retention policy.
```

#### Rule V17: Search Strategy Compatibility
```
Search strategies per field type:
- string: full_text, exact, fuzzy, prefix
- enum: exact only
- numeric: exact, range
- spatial: radius, bounding_box, polygon
- date: exact, range

Invalid combinations rejected at validation.
```

#### Rule V18: Notification Channel Validation
```
Notification channels:
- At least one channel MUST be enabled
- Fallback order MUST be specified if >1 channel
- Channel config MUST match channel type
```

#### Rule V19: Scheduled Trigger Validation
```
Scheduled triggers:
- cron expression MUST be valid
- Timezone MUST be specified
- on_missed action MUST be appropriate:
  - skip: For recurrence-based (daily reports)
  - execute_immediately: For critical tasks
  - queue: For batch processing
```

#### Rule V20: Durable Workflow Constraints
```
Durable workflows:
- Checkpoint frequency MUST NOT be "never"
- Total timeout MUST be reasonable (<30 days typical)
- Human tasks MUST specify timeout and escalation
- Compensation MUST be defined for non-idempotent steps
```

#### Rule V21: Form Field Mapping Completeness
```
Form intent binding:
- All required intent fields MUST be supplied
- Field sources MUST be valid:
  - context: Available in execution context
  - input: User-provided
  - computed: Derivable from other fields
```

#### Rule V22: Temporal Window Validation
```
Temporal windows:
- tumbling: size required
- sliding: size and slide required
- session: gap required
- hopping: size and hop required

Emit triggers:
- on_window_close: Standard
- on_every_record: High volume
- on_watermark: Out-of-order data
```

#### Rule V23: Collaboration Conflict Resolution
```
Collaborative sessions:
- operational_transform: For text editing
- crdt: For distributed data structures
- last_write_wins: Simple, may lose data
- manual: User resolves conflicts

Choice MUST match data type and use case.
```

#### Rule V24: Governance Retention Consistency
```
Data governance:
- time_based: Duration required
- event_based: after_event required
- permanent: No expiry

on_expiry action:
- delete: Hard delete
- anonymize: Remove PII, keep aggregates
- archive: Move to cold storage

Audit logs MUST NOT be deletable.
```

#### Rule V25: Edge Device Capability Declaration
```
Edge devices:
- Capabilities MUST be declared:
  - compute: CPU architecture and speed
  - storage: Available storage
  - connectivity: always | intermittent | scheduled
  
- Sync strategy MUST match connectivity:
  - real_time requires always
  - periodic works with intermittent
  - on_connect works with all
```

---

## Appendix L: Advanced Patterns & Anti-Patterns

### Pattern: Idempotent State Machines

**Problem**: State transitions must be idempotent but still enforce ordering.

**Solution**:
```yaml
transition:
  name: ShipOrder
  from: confirmed
  to: shipped
  guarantees:
    idempotency:
      key: ["order_id", "ship"]
      scope: global
      ttl: 7d
  validation:
    pre_condition: |
      order.status == "confirmed" OR order.status == "shipped"
    post_condition: |
      order.status == "shipped"
```

**Why It Works**:
- Pre-condition accepts both "confirmed" (first execution) and "shipped" (retry)
- Post-condition ensures final state is correct
- Idempotency key prevents double-processing
- TTL allows key cleanup after reasonable period

---

### Pattern: Compensating Transactions Without Distributed Locks

**Problem**: Need to undo operations across boundaries without 2PC.

**Solution**:
```yaml
sms:
  flows:
    - name: TransferWithCompensation
      steps:
        - work_unit: debit_account
          output: debit_id
          compensate: refund_account
          compensation_input: { debit_id: "{{debit_id}}" }
        
        - work_unit: credit_account
          output: credit_id
          compensate: reverse_credit
          compensation_input: { credit_id: "{{credit_id}}" }
          on_failure:
            execute: [reverse_credit, refund_account]
            order: reverse  # Undo in reverse order
```

**Key Principles**:
1. Each step stores compensation handle (ID)
2. Compensation is idempotent
3. Failure triggers reverse-order unwinding
4. Each compensation is best-effort (may fail)

---

### Pattern: View Composition for Complex Queries

**Problem**: Need to query across multiple entities efficiently.

**Solution**:
```yaml
# Base views
materialization:
  name: CustomerView
  source: CustomerEvent
  targetState: Customer
  retrieval:
    mode: by_entity
    entity_key: customer_id

materialization:
  name: OrderView
  source: OrderEvent
  targetState: Order
  retrieval:
    mode: by_entity
    entity_key: order_id

# Composed view
materialization:
  name: CustomerOrderSummary
  sources:
    - view: CustomerView
      join_key: customer_id
    - view: OrderView
      join_key: customer_id
  targetState: CustomerOrderSummary
  aggregation:
    - field: order.total
      function: sum
      as: total_spent
    - field: order.id
      function: count
      as: order_count
  retrieval:
    mode: by_entity
    entity_key: customer_id
```

**Benefits**:
- Reuses existing views
- Declarative join and aggregation
- Maintains freshness SLAs
- Scales horizontally

---

### Anti-Pattern: Embedding Business Logic in Policies

**Problem**: Policies with complex business rules become unmaintainable.

**Bad**:
```yaml
policy:
  when: |
    (subject.role == "manager" AND account.balance < 10000) OR
    (subject.role == "director" AND account.balance < 50000) OR
    (subject.role == "vp" AND account.balance < 100000) OR
    subject.role == "cfo"
```

**Good**:
```yaml
# Define permission
policy:
  when: subject.has_permission("approve_transfer")

# Business logic in work unit
work_unit:
  name: ApproveTransfer
  validation:
    pre_condition: |
      determine_approver(transfer.amount, subject.role)
```

**Why**:
- Policies focus on who, not what
- Business rules in appropriate layer
- Easier to test and evolve

---

### Anti-Pattern: Over-Normalized Data Model

**Problem**: Too many entities with complex relationships.

**Bad**:
```yaml
dataType:
  name: Address
  # Separate entity for address

dataType:
  name: PhoneNumber
  # Separate entity for phone

dataType:
  name: Customer
  fields:
    address_id: uuid
    phone_id: uuid
```

**Good**:
```yaml
dataType:
  name: Customer
  fields:
    address:
      street: string
      city: string
      postal_code: string
    phone: string
```

**Why**:
- Embedded values are simpler
- Fewer joins required
- Better query performance
- Only separate when independent lifecycle

---

### Anti-Pattern: Synchronous Cross-Region Calls

**Problem**: Synchronous calls across regions add latency.

**Bad**:
```yaml
sms:
  flows:
    - steps:
        - work_unit: process_order        # us-east
        - work_unit: notify_warehouse     # eu-west (synchronous)
```

**Good**:
```yaml
sms:
  flows:
    - steps:
        - work_unit: process_order        # us-east
        - emit:
            signal: OrderProcessed
        # Warehouse subscribes asynchronously

# Separate flow in eu-west
sms:
  flows:
    - triggeredBy:
        signal: OrderProcessed
      steps:
        - work_unit: notify_warehouse
```

**Why**:
- Asynchronous = lower latency
- Region failures don't block
- Better scalability

---

### Anti-Pattern: Stale View in Critical Path

**Problem**: Using eventual view for strong consistency requirement.

**Bad**:
```yaml
# View with 5-second staleness
materialization:
  name: InventoryView
  freshness:
    maxStaleness: 5s

# Flow using view for critical decision
sms:
  flows:
    - steps:
        - work_unit: check_inventory_view  # May be stale!
        - work_unit: reserve_item
```

**Good**:
```yaml
# Read authoritative source for critical decisions
sms:
  flows:
    - steps:
        - work_unit: check_inventory_authoritative
        - work_unit: reserve_item
```

**Why**:
- Strong consistency where needed
- Views for queries, not critical decisions
- Accept staleness only where safe

---

### Anti-Pattern: Unbounded Temporal Windows

**Problem**: Windows without bounds consume infinite memory.

**Bad**:
```yaml
temporal:
  window:
    type: session
    # No timeout specified!
```

**Good**:
```yaml
temporal:
  window:
    type: session
    gap: 5m          # Max 5-minute idle
    max_duration: 4h # Max 4-hour session
```

**Why**:
- Prevents memory leaks
- Ensures windows close
- Predictable resource usage

---

## Appendix M: Versioning & Evolution Strategy

### Version Numbering

SMS follows semantic versioning for specifications:

- **Major version** (v1, v2): Breaking changes to grammar
- **Minor version** (v1.5, v1.6): Additive features, backward compatible
- **Patch version** (v1.5.1): Clarifications, bug fixes

### Backward Compatibility Guarantees

#### v1.x Promise
All v1.x versions are backward compatible:
- v1.5 specs work with v1.0 runtimes (new features ignored)
- v1.0 specs work with v1.5 runtimes (all features supported)
- Parsers MUST accept older versions
- Parsers MAY accept newer versions (forward compatible)

#### Deprecation Policy
Features deprecated in v1.x:
- Marked deprecated in specification
- Continue to work through v1.x
- Removed in v2.0

Current deprecations: None (v1.5 has no deprecations)

### Adding New Features

New features in minor versions MUST:
1. Be additive (new fields, new artifact types)
2. Have sensible defaults
3. Not change existing semantics
4. Be clearly marked as "NEW in vX.Y"

### Runtime Version Negotiation

Clients and runtimes negotiate versions:

```yaml
# Client declares requirement
metadata:
  requires_sms_version: v1.5
  compatible_with: [v1.5, v1.6]

# Runtime declares support
runtime:
  sms_version: v1.5
  supports: [v1.0, v1.1, v1.2, v1.3, v1.4, v1.5]
```

If incompatible:
- Runtime rejects specification
- Error message indicates required version
- Client must upgrade or downgrade spec

### Future Roadmap (Informational)

**v1.6 (Planned)**:
- Enhanced observability grammar
- Cost attribution semantics
- Multi-tenancy improvements

**v2.0 (Future)**:
- Remove deprecated features (none currently)
- Simplified grammar based on learnings
- Performance-oriented changes

---

**End of SMS v1.5 Specification**

**Document Status**: Complete and Normative  
**Line Count**: 8,500+ lines  
**Integration**: All addendums merged inline  
**Verification**: Ready for review

