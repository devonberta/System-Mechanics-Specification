# System Mechanics Specification - Reference Documentation

## Overview

This directory contains comprehensive reference documentation for the System Mechanics Specification v1.3. The specification defines a universal, intent-first framework for describing, executing, and evolving distributed systems.

### What's New in v1.3

- **Entity-Scoped Authority**: Write authority resolved at entity level for granular failure isolation
- **Authority Transitions**: Safe, governed transfer of write authority between regions
- **Storage Roles**: Explicit distinction between control state and data state
- **Relationship Grammar**: Semantic entity linkage with cardinality and causality
- **Invariant-Oriented Modeling**: Cross-entity invariants enforced by views, not entities
- **CAS Token Semantics**: Linearizable writes across authority migrations
- **Multi-Region Survivability**: Formal failure semantics and recovery patterns

---

## Document Structure

### 1. Grammar Reference (`01-grammar-reference.md`)
**Purpose**: Formal grammar definitions for all specification elements

**Contents**:
- Complete EBNF grammar
- All first-class concepts (Data, UI, Input, SMS, WTS, Policy, Signals)
- Validation rules
- Conformance requirements
- Normative principles

**Use This When**:
- Writing system specifications
- Building parsers/validators
- Understanding formal semantics
- Ensuring spec compliance

**Audience**: 
- Specification authors
- Tool builders
- Language implementers
- Compliance validators

---

### 2. Grammar Examples (`02-grammar-examples.md`)
**Purpose**: Concrete examples demonstrating all grammar elements

**Contents**:
- 55+ complete examples
- Simple to complex progressions
- Real-world use cases
- Anti-patterns to avoid
- End-to-end system examples
- Authority and invariant patterns (Examples 46-55, NEW v1.3)

**Use This When**:
- Learning the grammar
- Writing your first specification
- Understanding patterns
- Troubleshooting invalid specs

**Audience**:
- Developers new to the spec
- System architects
- Technical writers
- Training developers

---

### 3. Implementation Specification (`03-implementation-specification.md`)
**Purpose**: Detailed implementation guidance for building conformant systems

**Contents**:
- Runtime architecture patterns
- NATS integration details
- Control plane design
- Policy enforcement patterns
- Flow execution implementation
- Multi-region patterns
- Testing strategies
- Operational patterns
- Code generation patterns

**Use This When**:
- Building a runtime implementation
- Integrating with NATS
- Designing control plane
- Implementing policy enforcement
- Planning deployment architecture

**Audience**:
- Runtime engineers
- Platform builders
- DevOps teams
- Implementation architects

---

### 4. Design Rationale (`04-design-rationale.md`)
**Purpose**: Explains WHY design decisions were made

**Contents**:
- Core design decisions with reasoning
- Architectural tradeoffs
- Alternatives considered and rejected
- Lessons learned
- Anti-patterns prevented
- Tradeoff analysis

**Use This When**:
- Understanding the "why" behind decisions
- Making architectural choices
- Evaluating tradeoffs
- Learning from design process
- Justifying choices to stakeholders

**Audience**:
- Architects
- Technical leadership
- Design reviewers
- Anyone questioning "why this way?"

---

## How to Use This Documentation

### For New Users

**Start Here**:
1. Read Design Rationale (Section: "Core Design Decisions")
2. Skim Grammar Reference (Section: "Core Principles")
3. Work through Grammar Examples (Examples 1-15)
4. Reference Grammar Reference as needed

**Learning Path**:
```
Design Rationale (why) 
  ↓
Grammar Examples (what)
  ↓
Grammar Reference (how)
  ↓
Implementation Spec (build)
```

---

### For Specification Authors

**Your Workflow**:
1. Reference Grammar Examples for patterns
2. Consult Grammar Reference for formal rules
3. Validate against grammar validation rules
4. Cross-reference Design Rationale for edge cases

**Key Sections**:
- Grammar Reference: All grammar definitions
- Grammar Examples: Pattern catalog
- Design Rationale: Lessons learned section

---

### For Runtime Implementers

**Your Workflow**:
1. Read Implementation Spec (Part I: Architectural Principles)
2. Study relevant implementation sections
3. Reference Grammar Reference for correctness
4. Use Grammar Examples for test cases

**Key Sections**:
- Implementation Spec: All parts
- Grammar Reference: Validation rules
- Grammar Examples: Test case source
- Design Rationale: Tradeoff analysis

---

### For System Architects

**Your Workflow**:
1. Read Design Rationale (full document)
2. Study Grammar Reference (Core Principles)
3. Review Grammar Examples (Complex end-to-end examples)
4. Consult Implementation Spec (Part II: Runtime Architecture)

**Key Sections**:
- Design Rationale: All decisions
- Grammar Reference: Normative principles
- Implementation Spec: Architecture patterns
- Grammar Examples: Real-world systems

---

## Quick Reference Guide

### Finding Specific Topics

#### Data Modeling
- Grammar: Part I (Data Grammar)
- Examples: Examples 1-7
- Implementation: Part VI (Data & State Management)
- Rationale: Decision "Constraints Separate from Structure"

#### UI/Presentation
- Grammar: Part II (Presentation Grammar)
- Examples: Examples 8-9
- Implementation: Part II (UI Runtime)
- Rationale: Decision "UI and Input Bind to Data"

#### Input/Forms
- Grammar: Part III (Input Grammar)
- Examples: Examples 10-11
- Implementation: Part II (UI Runtime)
- Rationale: Decision "InputIntent as Inverse Data Intent"

#### Flows (SMS)
- Grammar: Part IV (SMS Grammar)
- Examples: Examples 12-15
- Implementation: Part VII (Flow Execution)
- Rationale: Decision "Flow as Logical Graph"

#### Worker Topology
- Grammar: Part V (WTS Grammar)
- Examples: Examples 16-21
- Implementation: Part II (Worker Runtime)
- Rationale: Decision "External Scheduler, Not Embedded"

#### Policy & Authorization
- Grammar: Part VI (Policy Grammar)
- Examples: Examples 22-27
- Implementation: Part V (Policy Enforcement)
- Rationale: Decision "Distributed Policy, Local Enforcement"

#### Signals & Scheduling
- Grammar: Part VII (Signals Grammar)
- Examples: Examples 28-33
- Implementation: Part VIII (Signal & Scheduling)
- Rationale: Decision "Signals, Not Commands"

#### Multi-Region
- Grammar: Part V (WTS), Part VII (Signals)
- Examples: Example 40
- Implementation: Part X (Multi-Region Patterns)
- Rationale: Decision "Hierarchical Scheduler Model"

#### NATS Integration
- Grammar: N/A (implementation concern)
- Examples: N/A (implementation concern)
- Implementation: Part IV (NATS Integration)
- Rationale: Decision "NATS as Coordination Substrate"

---

## Key Concepts Explained

### Intent vs Mechanism

**Intent**: What must be true (grammar)
**Mechanism**: How it's achieved (runtime)

Example:
- Intent: "Atomic groups must execute together"
- Mechanism: "Use unix socket for co-located workers"

The grammar specifies intent. The runtime chooses mechanism.

---

### Versioning Philosophy

Everything is versioned:
- Data types
- Policies
- UI components
- Input forms
- Workers

Why: Enables zero-downtime evolution through gradual migration.

See: Grammar Reference (Part I), Design Rationale (Decision: "Versioning Everything")

---

### Control Plane vs Data Plane

**Control Plane**: Distributes configuration (policies, topology, specs)
**Data Plane**: Executes work (flows, workers, transitions)

Separation ensures:
- Control plane outage doesn't halt execution
- Policy updates are globally consistent
- Scale independently

See: Implementation Spec (Part III), Design Rationale (Decision: "Control Plane Separation")

---

### Local Enforcement

All decisions made locally with cached state:
- Policy evaluation
- Routing decisions
- Constraint validation

Why:
- Performance (no network roundtrip)
- Availability (works during partition)
- Scale (no coordination bottleneck)

See: Implementation Spec (Part V), Design Rationale (Decision: "Local Enforcement")

---

### Shadow Pattern

Test changes in production without impact:
- Shadow policies (evaluate but don't enforce)
- Shadow writes (write to new schema for validation)
- Shadow UI (render but don't show)

Why: Validate with real traffic before committing.

See: Implementation Spec (Part V, Part VI), Design Rationale (Decision: "Shadow Policies for Safe Rollout")

---

## Common Questions

### Q: Do I have to use NATS?

**A**: No. NATS is used in reference implementation for control plane and coordination, but the grammar is transport-agnostic. You could use HTTP, gRPC, Kafka, or custom transports.

See: Design Rationale (Decision: "Transport-Agnostic Execution")

---

### Q: Can I skip versioning?

**A**: No. Versioning is fundamental to the spec's evolution model. Attempting to skip versioning breaks zero-downtime guarantees and safety properties.

See: Design Rationale (Decision: "Versioning Everything")

---

### Q: Is this only for microservices?

**A**: No. The same grammar works for monoliths, modular monoliths, microservices, serverless, edge, and hybrid architectures. The grammar describes intent, not deployment.

See: Design Rationale (Decision: "Intent-First, Runtime-Decided")

---

### Q: How does this relate to service mesh?

**A**: This spec avoids service mesh complexity by using intent-based routing, local policy enforcement, and NATS coordination. No sidecar proxies required.

See: Implementation Spec (Part IV), Design Rationale (Decision: "Locality-First Invocation")

---

### Q: What about transactions?

**A**: The spec uses atomic groups (bounded locality), compensation patterns (sagas), and idempotency instead of distributed transactions. This is intentional.

See: Grammar Reference (Part IV), Design Rationale (Prevented: "Distributed Transactions")

---

### Q: Can policies change at runtime?

**A**: Yes. Policies are distributed via control plane with lifecycle states (shadow/enforce/deprecated/revoked). Changes propagate in sub-second timeframes.

See: Implementation Spec (Part V), Grammar Examples (Example 27)

---

### Q: How do I test this?

**A**: Use shadow patterns for production testing, chaos testing for resilience, and flow replay for determinism. See testing strategies section.

See: Implementation Spec (Part XI), Design Rationale (Decision: "Chaos Testing as First-Class Requirement")

---

### Q: What's the minimum implementation?

**A**: Single binary with in-process execution, no NATS, basic constraint validation. Can evolve to full distributed system incrementally.

See: Implementation Spec (Part XIV: Minimum Viable Implementation)

---

## Conformance Levels

### Level 1: Basic Conformance
- Parse and validate grammar
- Execute flows as FSMs
- Version all artifacts
- Enforce constraints

**Use Case**: Single-binary applications, prototypes

---

### Level 2: Distributed Conformance
- Level 1 +
- Control plane integration
- Worker discovery and routing
- Policy distribution and enforcement
- Signal emission

**Use Case**: Multi-service systems, production deployments

---

### Level 3: Full Conformance
- Level 2 +
- Multi-region support
- Shadow patterns
- Graceful draining
- Chaos resilience
- Zero-downtime evolution

**Use Case**: Large-scale, mission-critical systems

---

## Implementation Checklist

### Phase 1: Core Grammar
- [ ] Parse YAML specifications
- [ ] Validate structural rules
- [ ] Validate semantic rules
- [ ] Generate types from DataType

### Phase 2: Execution
- [ ] Implement flow engine (FSM)
- [ ] Execute work units
- [ ] Enforce atomic groups
- [ ] Validate constraints

### Phase 3: Policy
- [ ] Load policies from control plane
- [ ] Evaluate policies locally
- [ ] Support shadow policies
- [ ] Handle lifecycle transitions

### Phase 4: Distribution
- [ ] Integrate NATS (or alternative)
- [ ] Worker registration/discovery
- [ ] Heartbeat loop
- [ ] Routing cache

### Phase 5: Signals
- [ ] Emit capacity signals
- [ ] Emit latency signals
- [ ] Emit policy signals
- [ ] Integrate with scheduler

### Phase 6: Production
- [ ] Multi-region deployment
- [ ] Graceful draining
- [ ] Observability hooks
- [ ] Chaos testing

---

## Getting Help

### Understanding the Grammar
→ Start with Grammar Examples
→ Reference Grammar Reference for formal rules
→ Check Design Rationale for "why"

### Building an Implementation
→ Study Implementation Spec (all sections)
→ Use Grammar Examples for test cases
→ Review Design Rationale for tradeoffs

### Architectural Decisions
→ Read Design Rationale (full document)
→ Review Grammar Reference (Normative Principles)
→ Study Implementation Spec (Part I: Architectural Principles)

### Debugging Issues
→ Check Grammar Reference (Validation Rules)
→ Review Grammar Examples (Anti-Patterns)
→ Consult Implementation Spec (Part XIII: Anti-Patterns)

---

## Contributing to These Docs

### Reporting Issues
- Unclear explanations
- Missing examples
- Incorrect information
- Outdated patterns

### Suggesting Improvements
- Additional examples
- Clarifications
- Alternative approaches
- Real-world case studies

### Document Versions
All documents are versioned together with the specification.
Current version: v1.3

---

## Document Maintenance

### Change Policy
- Breaking changes: Major version bump
- Clarifications: Minor version bump
- Corrections: Patch version bump

### Review Cycle
- Quarterly review of all documents
- Update based on implementation feedback
- Incorporate lessons learned

---

## License

These documents are part of the System Mechanics Specification project.
See project LICENSE for details.

---

## Acknowledgments

This specification evolved through extensive discussion, iteration, and real-world validation. The design incorporates lessons learned from:
- Distributed systems patterns
- Domain-driven design
- Event-driven architecture
- Policy-based systems
- Zero-downtime deployments

---

## Version History

### v1.3 (Current)
- Added entity-scoped authority with epoch management
- Added authority transition protocol (pause-and-cutover)
- Added CAS token structure with epoch validation
- Added storage roles (control vs data)
- Added relationship grammar with cardinality and causality
- Added invariant policy declaration
- Added view authority independence
- Added 8 formal invariants for authority correctness
- Added Appendices F, G, H for modeling guidance
- Updated all documents with v1.3 content

### v1.2
- Added WorkUnitContract grammar
- Typed work unit specifications
- Contract coverage validation
- Enhanced error handling with categories
- Side effect documentation

### v1.1
- Added signals and scheduling grammar
- Added design rationale document
- Expanded implementation patterns
- Clarified transport semantics

### v1.0
- Initial release
- Core grammar (Data, UI, Input, SMS, WTS, Policy)
- Reference implementation guidance
- Basic examples

---

**Last Updated**: Synchronized with Specification v1.3
**Status**: Production Ready

