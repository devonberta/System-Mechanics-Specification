# System Mechanics Specification v1.3 - Documentation Index

## Quick Navigation

This index provides a visual map of all reference documentation and guidance for navigating the specification.

### v1.3 Additions

- Entity-Scoped Authority and Multi-Region Survivability
- Invariant-Oriented Modeling
- Relationship Grammar with Cardinality
- Storage Role Distinction
- CAS Token Semantics

---

## DOCUMENT MAP

```
System Mechanics Specification v1.3
│
├─ 00-index.md (THIS FILE)
│   └─ Navigation guide and document relationships
│
├─ README.md
│   └─ Overview, usage guidance, and getting started
│
├─ 01-grammar-reference.md ⭐ NORMATIVE
│   ├─ Complete formal grammar
│   ├─ All first-class concepts
│   ├─ Validation rules
│   ├─ Conformance requirements
│   └─ Lexical grammar
│
├─ 02-grammar-examples.md
│   ├─ 40+ concrete examples
│   ├─ Simple → Complex progression
│   ├─ Real-world use cases
│   ├─ Anti-patterns
│   └─ End-to-end examples
│
├─ 03-implementation-specification.md ⭐ CRITICAL
│   ├─ Runtime architecture patterns
│   ├─ NATS integration details
│   ├─ Control plane design
│   ├─ Policy enforcement
│   ├─ Flow execution
│   ├─ Multi-region patterns
│   ├─ Testing strategies
│   └─ Code generation patterns
│
├─ 04-design-rationale.md
│   ├─ Why decisions were made
│   ├─ Tradeoffs analyzed
│   ├─ Alternatives rejected
│   ├─ Lessons learned
│   └─ Anti-patterns prevented
│
├─ 05-formal-grammar-ebnf.md ⭐ NORMATIVE
│   ├─ Complete EBNF grammar
│   ├─ Suitable for parser generation
│   ├─ Formal semantics
│   └─ Validation rules
│
├─ 06-technical-implementation-details.md ⭐ CRITICAL
│   ├─ Deep technical patterns
│   ├─ NATS protocols
│   ├─ Performance optimization
│   ├─ Caching strategies
│   ├─ Streaming patterns
│   └─ Operational procedures
│
├─ 07-specification-addendums.md ⭐ NORMATIVE
│   ├─ Critical missing pieces
│   ├─ Transition grammar
│   ├─ Idempotency semantics
│   ├─ Error classification
│   ├─ Boundary clarifications
│   └─ Failure invariants
│
├─ 08-consolidated-specification.md ⭐ AUTHORITATIVE
│   ├─ Complete integrated reference
│   ├─ All grammar in one place
│   ├─ All addendums integrated
│   ├─ Single source of truth
│   └─ Implementation checklist
│
├─ 09-quick-reference.md
│   ├─ Cheat sheets
│   ├─ Decision trees
│   ├─ Templates
│   ├─ Troubleshooting
│   └─ Quick lookup tables
│
├─ Appendix-F-Invariant-Oriented-Modeling.md ⭐ NEW v1.3
│   ├─ Entity design rules
│   ├─ Relationship design rules
│   ├─ View design rules
│   ├─ Invariant placement table
│   └─ Anti-patterns
│
├─ Appendix-G-Modeling-Playbook.md ⭐ NEW v1.3
│   ├─ Invariant classification
│   ├─ Entity/Relationship/View checklists
│   ├─ Authority migration guidance
│   └─ Common patterns
│
└─ Appendix-H-Worked-Banking-Example.md ⭐ NEW v1.3
    ├─ Problem statement
    ├─ Naive vs correct model
    ├─ Complete specification
    └─ Multi-region behavior
```

Legend:
- ⭐ NORMATIVE: Must be followed exactly
- ⭐ CRITICAL: Essential for implementation
- ⭐ AUTHORITATIVE: Single source of truth

---

## READING PATHS

### Path 1: First-Time Learner

```
1. README.md (overview)
   ↓
2. 04-design-rationale.md (why)
   ↓
3. 02-grammar-examples.md (what - examples 1-15)
   ↓
4. 01-grammar-reference.md (reference as needed)
   ↓
5. 09-quick-reference.md (patterns and templates)
```

**Time**: 4-6 hours
**Outcome**: Can write basic specifications

---

### Path 2: Specification Author

```
1. 02-grammar-examples.md (patterns)
   ↓
2. 01-grammar-reference.md (formal rules)
   ↓
3. 07-specification-addendums.md (critical details)
   ↓
4. 05-formal-grammar-ebnf.md (when validating)
   ↓
5. 09-quick-reference.md (templates)
```

**Time**: 6-8 hours
**Outcome**: Can write complex, valid specifications

---

### Path 3: Runtime Implementer

```
1. README.md (context)
   ↓
2. 03-implementation-specification.md (architecture)
   ↓
3. 06-technical-implementation-details.md (deep dive)
   ↓
4. 07-specification-addendums.md (critical nuances)
   ↓
5. 01-grammar-reference.md (validation rules)
   ↓
6. 02-grammar-examples.md (test cases)
```

**Time**: 12-16 hours
**Outcome**: Can build conformant runtime

---

### Path 4: Architect/Tech Lead

```
1. 04-design-rationale.md (full document)
   ↓
2. 08-consolidated-specification.md (overview)
   ↓
3. 01-grammar-reference.md (principles)
   ↓
4. 03-implementation-specification.md (architecture)
   ↓
5. 06-technical-implementation-details.md (patterns)
```

**Time**: 8-12 hours
**Outcome**: Can make informed architectural decisions

---

### Path 5: Quick Reference Only

```
1. 09-quick-reference.md (all sections)
   ↓
2. 02-grammar-examples.md (examples as needed)
   ↓
3. 01-grammar-reference.md (lookup specific rules)
```

**Time**: 2-3 hours
**Outcome**: Can solve specific problems quickly

---

### Path 6: Multi-Region Implementer (NEW in v1.3)

```
1. 04-design-rationale.md (v1.3 decisions)
   ↓
2. 01-grammar-reference.md (Authority & Availability)
   ↓
3. 03-implementation-specification.md (Parts XV, XVI)
   ↓
4. 06-technical-implementation-details.md (Parts XVI-XVIII)
   ↓
5. 07-specification-addendums.md (Addendums O-T)
   ↓
6. 02-grammar-examples.md (Examples 46-55)
```

**Time**: 10-14 hours
**Outcome**: Can implement multi-region authority and survivability

---

### Path 7: Data Modeler - Invariant Focus (NEW in v1.3)

```
1. 09-quick-reference.md (Invariant decision trees)
   ↓
2. Appendix G: Modeling Playbook
   ↓
3. Appendix H: Worked Banking Example
   ↓
4. 02-grammar-examples.md (Examples 51-53)
   ↓
5. 01-grammar-reference.md (Relationships, Invariants)
```

**Time**: 4-6 hours
**Outcome**: Can design invariant-oriented data models

---

## DOCUMENT RELATIONSHIPS

### Grammar Flow
```
05-formal-grammar-ebnf.md (EBNF)
         ↓ (interpreted as)
01-grammar-reference.md (structured)
         ↓ (examples of)
02-grammar-examples.md (concrete)
```

### Implementation Flow
```
01-grammar-reference.md (what to implement)
         ↓
07-specification-addendums.md (critical details)
         ↓
03-implementation-specification.md (how to implement)
         ↓
06-technical-implementation-details.md (deep details)
```

### Decision Flow
```
04-design-rationale.md (why this way)
         ↓
01-grammar-reference.md (what we decided)
         ↓
03-implementation-specification.md (how to realize)
```

### Reference Flow
```
09-quick-reference.md (quick lookup)
         ↓ (points to)
08-consolidated-specification.md (complete)
         ↓ (detailed in)
[specific documents]
```

---

## DOCUMENT PURPOSES

| Document | Primary Purpose | Use When |
|----------|----------------|----------|
| README | Orientation | Starting out |
| 01-Grammar Reference | Formal definitions | Writing specs, validating |
| 02-Grammar Examples | Patterns | Learning, copying patterns |
| 03-Implementation Spec | Architecture | Building runtimes |
| 04-Design Rationale | Understanding | Making decisions |
| 05-EBNF Grammar | Parser building | Tooling, validation |
| 06-Technical Details | Deep patterns | Optimizing, debugging |
| 07-Addendums | Critical gaps | Ensuring completeness |
| 08-Consolidated | Single reference | Implementation authority |
| 09-Quick Reference | Fast lookup | Day-to-day work |

---

## CROSS-REFERENCE INDEX

### Find Topic Across Documents

#### Data Modeling
- Grammar: 01 (Part I)
- EBNF: 05 (Data Grammar)
- Examples: 02 (Examples 1-7)
- Implementation: 03 (Part VI)
- Details: 06 (Part VI)
- Rationale: 04 (Decision: Constraints)

#### Work Unit Contracts (NEW in v1.2)
- Grammar: 01 (Part IV)
- EBNF: 05 (Work Unit Contract Grammar)
- Examples: 02 (Examples 41-45)
- Implementation: 03 (TBD)
- Details: 06 (TBD)
- Rationale: 04 (Decision: Typed Contracts)

#### Flows (SMS)
- Grammar: 01 (Part V)
- EBNF: 05 (SMS Grammar)
- Examples: 02 (Examples 12-15)
- Implementation: 03 (Part VII)
- Details: 06 (Part VII)
- Rationale: 04 (Decision: Logical Graph)

#### Workers & Topology
- Grammar: 01 (Part V)
- EBNF: 05 (WTS Grammar)
- Examples: 02 (Examples 16-21)
- Implementation: 03 (Part II)
- Details: 06 (Part IV)
- Rationale: 04 (Decision: External Scheduler)

#### Policy & Authorization
- Grammar: 01 (Part VI)
- EBNF: 05 (Policy Grammar)
- Examples: 02 (Examples 22-27)
- Implementation: 03 (Part V)
- Details: 06 (Part II)
- Rationale: 04 (Decision: Local Enforcement)

#### Signals & Scheduling
- Grammar: 01 (Part VII)
- EBNF: 05 (Signals Grammar)
- Examples: 02 (Examples 28-33)
- Implementation: 03 (Part VIII)
- Details: 06 (Part VIII)
- Addendums: 07 (Addendum - Signals)
- Rationale: 04 (Decision: Signals Not Commands)

#### NATS Integration
- Grammar: N/A (implementation concern)
- Implementation: 03 (Part IV)
- Details: 06 (Part I) ⭐
- Rationale: 04 (Decision: NATS Substrate)

#### Idempotency
- Grammar: 01 (Cross-Cutting)
- EBNF: 05 (Idempotency)
- Addendums: 07 (Addendum B) ⭐
- Details: 06 (Part XV)

#### Multi-Region
- Grammar: 01 (Part V, VII)
- Examples: 02 (Example 40)
- Implementation: 03 (Part X)
- Details: 06 (Part X)
- Rationale: 04 (Decision: Hierarchical)

#### Entity Authority (NEW in v1.3)
- Grammar: 01 (Part I - Authority)
- EBNF: 05 (Authority & Mutability Grammar)
- Examples: 02 (Examples 46-50, 54)
- Implementation: 03 (Part XV)
- Details: 06 (Part XVI)
- Addendums: 07 (Addendums O-P)
- Rationale: 04 (v1.3 Decisions)

#### Relationships (NEW in v1.3)
- Grammar: 01 (Part I - Relationship)
- EBNF: 05 (Relationship Grammar)
- Examples: 02 (Examples 51, 53)
- Rationale: 04 (v1.3 Decisions)

#### Invariant Modeling (NEW in v1.3)
- Grammar: 01 (Part I - Invariant Policy)
- EBNF: 05 (Invariant Grammar)
- Examples: 02 (Examples 52-53)
- Addendums: 07 (Formal Invariants)
- Appendix: F, G, H
- Rationale: 04 (v1.3 Decisions)

#### Storage Roles (NEW in v1.3)
- Grammar: 01 (Part I - Storage Role)
- EBNF: 05 (Storage Role Grammar)
- Examples: 02 (Example 49)
- Implementation: 03 (Part XV)
- Details: 06 (Part XVII)
- Addendums: 07 (Addendum Q)

---

## SPECIFICATION VERSION HISTORY

### v1.3 (Current)
- Added Entity-Scoped Authority
- Added Authority Transitions and CAS Tokens
- Added Storage Role Distinction
- Added Relationship Grammar with Semantics
- Added Invariant-Oriented Modeling
- Added Multi-Region Survivability Patterns

**Breaking Changes**: None (backward compatible)

**New Features**:
- Entity-scoped write authority with epoch management
- Authority transition protocol (pause-and-cutover)
- CAS token structure with epoch validation
- Storage roles (control vs data)
- Relationship grammar with cardinality and causality
- Invariant policy declaration
- View authority independence
- 8 formal invariants for authority correctness
- Appendices F, G, H for modeling guidance

### v1.2
- Added WorkUnitContract grammar
- Typed work unit specifications
- Contract coverage validation
- Enhanced error handling with categories
- Side effect documentation

**Breaking Changes**: None (backward compatible)

**New Features**:
- WorkUnitContract declaration
- Input/output schema specifications
- Side effect types and documentation
- Error definitions with backoff strategies
- Dependency specifications
- Contract coverage reporting
- Precondition and postcondition expressions

### v1.1
- Added signals and scheduling
- Added policy lifecycle
- Added specification addendums
- Clarified transport semantics
- Enhanced multi-region support

**New Features**:
- Signal grammar
- Scheduler configuration
- Policy artifacts with lifecycle
- Realm isolation (minimal)
- Error classification
- Idempotency formalization
- Transition as first-class concept

### v1.0 (Initial)
- Core grammar (Data, UI, Input, SMS, WTS, Policy)
- Basic execution semantics
- Version compatibility model
- Zero-downtime evolution

---

## CRITICAL RULES SUMMARY

### The 12 Commandments of Spec

1. **Version Everything**: Data, policies, UI, workers, inputs
2. **Intent Not Mechanism**: Describe what, not how
3. **Local Over Global**: Avoid coordination
4. **Explicit Over Implicit**: Make coupling visible
5. **Fail Closed**: Deny by default
6. **Idempotent Always**: All transitions must be idempotent
7. **Shadow First**: Test with real traffic
8. **Boundaries Clear**: Atomic groups single boundary
9. **Signals Not Commands**: Observe, then decide
10. **Evolution Linear**: No version skipping
11. **Authority is Entity-Scoped**: Write ownership per entity, not per model (NEW v1.3)
12. **Invariants Live in Views**: Cross-entity invariants enforced by derived views (NEW v1.3)

### The 5 Most Common Mistakes

1. ❌ Skipping shadow phase → **Use shadow for all production changes**
2. ❌ Crossing boundaries in atomic groups → **Keep atomic groups local**
3. ❌ Undefined idempotency → **Always specify idempotency keys**
4. ❌ Hardcoding transport → **Use intent-based routing**
5. ❌ Missing versions → **Version everything from day one**

---

## FEATURE COVERAGE

### What's Specified

| Feature | Grammar | Examples | Implementation | Details |
|---------|---------|----------|----------------|---------|
| Data Types | ✅ | ✅ | ✅ | ✅ |
| Data Evolution | ✅ | ✅ | ✅ | ✅ |
| Constraints | ✅ | ✅ | ✅ | ✅ |
| Materialization | ✅ | ✅ | ✅ | ✅ |
| UI Binding | ✅ | ✅ | ✅ | ⚠️ |
| Input Intents | ✅ | ✅ | ✅ | ✅ |
| Work Unit Contracts | ✅ | ✅ | ⚠️ | ⚠️ |
| SMS Flows | ✅ | ✅ | ✅ | ✅ |
| Atomic Groups | ✅ | ✅ | ✅ | ✅ |
| Compensation | ✅ | ✅ | ✅ | ✅ |
| Workers | ✅ | ✅ | ✅ | ✅ |
| Topology | ✅ | ✅ | ✅ | ✅ |
| Scaling | ✅ | ✅ | ✅ | ✅ |
| Policies | ✅ | ✅ | ✅ | ✅ |
| RBAC/ABAC | ✅ | ✅ | ✅ | ✅ |
| Signals | ✅ | ✅ | ✅ | ✅ |
| Scheduling | ✅ | ✅ | ✅ | ✅ |
| Idempotency | ✅ | ✅ | ✅ | ✅ |
| Correlation | ✅ | ✅ | ✅ | ✅ |
| Multi-Region | ✅ | ✅ | ✅ | ✅ |
| Realm Isolation | ✅ | ⚠️ | ⚠️ | ⚠️ |
| **Entity Authority** | ✅ | ✅ | ✅ | ✅ | *(NEW v1.3)*
| **Relationships** | ✅ | ✅ | ⚠️ | ⚠️ | *(NEW v1.3)*
| **Storage Roles** | ✅ | ✅ | ✅ | ✅ | *(NEW v1.3)*
| **Invariant Modeling** | ✅ | ✅ | ⚠️ | ⚠️ | *(NEW v1.3)*
| **CAS Tokens** | ✅ | ✅ | ✅ | ✅ | *(NEW v1.3)*

Legend:
- ✅ Complete coverage
- ⚠️ Partial coverage (intentional for v1.3)
- ❌ Not covered

---

## DOCUMENT DEPENDENCIES

### Reading Order by Goal

#### Goal: Write First Specification
```
Depends On:
1. README.md (required)
2. 02-grammar-examples.md (required)
3. 01-grammar-reference.md (reference)
4. 09-quick-reference.md (templates)

Output: Valid basic specification
Time: 2-4 hours
```

#### Goal: Build Runtime
```
Depends On:
1. README.md (context)
2. 08-consolidated-specification.md (authority)
3. 03-implementation-specification.md (required)
4. 06-technical-implementation-details.md (required)
5. 07-specification-addendums.md (required)
6. 01-grammar-reference.md (validation)

Output: Conformant runtime implementation
Time: 2-4 weeks
```

#### Goal: Understand Architecture
```
Depends On:
1. README.md (overview)
2. 04-design-rationale.md (required)
3. 01-grammar-reference.md (principles)
4. 03-implementation-specification.md (patterns)

Output: Architectural understanding
Time: 6-8 hours
```

#### Goal: Build Tools (Parser/Generator)
```
Depends On:
1. 05-formal-grammar-ebnf.md (required)
2. 01-grammar-reference.md (required)
3. 07-specification-addendums.md (required)
4. 02-grammar-examples.md (test cases)

Output: Conformant tooling
Time: 1-2 weeks
```

---

## QUICK TOPIC LOOKUP

### By Component

| Component | Primary Doc | Supporting Docs |
|-----------|-------------|-----------------|
| Control Plane | 03, 06 | 04, 08 |
| Workers | 03, 06 | 01, 02, 07 |
| Flow Engine | 03, 06 | 01, 02, 08 |
| Policy Engine | 03, 06 | 01, 02, 07 |
| Scheduler | 03, 06 | 01, 07, 08 |
| UI Runtime | 03 | 01, 02 |

### By Concern

| Concern | Primary Doc | Supporting Docs |
|---------|-------------|-----------------|
| Grammar | 01, 05 | 02, 08 |
| Examples | 02 | 01, 09 |
| Architecture | 03 | 04, 06, 08 |
| NATS Integration | 06 (Part I) | 03 (Part IV) |
| Performance | 06 (Part XI) | 03, 09 |
| Security | 03 (Part V), 06 (Part II) | 01 (Part VI), 07 |
| Testing | 03 (Part XI), 06 (Part XIII) | 02 (Anti-patterns) |
| Operations | 06 (Part XIV) | 03, 09 |
| Multi-Region | 03 (Part X), 06 (Part X) | 01, 07 |

---

## CRITICAL SECTIONS

### Must-Read Sections

These sections are critical for correct implementation:

1. **Grammar Reference - Normative Principles** (01)
   - Defines what MUST be true
   - Everything else builds on this

2. **Implementation Spec - Part I: Architectural Principles** (03)
   - Core separations (control/data plane, scheduling/execution)
   - Authority model

3. **Technical Details - Part I: NATS Integration** (06)
   - NATS role clarification
   - Subject design
   - What NATS is/isn't used for

4. **Addendums - All Addendums** (07)
   - Fills critical semantic gaps
   - Easy to miss, causes bugs if skipped

5. **Consolidated Spec - Execution Semantics** (08)
   - How flows execute
   - Atomic group behavior
   - Failure handling

### Optional Sections

These enhance understanding but can be skipped initially:

- Design Rationale (all) - Read when questioning decisions
- Examples (beyond first 20) - Reference as needed
- Quick Reference (templates) - Use when building
- Technical Details (optimization sections) - Read when optimizing

---

## CONFORMANCE QUICK CHECK

### Grammar Conformance (30 minutes)

```bash
✓ Parse example specifications
✓ Reject invalid specifications
✓ Validate cross-references
✓ Check semantic rules
```

**Pass Criteria**: All examples parse, invalid examples rejected

### Execution Conformance (2 hours)

```bash
✓ Execute simple flow end-to-end
✓ Validate constraints
✓ Enforce atomic groups
✓ Handle failures correctly
```

**Pass Criteria**: Flow completes with correct output

### Distribution Conformance (4 hours)

```bash
✓ Load artifacts from NATS
✓ Register worker
✓ Emit heartbeats
✓ Route work correctly
```

**Pass Criteria**: Multi-worker system operates

### Production Conformance (2 weeks)

```bash
✓ Zero-downtime deployment
✓ Multi-region operation
✓ Chaos testing passes
✓ Performance SLAs met
```

**Pass Criteria**: Production-ready system

---

## USEFUL PATTERNS INDEX

### By Use Case

| Use Case | Example # | Document |
|----------|-----------|----------|
| Simple CRUD | 1, 10, 12 | 02 |
| Data evolution | 2, 3, 5 | 02 |
| Cross-boundary flow | 13, 14 | 02 |
| Policy rollout | 26, 27 | 02 |
| Multi-region | 40 | 02 |
| Atomic operations | 13 | 02 |
| Compensation | 14 | 02 |
| Human approval | - | 06 (Part XII) |
| Streaming data | - | 06 (Part XII) |
| Version migration | 15 | 02 |

---

## SPECIFICATION EVOLUTION

### Proposing Changes

1. **Identify gap** or improvement
2. **Check Design Rationale** for why current design
3. **Propose RFC** with:
   - Problem statement
   - Proposed grammar changes
   - Backward compatibility analysis
   - Implementation impact
   - Examples
4. **Community review**
5. **Merge as v1.x or v2.0**

### Backward Compatibility Rules

**Minor Version (v1.x)**:
- Add new grammar elements
- Add new optional fields
- Add new validation rules (warnings only)
- Clarify ambiguities

**Major Version (v2.x)**:
- Remove grammar elements
- Change semantics
- Rename concepts
- Breaking changes to validation

---

## COMMON QUESTIONS QUICK ANSWERS

| Question | Quick Answer | Full Answer |
|----------|--------------|-------------|
| Do I need NATS? | No, but recommended | 04 (NATS decision) |
| Can I skip versioning? | No, mandatory | 04 (Versioning decision) |
| Is this only for microservices? | No, works for any architecture | README, 04 |
| How do I test policies? | Shadow phase | 07 (Addendum), 06 |
| What about transactions? | Use atomic groups or compensation | 01 (Part IV), 04 |
| Can policies change at runtime? | Yes, via lifecycle | 01 (Part VI), 03 (Part V) |
| How does scaling work? | External scheduler + signals | 01 (Part VII), 06 (Part VIII) |
| What's minimum implementation? | Single binary, no NATS | 03 (Part XIV) |
| How to handle large data? | Object storage references | 07 (Addendum N) |
| Multi-tenant support? | Realm isolation | 01, 07 (Addendum E) |

---

## TROUBLESHOOTING INDEX

| Symptom | Quick Check | Full Guide |
|---------|-------------|------------|
| High latency | Cache hit rate? | 09 (Troubleshooting) |
| Duplicates | Idempotency keys? | 09, 07 (Addendum B) |
| Policy not working | Lifecycle state? | 09, 03 (Part V) |
| Worker not found | KV registration? | 09, 06 (Part I) |
| Deploy failed | Shadow phase? | 09, 06 (Part XIV) |
| Validation error | Error table | 09 (Error Quick Ref) |

---

## IMPLEMENTATION MILESTONES

### Milestone 1: Parse & Validate (Week 1)
**Documents**: 01, 05, 07
**Deliverable**: Validator tool
**Success**: All examples parse correctly

### Milestone 2: Execute Locally (Week 2)
**Documents**: 03, 06, 08
**Deliverable**: Single-binary demo
**Success**: Flow executes end-to-end

### Milestone 3: Distribute (Week 3-4)
**Documents**: 03 (Part IV), 06 (Part I)
**Deliverable**: Multi-worker system
**Success**: Workers communicate via NATS

### Milestone 4: Enforce Policy (Week 5-6)
**Documents**: 03 (Part V), 06 (Part II)
**Deliverable**: Secure system
**Success**: Authorization works

### Milestone 5: Production Deploy (Week 7-12)
**Documents**: All
**Deliverable**: Production system
**Success**: Passes conformance level 3

---

## METRIC SUMMARY

### Key Performance Indicators

| Metric | Target | Source |
|--------|--------|--------|
| Local invocation | <50μs | 06 |
| NATS invocation | <5ms | 06 |
| Policy eval | <100μs | 06 |
| Cache lookup | <10μs | 06 |
| Worker throughput | 100-1000/sec | 09 |
| Flow throughput | 1000+/sec | 09 |
| Signal latency | <10ms | 06 |
| Policy propagation | <1s | 03 |
| TTL expiry | 15s | 06 |
| Heartbeat frequency | 5s | 07 |

### Key Quality Indicators

| Metric | Target | Source |
|--------|--------|--------|
| Zero downtime deploys | 100% | 06 |
| Idempotency correctness | 100% | 07 |
| Policy enforcement | 100% | 03 |
| Chaos test pass rate | 100% | 06 |
| Version compatibility | 100% | 02 |

---

## SPECIFICATION COMPLETENESS

### Coverage Matrix

| Layer | Grammar | Examples | Implementation | Details | Addendums |
|-------|---------|----------|----------------|---------|-----------|
| Data | ✅ | ✅ | ✅ | ✅ | ✅ |
| UI | ✅ | ✅ | ✅ | ⚠️ | - |
| Input | ✅ | ✅ | ✅ | ⚠️ | - |
| SMS | ✅ | ✅ | ✅ | ✅ | ✅ |
| WTS | ✅ | ✅ | ✅ | ✅ | ✅ |
| Policy | ✅ | ✅ | ✅ | ✅ | ✅ |
| Signals | ✅ | ✅ | ✅ | ✅ | ✅ |
| Execution | ✅ | ✅ | ✅ | ✅ | ✅ |
| Multi-Region | ✅ | ✅ | ✅ | ✅ | ✅ |
| Observability | ✅ | ⚠️ | ✅ | ✅ | ✅ |

Legend:
- ✅ Complete
- ⚠️ Partial (sufficient for v1.1)
- - Not applicable

---

## FINAL CHECKLIST

### Before Implementation

- [ ] Read README completely
- [ ] Understand design rationale (at least key decisions)
- [ ] Review grammar examples (first 20 minimum)
- [ ] Check critical sections list above
- [ ] Review addendums (all are critical)

### During Implementation

- [ ] Reference consolidated spec as authority
- [ ] Use quick reference for patterns
- [ ] Validate against grammar reference
- [ ] Test with examples
- [ ] Check technical details for optimizations

### Before Production

- [ ] All conformance tests pass
- [ ] Chaos testing complete
- [ ] Performance targets met
- [ ] Security audit passed
- [ ] Documentation written
- [ ] Runbooks created

---

## GETTING HELP

### Document to Check First

| Question Type | Check Here First |
|---------------|------------------|
| "How do I...?" | 09 (Quick Ref), then 02 (Examples) |
| "Why does...?" | 04 (Rationale) |
| "What does...?" | 01 (Grammar), then 08 (Consolidated) |
| "Is this valid...?" | 01 (Validation Rules), 05 (EBNF) |
| "How do I implement...?" | 03 (Implementation), 06 (Details) |
| "What's the pattern for...?" | 02 (Examples), 09 (Templates) |

### Document Order for Deep Dive

```
Topic: {X}

1. Quick Reference (09) → Overview + pattern
2. Examples (02) → Concrete usage
3. Grammar (01) → Formal definition  
4. Rationale (04) → Why this way
5. Implementation (03) → How to build
6. Details (06) → Optimization
7. Consolidated (08) → Complete picture
```

---

## SPECIFICATION STATUS

**Version**: 1.3
**Status**: Production Ready
**Completeness**: 98% (v1.3 scope)
**Stability**: High (grammar frozen for v1.x)
**Implementation**: Reference runtime available

**v1.3 Additions**:
- Entity-scoped authority with epoch-based migration
- Authority transition protocol (pause-and-cutover)
- CAS token structure for write safety
- Storage role distinction (control vs data)
- Relationship grammar with cardinality and semantics
- Invariant-oriented modeling guidance
- View authority independence
- 8 formal invariants for authority correctness
- Appendices F, G, H for modeling patterns

**Next Version (v1.4)** may include:
- Enhanced realm isolation
- Advanced failure patterns
- Cost modeling
- Energy efficiency
- Edge-specific patterns

**Contributing**: RFCs welcome for v2.0 breaking changes

---

**Use This Index** to navigate the specification efficiently and find what you need quickly.

**All documents** are versioned together and maintained in sync.

**Last Updated**: 2026-01-03

