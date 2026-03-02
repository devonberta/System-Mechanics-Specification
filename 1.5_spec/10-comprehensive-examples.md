# System Mechanics Specification v1.5 - Comprehensive Examples

## Overview

This document contains complete, production-ready reference implementations demonstrating the full SMS v1.5 specification. Each example showcases real-world applications built using the intent-first philosophy, incorporating all v1.5 addendums and advanced features.

### Purpose

These examples serve as:

1. **Specification Validation** - Prove the grammar can express real-world systems
2. **Implementation Guidance** - Provide concrete targets for runtime development
3. **Learning Resources** - Help developers understand SMS patterns
4. **Test Fixtures** - Enable end-to-end specification parsing tests
5. **Best Practice Showcase** - Demonstrate production-quality specifications

### What's New in v1.5 Examples

All 15 examples have been enhanced to demonstrate:

- **Complete Application Development**: End-to-end experiences from intent to UI
- **Advanced Features**: All 28 v1.5 addendums integrated
- **Production Patterns**: Real-world scenarios with operational considerations
- **Multi-Modal Interfaces**: Web, mobile, voice, and conversational experiences
- **External Integration**: Third-party services, webhooks, and API composition
- **Data Governance**: GDPR, HIPAA, and regulatory compliance patterns
- **Edge Computing**: IoT, offline-first, and spatial semantics

---

## Example Catalog

| # | Example | Industry | Complexity | Primary Features |
|---|---------|----------|------------|------------------|
| 01 | [Enterprise Banking Platform](#example-01-enterprise-banking-platform) | Financial | High | Multi-region, workflows, delegation, compliance |
| 02 | [Telemedicine Platform](#example-02-telemedicine-platform) | Healthcare | High | Consent, scheduling, video, HIPAA compliance |
| 03 | [E-Commerce Marketplace](#example-03-e-commerce-marketplace) | Retail | Medium-High | Search, assets, payments, recommendations |
| 04 | [Logistics & Fleet Management](#example-04-logistics-fleet-management) | Transportation | High | IoT, spatial, real-time, offline mobile |
| 05 | [Customer Service Center](#example-05-customer-service-center) | Cross-Industry | Medium | Chatbot, voice, human tasks, knowledge base |
| 06 | [Corporate HR Platform](#example-06-corporate-hr-platform) | Enterprise | Medium | Graph hierarchy, workflows, GDPR, delegation |
| 07 | [Property Management](#example-07-property-management) | Real Estate | Medium | Spatial, IoT, assets, temporal access |
| 08 | [Learning Management System](#example-08-learning-management-system) | Education | Medium | Rich content, assets, collaboration, offline |
| 09 | [Social Networking Platform](#example-09-social-networking-platform) | Consumer | High | Graph, streaming, ML, governance |
| 10 | [Smart Factory / Industrial IoT](#example-10-smart-factory-industrial-iot) | Manufacturing | High | IoT, streaming, ML, safety, compliance |
| 11 | [Legal Case Management](#example-11-legal-case-management) | Legal | High | Workflows, legal holds, delegation, search |
| 12 | [Voice-First Smart Home](#example-12-voice-first-smart-home) | Consumer IoT | Medium | Voice, IoT, device groups, temporal access |
| 13 | [Subscription Commerce](#example-13-subscription-commerce) | SaaS | Medium | External APIs, webhooks, scheduling, metering |
| 14 | [Collaborative Document Editor](#example-14-collaborative-document-editor) | Productivity | High | Real-time collaboration, rich content, offline |
| 15 | [Government Permitting System](#example-15-government-permitting-system) | Government | High | Durable workflows, spatial, accessibility |

---

## Feature Coverage Matrix

This matrix shows which examples demonstrate each v1.5 feature:

| Feature Area | Addendum | Examples Using |
|--------------|----------|----------------|
| **Core Features (v1.4)** | | |
| Entities, Authority, CAS | v1.4 Core | All |
| Relationships & Cardinality | v1.4 Core | All |
| View Enforcement | v1.4 Core | All |
| Multi-Region Survivability | v1.4 Core | 01, 04, 09, 10 |
| **v1.5 Addendums** | | |
| Form-Intent Binding | X (01) | All |
| View Materialization Contract | Y (02) | All |
| Temporal Windows (Streaming) | Y (02) | 01, 04, 09, 10, 13 |
| Intent Delivery Contract | Z (03) | All |
| Session Contract | AA (04) | All |
| Multi-Device Sync | AA (04) | 01, 03, 04, 08, 09, 14 |
| Offline Support | AA (04) | 04, 06, 08, 14 |
| Indicator Source Binding | AB (05) | 01, 04, 10 |
| Composition Fetch Semantics | AC (06) | All |
| Mask Expression Grammar | AD (07) | 01, 02, 06 |
| Navigation Semantics | AE (08) | All |
| View Availability Policy | AF (09) | 01, 04 |
| Presentation Hints | AG (10) | All |
| Surface/Container Semantics | AH (11) | All |
| Accessibility Preferences | AI (12) | 02, 08, 15 |
| End-to-End Reference Flow | AJ (13) | All (narrative) |
| Asset Semantics | AK (14) | 03, 07, 08, 11, 14 |
| Search Semantics | AL (15) | 03, 05, 06, 09, 11 |
| Scheduled Trigger Semantics | AM (16) | 02, 06, 07, 13, 15 |
| Notification Channel Semantics | AN (17) | 01, 02, 03, 04, 05, 13 |
| Collaborative Session Semantics | AO (18) | 08, 14 |
| Structured Content Type | AP (19) | 03, 05, 08, 14 |
| Inference/Derived Fields | AQ (20) | 03, 09, 10, 13 |
| Spatial Type Semantics | AR (21) | 03, 04, 07, 15 |
| Graph Query Semantics | AS (22) | 04, 06, 09, 10 |
| Durable Workflow Semantics | AT (23) | 01, 02, 05, 06, 11, 15 |
| Delegation/Consent Semantics | AU (24) | 01, 02, 06, 07, 11, 12 |
| Conversational Experience | AV (25) | 05, 12 |
| External Integration Semantics | AW (26) | 01, 02, 03, 04, 06, 13 |
| Data Governance Semantics | AX (27) | 01, 02, 06, 09, 10, 11 |
| Edge Device Semantics | AY (28) | 04, 07, 10, 12 |

---

## Example Structure

Each example follows a consistent structure for clarity and completeness:

### Standard Sections

1. **Overview** - Business context and key capabilities
2. **Architecture Diagram** - High-level component relationships
3. **Data Model** - DataTypes and DataStates with governance
4. **Relationships** - Entity relationships including graph and delegation
5. **Intents** - InputIntents with scheduling and notifications
6. **Workflows** - smsFlows with durable and human task semantics
7. **Views** - Materializations with temporal and retrieval semantics
8. **Experiences** - Complete experience definitions with session and modality
9. **External Integrations** - Dependencies, webhooks, and credentials
10. **Policy & Authorization** - Policies with delegation and consent
11. **Key Specification Patterns** - Notable patterns demonstrated

### Reading Examples

**For Learning**:
- Start with simpler examples: 05 (Customer Service), 08 (LMS), 13 (Subscription)
- Progress to medium complexity: 03 (E-Commerce), 06 (HR), 07 (Property)
- Study advanced patterns: 01 (Banking), 10 (Factory), 14 (Collaboration)

**For Feature-Specific Learning**:
- IoT/Edge: Examples 04, 07, 10, 12
- Workflows: Examples 01, 02, 05, 06, 11, 15
- Governance: Examples 01, 02, 06, 09, 10, 11
- Search/Graph: Examples 03, 04, 05, 06, 09, 11
- Real-Time/Streaming: Examples 01, 04, 09, 10, 13, 14

---

## Using These Examples

### For Specification Validation

All examples contain complete, parseable YAML blocks that should validate against the v1.5 grammar:

```bash
# Parse all examples to validate grammar
sms validate specification/Breakdown/1.5_spec/10-comprehensive-examples.md

# Validate specific example
sms validate --example "01-banking-platform"
```

### For Runtime Implementation

Each example provides:
- Complete YAML specifications ready for runtime ingestion
- Expected runtime behavior descriptions
- Integration points with SMS components
- Performance and scaling considerations
- Error handling patterns

### For Test Development

Examples serve as test fixtures for:
- Parser validation
- Runtime behavior verification
- Policy enforcement testing
- Cross-component integration tests
- End-to-end workflow validation

### For Documentation Generation

Examples demonstrate best practices for:
- Inline documentation
- Semantic naming conventions
- Structural patterns
- Governance annotations
- Operational metadata

---

## Notes on Intent-First Philosophy

All examples maintain SMS core principles:

1. **Intent, Not Implementation** - Specifications declare what, not how
2. **Runtime Decides** - Implementation details determined at runtime
3. **Declarative Authority** - Write authority explicitly scoped to entities
4. **View Enforcement** - Invariants enforced by views, not entities
5. **Governance Embedded** - Compliance built into the specification
6. **Evolution-Friendly** - Changes made without breaking existing systems

---

## Examples Begin

The following sections contain the complete specifications for all 15 reference implementations.

---

## EXAMPLE 01: Enterprise Banking Platform

# Enterprise Banking Platform

**Example ID**: 01  
**Industry**: Financial Services  
**Complexity**: High  
**Key Features**: Multi-region, durable workflows, delegation, compliance, real-time

---

## Overview

A comprehensive retail and commercial banking platform supporting:

- **Account Management** - Checking, savings, loans, credit cards
- **Transfers & Payments** - Internal, external (ACH/wire), scheduled
- **Loan Origination** - Multi-week approval with underwriter review
- **Fraud Detection** - Real-time transaction scoring
- **Multi-Region** - Regional data sovereignty with global access
- **Power of Attorney** - Delegated account access
- **Regulatory Compliance** - SOX, PCI-DSS, data retention

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Banking Platform                              │
├─────────────────────────────────────────────────────────────────┤
│  Experiences                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │
│  │  Customer   │ │   Teller    │ │ Underwriter │                │
│  │   Portal    │ │   Console   │ │  Workbench  │                │
│  └─────────────┘ └─────────────┘ └─────────────┘                │
├─────────────────────────────────────────────────────────────────┤
│  Core Domains                                                    │
│  ┌─────────┐ ┌──────────┐ ┌────────┐ ┌─────────┐ ┌──────────┐  │
│  │ Account │ │ Transfer │ │  Loan  │ │ Payment │ │  Fraud   │  │
│  │ Domain  │ │  Domain  │ │ Domain │ │ Domain  │ │  Domain  │  │
│  └─────────┘ └──────────┘ └────────┘ └─────────┘ └──────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  External Integrations                                           │
│  ┌─────────┐ ┌──────────┐ ┌────────┐ ┌─────────┐               │
│  │   ACH   │ │   Wire   │ │ Credit │ │  Fraud  │               │
│  │ Network │ │ Network  │ │ Bureau │ │ Service │               │
│  └─────────┘ └──────────┘ └────────┘ └─────────┘               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Model

### Account Types

```yaml
dataType:
  name: Account
  version: v1
  description: "Bank account with balance and status"
  
  fields:
    account_id:
      type: string
      constraints:
        required: true
        unique: true
      presentation:
        display_as: identifier
        
    account_number:
      type: string
      constraints:
        required: true
        pattern: "^[0-9]{10,12}$"
      presentation:
        display_as: identifier
      accessibility:
        label:
          template: "Account number ending in {{value | reveal_last(4)}}"
          
    account_type:
      type: enum
      values: [checking, savings, money_market, cd, loan, credit_card]
      presentation:
        display_as: badge
        semantic_variants:
          checking: info
          savings: success
          loan: warning
          credit_card: neutral
          
    balance:
      type: decimal
      precision: 2
      presentation:
        display_as: currency
        currency_code: account.currency
        emphasis: primary
      accessibility:
        label:
          template: "Current balance: {{value | currency}}"
        announce_changes: true
        
    available_balance:
      type: decimal
      precision: 2
      presentation:
        display_as: currency
        
    currency:
      type: string
      constraints:
        pattern: "^[A-Z]{3}$"
        
    status:
      type: enum
      values: [active, frozen, closed, pending_closure]
      presentation:
        display_as: badge
        semantic_variants:
          active: success
          frozen: error
          closed: neutral
          pending_closure: warning
          
    owner_id:
      type: string
      constraints:
        required: true
        
    opened_at:
      type: timestamp
      presentation:
        display_as: date
        
    region:
      type: string
      description: "Primary region for data residency"
      
  governance:
    classification: financial
    retention:
      policy: time_based
      duration: 2555d                    # 7 years
      archive:
        enabled: true
        after: 365d
        tier: cold
      on_expiry: archive
    residency:
      primary_region: "account.region"
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 2555d
      immutable: true
    encryption:
      at_rest: required
      in_transit: required
```

### Transaction Type

```yaml
dataType:
  name: Transaction
  version: v1
  description: "Financial transaction record"
  
  fields:
    transaction_id:
      type: string
      constraints:
        required: true
        unique: true
        
    account_id:
      type: string
      constraints:
        required: true
        
    type:
      type: enum
      values: [deposit, withdrawal, transfer_in, transfer_out, payment, fee, interest]
      
    amount:
      type: decimal
      precision: 2
      presentation:
        display_as: currency
        
    running_balance:
      type: decimal
      precision: 2
      presentation:
        display_as: currency
        
    description:
      type: string
      
    counterparty:
      type: string
      
    timestamp:
      type: timestamp
      presentation:
        display_as: datetime
        
    fraud_score:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [amount, counterparty, timestamp, account_id]
        model:
          name: fraud_detector
          version: v3
        output:
          type: decimal
          range: { min: 0, max: 1 }
          confidence_field: fraud_confidence
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: default
          default_value: 0.5
          
  governance:
    classification: financial
    retention:
      policy: time_based
      duration: 2555d
    audit:
      access_logging: true
      immutable: true
```

### Loan Application Type

```yaml
dataType:
  name: LoanApplication
  version: v1
  description: "Loan application with underwriting data"
  
  fields:
    application_id:
      type: string
      constraints:
        required: true
        
    applicant_id:
      type: string
      constraints:
        required: true
        
    loan_type:
      type: enum
      values: [personal, mortgage, auto, business, heloc]
      
    requested_amount:
      type: decimal
      precision: 2
      presentation:
        display_as: currency
        
    term_months:
      type: integer
      
    status:
      type: enum
      values: [draft, submitted, under_review, approved, denied, funded, cancelled]
      presentation:
        display_as: badge
        semantic_variants:
          draft: neutral
          submitted: info
          under_review: warning
          approved: success
          denied: error
          funded: success
          cancelled: neutral
          
    credit_score:
      type: integer
      
    risk_score:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [requested_amount, credit_score, applicant_id]
        model:
          name: loan_risk_scorer
          version: v2
        output:
          type: decimal
          range: { min: 0, max: 1000 }
          
    underwriter_decision:
      type: enum
      values: [pending, approved, approved_with_conditions, denied]
      
    underwriter_notes:
      type: string
      
    decision_date:
      type: timestamp
      
  search:
    indexed: true
    fields:
      - field: applicant_id
        strategy: exact
      - field: status
        strategy: exact
        facet:
          enabled: true
          type: terms
      - field: loan_type
        strategy: exact
        facet:
          enabled: true
          type: terms
```

---

## Data States

```yaml
dataState:
  name: AccountState
  type: Account
  lifecycle: persistent
  
  authority:
    scope: entity
    entity_key: account_id
    home_region: "account.region"
    
  placement:
    replicas:
      - region: us-east-1
        role: primary
      - region: us-west-2
        role: replica
      - region: eu-west-1
        role: replica

dataState:
  name: TransactionState
  type: Transaction
  lifecycle: persistent
  
  authority:
    scope: entity
    entity_key: transaction_id
    
dataState:
  name: LoanApplicationState
  type: LoanApplication
  lifecycle: persistent
  
  authority:
    scope: entity
    entity_key: application_id

dataState:
  name: LoanApplicationFlowState
  type: LoanApplicationFlowData
  lifecycle: persistent
  description: "Durable workflow state for loan processing"
```

---

## Relationships

### Account Ownership

```yaml
relationship:
  name: Owns_Account
  from: CustomerState
  to: AccountState
  type: ownership
  cardinality: one-to-many
  semantics:
    causal: true
    cascade_delete: false
```

### Power of Attorney Delegation

```yaml
relationship:
  name: Has_Power_Of_Attorney_For
  from: SubjectState
  to: SubjectState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view_accounts, view_transactions, initiate_transfer, manage_beneficiaries]
    constraints:
      transitive: false
      max_depth: 1
      same_realm: true
    requires:
      approval: both
      verification: mfa
    validity:
      type: temporal
      duration: 1825d              # 5 years
    revocation:
      by: either
      cascade: true
```

### Account Beneficiary

```yaml
relationship:
  name: Beneficiary_Of
  from: BeneficiaryState
  to: AccountState
  type: association
  cardinality: many-to-one
```

---

## Intents

### Transfer Intent

```yaml
inputIntent:
  name: InitiateTransferIntent
  version: v1
  description: "Initiate a fund transfer between accounts"
  
  proposal:
    creates: TransferState
    fields:
      from_account_id:
        type: string
        required: true
      to_account_id:
        type: string
        required: true
      amount:
        type: decimal
        required: true
        constraints:
          min: 0.01
      memo:
        type: string
        required: false
        
  delivery:
    guarantee: exactly_once
    acknowledgment: required
    timeout: 30s
    
  response:
    on_success:
      includes: [transfer_id, status, estimated_completion]
    on_error:
      includes: [error_code, error_message]
      
  notification:
    enabled: true
    trigger:
      on: intent_success
    channels:
      - type: push
        priority: 1
      - type: email
        priority: 2
        fallback: true
    recipient:
      type: subject
      subject_field: initiator_id
```

### Loan Application Intent

```yaml
inputIntent:
  name: SubmitLoanApplicationIntent
  version: v1
  description: "Submit a new loan application"
  
  proposal:
    creates: LoanApplicationState
    fields:
      loan_type:
        type: enum
        values: [personal, mortgage, auto, business, heloc]
        required: true
      requested_amount:
        type: decimal
        required: true
      term_months:
        type: integer
        required: true
      purpose:
        type: string
        required: false
        
  delivery:
    guarantee: exactly_once
    
  notification:
    enabled: true
    trigger:
      on: intent_success
    channels:
      - type: email
        email:
          template: loan_application_received
          subject: "Your loan application has been received"
```

### Underwriter Decision Intent

```yaml
inputIntent:
  name: UnderwriterDecisionIntent
  version: v1
  description: "Underwriter decision on loan application"
  
  proposal:
    updates: LoanApplicationState
    fields:
      application_id:
        type: string
        required: true
      decision:
        type: enum
        values: [approved, approved_with_conditions, denied]
        required: true
      conditions:
        type: array
        items:
          type: string
        required: false
      notes:
        type: string
        required: false
        
  delivery:
    guarantee: exactly_once
```

---

## Workflows

### Transfer Flow

```yaml
smsFlow:
  name: TransferFlow
  version: v1
  description: "Process fund transfer with fraud check"
  
  triggeredBy:
    inputIntent: InitiateTransferIntent
    
  steps:
    - work_unit: ValidateTransfer
      input:
        from_account: "intent.from_account_id"
        to_account: "intent.to_account_id"
        amount: "intent.amount"
        
    - work_unit: CheckFraud
      input:
        transaction: "transfer"
      output:
        fraud_score: "fraud_result.score"
        
    - if: "fraud_score > 0.8"
      then:
        - work_unit: FlagForReview
        - signal: TransferFlaggedSignal
      else:
        - atomicGroup:
            - work_unit: DebitAccount
              input:
                account_id: "intent.from_account_id"
                amount: "intent.amount"
            - work_unit: CreditAccount
              input:
                account_id: "intent.to_account_id"
                amount: "intent.amount"
          compensation:
            - work_unit: ReverseDebit
            - work_unit: ReverseCredit
            
    - work_unit: RecordTransaction
    
    - signal: TransferCompletedSignal
```

### Loan Origination Flow (Durable)

```yaml
smsFlow:
  name: LoanOriginationFlow
  version: v1
  description: "Multi-week loan approval with human review"
  
  triggeredBy:
    inputIntent: SubmitLoanApplicationIntent
    
  durability:
    enabled: true
    state: LoanApplicationFlowState
    ttl: 60d
    versioning:
      strategy: run_to_completion
      
  steps:
    - work_unit: ValidateApplication
      input:
        application: "intent"
        
    - checkpoint: post_validation
    
    - work_unit: PullCreditReport
      external_dependency: CreditBureau
      
    - work_unit: CalculateRiskScore
    
    - if: "risk_score < 300"
      then:
        - work_unit: AutoApprove
        - goto: generate_documents
      else:
        - if: "risk_score > 700"
          then:
            - work_unit: AutoDeny
            - goto: send_denial
          else:
            - await:
                name: await_underwriter_review
                description: "Manual underwriter review required"
                trigger:
                  type: inputIntent
                  intent: UnderwriterDecisionIntent
                  assignment:
                    policy: skill_based
                    pool: UnderwriterPool
                sla:
                  target: 24h
                  warning_at: 20h
                escalation:
                  - after: 24h
                    action: notify
                    to: UnderwriterManager
                  - after: 48h
                    action: escalate_pool
                    to: SeniorUnderwriterPool
                timeout: 72h
                on_timeout: escalate
                result:
                  captures: [decision, conditions, notes]
                  
    - checkpoint: post_underwriting
    
    - if: "underwriter_decision == 'approved' OR underwriter_decision == 'approved_with_conditions'"
      then:
        - label: generate_documents
        - work_unit: GenerateLoanDocuments
        
        - await:
            name: await_document_signing
            trigger:
              type: inputIntent
              intent: SignLoanDocumentsIntent
              assignment:
                policy: direct
                subject: "application.applicant_id"
            sla:
              target: 7d
            timeout: 14d
            on_timeout: cancel
            
        - work_unit: FundLoan
        - signal: LoanFundedSignal
        
      else:
        - label: send_denial
        - work_unit: SendDenialNotice
        
  checkpoints:
    - name: post_validation
      after_step: ValidateApplication
    - name: post_underwriting
      after_step: await_underwriter_review
      state_snapshot: full
      
  compensation:
    - from_checkpoint: post_underwriting
      do: ReverseApproval
    - from_checkpoint: post_validation
      do: CancelApplication

taskPool:
  name: UnderwriterPool
  version: v1
  
  membership:
    type: role_based
    roles: [underwriter, senior_underwriter]
    
  capacity:
    max_concurrent_per_member: 10
    
  skills:
    - name: credit_analysis
      required: true
    - name: high_value_loans
      required: false
    - name: commercial_lending
      required: false

taskPool:
  name: SeniorUnderwriterPool
  version: v1
  
  membership:
    type: role_based
    roles: [senior_underwriter]
    
  capacity:
    max_concurrent_per_member: 5
```

---

## Views

### Account Balance View

```yaml
materialization:
  name: AccountBalanceView
  version: v1
  source: AccountState
  targetState: AccountBalanceMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: account_id
    
    freshness:
      max_staleness: 5s
      
    on_unavailable:
      strategy: use_cached
      cache_ttl: 60s
      
  availability:
    read_preference: local_preferred
    staleness_budget: 10s
    on_region_unavailable: fallback_remote
```

### Transaction History View

```yaml
materialization:
  name: TransactionHistoryView
  version: v1
  source: [TransactionState, AccountState]
  targetState: TransactionHistoryMaterialized
  
  retrieval:
    mode: by_query
    query_fields: [account_id, date_range, transaction_type]
    
    list:
      max_items: 100
      default_order: timestamp
      order_direction: desc
      
    freshness:
      max_staleness: 30s
```

### Real-Time Balance Stream

```yaml
materialization:
  name: RealTimeBalanceStream
  version: v1
  source: TransactionState
  targetState: RealTimeBalanceMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: account_id
    
    temporal:
      window:
        type: sliding
        size: 5m
        slide: 10s
        
      aggregation:
        - field: amount
          function: sum
          as: net_change
        - field: transaction_id
          function: count
          as: transaction_count
          
      timestamp_field: timestamp
      watermark: 5s
      
      overflow:
        strategy: drop_oldest
        
      emit:
        trigger: periodic
        periodic_interval: 1s
        include_partial: true
        
    freshness:
      max_staleness: 1s
```

### Loan Pipeline View

```yaml
materialization:
  name: LoanPipelineView
  version: v1
  source: LoanApplicationState
  targetState: LoanPipelineMaterialized
  
  retrieval:
    mode: by_query
    query_fields: [status, underwriter_id, loan_type]
    
    list:
      max_items: 50
      default_order: submitted_at
      order_direction: asc
```

---

## Experiences

### Customer Banking Experience

```yaml
experience:
  name: CustomerBanking
  version: v1
  description: "Retail customer banking portal"
  
  entry_point: CustomerDashboard
  
  includes:
    - CustomerDashboard
    - AccountList
    - AccountDetail
    - TransactionHistory
    - TransferWorkspace
    - LoanApplicationWorkspace
    - ProfileSettings
    
  session:
    required: true
    
    subject_binding:
      mode: authenticated
      
    contains:
      required:
        - subject_id
        - roles
        - realm
      optional:
        - attributes
        - preferences
        - mfa_verified
        
    lifetime:
      mode: sliding
      idle_timeout: 30m
      max_duration: 24h
      
    on_expired:
      action: redirect_to_auth
      preserve_location: true
      
    devices:
      enabled: true
      max_active: 5
      
      identification:
        by: device_id
        
      state_scope:
        per_device:
          - ui.theme
          - ui.last_account_viewed
        shared:
          - pending_transfers
          - notification_preferences
          
      handoff:
        enabled: true
        state_transfer: immediate
        transferable:
          - transfer.in_progress
          
      concurrent_limits:
        on_new_device: allow
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: reject
        
      cache:
        views: [AccountBalanceView, TransactionHistoryView]
        max_size: 10MB
        ttl: 4h
        
  policy_set:
    - CustomerAccountAccessPolicy
    - CustomerTransferPolicy
    
  preferences:
    schema:
      theme:
        type: enum
        values: [light, dark, system]
        default: system
      notifications:
        type: object
        properties:
          transfer_alerts: { type: boolean, default: true }
          statement_ready: { type: boolean, default: true }
          security_alerts: { type: boolean, default: true }
```

### Underwriter Workbench Experience

```yaml
experience:
  name: UnderwriterWorkbench
  version: v1
  description: "Loan underwriter workspace"
  
  entry_point: UnderwriterDashboard
  
  includes:
    - UnderwriterDashboard
    - LoanQueue
    - ApplicationReview
    - RiskAnalysis
    - DecisionForm
    
  session:
    required: true
    
    subject_binding:
      mode: authenticated
      
    contains:
      required:
        - subject_id
        - roles
        - realm
        - mfa_verified
        
    lifetime:
      mode: bounded
      idle_timeout: 15m
      max_duration: 8h
      
    on_expired:
      action: prompt_reauth
      preserve_location: true
      
  policy_set:
    - UnderwriterAccessPolicy
```

---

## External Integrations

### Credit Bureau

```yaml
externalDependency:
  name: CreditBureau
  version: v1
  description: "Credit report and score provider"
  
  type: api
  
  capabilities:
    - pull_credit_report
    - get_credit_score
    
  contract:
    pull_credit_report:
      input:
        schema:
          ssn: { type: string, required: true }
          full_name: { type: string, required: true }
          dob: { type: date, required: true }
      output:
        schema:
          report_id: { type: string }
          score: { type: integer }
          accounts: { type: array }
      errors:
        - code: identity_not_found
          retryable: false
        - code: rate_limited
          retryable: true
          
  sla:
    availability: 99.9%
    latency_p99: 5s
    
  resilience:
    timeout: 10s
    retry:
      enabled: true
      max_attempts: 2
      backoff:
        strategy: exponential
        initial: 500ms
        max: 3s
    circuit_breaker:
      enabled: true
      threshold: 5
      reset_after: 60s
      
  rate_limit:
    requests_per_minute: 100
    
  credentials:
    type: mtls
    secret_ref: credit_bureau_cert
```

### ACH Network

```yaml
externalDependency:
  name: ACHNetwork
  version: v1
  description: "ACH payment processing"
  
  type: api
  
  capabilities:
    - submit_ach_transfer
    - check_transfer_status
    - cancel_transfer
    
  resilience:
    timeout: 30s
    retry:
      enabled: true
      max_attempts: 3
    circuit_breaker:
      enabled: true
      threshold: 10
      reset_after: 120s
      
  fallback:
    on_unavailable: queue
    queue_ttl: 24h
    
  credentials:
    type: api_key
    secret_ref: ach_api_key

webhookReceiver:
  name: ACHStatusWebhook
  version: v1
  description: "Receive ACH transfer status updates"
  
  source: ACHNetwork
  
  maps_to:
    inputIntent: ACHStatusUpdateIntent
    
  authentication:
    type: signature
    signature:
      algorithm: hmac_sha256
      header: X-ACH-Signature
      secret_ref: ach_webhook_secret
      
  event_mapping:
    - external_event: transfer.completed
      intent_type: ACHTransferCompleted
      field_mapping:
        transfer_id: "$.data.transfer_id"
        status: "$.data.status"
        completed_at: "$.data.completed_at"
        
    - external_event: transfer.failed
      intent_type: ACHTransferFailed
      field_mapping:
        transfer_id: "$.data.transfer_id"
        error_code: "$.data.error.code"
        error_message: "$.data.error.message"
        
  idempotency:
    key_path: "$.id"
    ttl: 24h
```

### Fraud Detection Service

```yaml
externalDependency:
  name: FraudDetectionService
  version: v1
  description: "Real-time fraud scoring"
  
  type: api
  
  capabilities:
    - score_transaction
    
  sla:
    availability: 99.99%
    latency_p99: 200ms
    
  resilience:
    timeout: 500ms
    retry:
      enabled: false              # Too slow to retry
    circuit_breaker:
      enabled: true
      threshold: 5
      reset_after: 30s
      
  fallback:
    on_unavailable: degrade
    degrade_to: DefaultFraudScore  # Return 0.5 score
    
  credentials:
    type: api_key
    secret_ref: fraud_service_key
```

---

## Policies

### Customer Account Access

```yaml
policy:
  name: CustomerAccountAccessPolicy
  version: v1
  
  appliesTo:
    type: dataState
    name: AccountState
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Has_Power_Of_Attorney_For
    max_chain: 1
    require_active: true
    
  when: |
    (subject.id == account.owner_id) OR
    (delegation.active == true AND 
     delegation.scope CONTAINS 'view_accounts')
```

### Transfer Authorization

```yaml
policy:
  name: CustomerTransferPolicy
  version: v1
  
  appliesTo:
    type: inputIntent
    name: InitiateTransferIntent
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Has_Power_Of_Attorney_For
    max_chain: 1
    
  when: |
    (subject.id == from_account.owner_id AND
     transfer.amount <= 10000) OR
    (delegation.active == true AND
     delegation.scope CONTAINS 'initiate_transfer' AND
     transfer.amount <= delegation.max_amount)
```

### Field Permissions

```yaml
fieldPermissions:
  name: AccountFieldPermissions
  version: v1
  appliesTo: AccountState
  
  field_rules:
    - field: account_number
      when: "subject.id != account.owner_id AND NOT subject.has_role('teller')"
      mask:
        transform: reveal_last(4)
        placeholder: "****"
        
    - field: balance
      when: "NOT (subject.id == account.owner_id OR delegation.scope CONTAINS 'view_accounts')"
      mask:
        transform: redact
        placeholder: "***"
```

---

## Key Specification Patterns

### 1. Multi-Region with Local Reads

```yaml
# Pattern: Regional data placement with fallback
placement:
  replicas:
    - region: us-east-1
      role: primary
    - region: eu-west-1
      role: replica
      
availability:
  read_preference: local_preferred
  on_region_unavailable: fallback_remote
```

### 2. Durable Workflow with Human Tasks

```yaml
# Pattern: Long-running process with await points
durability:
  enabled: true
  ttl: 60d
  
await:
  trigger:
    type: inputIntent
    assignment:
      policy: skill_based
      pool: UnderwriterPool
  sla:
    target: 24h
  escalation:
    - after: 48h
      action: escalate_pool
```

### 3. Delegation-Aware Authorization

```yaml
# Pattern: Policy evaluates delegation chain
delegation:
  enabled: true
  via_relationship: Has_Power_Of_Attorney_For
  max_chain: 1
  
when: |
  subject.id == owner_id OR
  (delegation.active AND delegation.scope CONTAINS 'action')
```

### 4. Real-Time Streaming View

```yaml
# Pattern: Sliding window aggregation
temporal:
  window:
    type: sliding
    size: 5m
    slide: 10s
  aggregation:
    - field: amount
      function: sum
  emit:
    trigger: periodic
    periodic_interval: 1s
```

### 5. External API with Resilience

```yaml
# Pattern: Circuit breaker with fallback
resilience:
  timeout: 500ms
  circuit_breaker:
    threshold: 5
    reset_after: 30s
fallback:
  on_unavailable: degrade
  degrade_to: DefaultScore
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 02: Telemedicine Platform

# Telemedicine Platform

**Example ID**: 02  
**Industry**: Healthcare  
**Complexity**: High  
**Key Features**: HIPAA consent, scheduling, video sessions, delegation, accessibility

---

## Overview

A comprehensive telemedicine platform supporting:

- **Patient Portal** - Medical records, appointments, prescriptions
- **Video Consultations** - Real-time telehealth visits
- **Provider Dashboard** - Patient lists, schedules, charting
- **Care Team Delegation** - Shared patient access
- **Insurance Integration** - Eligibility, claims
- **HIPAA Compliance** - Consent, audit, data governance

---

## Data Model

### Patient Record

```yaml
dataType:
  name: PatientRecord
  version: v1
  
  fields:
    patient_id:
      type: string
      constraints:
        required: true
        
    mrn:
      type: string
      description: "Medical Record Number"
      
    demographics:
      type: object
      properties:
        first_name: { type: string }
        last_name: { type: string }
        dob: { type: date }
        gender: { type: enum, values: [male, female, other, prefer_not_to_say] }
        
    contact:
      type: object
      properties:
        phone: { type: string }
        email: { type: string }
        address: { type: object }
        
    emergency_contact:
      type: object
      
    primary_provider_id:
      type: string
      
    insurance:
      type: object
      properties:
        carrier: { type: string }
        member_id: { type: string }
        group_number: { type: string }
        
  governance:
    classification: phi
    retention:
      policy: time_based
      duration: 2555d             # 7 years
      archive:
        enabled: true
        after: 365d
        tier: cold
    residency:
      allowed_regions: [us-east-1, us-west-2]
      transfer:
        allowed: false
    audit:
      access_logging: true
      modification_logging: true
      retention_for_logs: 2555d
      immutable: true
    encryption:
      at_rest: required
      in_transit: required
```

### Appointment

```yaml
dataType:
  name: Appointment
  version: v1
  
  fields:
    appointment_id:
      type: string
      
    patient_id:
      type: string
      
    provider_id:
      type: string
      
    appointment_type:
      type: enum
      values: [video_visit, phone_call, in_person, follow_up]
      
    scheduled_start:
      type: timestamp
      presentation:
        display_as: datetime
        
    scheduled_end:
      type: timestamp
      
    status:
      type: enum
      values: [scheduled, confirmed, in_progress, completed, cancelled, no_show]
      presentation:
        display_as: badge
        semantic_variants:
          scheduled: info
          confirmed: success
          in_progress: warning
          completed: neutral
          cancelled: error
          no_show: error
          
    reason:
      type: string
      
    notes:
      type: string
      
    video_session_id:
      type: string
```

---

## Consent Management

```yaml
consentRecord:
  name: HIPAAConsent
  version: v1
  description: "HIPAA-compliant consent for healthcare services"
  
  subject_field: patient_id
  
  grantor:
    type: subject
    
  scopes:
    - name: treatment
      description: "Use and share data for treatment purposes"
      required: true
      revocable: false
      
    - name: payment
      description: "Use data for insurance and billing"
      required: true
      revocable: false
      
    - name: care_coordination
      description: "Share data with care team members"
      required: false
      default: true
      revocable: true
      
    - name: research
      description: "Use anonymized data for medical research"
      required: false
      default: false
      revocable: true
      
    - name: marketing
      description: "Receive health-related marketing communications"
      required: false
      default: false
      revocable: true
      
  purposes:
    treatment: [diagnosis, treatment_planning, prescription, referral]
    payment: [insurance_claim, billing, eligibility_check]
    care_coordination: [care_team_sharing, specialist_referral]
    research: [anonymized_studies, clinical_trials_eligibility]
    
  data_categories:
    treatment: [medical_history, vitals, lab_results, imaging, medications]
    payment: [demographics, insurance, diagnosis_codes]
    care_coordination: [treatment_summary, care_plan]
    research: [anonymized_demographics, diagnosis_codes]
    
  lifecycle:
    collection_point: PatientOnboardingComposition
    expires_after: 730d
    renewal:
      type: prompt
      notice_before: 30d
      
  withdrawal:
    effect: immediate
    cascade_to: [ResearchDataSharing, MarketingOptIn]
    
  audit:
    track_changes: true
    retention: 7y
```

### Care Team Delegation

```yaml
relationship:
  name: Authorized_Care_Team
  from: ProviderState
  to: PatientState
  type: delegation
  cardinality: many-to-many
  
  delegation:
    scope: [view_records, add_notes, order_labs, prescribe]
    constraints:
      transitive: false
      max_depth: 1
    requires:
      approval: none              # Provider initiates
      verification: identity
    validity:
      type: temporal
      duration: 365d
    revocation:
      by: grantor
      cascade: false
```

---

## Scheduling

```yaml
inputIntent:
  name: ScheduleAppointmentIntent
  version: v1
  
  proposal:
    creates: AppointmentState
    fields:
      patient_id:
        type: string
        required: true
      provider_id:
        type: string
        required: true
      appointment_type:
        type: enum
        required: true
      preferred_date:
        type: date
        required: true
      preferred_time_range:
        type: enum
        values: [morning, afternoon, evening]
        
  notification:
    enabled: true
    trigger:
      on: intent_success
    channels:
      - type: email
        email:
          template: appointment_confirmation
      - type: sms
        sms:
          template: appointment_reminder_sms
          
inputIntent:
  name: AppointmentReminderIntent
  version: v1
  description: "Send appointment reminders"
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: appointment.scheduled_start
        offset: 24h
    batch:
      enabled: true
      source:
        query: "appointment.status == 'scheduled'"
        
  notification:
    enabled: true
    channels:
      - type: sms
        priority: 1
      - type: email
        priority: 2
```

---

## Video Consultation Session

```yaml
collaborativeSession:
  name: TelehealthSession
  version: v1
  description: "Real-time video consultation"
  
  binds_to:
    dataState: AppointmentState
    entity_key: appointment_id
    
  presence:
    enabled: true
    fields:
      video_status:
        type: enum
        values: [connecting, connected, muted, disconnected]
        scope: per_client
      audio_status:
        type: enum
        values: [on, muted]
        scope: per_client
    ttl: 30s
    broadcast:
      to: same_entity
      
  operations:
    mode: stream
    types:
      - name: chat_message
        payload:
          text: string
          attachments: [asset_ref]
      - name: share_document
        payload:
          document_id: string
          
  access:
    max_participants: 4
    roles:
      patient: [join, chat, share_screen]
      provider: [join, chat, share_screen, share_document, record]
      
  lifecycle:
    auto_close_after: 2h
    on_all_leave: end_session
```

---

## Experiences

```yaml
experience:
  name: PatientPortal
  version: v1
  
  entry_point: PatientDashboard
  
  includes:
    - PatientDashboard
    - AppointmentScheduler
    - VideoVisit
    - MedicalRecords
    - Prescriptions
    - Messages
    - Billing
    
  session:
    required: true
    subject_binding:
      mode: authenticated
    lifetime:
      mode: sliding
      idle_timeout: 30m
      max_duration: 8h
    on_expired:
      action: redirect_to_auth
      
  preferences:
    schema:
      notifications:
        type: object
        properties:
          appointment_reminders: { type: boolean, default: true }
          lab_results: { type: boolean, default: true }
          
  accessibility:
    presets:
      - name: vision_impaired
        settings:
          font_scale: 1.5
          high_contrast: true
          motion: none
```

---

## External Integrations

```yaml
externalDependency:
  name: InsuranceEligibilityService
  version: v1
  type: api
  
  capabilities:
    - check_eligibility
    - get_coverage_details
    
  resilience:
    timeout: 10s
    circuit_breaker:
      enabled: true
      threshold: 5
      
  credentials:
    type: oauth2
    oauth2:
      grant_type: client_credentials
      scopes: [eligibility.read]
```

---

## Key Patterns

### 1. HIPAA Consent with Purpose Binding

```yaml
consent:
  enabled: true
  record: HIPAAConsent
  scope: treatment
  
purpose:
  required: true
  allowed: [diagnosis, treatment_planning]
```

### 2. Care Team Delegation

```yaml
delegation:
  enabled: true
  via_relationship: Authorized_Care_Team
  require_active: true
```

### 3. Scheduled Reminders

```yaml
schedule:
  trigger:
    type: relative
    before_event: appointment.scheduled_start
    offset: 24h
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 03: E-Commerce Marketplace

# E-Commerce Marketplace

**Example ID**: 03  
**Industry**: Retail  
**Complexity**: Medium-High  
**Key Features**: Search, assets, payments, recommendations, multi-device

---

## Overview

A multi-vendor marketplace supporting:

- **Product Catalog** - Rich content, images, variants
- **Search & Discovery** - Full-text, faceted, spatial
- **Shopping Cart** - Cross-device sync
- **Checkout & Payments** - Stripe integration
- **Order Tracking** - Status updates, shipping
- **Seller Dashboard** - Inventory, analytics
- **Recommendations** - ML-powered suggestions

---

## Data Model

### Product

```yaml
dataType:
  name: Product
  version: v1
  
  fields:
    product_id:
      type: string
      constraints:
        required: true
        
    seller_id:
      type: string
      
    name:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 3.0
        suggest:
          enabled: true
          type: completion
          
    description:
      type: structured_content
      structured_content:
        schema: product_description
        blocks:
          allowed: [paragraph, heading, list, image]
        marks:
          allowed: [bold, italic, link]
        constraints:
          max_length: 10000
          
    category:
      type: string
      search:
        indexed: true
        strategy: exact
        facet:
          enabled: true
          type: terms
          
    brand:
      type: string
      search:
        indexed: true
        facet:
          enabled: true
          type: terms
          
    price:
      type: decimal
      presentation:
        display_as: currency
        emphasis: primary
      search:
        facet:
          enabled: true
          type: range
          ranges: [0-25, 25-50, 50-100, 100-500, 500+]
          
    images:
      type: array
      items:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 10MB
            mime_types: [image/jpeg, image/png, image/webp]
          variants:
            - name: thumbnail
              transform: resize
              params: { width: 150, height: 150, fit: cover }
            - name: medium
              transform: resize
              params: { width: 500, height: 500, fit: contain }
            - name: large
              transform: resize
              params: { width: 1200, height: 1200, fit: contain }
          access:
            read: public
            
    inventory_count:
      type: integer
      presentation:
        display_as: badge
        semantic_variants:
          "0": error
          "1-5": warning
          "default": neutral
          
    rating:
      type: decimal
      presentation:
        display_as: stars
        
    location:
      type: point
      point:
        coordinate_system: wgs84
        search:
          indexed: true
          strategy: spatial
          
    embedding:
      type: vector
      vector:
        dimensions: 768
        source:
          fields: [name, description, category]
        model:
          name: product_embeddings
          version: v1
        search:
          indexed: true
          strategy: semantic
```

### Shopping Cart

```yaml
dataType:
  name: ShoppingCart
  version: v1
  
  fields:
    cart_id:
      type: string
      
    customer_id:
      type: string
      
    items:
      type: array
      items:
        type: object
        properties:
          product_id: { type: string }
          quantity: { type: integer }
          unit_price: { type: decimal }
          
    subtotal:
      type: decimal
      presentation:
        display_as: currency
        
    created_at:
      type: timestamp
      
    updated_at:
      type: timestamp

dataState:
  name: ShoppingCartState
  type: ShoppingCart
  lifecycle: intermediate
  ttl: 30d
  
  authority:
    scope: entity
    entity_key: cart_id
```

---

## Search

```yaml
searchIntent:
  name: ProductSearchIntent
  version: v1
  description: "Search products with filters"
  
  searches: ProductState
  
  query:
    fields: [name, description, brand]
    strategy: full_text
    highlight:
      enabled: true
      fields: [name, description]
      
  semantic:
    enabled: true
    field: embedding
    weight: 0.3
    
  spatial:
    field: location
    type: within_distance
    within_distance:
      from: "request.user_location"
      max_distance: 50km
    sort:
      by_distance_from: "request.user_location"
      
  facets:
    include: [category, brand, price]
    
  result:
    includes: [product_id, name, price, images, rating, inventory_count]
    max_results: 50
    
  suggestions:
    enabled: true
    field: name
    max_suggestions: 5
```

---

## Payment Integration

```yaml
externalDependency:
  name: StripePayments
  version: v1
  type: api
  
  capabilities:
    - create_payment_intent
    - confirm_payment
    - create_refund
    
  contract:
    create_payment_intent:
      input:
        schema:
          amount: { type: integer, required: true }
          currency: { type: string, required: true }
          customer_id: { type: string }
      output:
        schema:
          payment_intent_id: { type: string }
          client_secret: { type: string }
      errors:
        - code: card_declined
          retryable: false
          
  resilience:
    timeout: 30s
    circuit_breaker:
      enabled: true
      threshold: 5
      
  credentials:
    type: api_key
    secret_ref: stripe_secret_key

webhookReceiver:
  name: StripeWebhook
  version: v1
  
  source: StripePayments
  
  maps_to:
    inputIntent: PaymentEventIntent
    
  authentication:
    type: signature
    signature:
      algorithm: hmac_sha256
      header: Stripe-Signature
      secret_ref: stripe_webhook_secret
      
  event_mapping:
    - external_event: payment_intent.succeeded
      intent_type: PaymentSucceeded
      field_mapping:
        payment_id: "$.data.object.id"
        amount: "$.data.object.amount"
        order_id: "$.data.object.metadata.order_id"
        
    - external_event: charge.refunded
      intent_type: RefundProcessed
      
  idempotency:
    key_path: "$.id"
    ttl: 24h
```

---

## Recommendations

```yaml
dataType:
  name: ProductRecommendation
  version: v1
  
  fields:
    user_id:
      type: string
      
    recommended_products:
      type: array
      items:
        type: object
        properties:
          product_id: { type: string }
          score: { type: decimal }
          reason: { type: string }
          
    generated_at:
      type: timestamp
      
    model_version:
      type: string

materialization:
  name: PersonalizedRecommendationsView
  version: v1
  source: [UserActivityState, ProductState, PurchaseHistoryState]
  targetState: ProductRecommendationMaterialized
  
  inference:
    model:
      name: recommendation_engine
      version: v2
    input_fields: [user_id, recent_views, purchase_history]
    output_field: recommended_products
    freshness:
      strategy: lazy
      ttl: 1h
      
  retrieval:
    mode: by_entity
    entity_key: user_id
    freshness:
      max_staleness: 1h
```

---

## Order Workflow

```yaml
smsFlow:
  name: OrderFulfillmentFlow
  version: v1
  
  triggeredBy:
    inputIntent: PlaceOrderIntent
    
  steps:
    - work_unit: ValidateOrder
    
    - work_unit: ReserveInventory
    
    - work_unit: ProcessPayment
      external_dependency: StripePayments
      
    - if: "payment.status == 'succeeded'"
      then:
        - work_unit: ConfirmOrder
        - work_unit: NotifyCustomer
        - work_unit: NotifySeller
        - work_unit: CreateShipment
      else:
        - work_unit: ReleaseInventory
        - work_unit: NotifyPaymentFailure
        
  compensation:
    - do: RefundPayment
    - do: ReleaseInventory
    - do: CancelOrder
```

---

## Multi-Device Experience

```yaml
experience:
  name: ShoppingApp
  version: v1
  
  entry_point: HomePage
  
  session:
    required: false
    
    subject_binding:
      mode: either
      
    lifetime:
      mode: sliding
      idle_timeout: 30d
      
    devices:
      enabled: true
      max_active: 10
      
      state_scope:
        per_device:
          - ui.recent_searches
          - ui.scroll_position
        shared:
          - cart.items
          - wishlist
          - recently_viewed
          
      handoff:
        enabled: true
        state_transfer: immediate
        transferable:
          - cart.items
          - checkout.step
          
    offline:
      enabled: true
      capabilities:
        read: true
        write: queue
      sync:
        on_reconnect: immediate
        field_strategies:
          - field: cart.items
            strategy: union
          - field: wishlist
            strategy: union
      cache:
        views: [ProductCatalogView, CategoryView]
        max_size: 50MB
        ttl: 24h
```

---

## Key Patterns

### 1. Full-Text + Semantic Search

```yaml
query:
  fields: [name, description]
  strategy: full_text
  
semantic:
  enabled: true
  field: embedding
  weight: 0.3
```

### 2. Asset Variants

```yaml
variants:
  - name: thumbnail
    transform: resize
    params: { width: 150, height: 150 }
```

### 3. Cart as Intermediate State

```yaml
dataState:
  lifecycle: intermediate
  ttl: 30d
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 04: Logistics & Fleet Management

# Logistics & Fleet Management

**Example ID**: 04  
**Industry**: Transportation  
**Complexity**: High  
**Key Features**: IoT devices, spatial queries, real-time streaming, offline mobile

---

## Overview

A comprehensive logistics platform supporting:

- **Fleet Tracking** - Real-time GPS and telemetry
- **Route Optimization** - Shortest path, traffic-aware
- **Delivery Management** - Scheduling, proof of delivery
- **Driver App** - Mobile with offline support
- **Warehouse Integration** - Sensor monitoring
- **Partner APIs** - Shipping carriers

---

## Data Model

### Vehicle

```yaml
dataType:
  name: Vehicle
  version: v1
  
  fields:
    vehicle_id:
      type: string
      
    vin:
      type: string
      
    license_plate:
      type: string
      
    vehicle_type:
      type: enum
      values: [van, truck, semi, motorcycle]
      
    capacity_kg:
      type: decimal
      
    current_location:
      type: point
      point:
        coordinate_system: wgs84
        presentation:
          display_as: map_marker
          
    status:
      type: enum
      values: [available, in_transit, maintenance, offline]
      presentation:
        display_as: badge
        
    driver_id:
      type: string
      
    fuel_level:
      type: decimal
      presentation:
        display_as: gauge
        thresholds:
          warning: 25
          critical: 10
```

### Delivery Zone

```yaml
dataType:
  name: DeliveryZone
  version: v1
  
  fields:
    zone_id:
      type: string
      
    name:
      type: string
      
    boundary:
      type: polygon
      polygon:
        coordinate_system: wgs84
        constraints:
          max_vertices: 100
        presentation:
          display_as: region
          fill_opacity: 0.3
          stroke_color: "#0066CC"
          
    hub_location:
      type: point
      point:
        presentation:
          display_as: map_marker
          icon: warehouse
          
    service_level:
      type: enum
      values: [same_day, next_day, standard]
```

---

## IoT Devices

### Vehicle Tracker

```yaml
worker_class:
  name: VehicleTracker
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_gps
      - read_speed
      - read_fuel_level
      - read_engine_status
      
    hardware:
      category: automotive
      certifications: [FCC, CE]
      
    constraints:
      power:
        source: vehicle
        
      connectivity:
        type: always_on
        protocols: [lte, wifi]
        
    health:
      heartbeat_interval: 1m
      offline_threshold: 5m

deviceCapability:
  name: read_gps
  version: v1
  type: sensor
  
  sensor:
    measures: gps
    output:
      type: point
      precision: 6
    sampling:
      min_interval: 1s
      max_interval: 30s

deviceTelemetry:
  name: VehicleLocationStream
  version: v1
  
  source:
    device_type: VehicleTracker
    capability: read_gps
    
  collection:
    mode: continuous
    continuous:
      interval: 10s
      
  buffering:
    on_offline: buffer_local
    max_buffer_size: 10000
    max_buffer_duration: 24h
    sync_on_reconnect: batched
    
  delivery:
    guarantee: at_least_once
    batching:
      enabled: true
      max_size: 100
      max_delay: 30s
```

### Warehouse Sensor

```yaml
worker_class:
  name: WarehouseTempSensor
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_temperature
      - read_humidity
      
    constraints:
      power:
        source: poe
        
      connectivity:
        type: always_on
        protocols: [ethernet]
        
      environment:
        temperature_range: { min: -30, max: 50 }
        ip_rating: IP65
        
    health:
      heartbeat_interval: 5m

deviceTelemetry:
  name: WarehouseEnvironmentStream
  version: v1
  
  source:
    device_type: WarehouseTempSensor
    capability: read_temperature
    
  collection:
    mode: threshold
    threshold:
      above: 25
      below: 2
      
  aggregation:
    enabled: true
    window: 5m
    functions: [min, max, avg]
```

---

## Spatial Queries

### Route Planning

```yaml
relationship:
  name: Connected_To
  from: LocationState
  to: LocationState
  type: network
  cardinality: many-to-many
  semantics:
    edge_weight: distance_km
    bidirectional: true
    
graphQuery:
  name: ShortestRouteQuery
  version: v1
  description: "Find optimal route between stops"
  
  traverses: Connected_To
  
  start:
    from: LocationState
    where: "location.id == request.origin_id"
    
  pattern:
    type: shortest_path
    shortest_path:
      to: "location.id == request.destination_id"
      weight_field: distance_km
      max_cost: 500
      
  result:
    includes: [id, name, coordinates]
    path_field: route
    
  aggregation:
    - type: sum
      field: distance_km
      as: total_distance
```

### Nearby Vehicles

```yaml
searchIntent:
  name: NearbyVehiclesQuery
  version: v1
  
  searches: VehicleState
  
  query:
    spatial:
      field: current_location
      type: within_distance
      within_distance:
        from: "request.location"
        max_distance: 10km
      sort:
        by_distance_from: "request.location"
        
  filter:
    expression: "vehicle.status == 'available'"
    
  result:
    includes: [vehicle_id, vehicle_type, current_location, driver_id]
    max_results: 20
```

### Zone Containment

```yaml
searchIntent:
  name: DeliveryZoneLookup
  version: v1
  
  searches: DeliveryZoneState
  
  query:
    spatial:
      field: boundary
      type: contains_point
      contains_point:
        point: "request.delivery_address"
        
  result:
    includes: [zone_id, name, service_level, hub_location]
```

---

## Real-Time Views

```yaml
materialization:
  name: FleetStatusView
  version: v1
  source: [VehicleLocationStream, VehicleState]
  targetState: FleetStatusMaterialized
  
  retrieval:
    mode: singleton
    
    temporal:
      window:
        type: tumbling
        size: 1m
        
      aggregation:
        - field: vehicle_id
          function: count
          as: active_vehicles
        - field: speed
          function: avg
          as: avg_speed
          
      emit:
        trigger: on_window_close
        
    freshness:
      max_staleness: 1m

materialization:
  name: VehicleTrailView
  version: v1
  source: VehicleLocationStream
  targetState: VehicleTrailMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: vehicle_id
    
    temporal:
      window:
        type: sliding
        size: 1h
        slide: 1m
        
      aggregation:
        - field: location
          function: collect
          as: trail_points
          
      emit:
        trigger: periodic
        periodic_interval: 1m
```

---

## Driver Mobile App

```yaml
experience:
  name: DriverApp
  version: v1
  
  entry_point: DriverDashboard
  
  includes:
    - DriverDashboard
    - DeliveryList
    - Navigation
    - ProofOfDelivery
    - VehicleInspection
    
  session:
    required: true
    
    subject_binding:
      mode: authenticated
      
    lifetime:
      mode: bounded
      max_duration: 12h
      
    devices:
      enabled: true
      max_active: 2
      
      state_scope:
        per_device:
          - ui.map_zoom
        shared:
          - current_route
          - completed_deliveries
          
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: immediate
        conflict_resolution: merge
        field_strategies:
          - field: delivery.status
            strategy: server_wins
          - field: proof_of_delivery
            strategy: client_wins
            
      queue:
        max_size: 500
        max_age: 48h
        
      cache:
        views: [RouteView, DeliveryListView, CustomerInfoView]
        max_size: 100MB
        ttl: 24h
```

---

## External Integrations

```yaml
externalDependency:
  name: ShippingCarrier
  version: v1
  type: api
  
  capabilities:
    - get_rates
    - create_shipment
    - create_label
    - track_package
    
  resilience:
    timeout: 10s
    retry:
      enabled: true
      max_attempts: 3
    circuit_breaker:
      enabled: true
      threshold: 5
      
  credentials:
    type: api_key
    secret_ref: carrier_api_key

webhookReceiver:
  name: CarrierTrackingWebhook
  version: v1
  
  source: ShippingCarrier
  
  maps_to:
    inputIntent: TrackingUpdateIntent
    
  event_mapping:
    - external_event: tracking.updated
      intent_type: DeliveryStatusUpdate
      field_mapping:
        tracking_number: "$.tracking_number"
        status: "$.status"
        location: "$.current_location"
```

---

## Key Patterns

### 1. IoT Telemetry with Offline Buffering

```yaml
buffering:
  on_offline: buffer_local
  max_buffer_size: 10000
  sync_on_reconnect: batched
```

### 2. Spatial Shortest Path

```yaml
pattern:
  type: shortest_path
  shortest_path:
    weight_field: distance_km
```

### 3. Real-Time Fleet Aggregation

```yaml
temporal:
  window:
    type: tumbling
    size: 1m
  aggregation:
    - field: vehicle_id
      function: count
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 05: Customer Service Center

# Customer Service Center

**Example ID**: 05  
**Industry**: Cross-Industry  
**Complexity**: Medium  
**Key Features**: Chatbot, voice, human tasks, knowledge base, omnichannel

---

## Overview

An omnichannel customer support platform supporting:

- **Ticketing System** - Create, assign, resolve tickets
- **AI Chatbot** - Automated first response
- **Voice Assistant** - IVR and voice support
- **Live Agent Handoff** - Seamless escalation
- **Knowledge Base** - Self-service articles
- **Customer 360** - Unified customer view

---

## Data Model

### Support Ticket

```yaml
dataType:
  name: SupportTicket
  version: v1
  
  fields:
    ticket_id:
      type: string
      
    customer_id:
      type: string
      
    subject:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    description:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    category:
      type: enum
      values: [billing, technical, shipping, returns, general]
      search:
        facet:
          enabled: true
          type: terms
          
    priority:
      type: enum
      values: [low, medium, high, urgent]
      presentation:
        display_as: badge
        semantic_variants:
          low: neutral
          medium: info
          high: warning
          urgent: error
          
    status:
      type: enum
      values: [open, in_progress, pending_customer, resolved, closed]
      presentation:
        display_as: badge
        
    assigned_agent_id:
      type: string
      
    channel:
      type: enum
      values: [web, chat, voice, email, social]
      
    created_at:
      type: timestamp
      
    resolved_at:
      type: timestamp
      
    sla_due_at:
      type: timestamp
      presentation:
        display_as: countdown
```

### Knowledge Article

```yaml
dataType:
  name: KnowledgeArticle
  version: v1
  
  fields:
    article_id:
      type: string
      
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 3.0
        suggest:
          enabled: true
          
    content:
      type: structured_content
      structured_content:
        schema: help_article
        blocks:
          allowed: [paragraph, heading, list, code_block, image, callout]
        marks:
          allowed: [bold, italic, link, code]
        links:
          internal:
            enabled: true
            targets: [KnowledgeArticle]
            
    category:
      type: string
      search:
        facet:
          enabled: true
          
    tags:
      type: array
      items:
        type: string
      search:
        indexed: true
        
    helpful_votes:
      type: integer
      
    view_count:
      type: integer
      
    published:
      type: boolean
```

---

## Conversational Experience

### Chatbot

```yaml
experience:
  name: CustomerChatbot
  version: v1
  
  modality:
    type: text
    text:
      platforms: [web, slack, whatsapp]
      
  conversation:
    context:
      ttl: 30m
      carry_forward: [customer_id, ticket_id, issue_category]
      
    turn_taking:
      model: user_initiated
      
    error_handling:
      no_match: fallback
      max_reprompts: 3
      
    disambiguation:
      strategy: list_options
      max_options: 3
      
  entry_point: GreetingDialog

presentationComposition:
  name: GreetingDialog
  version: v1
  
  dialog:
    type: statement
    
    prompts:
      initial: GreetingPrompt
      
    transitions:
      on_complete: IssueClassificationDialog

presentationComposition:
  name: IssueClassificationDialog
  version: v1
  
  dialog:
    type: question
    
    prompts:
      initial: WhatCanIHelpWithPrompt
      
    slots:
      - name: issue_category
        maps_to: ClassifyIssueIntent.category
        required: true
        type: enum
        enum:
          values: [billing, technical, shipping, returns, other]
        synonyms:
          billing: [payment, charge, invoice, refund, subscription]
          technical: [bug, error, not working, broken, crash, login]
          shipping: [delivery, tracking, package, lost, late]
          returns: [return, exchange, wrong item, damaged]
          
      - name: description
        maps_to: ClassifyIssueIntent.description
        required: true
        elicitation:
          prompt: DescribeIssuePrompt
        type: text
        validation:
          expression: "length(description) >= 10"
          
    transitions:
      on_complete: CheckKnowledgeBaseDialog
      on_fallback: HandoffToAgentDialog

presentationComposition:
  name: HandoffToAgentDialog
  version: v1
  
  dialog:
    type: handoff
    
    prompts:
      initial: HandoffPrompt
      
    handoff:
      to: live_agent
      queue: GeneralSupportQueue
      context_transfer:
        - customer_id
        - issue_category
        - conversation_history

prompt:
  name: GreetingPrompt
  version: v1
  content:
    text: "Hi! I'm your support assistant. How can I help you today?"
  variations:
    - text: "Hello! What can I help you with?"
    - text: "Welcome! I'm here to help. What's on your mind?"

prompt:
  name: HandoffPrompt
  version: v1
  content:
    text: "I'll connect you with a support agent who can help further. Please hold on for a moment..."
```

### Voice Assistant

```yaml
experience:
  name: CustomerVoiceSupport
  version: v1
  
  modality:
    type: voice
    voice:
      languages: [en-US, es-US]
      
  conversation:
    context:
      ttl: 10m
      
    turn_taking:
      model: mixed
      timeout: 8s
      
    error_handling:
      no_match: reprompt
      no_input: reprompt
      max_reprompts: 2
      
  entry_point: VoiceGreetingDialog

presentationComposition:
  name: VoiceGreetingDialog
  version: v1
  
  dialog:
    type: statement
    
    prompts:
      initial: VoiceGreetingPrompt
      
prompt:
  name: VoiceGreetingPrompt
  version: v1
  content:
    speech:
      ssml: |
        <speak>
          Thank you for calling customer support.
          <break time="300ms"/>
          For billing questions, say "billing".
          For technical support, say "technical".
          Or describe your issue in a few words.
        </speak>
```

---

## Agent Workflow

```yaml
smsFlow:
  name: TicketResolutionFlow
  version: v1
  
  triggeredBy:
    inputIntent: CreateTicketIntent
    
  durability:
    enabled: true
    state: TicketFlowState
    ttl: 7d
    
  steps:
    - work_unit: ClassifyTicket
    
    - work_unit: CalculateSLA
    
    - work_unit: SearchKnowledgeBase
    
    - if: "kb_match.confidence > 0.9"
      then:
        - work_unit: SuggestArticle
        - await:
            name: await_customer_feedback
            trigger:
              type: inputIntent
              intent: ArticleFeedbackIntent
              assignment:
                policy: direct
                subject: "ticket.customer_id"
            timeout: 24h
            on_timeout: continue_default
            default_result:
              helpful: false
        - if: "feedback.helpful"
          then:
            - work_unit: AutoResolveTicket
          else:
            - goto: assign_agent
      else:
        - label: assign_agent
        - await:
            name: await_agent_resolution
            trigger:
              type: inputIntent
              intent: ResolveTicketIntent
              assignment:
                policy: least_loaded
                pool: SupportAgentPool
            sla:
              target: "ticket.sla_due_at"
              warning_at: "ticket.sla_due_at - 2h"
            escalation:
              - after: "ticket.sla_due_at - 1h"
                action: notify
                to: SupportSupervisor
              - after: "ticket.sla_due_at"
                action: escalate_pool
                to: SeniorAgentPool
                
    - work_unit: SendResolutionSurvey
    
taskPool:
  name: SupportAgentPool
  version: v1
  
  membership:
    type: role_based
    roles: [support_agent, senior_agent]
    
  capacity:
    max_concurrent_per_member: 5
    
  availability:
    schedule: "0 9-17 * * 1-5"
    timezone: America/New_York
    
  skills:
    - name: billing
      required: false
    - name: technical
      required: false
```

---

## Search

```yaml
searchIntent:
  name: KnowledgeBaseSearch
  version: v1
  
  searches: KnowledgeArticleState
  
  query:
    fields: [title, content, tags]
    strategy: full_text
    highlight:
      enabled: true
      
  filter:
    expression: "article.published == true"
    
  facets:
    include: [category]
    
  suggestions:
    enabled: true
    field: title
```

---

## Key Patterns

### 1. Chatbot with Slot Filling

```yaml
slots:
  - name: issue_category
    type: enum
    synonyms:
      billing: [payment, charge, refund]
```

### 2. Agent Task Assignment

```yaml
assignment:
  policy: least_loaded
  pool: SupportAgentPool
```

### 3. Handoff to Human

```yaml
handoff:
  to: live_agent
  queue: GeneralSupportQueue
  context_transfer: [customer_id, conversation_history]
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 06: Corporate HR Platform

# Corporate HR Platform

**Example ID**: 06  
**Industry**: Enterprise  
**Complexity**: Medium  
**Key Features**: Graph hierarchy, approval workflows, GDPR, delegation

---

## Overview

A comprehensive HR management platform supporting:

- **Employee Directory** - Org chart, search
- **Onboarding/Offboarding** - Multi-step workflows
- **Time & Attendance** - Clock in/out, leave requests
- **Performance Reviews** - 360 feedback, goals
- **Payroll Integration** - External system sync
- **GDPR Compliance** - Erasure, consent

---

## Data Model

### Employee

```yaml
dataType:
  name: Employee
  version: v1
  
  fields:
    employee_id:
      type: string
      
    email:
      type: string
      search:
        indexed: true
        
    first_name:
      type: string
      search:
        indexed: true
        
    last_name:
      type: string
      search:
        indexed: true
        
    title:
      type: string
      
    department:
      type: string
      search:
        facet:
          enabled: true
          
    manager_id:
      type: string
      
    hire_date:
      type: date
      
    employment_status:
      type: enum
      values: [active, on_leave, terminated]
      
    profile_photo:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
        variants:
          - name: avatar
            transform: resize
            params: { width: 100, height: 100, fit: cover }
            
  governance:
    classification: pii
    retention:
      policy: event_based
      after_event: employment_terminated
      grace_period: 2555d         # 7 years after termination
      on_expiry: anonymize
    audit:
      access_logging: true
      modification_logging: true
```

### Leave Request

```yaml
dataType:
  name: LeaveRequest
  version: v1
  
  fields:
    request_id:
      type: string
      
    employee_id:
      type: string
      
    leave_type:
      type: enum
      values: [vacation, sick, personal, bereavement, parental]
      
    start_date:
      type: date
      
    end_date:
      type: date
      
    status:
      type: enum
      values: [pending, approved, denied, cancelled]
      presentation:
        display_as: badge
        
    approver_id:
      type: string
      
    notes:
      type: string
```

---

## Organizational Hierarchy

```yaml
relationship:
  name: Reports_To
  from: EmployeeState
  to: EmployeeState
  type: hierarchical
  cardinality: many-to-one
  semantics:
    causal: true
    
  recursion:
    enabled: true
    max_depth: 15
    direction: up
    cycle_handling: error

graphQuery:
  name: ManagementChainQuery
  version: v1
  description: "Find all managers above an employee"
  
  traverses: Reports_To
  
  start:
    from: EmployeeState
    where: "employee.id == request.employee_id"
    
  pattern:
    type: path
    path:
      direction: up
      stop_when: "employee.manager_id == null"
      max_hops: 15
      
  result:
    includes: [employee_id, first_name, last_name, title]
    depth_field: level

graphQuery:
  name: DirectReportsQuery
  version: v1
  description: "Find all employees reporting to a manager"
  
  traverses: Reports_To
  
  start:
    from: EmployeeState
    where: "employee.id == request.manager_id"
    
  pattern:
    type: neighbors
    neighbors:
      depth: 1
      direction: inbound
      
  result:
    includes: [employee_id, first_name, last_name, title, department]

graphQuery:
  name: TeamHierarchyQuery
  version: v1
  description: "Find entire team tree under a manager"
  
  traverses: Reports_To
  
  start:
    from: EmployeeState
    where: "employee.id == request.manager_id"
    
  pattern:
    type: path
    path:
      direction: down
      max_hops: 5
      
  result:
    includes: [employee_id, first_name, last_name, title]
    depth_field: org_level
```

---

## Manager Delegation

```yaml
relationship:
  name: Delegates_Approval_To
  from: EmployeeState
  to: EmployeeState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [approve_leave, approve_expense, approve_timesheet]
    constraints:
      transitive: false
      max_depth: 1
    requires:
      approval: grantor
    validity:
      type: temporal
      duration: 30d
    revocation:
      by: grantor
```

---

## Approval Workflow

```yaml
smsFlow:
  name: LeaveApprovalFlow
  version: v1
  
  triggeredBy:
    inputIntent: SubmitLeaveRequestIntent
    
  durability:
    enabled: true
    state: LeaveRequestFlowState
    ttl: 30d
    
  steps:
    - work_unit: ValidateLeaveBalance
    
    - work_unit: DetermineApprover
      output:
        approver_id: "manager.id"
        
    - await:
        name: await_manager_approval
        trigger:
          type: inputIntent
          intent: ApproveLeaveIntent
          assignment:
            policy: direct
            subject: "approver_id"
        sla:
          target: 48h
        escalation:
          - after: 48h
            action: notify
            to: "approver.manager_id"
        timeout: 7d
        on_timeout: continue_default
        default_result:
          decision: auto_approved
        result:
          captures: [decision, denial_reason]
          
    - if: "decision == 'approved' OR decision == 'auto_approved'"
      then:
        - work_unit: UpdateLeaveBalance
        - work_unit: UpdateCalendar
        - work_unit: NotifyEmployee
      else:
        - work_unit: NotifyDenial
        
  compensation:
    - do: RestoreLeaveBalance
    - do: RemoveCalendarEntry

policy:
  name: LeaveApprovalPolicy
  version: v1
  
  appliesTo:
    type: inputIntent
    name: ApproveLeaveIntent
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Delegates_Approval_To
    max_chain: 1
    
  when: |
    (subject.id == leave_request.approver_id) OR
    (delegation.active AND delegation.scope CONTAINS 'approve_leave')
```

---

## GDPR Compliance

```yaml
erasureRequest:
  name: EmployeeDataErasure
  version: v1
  description: "GDPR erasure for former employees"
  
  scope:
    subject_field: employee_id
    
  strategy: hybrid
  
  hybrid:
    delete: [email, phone, address, emergency_contact, profile_photo]
    anonymize: [first_name, last_name, performance_reviews]
    retain: [employee_id, hire_date, termination_date, department, title]
    
  cascade:
    behavior: automatic
    entities: [LeaveRequestState, TimesheetState, ExpenseState]
    relationship_handling: orphan
    
  verification:
    required: true
    methods: [identity_verification]
    
  processing:
    sla: 720h                          # 30 days
    notification:
      on_complete: true
      
  exceptions:
    - reason: legal_hold
      action: defer
    - reason: ongoing_litigation
      action: refuse
      
  completion:
    certificate: true
    retention_of_proof: 3y

anonymizationProfile:
  name: EmployeeAnonymization
  version: v1
  
  applies_to: EmployeeState
  
  transforms:
    - field: first_name
      method: pseudonymize
      params:
        key_ref: employee_anon_key
        
    - field: last_name
      method: pseudonymize
      params:
        key_ref: employee_anon_key
        
    - field: email
      method: hash
      params:
        algorithm: sha256
        salt_source: entity_id
        
    - field: phone
      method: redact
      params:
        placeholder: "[REDACTED]"
        
  reversible: false
```

---

## Payroll Integration

```yaml
externalDependency:
  name: PayrollSystem
  version: v1
  type: api
  
  capabilities:
    - sync_employee
    - submit_hours
    - get_pay_stub
    
  resilience:
    timeout: 30s
    circuit_breaker:
      enabled: true
      threshold: 5
      
  credentials:
    type: oauth2
    oauth2:
      grant_type: client_credentials
      scopes: [employees.write, timesheets.write]

inputIntent:
  name: SyncPayrollIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 2 * * *"       # 2 AM daily
        timezone: America/New_York
    batch:
      enabled: true
      source:
        query: "employee.status == 'active'"
```

---

## Search

```yaml
searchIntent:
  name: EmployeeDirectorySearch
  version: v1
  
  searches: EmployeeState
  
  query:
    fields: [first_name, last_name, email, title]
    strategy: full_text
    
  facets:
    include: [department, employment_status]
    
  result:
    includes: [employee_id, first_name, last_name, title, department, profile_photo]
```

---

## Key Patterns

### 1. Recursive Org Hierarchy

```yaml
recursion:
  enabled: true
  max_depth: 15
  direction: up
```

### 2. Manager Delegation

```yaml
delegation:
  scope: [approve_leave]
  validity:
    type: temporal
    duration: 30d
```

### 3. GDPR Hybrid Erasure

```yaml
strategy: hybrid
hybrid:
  delete: [email, phone]
  anonymize: [name]
  retain: [employee_id, hire_date]
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 07: Property Management

# Property Management

**Example ID**: 07  
**Industry**: Real Estate  
**Complexity**: Medium  
**Key Features**: Spatial, IoT, assets, temporal access, scheduling

---

## Overview

A property management platform supporting:

- **Property Listings** - Location-based search
- **Tenant Management** - Leases, payments
- **Maintenance Requests** - Ticketing, assignment
- **Smart Building** - IoT sensors, access control
- **Guest Access** - Time-bound entry

---

## Data Model

### Property

```yaml
dataType:
  name: Property
  version: v1
  
  fields:
    property_id:
      type: string
      
    name:
      type: string
      search:
        indexed: true
        
    property_type:
      type: enum
      values: [apartment, house, condo, commercial, industrial]
      search:
        facet:
          enabled: true
          
    address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        state: { type: string }
        zip: { type: string }
        country: { type: string }
        
    location:
      type: point
      point:
        coordinate_system: wgs84
        search:
          indexed: true
          strategy: spatial
        presentation:
          display_as: map_marker
          
    bedrooms:
      type: integer
      search:
        facet:
          enabled: true
          type: terms
          
    bathrooms:
      type: decimal
      
    square_feet:
      type: integer
      search:
        facet:
          enabled: true
          type: range
          ranges: [0-500, 500-1000, 1000-1500, 1500-2000, 2000+]
          
    rent_amount:
      type: decimal
      presentation:
        display_as: currency
        emphasis: primary
      search:
        facet:
          enabled: true
          type: range
          
    photos:
      type: array
      items:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 10MB
          variants:
            - name: thumbnail
              transform: resize
              params: { width: 300, height: 200 }
            - name: gallery
              transform: resize
              params: { width: 1200, height: 800 }
          access:
            read: public
            
    virtual_tour:
      type: asset
      asset:
        category: media
        constraints:
          mime_types: [video/mp4, application/x-mpegURL]
        access:
          read: signed
          url_expiry: 1h
          
    status:
      type: enum
      values: [available, occupied, maintenance, off_market]
```

### Lease

```yaml
dataType:
  name: Lease
  version: v1
  
  fields:
    lease_id:
      type: string
      
    property_id:
      type: string
      
    tenant_id:
      type: string
      
    start_date:
      type: date
      
    end_date:
      type: date
      
    monthly_rent:
      type: decimal
      
    security_deposit:
      type: decimal
      
    status:
      type: enum
      values: [draft, pending_signature, active, expired, terminated]
      
    lease_document:
      type: asset
      asset:
        category: document
        constraints:
          mime_types: [application/pdf]
        lifecycle:
          type: permanent
        access:
          read: authenticated
```

---

## Spatial Search

```yaml
searchIntent:
  name: PropertySearchIntent
  version: v1
  
  searches: PropertyState
  
  query:
    fields: [name, address.city]
    strategy: full_text
    
  spatial:
    field: location
    type: within_distance
    within_distance:
      from: "request.search_location"
      max_distance: "request.radius_km"
    sort:
      by_distance_from: "request.search_location"
      
  facets:
    include: [property_type, bedrooms, square_feet, rent_amount]
    
  filter:
    expression: "property.status == 'available'"
```

---

## Smart Building IoT

### Smart Lock

```yaml
worker_class:
  name: SmartLock
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - lock
      - unlock
      - check_status
      - read_battery
      
    constraints:
      power:
        source: battery
        battery_capacity: 4000
        
      connectivity:
        type: always_on
        protocols: [ble, wifi]
        
    health:
      heartbeat_interval: 1h
      battery_low_threshold: 20

deviceCapability:
  name: unlock
  version: v1
  type: actuator
  
  actuator:
    controls: lock
    input:
      type: boolean
    feedback:
      confirmation: required
      timeout: 10s
      
  safety:
    constraints:
      - expression: "request.authorized == true"
        on_violation: reject

actuatorCommand:
  name: UnlockDoorCommand
  version: v1
  
  target:
    device_type: SmartLock
    capability: unlock
    selector: "device.property_id == request.property_id"
    
  command:
    type: set_state
    set_state:
      value: true
      
  delivery:
    guarantee: at_least_once
    timeout: 30s
    
  confirmation:
    required: true
    timeout: 10s
```

### Thermostat

```yaml
worker_class:
  name: SmartThermostat
  runtime: edge
  
  device:
    type: hybrid
    
    capabilities:
      - read_temperature
      - set_target_temperature
      - read_humidity
      
    constraints:
      power:
        source: mains
      connectivity:
        type: always_on
        protocols: [wifi]
        
    health:
      heartbeat_interval: 5m

deviceCapability:
  name: set_target_temperature
  version: v1
  type: actuator
  
  actuator:
    controls: hvac
    input:
      type: decimal
      range: { min: 55, max: 85 }
    feedback:
      confirmation: required
      
  safety:
    constraints:
      - expression: "target >= 55 AND target <= 85"
        on_violation: clamp
    clamp_to: { min: 55, max: 85 }
```

---

## Guest Access

```yaml
temporalAccess:
  name: GuestPropertyAccess
  version: v1
  description: "Time-bound guest access to property"
  
  grants:
    policy: PropertyAccessPolicy
    to: "request.guest_email"
    
  validity:
    type: time_bound
    starts_at: "booking.check_in_time"
    expires_at: "booking.check_out_time"
    
  constraints:
    device_bound: false
    
  revocation:
    on_breach: immediate

inputIntent:
  name: GrantGuestAccessIntent
  version: v1
  
  proposal:
    creates: GuestAccessState
    fields:
      property_id:
        type: string
        required: true
      guest_email:
        type: string
        required: true
      access_start:
        type: timestamp
        required: true
      access_end:
        type: timestamp
        required: true
        
  notification:
    enabled: true
    trigger:
      on: intent_success
    channels:
      - type: email
        email:
          template: guest_access_granted
          to_field: guest_email
      - type: sms
```

---

## Maintenance Workflow

```yaml
smsFlow:
  name: MaintenanceRequestFlow
  version: v1
  
  triggeredBy:
    inputIntent: SubmitMaintenanceRequestIntent
    
  durability:
    enabled: true
    state: MaintenanceFlowState
    ttl: 30d
    
  steps:
    - work_unit: ClassifyRequest
    
    - work_unit: AssignPriority
    
    - await:
        name: await_technician_assignment
        trigger:
          type: inputIntent
          intent: AcceptMaintenanceJobIntent
          assignment:
            policy: skill_based
            pool: MaintenanceTechPool
        sla:
          target: 24h
        escalation:
          - after: 24h
            action: notify
            to: PropertyManager
            
    - await:
        name: await_completion
        trigger:
          type: inputIntent
          intent: CompleteMaintenanceIntent
        timeout: 7d
        
    - work_unit: RequestTenantFeedback
    
taskPool:
  name: MaintenanceTechPool
  version: v1
  
  membership:
    type: role_based
    roles: [maintenance_tech]
    
  skills:
    - name: plumbing
    - name: electrical
    - name: hvac
    - name: general
```

---

## Scheduled Tasks

```yaml
inputIntent:
  name: RentReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 9 1 * *"       # 1st of month at 9 AM
        timezone: America/New_York
    batch:
      enabled: true
      source:
        query: "lease.status == 'active'"
        
  notification:
    enabled: true
    channels:
      - type: email
        email:
          template: rent_reminder
      - type: sms
```

---

## Key Patterns

### 1. Location-Based Property Search

```yaml
spatial:
  field: location
  type: within_distance
  sort:
    by_distance_from: "request.search_location"
```

### 2. Time-Bound Guest Access

```yaml
validity:
  type: time_bound
  starts_at: "booking.check_in_time"
  expires_at: "booking.check_out_time"
```

### 3. Smart Lock Control

```yaml
actuator:
  controls: lock
  safety:
    constraints:
      - expression: "request.authorized == true"
        on_violation: reject
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 08: Learning Management System

# Learning Management System

**Example ID**: 08  
**Industry**: Education  
**Complexity**: Medium  
**Key Features**: Rich content, assets, collaboration, offline, accessibility

---

## Overview

A learning management system supporting:

- **Course Creation** - Rich content authoring
- **Video Lessons** - Streaming with progress
- **Assessments** - Quizzes, assignments
- **Progress Tracking** - Completion, certificates
- **Collaborative Notes** - Real-time editing
- **Mobile Learning** - Offline access

---

## Data Model

### Course

```yaml
dataType:
  name: Course
  version: v1
  
  fields:
    course_id:
      type: string
      
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 3.0
        
    description:
      type: structured_content
      structured_content:
        schema: course_description
        blocks:
          allowed: [paragraph, heading, list, image]
        constraints:
          max_length: 5000
          
    instructor_id:
      type: string
      
    category:
      type: string
      search:
        facet:
          enabled: true
          
    difficulty:
      type: enum
      values: [beginner, intermediate, advanced]
      search:
        facet:
          enabled: true
          
    duration_hours:
      type: decimal
      
    thumbnail:
      type: asset
      asset:
        category: image
        variants:
          - name: card
            transform: resize
            params: { width: 400, height: 225 }
        access:
          read: public
          
    status:
      type: enum
      values: [draft, published, archived]
      
    rating:
      type: decimal
      presentation:
        display_as: stars
        
    enrollment_count:
      type: integer
```

### Lesson

```yaml
dataType:
  name: Lesson
  version: v1
  
  fields:
    lesson_id:
      type: string
      
    course_id:
      type: string
      
    title:
      type: string
      
    sequence_order:
      type: integer
      
    content:
      type: structured_content
      structured_content:
        schema: lesson_content
        blocks:
          allowed: [paragraph, heading, list, code_block, image, video_embed, callout, quiz_embed]
        marks:
          allowed: [bold, italic, link, code, highlight]
        nesting:
          max_depth: 3
          
    video:
      type: asset
      asset:
        category: media
        constraints:
          mime_types: [video/mp4, application/x-mpegURL]
          max_size: 2GB
        lifecycle:
          type: permanent
        access:
          read: authenticated
          
    duration_minutes:
      type: integer
      
    attachments:
      type: array
      items:
        type: asset
        asset:
          category: document
          constraints:
            max_size: 50MB
          access:
            read: authenticated
```

### Quiz

```yaml
dataType:
  name: Quiz
  version: v1
  
  fields:
    quiz_id:
      type: string
      
    lesson_id:
      type: string
      
    questions:
      type: array
      items:
        type: object
        properties:
          question_text: { type: string }
          question_type: { type: enum, values: [multiple_choice, true_false, short_answer] }
          options: { type: array, items: { type: string } }
          correct_answer: { type: string }
          points: { type: integer }
          
    passing_score:
      type: integer
      
    time_limit_minutes:
      type: integer
```

### Student Progress

```yaml
dataType:
  name: StudentProgress
  version: v1
  
  fields:
    student_id:
      type: string
      
    course_id:
      type: string
      
    enrolled_at:
      type: timestamp
      
    completed_lessons:
      type: array
      items:
        type: string
        
    quiz_scores:
      type: object
      
    overall_progress:
      type: decimal
      presentation:
        display_as: progress_bar
        
    completed_at:
      type: timestamp
      
    certificate_id:
      type: string
```

---

## Collaborative Notes

```yaml
collaborativeSession:
  name: SharedNotesSession
  version: v1
  description: "Real-time collaborative note-taking"
  
  binds_to:
    dataState: StudentNotesState
    entity_key: notes_id
    
  presence:
    enabled: true
    fields:
      cursor:
        type: position
        scope: per_client
      selection:
        type: range
        scope: per_client
      user_color:
        type: string
        scope: per_session
    ttl: 30s
    broadcast:
      to: same_entity
      
  operations:
    mode: stream
    ordering:
      guarantee: causal
    conflict:
      hint: merge
    types:
      - name: text_insert
        payload:
          position: integer
          text: string
      - name: text_delete
        payload:
          position: integer
          length: integer
      - name: format
        payload:
          range: object
          mark: string
          
  access:
    max_participants: 10
    roles:
      owner: [edit, share, delete]
      collaborator: [edit]
      viewer: [view]
```

---

## Certificate Generation

```yaml
inputIntent:
  name: GenerateCertificateIntent
  version: v1
  description: "Generate course completion certificate"
  
  schedule:
    enabled: true
    trigger:
      type: event
      event:
        type: CourseCompleted
        filter: "progress.overall_progress == 100"
        
smsFlow:
  name: CertificateGenerationFlow
  version: v1
  
  triggeredBy:
    signal: CourseCompletedSignal
    
  steps:
    - work_unit: VerifyCompletion
    
    - work_unit: GenerateCertificatePDF
      output:
        certificate_url: "certificate.url"
        
    - work_unit: UpdateStudentRecord
    
    - signal: CertificateReadySignal
    
  notification:
    enabled: true
    trigger:
      on: flow_success
    channels:
      - type: email
        email:
          template: certificate_ready
          attachments:
            - field: certificate_url
```

---

## Offline Learning

```yaml
experience:
  name: LearningApp
  version: v1
  
  entry_point: Dashboard
  
  includes:
    - Dashboard
    - CourseList
    - CourseDetail
    - LessonPlayer
    - Notes
    - Quizzes
    - Progress
    
  session:
    required: true
    
    subject_binding:
      mode: authenticated
      
    lifetime:
      mode: sliding
      idle_timeout: 7d
      
    devices:
      enabled: true
      max_active: 5
      
      state_scope:
        per_device:
          - video.playback_position
          - ui.font_size
        shared:
          - progress
          - notes
          - quiz_answers
          
      handoff:
        enabled: true
        state_transfer: on_request
        transferable:
          - current_lesson
          - video.playback_position
          
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: background
        conflict_resolution: merge
        field_strategies:
          - field: notes.content
            strategy: merge
          - field: progress.completed_lessons
            strategy: union
          - field: quiz_answers
            strategy: client_wins
            
      queue:
        max_size: 200
        max_age: 30d
        
      cache:
        views: [EnrolledCoursesView, LessonContentView]
        max_size: 500MB
        ttl: 168h
        
  accessibility:
    presets:
      - name: vision_impaired
        settings:
          font_scale: 1.5
          high_contrast: true
      - name: hearing_impaired
        settings:
          captions: always
          visual_alerts: true
          
  preferences:
    schema:
      playback_speed:
        type: enum
        values: [0.5, 0.75, 1.0, 1.25, 1.5, 2.0]
        default: 1.0
      captions:
        type: enum
        values: [off, auto, always]
        default: auto
      download_quality:
        type: enum
        values: [low, medium, high]
        default: medium
```

---

## Search

```yaml
searchIntent:
  name: CourseSearchIntent
  version: v1
  
  searches: CourseState
  
  query:
    fields: [title, description]
    strategy: full_text
    highlight:
      enabled: true
      
  filter:
    expression: "course.status == 'published'"
    
  facets:
    include: [category, difficulty]
    
  suggestions:
    enabled: true
    field: title
```

---

## Key Patterns

### 1. Rich Content with Embedded Elements

```yaml
blocks:
  allowed: [paragraph, heading, list, code_block, video_embed, quiz_embed]
```

### 2. Collaborative Note-Taking

```yaml
operations:
  mode: stream
  conflict:
    hint: merge
```

### 3. Offline Course Caching

```yaml
cache:
  views: [EnrolledCoursesView, LessonContentView]
  max_size: 500MB
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Social Networking Platform

**Example ID**: 09  
**Industry**: Consumer Tech  
**Complexity**: High  
**Key Features**: Graph, streaming, ML, governance, multi-device

---

## Overview

A social networking platform supporting:

- **User Profiles** - Bios, photos, settings
- **Social Graph** - Followers, friends, connections
- **Posts & Feed** - Rich content, real-time updates
- **Messaging** - Direct messages, group chats
- **Recommendations** - Friend suggestions
- **Content Moderation** - AI-powered safety

---

## Data Model

### User Profile

```yaml
dataType:
  name: UserProfile
  version: v1
  
  fields:
    user_id:
      type: string
      
    username:
      type: string
      constraints:
        unique: true
        pattern: "^[a-zA-Z0-9_]{3,30}$"
      search:
        indexed: true
        strategy: prefix
        suggest:
          enabled: true
          
    display_name:
      type: string
      search:
        indexed: true
        
    bio:
      type: string
      constraints:
        max_length: 500
        
    avatar:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
        variants:
          - name: small
            transform: resize
            params: { width: 50, height: 50, fit: cover }
          - name: medium
            transform: resize
            params: { width: 150, height: 150, fit: cover }
        access:
          read: public
          
    follower_count:
      type: integer
      
    following_count:
      type: integer
      
    is_verified:
      type: boolean
      presentation:
        display_as: badge
        
  governance:
    classification: pii
    retention:
      policy: event_based
      after_event: account_deleted
      grace_period: 30d
      on_expiry: delete
```

### Post

```yaml
dataType:
  name: Post
  version: v1
  
  fields:
    post_id:
      type: string
      
    author_id:
      type: string
      
    content:
      type: structured_content
      structured_content:
        schema: social_post
        blocks:
          allowed: [paragraph, image, video, poll]
        marks:
          allowed: [bold, italic, link, mention, hashtag]
        constraints:
          max_length: 5000
          
    media:
      type: array
      items:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 10MB
          variants:
            - name: thumbnail
              transform: resize
              params: { width: 200, height: 200 }
            - name: feed
              transform: resize
              params: { width: 600, height: 600 }
              
    visibility:
      type: enum
      values: [public, followers, private]
      
    like_count:
      type: integer
      
    comment_count:
      type: integer
      
    share_count:
      type: integer
      
    created_at:
      type: timestamp
      
    moderation_status:
      type: enum
      values: [pending, approved, flagged, removed]
      
    toxicity_score:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [content]
        model:
          name: content_moderator
          version: v3
        output:
          type: decimal
          range: { min: 0, max: 1 }
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: queue
```

---

## Social Graph

```yaml
relationship:
  name: Follows
  from: UserState
  to: UserState
  type: social
  cardinality: many-to-many
  
  recursion:
    enabled: true
    max_depth: 6
    direction: both
    cycle_handling: detect

graphQuery:
  name: MutualFollowersQuery
  version: v1
  description: "Find mutual connections between two users"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_a_id"
    
  pattern:
    type: neighbors
    neighbors:
      depth: 1
      direction: outbound
      
  filter: "user.id IN followers_of(request.user_b_id)"
  
  result:
    includes: [user_id, username, display_name, avatar]

graphQuery:
  name: FriendSuggestionsQuery
  version: v1
  description: "Find friends-of-friends for recommendations"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_id"
    
  pattern:
    type: path
    path:
      direction: both
      min_hops: 2
      max_hops: 2
      filter: "user.id NOT IN following_of(request.user_id)"
      
  result:
    includes: [user_id, username, display_name, avatar]
    
  aggregation:
    - type: count
      field: connection_paths
      as: mutual_count

graphQuery:
  name: InfluencerPathQuery
  version: v1
  description: "Find path to an influencer"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_id"
    
  pattern:
    type: shortest_path
    shortest_path:
      to: "user.id == request.influencer_id"
      max_cost: 6
      
  result:
    includes: [user_id, username, avatar]
    path_field: connection_chain
```

---

## Real-Time Feed

```yaml
materialization:
  name: UserFeedView
  version: v1
  source: [PostState, FollowsRelationship]
  targetState: UserFeedMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: user_id
    
    temporal:
      window:
        type: sliding
        size: 24h
        slide: 5m
        
      aggregation:
        - field: post_id
          function: collect
          as: feed_posts
          
      timestamp_field: created_at
      
      emit:
        trigger: periodic
        periodic_interval: 1m
        include_partial: true
        
    list:
      max_items: 100
      default_order: created_at
      order_direction: desc
      
    freshness:
      max_staleness: 30s

materialization:
  name: TrendingTopicsView
  version: v1
  source: PostState
  targetState: TrendingTopicsMaterialized
  
  retrieval:
    mode: singleton
    
    temporal:
      window:
        type: tumbling
        size: 1h
        
      aggregation:
        - field: hashtag
          function: count
          as: hashtag_count
          
      emit:
        trigger: on_window_close
        
    freshness:
      max_staleness: 5m
```

---

## Content Moderation

```yaml
smsFlow:
  name: ContentModerationFlow
  version: v1
  
  triggeredBy:
    inputIntent: CreatePostIntent
    
  steps:
    - work_unit: AnalyzeContent
      output:
        toxicity_score: "analysis.toxicity"
        
    - if: "toxicity_score > 0.9"
      then:
        - work_unit: AutoRemovePost
        - work_unit: NotifyUser
        - signal: ContentRemovedSignal
      else:
        - if: "toxicity_score > 0.7"
          then:
            - work_unit: FlagForReview
            - await:
                name: await_moderator_review
                trigger:
                  type: inputIntent
                  intent: ModerationDecisionIntent
                  assignment:
                    policy: round_robin
                    pool: ModeratorPool
                sla:
                  target: 4h
                result:
                  captures: [decision, reason]
          else:
            - work_unit: ApprovePost
            
    - work_unit: UpdateFeed
    
taskPool:
  name: ModeratorPool
  version: v1
  
  membership:
    type: role_based
    roles: [content_moderator, senior_moderator]
    
  capacity:
    max_concurrent_per_member: 20
```

---

## Privacy & Governance

```yaml
consentRecord:
  name: DataUsageConsent
  version: v1
  
  subject_field: user_id
  
  scopes:
    - name: personalization
      description: "Personalize your feed and recommendations"
      required: false
      default: true
      revocable: true
      
    - name: analytics
      description: "Help us improve with usage analytics"
      required: false
      default: true
      revocable: true
      
    - name: advertising
      description: "Show personalized ads"
      required: false
      default: false
      revocable: true
      
  purposes:
    personalization: [feed_ranking, friend_suggestions]
    analytics: [usage_patterns, feature_adoption]
    advertising: [ad_targeting, conversion_tracking]
    
  lifecycle:
    collection_point: OnboardingComposition
    expires_after: 365d
    renewal:
      type: prompt
      notice_before: 30d

erasureRequest:
  name: AccountDeletionRequest
  version: v1
  
  scope:
    subject_field: user_id
    
  strategy: hybrid
  
  hybrid:
    delete: [email, phone, bio, avatar, messages]
    anonymize: [posts, comments]
    retain: [user_id]
    
  cascade:
    behavior: automatic
    entities: [PostState, CommentState, MessageState]
    
  processing:
    sla: 720h
    
  completion:
    certificate: true
```

---

## Multi-Device

```yaml
experience:
  name: SocialApp
  version: v1
  
  entry_point: FeedScreen
  
  session:
    required: true
    
    devices:
      enabled: true
      max_active: 10
      
      state_scope:
        per_device:
          - feed.scroll_position
          - ui.theme
        shared:
          - draft_posts
          - notification_state
          
      handoff:
        enabled: true
        state_transfer: immediate
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: immediate
        field_strategies:
          - field: draft_posts
            strategy: union
          - field: likes
            strategy: union
            
      cache:
        views: [UserFeedView, ProfileView]
        max_size: 100MB
        ttl: 24h
```

---

## Key Patterns

### 1. Social Graph Queries

```yaml
pattern:
  type: path
  path:
    min_hops: 2
    max_hops: 2
```

### 2. ML Content Moderation

```yaml
toxicity_score:
  inference:
    model: content_moderator
    freshness:
      strategy: on_write
```

### 3. Trending Aggregation

```yaml
temporal:
  window:
    type: tumbling
    size: 1h
  aggregation:
    - field: hashtag
      function: count
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Smart Factory / Industrial IoT

**Example ID**: 10  
**Industry**: Manufacturing  
**Complexity**: High  
**Key Features**: IoT, streaming, ML, safety, compliance

---

## Overview

A smart factory platform supporting:

- **Production Monitoring** - Real-time machine status
- **Predictive Maintenance** - ML-based failure prediction
- **Quality Control** - Sensor-based defect detection
- **Safety Systems** - Interlocks and emergency stops
- **Energy Management** - Usage optimization
- **Compliance** - Audit trails, data retention

---

## Data Model

### Machine

```yaml
dataType:
  name: Machine
  version: v1
  
  fields:
    machine_id:
      type: string
      
    name:
      type: string
      
    machine_type:
      type: enum
      values: [cnc, robot, conveyor, press, welder, assembler]
      
    location:
      type: object
      properties:
        building: { type: string }
        floor: { type: integer }
        zone: { type: string }
        coordinates: { type: point }
        
    status:
      type: enum
      values: [running, idle, maintenance, fault, emergency_stop]
      presentation:
        display_as: badge
        semantic_variants:
          running: success
          idle: neutral
          maintenance: warning
          fault: error
          emergency_stop: error
          
    production_rate:
      type: decimal
      presentation:
        display_as: gauge
        
    oee_score:
      type: decimal
      description: "Overall Equipment Effectiveness"
      presentation:
        display_as: percentage
        
    last_maintenance:
      type: timestamp
      
    next_scheduled_maintenance:
      type: timestamp
      
    failure_probability:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [vibration_data, temperature_data, runtime_hours]
        model:
          name: predictive_maintenance
          version: v2
        output:
          type: decimal
          range: { min: 0, max: 1 }
        freshness:
          strategy: periodic
          interval: 1h
```

---

## IoT Sensors

### Vibration Sensor

```yaml
worker_class:
  name: VibrationSensor
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_vibration
      - read_frequency_spectrum
      
    hardware:
      category: industrial
      certifications: [ATEX, IECEx]
      
    constraints:
      power:
        source: poe
        
      connectivity:
        type: always_on
        protocols: [ethernet]
        
      environment:
        temperature_range: { min: -20, max: 80 }
        ip_rating: IP67
        
    health:
      heartbeat_interval: 1m
      offline_threshold: 5m

deviceCapability:
  name: read_vibration
  version: v1
  type: sensor
  
  sensor:
    measures: vibration
    output:
      type: decimal
      unit: mm/s
      precision: 3
      range: { min: 0, max: 100 }
    sampling:
      min_interval: 100ms
      max_interval: 1s

deviceTelemetry:
  name: VibrationStream
  version: v1
  
  source:
    device_type: VibrationSensor
    capability: read_vibration
    
  collection:
    mode: continuous
    continuous:
      interval: 1s
      
  aggregation:
    enabled: true
    window: 1m
    functions: [min, max, avg, stddev]
    
  delivery:
    guarantee: at_least_once
    compression: lz4
    batching:
      enabled: true
      max_size: 100
```

### Temperature Sensor

```yaml
worker_class:
  name: IndustrialTempSensor
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_temperature
      
    hardware:
      category: industrial
      certifications: [ATEX]
      
    constraints:
      environment:
        temperature_range: { min: -40, max: 200 }

deviceTelemetry:
  name: TemperatureStream
  version: v1
  
  source:
    device_type: IndustrialTempSensor
    capability: read_temperature
    
  collection:
    mode: threshold
    threshold:
      above: 80
      below: 5
      
  buffering:
    on_offline: buffer_local
    max_buffer_size: 10000
```

---

## Actuators with Safety

### Industrial Valve

```yaml
worker_class:
  name: IndustrialValve
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - set_valve_position
      - emergency_close
      - read_position
      
    hardware:
      category: industrial
      certifications: [SIL2, ATEX]
      
    health:
      heartbeat_interval: 1s
      offline_threshold: 5s

deviceCapability:
  name: set_valve_position
  version: v1
  type: actuator
  
  actuator:
    controls: valve
    input:
      type: integer
      range: { min: 0, max: 100 }
    feedback:
      confirmation: required
      timeout: 5s
      
  safety:
    constraints:
      - expression: "pressure_sensor.value < max_safe_pressure"
        on_violation: emergency_stop
      - expression: "rate_of_change(position) < 20"
        on_violation: reject
        
    interlocks:
      - condition: "upstream_valve.closed == true"
        action: prevent
      - condition: "emergency_mode == true"
        action: prevent

deviceCapability:
  name: emergency_close
  version: v1
  type: actuator
  
  actuator:
    controls: valve
    input:
      type: boolean
    feedback:
      confirmation: required
      timeout: 2s
      
  safety:
    constraints: []                    # No constraints - this IS safety

actuatorCommand:
  name: EmergencyShutdown
  version: v1
  
  target:
    device_type: IndustrialValve
    capability: emergency_close
    selector: "device.zone == request.zone"
    
  command:
    type: set_state
    set_state:
      value: true
      
  delivery:
    guarantee: exactly_once
    timeout: 2s
```

---

## Real-Time Analytics

```yaml
materialization:
  name: MachineHealthDashboard
  version: v1
  source: [VibrationStream, TemperatureStream, MachineState]
  targetState: MachineHealthMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: machine_id
    
    temporal:
      window:
        type: sliding
        size: 5m
        slide: 30s
        
      aggregation:
        - field: vibration
          function: avg
          as: avg_vibration
        - field: vibration
          function: stddev
          as: vibration_variance
        - field: temperature
          function: max
          as: max_temperature
          
      timestamp_field: reading_time
      watermark: 10s
      
      overflow:
        strategy: sample
        sample_rate: 0.1
        
      emit:
        trigger: periodic
        periodic_interval: 30s
        
    freshness:
      max_staleness: 1m

materialization:
  name: ProductionMetricsView
  version: v1
  source: ProductionEventState
  targetState: ProductionMetricsMaterialized
  
  retrieval:
    mode: singleton
    
    temporal:
      window:
        type: tumbling
        size: 1h
        
      aggregation:
        - field: units_produced
          function: sum
          as: hourly_production
        - field: defects
          function: count
          as: defect_count
        - field: cycle_time
          function: avg
          as: avg_cycle_time
          
      emit:
        trigger: on_window_close
```

---

## Equipment Hierarchy

```yaml
relationship:
  name: Part_Of
  from: EquipmentState
  to: EquipmentState
  type: hierarchical
  cardinality: many-to-one
  
  recursion:
    enabled: true
    max_depth: 10
    direction: up

graphQuery:
  name: EquipmentLineageQuery
  version: v1
  description: "Find all parent equipment"
  
  traverses: Part_Of
  
  start:
    from: EquipmentState
    where: "equipment.id == request.equipment_id"
    
  pattern:
    type: path
    path:
      direction: up
      max_hops: 10
      
  result:
    includes: [id, name, equipment_type]
    depth_field: hierarchy_level
```

---

## Predictive Maintenance Workflow

```yaml
smsFlow:
  name: PredictiveMaintenanceFlow
  version: v1
  
  triggeredBy:
    signal: HighFailureProbabilitySignal
    when: "machine.failure_probability > 0.8"
    
  durability:
    enabled: true
    state: MaintenanceFlowState
    ttl: 7d
    
  steps:
    - work_unit: AnalyzeFailureMode
    
    - work_unit: CreateMaintenanceOrder
    
    - await:
        name: await_technician_assignment
        trigger:
          type: inputIntent
          intent: AcceptMaintenanceOrderIntent
          assignment:
            policy: skill_based
            pool: MaintenanceTechPool
        sla:
          target: 4h
        escalation:
          - after: 4h
            action: notify
            to: MaintenanceManager
            
    - await:
        name: await_completion
        trigger:
          type: inputIntent
          intent: CompleteMaintenanceIntent
        sla:
          target: 24h
          
    - work_unit: UpdateMachineRecord
    - work_unit: RecordMaintenanceHistory

taskPool:
  name: MaintenanceTechPool
  version: v1
  
  membership:
    type: role_based
    roles: [maintenance_tech, senior_tech]
    
  skills:
    - name: mechanical
    - name: electrical
    - name: plc_programming
    - name: robotics
```

---

## Compliance & Governance

```yaml
dataState:
  name: ProductionLogState
  type: ProductionLog
  lifecycle: persistent
  
  governance:
    classification: financial
    retention:
      policy: time_based
      duration: 3650d             # 10 years
      archive:
        enabled: true
        after: 365d
        tier: cold
      on_expiry: archive
    audit:
      access_logging: true
      modification_logging: true
      immutable: true
```

---

## Key Patterns

### 1. Safety Interlocks

```yaml
safety:
  constraints:
    - expression: "pressure < max_safe"
      on_violation: emergency_stop
  interlocks:
    - condition: "upstream.closed"
      action: prevent
```

### 2. Streaming Aggregation with Sampling

```yaml
overflow:
  strategy: sample
  sample_rate: 0.1
```

### 3. ML Predictive Maintenance

```yaml
failure_probability:
  inference:
    model: predictive_maintenance
    freshness:
      strategy: periodic
      interval: 1h
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Legal Case Management

**Example ID**: 11  
**Industry**: Legal  
**Complexity**: High  
**Key Features**: Workflows, legal holds, delegation, search, compliance

---

## Overview

A legal case management platform supporting:

- **Matter Management** - Cases, clients, billing
- **Document Repository** - Storage, versioning, search
- **Workflow Automation** - Deadlines, approvals
- **E-Discovery** - Legal holds, collections
- **Time & Billing** - Time tracking, invoicing
- **Client Portal** - Secure document sharing

---

## Data Model

### Matter

```yaml
dataType:
  name: Matter
  version: v1
  
  fields:
    matter_id:
      type: string
      
    matter_number:
      type: string
      constraints:
        unique: true
        
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    client_id:
      type: string
      
    matter_type:
      type: enum
      values: [litigation, corporate, ip, real_estate, employment, regulatory]
      search:
        facet:
          enabled: true
          
    status:
      type: enum
      values: [active, pending, closed, on_hold]
      presentation:
        display_as: badge
        
    responsible_attorney:
      type: string
      
    practice_group:
      type: string
      search:
        facet:
          enabled: true
          
    opened_date:
      type: date
      
    closed_date:
      type: date
      
    billing_type:
      type: enum
      values: [hourly, fixed_fee, contingency, pro_bono]
      
  governance:
    classification: confidential
    retention:
      policy: time_based
      duration: 3650d             # 10 years
      archive:
        enabled: true
        after: 730d
        tier: cold
    audit:
      access_logging: true
      modification_logging: true
      immutable: true
```

### Legal Document

```yaml
dataType:
  name: LegalDocument
  version: v1
  
  fields:
    document_id:
      type: string
      
    matter_id:
      type: string
      
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 2.0
        
    document_type:
      type: enum
      values: [pleading, motion, brief, contract, correspondence, memo, evidence]
      search:
        facet:
          enabled: true
          
    content:
      type: structured_content
      structured_content:
        schema: legal_document
        blocks:
          allowed: [paragraph, heading, list, blockquote, table]
        marks:
          allowed: [bold, italic, underline, strikethrough, highlight]
          
    file:
      type: asset
      asset:
        category: document
        constraints:
          mime_types: [application/pdf, application/msword, application/vnd.openxmlformats-officedocument.wordprocessingml.document]
          max_size: 100MB
        lifecycle:
          type: permanent
        access:
          read: authenticated
          
    version:
      type: integer
      
    created_by:
      type: string
      
    created_at:
      type: timestamp
      
    tags:
      type: array
      items:
        type: string
      search:
        indexed: true
        
    privileged:
      type: boolean
      
    bates_start:
      type: string
      
    bates_end:
      type: string
```

---

## Attorney Delegation

```yaml
relationship:
  name: Delegates_Matter_Access
  from: AttorneyState
  to: AttorneyState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view_matter, edit_documents, bill_time, approve_expenses]
    constraints:
      transitive: false
      max_depth: 1
      same_realm: true
    requires:
      approval: grantor
    validity:
      type: temporal
      duration: 365d
    revocation:
      by: either

policy:
  name: MatterAccessPolicy
  version: v1
  
  appliesTo:
    type: dataState
    name: MatterState
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Delegates_Matter_Access
    max_chain: 1
    
  when: |
    (subject.id == matter.responsible_attorney) OR
    (subject.id IN matter.team_members) OR
    (delegation.active AND delegation.scope CONTAINS 'view_matter')
```

---

## Legal Holds

```yaml
legalHold:
  name: LitigationHold
  version: v1
  
  scope:
    entities: [LegalDocumentState, EmailState, MatterState]
    filter: "matter.id == hold.matter_id"
    
  reason: "Pending litigation - preservation required"
  reference: "hold.case_number"
  
  custodian: legal_department
  
  preserves:
    - dataState: LegalDocumentState
      fields: all
    - dataState: EmailState
      fields: all
    - dataState: MatterState
      fields: all
      
  overrides:
    retention: true
    erasure: true
    
  lifecycle:
    start_date: "hold.effective_date"
    review_interval: 90d
    
  release:
    requires_approval: true
    approvers: [partner, general_counsel]

inputIntent:
  name: InitiateLegalHoldIntent
  version: v1
  
  proposal:
    creates: LegalHoldState
    fields:
      matter_id:
        type: string
        required: true
      case_number:
        type: string
        required: true
      custodians:
        type: array
        required: true
      scope_description:
        type: string
        required: true
        
  notification:
    enabled: true
    trigger:
      on: intent_success
    channels:
      - type: email
        email:
          template: legal_hold_notice
          to_field: custodians
```

---

## Court Deadlines

```yaml
inputIntent:
  name: CourtDeadlineReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: deadline.due_date
        offset: 7d
    batch:
      enabled: true
      source:
        query: "deadline.status == 'pending'"
        
  notification:
    enabled: true
    channels:
      - type: email
        priority: 1
      - type: push
        priority: 2

smsFlow:
  name: DeadlineManagementFlow
  version: v1
  
  triggeredBy:
    inputIntent: CreateDeadlineIntent
    
  durability:
    enabled: true
    state: DeadlineFlowState
    ttl: 365d
    
  steps:
    - work_unit: ValidateDeadline
    
    - work_unit: CalculateInternalDeadlines
      output:
        internal_deadlines: "calculated_dates"
        
    - work_unit: AssignResponsibilities
    
    - for_each: "internal_deadlines"
      do:
        - work_unit: ScheduleReminder
        - await:
            name: await_task_completion
            trigger:
              type: inputIntent
              intent: CompleteDeadlineTaskIntent
              assignment:
                policy: direct
                subject: "task.assigned_to"
            sla:
              target: "task.due_date"
              warning_at: "task.due_date - 2d"
            escalation:
              - after: "task.due_date - 1d"
                action: notify
                to: "matter.responsible_attorney"
            timeout: "task.due_date + 1d"
            on_timeout: escalate
```

---

## Document Search

```yaml
searchIntent:
  name: DocumentSearchIntent
  version: v1
  
  searches: LegalDocumentState
  
  query:
    fields: [title, content, tags]
    strategy: full_text
    highlight:
      enabled: true
      fields: [title, content]
      
  filter:
    expression: "document.matter_id IN accessible_matters(subject.id)"
    
  facets:
    include: [document_type, practice_group, created_at]
    
  result:
    includes: [document_id, title, document_type, created_at, matter_id]
    max_results: 100
```

---

## Time Entry Workflow

```yaml
smsFlow:
  name: TimeEntryApprovalFlow
  version: v1
  
  triggeredBy:
    inputIntent: SubmitTimeEntryIntent
    
  durability:
    enabled: true
    state: TimeEntryFlowState
    ttl: 30d
    
  steps:
    - work_unit: ValidateTimeEntry
    
    - if: "time_entry.amount > 1000"
      then:
        - await:
            name: await_partner_approval
            trigger:
              type: inputIntent
              intent: ApproveTimeEntryIntent
              assignment:
                policy: direct
                subject: "matter.billing_partner"
            sla:
              target: 48h
            timeout: 7d
            on_timeout: continue_default
            default_result:
              decision: auto_approved
              
    - work_unit: UpdateBillingRecord
    
    - work_unit: UpdateMatterBalance
```

---

## Key Patterns

### 1. Legal Hold Preservation

```yaml
overrides:
  retention: true
  erasure: true
  
release:
  requires_approval: true
  approvers: [partner, general_counsel]
```

### 2. Court Deadline Reminders

```yaml
schedule:
  trigger:
    type: relative
    before_event: deadline.due_date
    offset: 7d
```

### 3. Matter Access Delegation

```yaml
delegation:
  scope: [view_matter, edit_documents, bill_time]
  same_realm: true
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Voice-First Smart Home

**Example ID**: 12  
**Industry**: Consumer IoT  
**Complexity**: Medium  
**Key Features**: Voice, IoT, device groups, temporal access, scheduling

---

## Overview

A voice-controlled smart home platform supporting:

- **Voice Control** - Natural language commands
- **Device Management** - Lights, HVAC, locks, cameras
- **Routines** - Scheduled and triggered automations
- **Guest Access** - Time-bound permissions
- **Energy Monitoring** - Usage tracking

---

## Data Model

### Home

```yaml
dataType:
  name: Home
  version: v1
  
  fields:
    home_id:
      type: string
      
    name:
      type: string
      
    address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        state: { type: string }
        zip: { type: string }
        
    location:
      type: point
      point:
        coordinate_system: wgs84
        
    timezone:
      type: string
      
    owner_id:
      type: string
      
    members:
      type: array
      items:
        type: object
        properties:
          user_id: { type: string }
          role: { type: enum, values: [owner, admin, member, guest] }
```

### Room

```yaml
dataType:
  name: Room
  version: v1
  
  fields:
    room_id:
      type: string
      
    home_id:
      type: string
      
    name:
      type: string
      
    room_type:
      type: enum
      values: [living_room, bedroom, kitchen, bathroom, garage, office, outdoor]
      
    floor:
      type: integer
```

---

## IoT Devices

### Smart Light

```yaml
worker_class:
  name: SmartLight
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - set_power
      - set_brightness
      - set_color
      
    constraints:
      power:
        source: mains
        
      connectivity:
        type: always_on
        protocols: [wifi, zigbee]
        
    health:
      heartbeat_interval: 5m

deviceCapability:
  name: set_brightness
  version: v1
  type: actuator
  
  actuator:
    controls: dimmer
    input:
      type: integer
      range: { min: 0, max: 100 }
    feedback:
      confirmation: optional
      timeout: 5s
```

### Smart Thermostat

```yaml
worker_class:
  name: SmartThermostat
  runtime: edge
  
  device:
    type: hybrid
    
    capabilities:
      - read_temperature
      - read_humidity
      - set_target_temperature
      - set_mode
      
    health:
      heartbeat_interval: 5m

deviceCapability:
  name: set_target_temperature
  version: v1
  type: actuator
  
  actuator:
    controls: hvac
    input:
      type: decimal
      range: { min: 50, max: 85 }
    feedback:
      confirmation: required
      
  safety:
    constraints:
      - expression: "target >= 50 AND target <= 85"
        on_violation: clamp
```

### Smart Lock

```yaml
worker_class:
  name: SmartDoorLock
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - lock
      - unlock
      - check_status
      - read_battery
      
    constraints:
      power:
        source: battery
        battery_capacity: 4000
        
      connectivity:
        type: always_on
        protocols: [ble, wifi]
        
    health:
      heartbeat_interval: 1h
      battery_low_threshold: 20

deviceCapability:
  name: unlock
  version: v1
  type: actuator
  
  actuator:
    controls: lock
    input:
      type: boolean
    feedback:
      confirmation: required
      timeout: 10s
      
  safety:
    constraints:
      - expression: "request.authorized == true"
        on_violation: reject
```

---

## Device Groups

```yaml
deviceGroup:
  name: LivingRoomLights
  version: v1
  
  membership:
    type: dynamic
    filter: "device.room_id == 'living_room' AND device.type == 'SmartLight'"
    
  properties:
    location:
      type: room
      room_id: living_room
      
  commands:
    broadcast: true
    sequential: false

deviceGroup:
  name: AllLights
  version: v1
  
  membership:
    type: dynamic
    filter: "device.type == 'SmartLight'"
    
  commands:
    broadcast: true

actuatorCommand:
  name: TurnOffAllLights
  version: v1
  
  target:
    device_type: SmartLight
    capability: set_power
    selector: "device IN group('AllLights')"
    
  command:
    type: set_state
    set_state:
      value: false
      
  delivery:
    guarantee: at_least_once
    timeout: 10s
```

---

## Voice Experience

```yaml
experience:
  name: SmartHomeVoice
  version: v1
  
  modality:
    type: voice
    voice:
      wake_word: "Hey Home"
      languages: [en-US]
      
  conversation:
    context:
      ttl: 5m
      carry_forward: [room_context, device_context]
      
    turn_taking:
      model: mixed
      timeout: 5s
      
    error_handling:
      no_match: reprompt
      no_input: close
      max_reprompts: 2
      
  entry_point: HomeAssistantDialog

presentationComposition:
  name: HomeAssistantDialog
  version: v1
  
  dialog:
    type: question
    
    prompts:
      initial: ReadyPrompt
      reprompt: DidntCatchPrompt
      help: HelpPrompt
      
    slots:
      - name: action
        maps_to: DeviceControlIntent.action
        required: true
        type: enum
        enum:
          values: [turn_on, turn_off, set, dim, brighten, lock, unlock]
        synonyms:
          turn_on: [switch on, enable, activate, power on]
          turn_off: [switch off, disable, deactivate, power off, kill]
          dim: [lower, reduce, decrease]
          brighten: [raise, increase]
          lock: [secure, close]
          unlock: [open]
          
      - name: device_target
        maps_to: DeviceControlIntent.target
        required: true
        type: entity
        entity:
          entity_type: DeviceEntity
        synonyms:
          living_room_lights: [living room, front room lights]
          bedroom_lights: [bedroom, master bedroom]
          all_lights: [all lights, every light, everywhere]
          thermostat: [temperature, heat, ac, air conditioning]
          front_door: [front lock, main door]
          
      - name: value
        maps_to: DeviceControlIntent.value
        required: false
        type: number
        elicitation:
          prompt: WhatValuePrompt
          
    confirmation:
      required: false
      
    fulfillment:
      intent: DeviceControlIntent
      response: ConfirmationPrompt
      
  transitions:
    on_complete: ReadyForMoreDialog
    on_help: HelpDialog

prompt:
  name: ReadyPrompt
  version: v1
  content:
    speech:
      ssml: |
        <speak>
          <audio src="listening_tone.mp3"/>
        </speak>

prompt:
  name: ConfirmationPrompt
  version: v1
  content:
    speech:
      ssml: "Done. The {{device_target}} is now {{action | past_tense}}."
  context_variables:
    - name: device_target
      from: slot
      field: device_target
    - name: action
      from: slot
      field: action
```

---

## Guest Access

```yaml
temporalAccess:
  name: GuestHomeAccess
  version: v1
  
  grants:
    policy: HomeDevicePolicy
    to: "request.guest_email"
    
  validity:
    type: time_bound
    starts_at: "request.access_start"
    expires_at: "request.access_end"
    
  constraints:
    device_bound: false
    
  revocation:
    on_breach: immediate

consentRecord:
  name: HomeDataConsent
  version: v1
  
  subject_field: user_id
  
  scopes:
    - name: voice_history
      description: "Store voice command history"
      required: false
      revocable: true
      
    - name: energy_analytics
      description: "Analyze energy usage patterns"
      required: false
      default: true
      revocable: true
```

---

## Scheduled Routines

```yaml
inputIntent:
  name: GoodNightRoutineIntent
  version: v1
  description: "Execute good night routine"
  
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 22 * * *"
        timezone: "home.timezone"
        
smsFlow:
  name: GoodNightRoutineFlow
  version: v1
  
  triggeredBy:
    inputIntent: GoodNightRoutineIntent
    
  steps:
    - work_unit: TurnOffAllLights
      target:
        group: AllLights
        
    - work_unit: LockAllDoors
      target:
        group: AllLocks
        
    - work_unit: SetThermostatNight
      params:
        temperature: 68
        
    - work_unit: ArmSecuritySystem
```

---

## Key Patterns

### 1. Voice Slot Synonyms

```yaml
synonyms:
  turn_on: [switch on, enable, activate]
  living_room_lights: [living room, front room]
```

### 2. Device Group Broadcast

```yaml
commands:
  broadcast: true
  sequential: false
```

### 3. Time-Bound Guest Access

```yaml
validity:
  type: time_bound
  starts_at: "request.access_start"
  expires_at: "request.access_end"
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Subscription Commerce Platform

**Example ID**: 13  
**Industry**: SaaS / Subscription  
**Complexity**: Medium  
**Key Features**: External APIs, webhooks, scheduling, metering, streaming

---

## Overview

A subscription management platform supporting:

- **Subscription Lifecycle** - Trials, upgrades, cancellations
- **Billing Integration** - Payment processing, invoicing
- **Usage Metering** - Track and bill by consumption
- **Churn Prediction** - ML-based risk scoring
- **Renewal Management** - Automated renewals and reminders

---

## Data Model

### Subscription

```yaml
dataType:
  name: Subscription
  version: v1
  
  fields:
    subscription_id:
      type: string
      
    customer_id:
      type: string
      
    plan_id:
      type: string
      
    status:
      type: enum
      values: [trial, active, past_due, cancelled, expired]
      presentation:
        display_as: badge
        semantic_variants:
          trial: info
          active: success
          past_due: warning
          cancelled: error
          expired: neutral
          
    billing_cycle:
      type: enum
      values: [monthly, quarterly, annual]
      
    current_period_start:
      type: timestamp
      
    current_period_end:
      type: timestamp
      
    trial_end:
      type: timestamp
      
    cancelled_at:
      type: timestamp
      
    cancel_at_period_end:
      type: boolean
      
    mrr:
      type: decimal
      description: "Monthly Recurring Revenue"
      presentation:
        display_as: currency
        
    payment_method_id:
      type: string
      
    churn_risk_score:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [customer_id, plan_id, usage_data, support_tickets]
        model:
          name: churn_predictor
          version: v2
        output:
          type: decimal
          range: { min: 0, max: 1 }
        freshness:
          strategy: periodic
          interval: 24h
        fallback:
          on_unavailable: default
          default_value: 0.5
```

### Usage Record

```yaml
dataType:
  name: UsageRecord
  version: v1
  
  fields:
    record_id:
      type: string
      
    subscription_id:
      type: string
      
    metric:
      type: string
      description: "e.g., api_calls, storage_gb, seats"
      
    quantity:
      type: decimal
      
    timestamp:
      type: timestamp
      
    idempotency_key:
      type: string
```

### Invoice

```yaml
dataType:
  name: Invoice
  version: v1
  
  fields:
    invoice_id:
      type: string
      
    subscription_id:
      type: string
      
    customer_id:
      type: string
      
    status:
      type: enum
      values: [draft, open, paid, void, uncollectible]
      
    amount_due:
      type: decimal
      presentation:
        display_as: currency
        
    amount_paid:
      type: decimal
      
    due_date:
      type: date
      
    paid_at:
      type: timestamp
      
    pdf:
      type: asset
      asset:
        category: document
        lifecycle:
          type: permanent
        access:
          read: authenticated
```

---

## Payment Integration

```yaml
externalDependency:
  name: StripeSubscriptions
  version: v1
  type: api
  
  capabilities:
    - create_subscription
    - update_subscription
    - cancel_subscription
    - create_invoice
    - charge_invoice
    - create_usage_record
    
  contract:
    create_subscription:
      input:
        schema:
          customer_id: { type: string, required: true }
          price_id: { type: string, required: true }
          trial_days: { type: integer }
      output:
        schema:
          subscription_id: { type: string }
          status: { type: string }
      errors:
        - code: payment_failed
          retryable: true
        - code: card_declined
          retryable: false
          
  sla:
    availability: 99.99%
    latency_p99: 2s
    
  resilience:
    timeout: 30s
    retry:
      enabled: true
      max_attempts: 3
      backoff:
        strategy: exponential
        initial: 500ms
    circuit_breaker:
      enabled: true
      threshold: 5
      reset_after: 60s
      
  fallback:
    on_unavailable: queue
    queue_ttl: 24h
    
  credentials:
    type: api_key
    secret_ref: stripe_secret_key

webhookReceiver:
  name: StripeSubscriptionWebhook
  version: v1
  
  source: StripeSubscriptions
  
  maps_to:
    inputIntent: SubscriptionEventIntent
    
  authentication:
    type: signature
    signature:
      algorithm: hmac_sha256
      header: Stripe-Signature
      secret_ref: stripe_webhook_secret
      timestamp_tolerance: 5m
      
  event_mapping:
    - external_event: customer.subscription.created
      intent_type: SubscriptionCreated
      field_mapping:
        subscription_id: "$.data.object.id"
        customer_id: "$.data.object.customer"
        status: "$.data.object.status"
        
    - external_event: customer.subscription.updated
      intent_type: SubscriptionUpdated
      field_mapping:
        subscription_id: "$.data.object.id"
        status: "$.data.object.status"
        cancel_at_period_end: "$.data.object.cancel_at_period_end"
        
    - external_event: invoice.payment_succeeded
      intent_type: PaymentSucceeded
      field_mapping:
        invoice_id: "$.data.object.id"
        subscription_id: "$.data.object.subscription"
        amount_paid: "$.data.object.amount_paid"
        
    - external_event: invoice.payment_failed
      intent_type: PaymentFailed
      field_mapping:
        invoice_id: "$.data.object.id"
        subscription_id: "$.data.object.subscription"
        
  idempotency:
    key_path: "$.id"
    ttl: 24h
```

---

## Usage Metering

```yaml
materialization:
  name: UsageAggregateView
  version: v1
  source: UsageRecordState
  targetState: UsageAggregateMaterialized
  
  retrieval:
    mode: by_query
    query_fields: [subscription_id, metric, period]
    
    temporal:
      window:
        type: tumbling
        size: 1h
        
      aggregation:
        - field: quantity
          function: sum
          as: total_quantity
        - field: record_id
          function: count
          as: record_count
          
      timestamp_field: timestamp
      
      emit:
        trigger: on_window_close
        
    freshness:
      max_staleness: 5m

inputIntent:
  name: ReportUsageIntent
  version: v1
  
  proposal:
    creates: UsageRecordState
    fields:
      subscription_id:
        type: string
        required: true
      metric:
        type: string
        required: true
      quantity:
        type: decimal
        required: true
      idempotency_key:
        type: string
        required: true
        
  delivery:
    guarantee: exactly_once
    deduplication:
      enabled: true
      key_field: idempotency_key
      ttl: 24h
```

---

## Scheduled Tasks

```yaml
inputIntent:
  name: RenewalReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: subscription.current_period_end
        offset: 7d
    batch:
      enabled: true
      source:
        query: "subscription.status == 'active' AND subscription.cancel_at_period_end == false"
        
  notification:
    enabled: true
    channels:
      - type: email
        email:
          template: renewal_reminder
          subject: "Your subscription renews in 7 days"

inputIntent:
  name: TrialEndingReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: subscription.trial_end
        offset: 3d
    batch:
      enabled: true
      source:
        query: "subscription.status == 'trial'"
        
  notification:
    enabled: true
    channels:
      - type: email
        priority: 1
      - type: push
        priority: 2

inputIntent:
  name: BillingCycleProcessIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 0 * * *"         # Daily at midnight
        timezone: UTC
    batch:
      enabled: true
      source:
        query: "subscription.current_period_end <= now() AND subscription.status == 'active'"
      chunking:
        size: 1000
        parallel: 4
```

---

## Subscription Workflow

```yaml
smsFlow:
  name: SubscriptionCancellationFlow
  version: v1
  
  triggeredBy:
    inputIntent: CancelSubscriptionIntent
    
  steps:
    - work_unit: ValidateCancellation
    
    - work_unit: CalculateProration
      output:
        refund_amount: "calculation.refund"
        
    - if: "intent.immediate == true"
      then:
        - work_unit: ProcessImmediateCancellation
        - if: "refund_amount > 0"
          then:
            - work_unit: ProcessRefund
              external_dependency: StripeSubscriptions
      else:
        - work_unit: SchedulePeriodEndCancellation
        
    - work_unit: UpdateSubscriptionRecord
    
    - signal: SubscriptionCancelledSignal
    
  notification:
    enabled: true
    trigger:
      on: flow_success
    channels:
      - type: email
        email:
          template: cancellation_confirmation
```

---

## Churn Prevention

```yaml
smsFlow:
  name: ChurnPreventionFlow
  version: v1
  
  triggeredBy:
    signal: HighChurnRiskSignal
    when: "subscription.churn_risk_score > 0.7"
    
  steps:
    - work_unit: AnalyzeChurnFactors
    
    - work_unit: DetermineIntervention
      output:
        intervention_type: "analysis.recommendation"
        
    - if: "intervention_type == 'discount'"
      then:
        - work_unit: GenerateDiscountOffer
        - work_unit: SendDiscountEmail
    else:
      - if: "intervention_type == 'outreach'"
        then:
          - work_unit: CreateOutreachTask
          - await:
              name: await_outreach_completion
              trigger:
                type: inputIntent
                intent: CompleteOutreachIntent
                assignment:
                  policy: round_robin
                  pool: CustomerSuccessPool
              timeout: 7d
              
    - work_unit: RecordIntervention
```

---

## Key Patterns

### 1. Webhook Signature Verification

```yaml
authentication:
  type: signature
  signature:
    algorithm: hmac_sha256
    timestamp_tolerance: 5m
```

### 2. Usage Metering with Idempotency

```yaml
delivery:
  guarantee: exactly_once
  deduplication:
    key_field: idempotency_key
    ttl: 24h
```

### 3. Relative Scheduled Triggers

```yaml
trigger:
  type: relative
  before_event: subscription.trial_end
  offset: 3d
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Collaborative Document Editor

**Example ID**: 14  
**Industry**: Productivity  
**Complexity**: High  
**Key Features**: Real-time collaboration, rich content, offline, multi-device

---

## Overview

A real-time collaborative document editor supporting:

- **Real-Time Editing** - Multiple cursors, live updates
- **Rich Content** - Formatting, tables, images, embeds
- **Version History** - Track changes, restore versions
- **Offline Editing** - Work without connection
- **Sharing & Permissions** - Granular access control
- **Comments & Discussions** - Threaded conversations

---

## Data Model

### Document

```yaml
dataType:
  name: Document
  version: v1
  
  fields:
    document_id:
      type: string
      
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    content:
      type: structured_content
      structured_content:
        schema: rich_document
        blocks:
          allowed:
            - paragraph
            - heading
            - list
            - blockquote
            - code_block
            - table
            - image
            - divider
            - callout
            - embed
        marks:
          allowed:
            - bold
            - italic
            - underline
            - strikethrough
            - code
            - link
            - highlight
            - comment
        nesting:
          max_depth: 5
        links:
          internal:
            enabled: true
            targets: [Document]
          external:
            enabled: true
            validate: true
        constraints:
          max_length: 500000
        structure:
          outline: true
          
    owner_id:
      type: string
      
    folder_id:
      type: string
      
    created_at:
      type: timestamp
      
    updated_at:
      type: timestamp
      
    version:
      type: integer
      
    word_count:
      type: integer
      
    cover_image:
      type: asset
      asset:
        category: image
        variants:
          - name: thumbnail
            transform: resize
            params: { width: 400, height: 300 }
        access:
          read: authenticated

dataState:
  name: DocumentState
  type: Document
  lifecycle: persistent
  
  authority:
    scope: entity
    entity_key: document_id
```

### Document Version

```yaml
dataType:
  name: DocumentVersion
  version: v1
  
  fields:
    version_id:
      type: string
      
    document_id:
      type: string
      
    version_number:
      type: integer
      
    content_snapshot:
      type: structured_content
      
    created_by:
      type: string
      
    created_at:
      type: timestamp
      
    change_summary:
      type: string
```

### Comment

```yaml
dataType:
  name: Comment
  version: v1
  
  fields:
    comment_id:
      type: string
      
    document_id:
      type: string
      
    parent_comment_id:
      type: string
      description: "For threaded replies"
      
    author_id:
      type: string
      
    content:
      type: string
      
    anchor:
      type: object
      properties:
        type: { type: enum, values: [range, point] }
        start: { type: integer }
        end: { type: integer }
        
    status:
      type: enum
      values: [open, resolved]
      
    created_at:
      type: timestamp
```

---

## Real-Time Collaboration

```yaml
collaborativeSession:
  name: DocumentEditSession
  version: v1
  description: "Real-time collaborative document editing"
  
  binds_to:
    dataState: DocumentState
    entity_key: document_id
    
  presence:
    enabled: true
    fields:
      cursor:
        type: position
        scope: per_client
      selection:
        type: range
        scope: per_client
      user_color:
        type: string
        scope: per_session
      user_name:
        type: string
        scope: per_session
      is_typing:
        type: boolean
        scope: per_client
    ttl: 30s
    broadcast:
      to: same_entity
      throttle: 50ms
      
  operations:
    mode: stream
    ordering:
      guarantee: causal
    conflict:
      hint: merge
    types:
      - name: insert
        payload:
          position: integer
          content: string
      - name: delete
        payload:
          position: integer
          length: integer
      - name: format
        payload:
          range: object
          mark: string
          value: any
      - name: replace_block
        payload:
          block_id: string
          block_data: object
          
  subscription:
    to: all
    state:
      mode: delta
      buffer_size: 100
      
  access:
    max_participants: 50
    roles:
      owner: [edit, share, delete, manage_comments]
      editor: [edit, comment]
      commenter: [comment, view]
      viewer: [view]
      
  lifecycle:
    auto_close_after: 1h
    on_all_leave: keep_open
    inactivity_timeout: 30m
```

---

## Permission Delegation

```yaml
relationship:
  name: Shared_With
  from: DocumentState
  to: SubjectState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view, comment, edit]
    constraints:
      transitive: false
      max_depth: 1
    requires:
      approval: none
    validity:
      type: permanent
    revocation:
      by: grantor

policy:
  name: DocumentAccessPolicy
  version: v1
  
  appliesTo:
    type: dataState
    name: DocumentState
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Shared_With
    max_chain: 1
    
  when: |
    (subject.id == document.owner_id) OR
    (delegation.active AND delegation.scope CONTAINS 'view')
```

---

## Offline Support

```yaml
experience:
  name: DocumentEditor
  version: v1
  
  entry_point: DocumentList
  
  includes:
    - DocumentList
    - DocumentEditor
    - VersionHistory
    - Comments
    - Sharing
    
  session:
    required: true
    
    subject_binding:
      mode: authenticated
      
    lifetime:
      mode: sliding
      idle_timeout: 30d
      
    devices:
      enabled: true
      max_active: 10
      
      identification:
        by: device_id
        
      state_scope:
        per_device:
          - ui.sidebar_collapsed
          - ui.zoom_level
          - editor.cursor_position
        shared:
          - documents
          - folders
          - recent_documents
          
      handoff:
        enabled: true
        state_transfer: immediate
        transferable:
          - current_document_id
          - editor.cursor_position
          - editor.scroll_position
          
      concurrent_limits:
        on_new_device: allow
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: background
        conflict_resolution: merge
        
        field_strategies:
          - field: document.content
            strategy: merge
          - field: document.title
            strategy: server_wins
          - field: comments
            strategy: union
            
      queue:
        max_size: 1000
        max_age: 30d
        on_overflow: reject_new
        
      cache:
        views: [RecentDocumentsView, FolderContentsView]
        max_size: 200MB
        ttl: 168h
```

---

## Version History

```yaml
inputIntent:
  name: CreateVersionSnapshotIntent
  version: v1
  
  proposal:
    creates: DocumentVersionState
    fields:
      document_id:
        type: string
        required: true
      change_summary:
        type: string
        required: false

smsFlow:
  name: AutoVersionFlow
  version: v1
  description: "Automatically create version snapshots"
  
  triggeredBy:
    schedule:
      type: interval
      interval: 1h
      
  steps:
    - work_unit: FindModifiedDocuments
      output:
        modified_docs: "query.results"
        
    - for_each: "modified_docs"
      do:
        - work_unit: CreateVersionSnapshot
        - work_unit: PruneOldVersions
          params:
            keep_count: 100
```

---

## Search

```yaml
searchIntent:
  name: DocumentSearchIntent
  version: v1
  
  searches: DocumentState
  
  query:
    fields: [title, content]
    strategy: full_text
    highlight:
      enabled: true
      fields: [title, content]
      
  filter:
    expression: "document.id IN accessible_documents(subject.id)"
    
  result:
    includes: [document_id, title, updated_at, owner_id, cover_image]
    max_results: 50
    
  suggestions:
    enabled: true
    field: title
```

---

## Embedded Assets

```yaml
inputIntent:
  name: UploadDocumentImageIntent
  version: v1
  
  proposal:
    creates: DocumentAssetState
    fields:
      document_id:
        type: string
        required: true
      image:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 10MB
            mime_types: [image/jpeg, image/png, image/gif, image/webp]
          variants:
            - name: inline
              transform: resize
              params: { width: 800, max_height: 600 }
            - name: thumbnail
              transform: resize
              params: { width: 100, height: 100 }
          access:
            read: authenticated
```

---

## Key Patterns

### 1. Real-Time Presence with Throttling

```yaml
presence:
  fields:
    cursor:
      type: position
      scope: per_client
  broadcast:
    to: same_entity
    throttle: 50ms
```

### 2. Causal Operation Ordering

```yaml
operations:
  mode: stream
  ordering:
    guarantee: causal
  conflict:
    hint: merge
```

### 3. Offline Content Merge

```yaml
field_strategies:
  - field: document.content
    strategy: merge
  - field: comments
    strategy: union
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

# Government Permitting System

**Example ID**: 15  
**Industry**: Government  
**Complexity**: High  
**Key Features**: Durable workflows, spatial, accessibility, compliance

---

## Overview

A government permitting system supporting:

- **Permit Applications** - Online submission with documents
- **Multi-Department Review** - Parallel and sequential approvals
- **Public Hearings** - Scheduling and notifications
- **Fee Collection** - Online payments
- **GIS Integration** - Property lookups, zoning
- **Accessibility** - WCAG 2.1 AA compliance

---

## Data Model

### Permit Application

```yaml
dataType:
  name: PermitApplication
  version: v1
  
  fields:
    application_id:
      type: string
      
    application_number:
      type: string
      constraints:
        unique: true
      presentation:
        display_as: identifier
        
    permit_type:
      type: enum
      values: [building, electrical, plumbing, mechanical, demolition, grading, special_use]
      search:
        facet:
          enabled: true
          
    status:
      type: enum
      values: [draft, submitted, under_review, pending_info, approved, denied, expired, revoked]
      presentation:
        display_as: badge
        semantic_variants:
          draft: neutral
          submitted: info
          under_review: warning
          pending_info: warning
          approved: success
          denied: error
          expired: neutral
          revoked: error
      accessibility:
        label:
          template: "Application status: {{value}}"
          
    applicant_id:
      type: string
      
    property_address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        state: { type: string }
        zip: { type: string }
        
    property_location:
      type: point
      point:
        coordinate_system: wgs84
        search:
          indexed: true
          strategy: spatial
          
    parcel_id:
      type: string
      
    project_description:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    estimated_value:
      type: decimal
      presentation:
        display_as: currency
        
    documents:
      type: array
      items:
        type: asset
        asset:
          category: document
          constraints:
            mime_types: [application/pdf, image/jpeg, image/png]
            max_size: 50MB
          lifecycle:
            type: permanent
          access:
            read: authenticated
            
    submitted_at:
      type: timestamp
      
    decision_date:
      type: timestamp
      
  governance:
    classification: public
    retention:
      policy: time_based
      duration: 3650d             # 10 years
      archive:
        enabled: true
        after: 730d
        tier: cold
    audit:
      access_logging: true
      modification_logging: true
```

### Department Review

```yaml
dataType:
  name: DepartmentReview
  version: v1
  
  fields:
    review_id:
      type: string
      
    application_id:
      type: string
      
    department:
      type: enum
      values: [planning, building, fire, public_works, health, environmental]
      
    reviewer_id:
      type: string
      
    status:
      type: enum
      values: [pending, in_progress, approved, denied, needs_revision]
      
    comments:
      type: string
      
    conditions:
      type: array
      items:
        type: string
        
    reviewed_at:
      type: timestamp
```

### Zoning District

```yaml
dataType:
  name: ZoningDistrict
  version: v1
  
  fields:
    district_id:
      type: string
      
    name:
      type: string
      
    code:
      type: string
      
    boundary:
      type: polygon
      polygon:
        coordinate_system: wgs84
        presentation:
          display_as: region
          fill_opacity: 0.2
          stroke_color: "#0066CC"
          
    permitted_uses:
      type: array
      items:
        type: string
        
    max_height_feet:
      type: integer
      
    max_lot_coverage:
      type: decimal
```

---

## Spatial Queries

```yaml
searchIntent:
  name: ZoningLookupIntent
  version: v1
  description: "Find zoning district for a property"
  
  searches: ZoningDistrictState
  
  query:
    spatial:
      field: boundary
      type: contains_point
      contains_point:
        point: "request.property_location"
        
  result:
    includes: [district_id, name, code, permitted_uses, max_height_feet]

searchIntent:
  name: NearbyPermitsIntent
  version: v1
  description: "Find permits near a location"
  
  searches: PermitApplicationState
  
  query:
    spatial:
      field: property_location
      type: within_distance
      within_distance:
        from: "request.location"
        max_distance: 500m
    sort:
      by_distance_from: "request.location"
      
  filter:
    expression: "application.status IN ['submitted', 'under_review', 'approved']"
    
  result:
    includes: [application_id, application_number, permit_type, status, property_address]
```

---

## Multi-Department Workflow

```yaml
smsFlow:
  name: PermitReviewFlow
  version: v1
  description: "Multi-department permit review process"
  
  triggeredBy:
    inputIntent: SubmitPermitApplicationIntent
    
  durability:
    enabled: true
    state: PermitReviewFlowState
    ttl: 365d
    versioning:
      strategy: run_to_completion
      
  steps:
    - work_unit: ValidateApplication
    
    - work_unit: CalculateFees
      output:
        fee_amount: "calculation.total"
        
    - await:
        name: await_fee_payment
        trigger:
          type: inputIntent
          intent: PayPermitFeeIntent
          assignment:
            policy: direct
            subject: "application.applicant_id"
        sla:
          target: 30d
        timeout: 30d
        on_timeout: cancel
        
    - checkpoint: post_payment
    
    - work_unit: DetermineRequiredReviews
      output:
        required_departments: "routing.departments"
        
    # Parallel department reviews
    - parallel:
        - for_each: "required_departments"
          do:
            - await:
                name: await_department_review
                description: "Review by {{department.name}}"
                trigger:
                  type: inputIntent
                  intent: CompleteDepartmentReviewIntent
                  assignment:
                    policy: round_robin
                    pool: "{{department.code}}ReviewerPool"
                sla:
                  target: 14d
                  warning_at: 10d
                escalation:
                  - after: 14d
                    action: notify
                    to: "{{department.code}}Supervisor"
                  - after: 21d
                    action: escalate_pool
                    to: SeniorReviewerPool
                timeout: 30d
                result:
                  captures: [decision, conditions, comments]
                  
    - checkpoint: post_reviews
    
    - work_unit: ConsolidateReviews
      output:
        all_approved: "consolidation.all_approved"
        combined_conditions: "consolidation.conditions"
        
    - if: "all_approved"
      then:
        - if: "application.requires_hearing"
          then:
            - work_unit: SchedulePublicHearing
            - await:
                name: await_hearing_decision
                trigger:
                  type: inputIntent
                  intent: RecordHearingDecisionIntent
                sla:
                  target: 60d
                result:
                  captures: [hearing_decision, hearing_conditions]
                  
        - work_unit: IssuePermit
        - work_unit: NotifyApplicant
        - signal: PermitIssuedSignal
        
      else:
        - work_unit: CompileDenialReasons
        - work_unit: NotifyDenial
        - signal: PermitDeniedSignal
        
  checkpoints:
    - name: post_payment
      after_step: await_fee_payment
    - name: post_reviews
      after_step: parallel
      state_snapshot: full
      
  compensation:
    - from_checkpoint: post_payment
      do: RefundFees
    - from_checkpoint: post_reviews
      do: RevokeApprovals

taskPool:
  name: PlanningReviewerPool
  version: v1
  
  membership:
    type: role_based
    roles: [planning_reviewer, senior_planner]
    
  capacity:
    max_concurrent_per_member: 15
    
  availability:
    schedule: "0 8-17 * * 1-5"
    timezone: America/New_York
    
  skills:
    - name: residential
    - name: commercial
    - name: special_use
```

---

## Payment Integration

```yaml
externalDependency:
  name: GovPaymentProcessor
  version: v1
  type: api
  
  capabilities:
    - create_payment
    - check_status
    - process_refund
    
  resilience:
    timeout: 30s
    circuit_breaker:
      enabled: true
      threshold: 5
      
  credentials:
    type: api_key
    secret_ref: gov_pay_api_key
```

---

## Accessibility

```yaml
experience:
  name: PermitPortal
  version: v1
  description: "Public permit application portal"
  
  entry_point: PermitHome
  
  includes:
    - PermitHome
    - ApplicationForm
    - ApplicationStatus
    - DocumentUpload
    - PaymentForm
    - PermitSearch
    
  session:
    required: false
    
    subject_binding:
      mode: either
      
  accessibility:
    presets:
      - name: standard
        settings: {}
      - name: vision_impaired
        settings:
          font_scale: 1.5
          high_contrast: true
          motion: none
      - name: motor_impaired
        settings:
          focus_indicators: enhanced
          click_targets: large
          
  preferences:
    schema:
      language:
        type: enum
        values: [en, es, zh, vi, ko]
        default: en
      text_size:
        type: enum
        values: [normal, large, extra_large]
        default: normal

presentationView:
  name: ApplicationFormView
  version: v1
  
  accessibility:
    landmark: main
    skip_link:
      enabled: true
      label: "Skip to application form"
    keyboard_navigation:
      tab_order: sequential
      shortcuts:
        - key: "Alt+S"
          action: submit
          label: "Submit application"
        - key: "Alt+D"
          action: save_draft
          label: "Save as draft"
```

---

## Public Records Search

```yaml
searchIntent:
  name: PublicPermitSearchIntent
  version: v1
  
  searches: PermitApplicationState
  
  query:
    fields: [application_number, property_address, project_description]
    strategy: full_text
    
  filter:
    expression: "application.status NOT IN ['draft']"
    
  facets:
    include: [permit_type, status]
    
  result:
    includes: [application_id, application_number, permit_type, status, property_address, submitted_at]
```

---

## Scheduled Notifications

```yaml
inputIntent:
  name: PermitExpirationReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: permit.expiration_date
        offset: 30d
    batch:
      enabled: true
      source:
        query: "permit.status == 'approved'"
        
  notification:
    enabled: true
    channels:
      - type: email
        email:
          template: permit_expiration_reminder
      - type: mail
        mail:
          template: permit_expiration_letter
```

---

## Key Patterns

### 1. Parallel Department Reviews

```yaml
parallel:
  - for_each: "required_departments"
    do:
      - await:
          assignment:
            pool: "{{department.code}}ReviewerPool"
```

### 2. Zoning Polygon Containment

```yaml
spatial:
  field: boundary
  type: contains_point
  contains_point:
    point: "request.property_location"
```

### 3. Accessibility-First Design

```yaml
accessibility:
  landmark: main
  skip_link:
    enabled: true
  keyboard_navigation:
    shortcuts:
      - key: "Alt+S"
        action: submit
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 09: Social Networking Platform

# Social Networking Platform

**Example ID**: 09  
**Industry**: Consumer Tech  
**Complexity**: High  
**Key Features**: Graph, streaming, ML, governance, multi-device

---

## Overview

A social networking platform supporting:

- **User Profiles** - Bios, photos, settings
- **Social Graph** - Followers, friends, connections
- **Posts & Feed** - Rich content, real-time updates
- **Messaging** - Direct messages, group chats
- **Recommendations** - Friend suggestions
- **Content Moderation** - AI-powered safety

---

## Data Model

### User Profile

```yaml
dataType:
  name: UserProfile
  version: v1
  
  fields:
    user_id:
      type: string
      
    username:
      type: string
      constraints:
        unique: true
        pattern: "^[a-zA-Z0-9_]{3,30}$"
      search:
        indexed: true
        strategy: prefix
        suggest:
          enabled: true
          
    display_name:
      type: string
      search:
        indexed: true
        
    bio:
      type: string
      constraints:
        max_length: 500
        
    avatar:
      type: asset
      asset:
        category: image
        constraints:
          max_size: 5MB
        variants:
          - name: small
            transform: resize
            params: { width: 50, height: 50, fit: cover }
          - name: medium
            transform: resize
            params: { width: 150, height: 150, fit: cover }
        access:
          read: public
          
    follower_count:
      type: integer
      
    following_count:
      type: integer
      
    is_verified:
      type: boolean
      presentation:
        display_as: badge
        
  governance:
    classification: pii
    retention:
      policy: event_based
      after_event: account_deleted
      grace_period: 30d
      on_expiry: delete
```

### Post

```yaml
dataType:
  name: Post
  version: v1
  
  fields:
    post_id:
      type: string
      
    author_id:
      type: string
      
    content:
      type: structured_content
      structured_content:
        schema: social_post
        blocks:
          allowed: [paragraph, image, video, poll]
        marks:
          allowed: [bold, italic, link, mention, hashtag]
        constraints:
          max_length: 5000
          
    media:
      type: array
      items:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 10MB
          variants:
            - name: thumbnail
              transform: resize
              params: { width: 200, height: 200 }
            - name: feed
              transform: resize
              params: { width: 600, height: 600 }
              
    visibility:
      type: enum
      values: [public, followers, private]
      
    like_count:
      type: integer
      
    comment_count:
      type: integer
      
    share_count:
      type: integer
      
    created_at:
      type: timestamp
      
    moderation_status:
      type: enum
      values: [pending, approved, flagged, removed]
      
    toxicity_score:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [content]
        model:
          name: content_moderator
          version: v3
        output:
          type: decimal
          range: { min: 0, max: 1 }
        freshness:
          strategy: on_write
        fallback:
          on_unavailable: queue
```

---

## Social Graph

```yaml
relationship:
  name: Follows
  from: UserState
  to: UserState
  type: social
  cardinality: many-to-many
  
  recursion:
    enabled: true
    max_depth: 6
    direction: both
    cycle_handling: detect

graphQuery:
  name: MutualFollowersQuery
  version: v1
  description: "Find mutual connections between two users"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_a_id"
    
  pattern:
    type: neighbors
    neighbors:
      depth: 1
      direction: outbound
      
  filter: "user.id IN followers_of(request.user_b_id)"
  
  result:
    includes: [user_id, username, display_name, avatar]

graphQuery:
  name: FriendSuggestionsQuery
  version: v1
  description: "Find friends-of-friends for recommendations"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_id"
    
  pattern:
    type: path
    path:
      direction: both
      min_hops: 2
      max_hops: 2
      filter: "user.id NOT IN following_of(request.user_id)"
      
  result:
    includes: [user_id, username, display_name, avatar]
    
  aggregation:
    - type: count
      field: connection_paths
      as: mutual_count

graphQuery:
  name: InfluencerPathQuery
  version: v1
  description: "Find path to an influencer"
  
  traverses: Follows
  
  start:
    from: UserState
    where: "user.id == request.user_id"
    
  pattern:
    type: shortest_path
    shortest_path:
      to: "user.id == request.influencer_id"
      max_cost: 6
      
  result:
    includes: [user_id, username, avatar]
    path_field: connection_chain
```

---

## Real-Time Feed

```yaml
materialization:
  name: UserFeedView
  version: v1
  source: [PostState, FollowsRelationship]
  targetState: UserFeedMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: user_id
    
    temporal:
      window:
        type: sliding
        size: 24h
        slide: 5m
        
      aggregation:
        - field: post_id
          function: collect
          as: feed_posts
          
      timestamp_field: created_at
      
      emit:
        trigger: periodic
        periodic_interval: 1m
        include_partial: true
        
    list:
      max_items: 100
      default_order: created_at
      order_direction: desc
      
    freshness:
      max_staleness: 30s

materialization:
  name: TrendingTopicsView
  version: v1
  source: PostState
  targetState: TrendingTopicsMaterialized
  
  retrieval:
    mode: singleton
    
    temporal:
      window:
        type: tumbling
        size: 1h
        
      aggregation:
        - field: hashtag
          function: count
          as: hashtag_count
          
      emit:
        trigger: on_window_close
        
    freshness:
      max_staleness: 5m
```

---

## Content Moderation

```yaml
smsFlow:
  name: ContentModerationFlow
  version: v1
  
  triggeredBy:
    inputIntent: CreatePostIntent
    
  steps:
    - work_unit: AnalyzeContent
      output:
        toxicity_score: "analysis.toxicity"
        
    - if: "toxicity_score > 0.9"
      then:
        - work_unit: AutoRemovePost
        - work_unit: NotifyUser
        - signal: ContentRemovedSignal
      else:
        - if: "toxicity_score > 0.7"
          then:
            - work_unit: FlagForReview
            - await:
                name: await_moderator_review
                trigger:
                  type: inputIntent
                  intent: ModerationDecisionIntent
                  assignment:
                    policy: round_robin
                    pool: ModeratorPool
                sla:
                  target: 4h
                result:
                  captures: [decision, reason]
          else:
            - work_unit: ApprovePost
            
    - work_unit: UpdateFeed
    
taskPool:
  name: ModeratorPool
  version: v1
  
  membership:
    type: role_based
    roles: [content_moderator, senior_moderator]
    
  capacity:
    max_concurrent_per_member: 20
```

---

## Privacy & Governance

```yaml
consentRecord:
  name: DataUsageConsent
  version: v1
  
  subject_field: user_id
  
  scopes:
    - name: personalization
      description: "Personalize your feed and recommendations"
      required: false
      default: true
      revocable: true
      
    - name: analytics
      description: "Help us improve with usage analytics"
      required: false
      default: true
      revocable: true
      
    - name: advertising
      description: "Show personalized ads"
      required: false
      default: false
      revocable: true
      
  purposes:
    personalization: [feed_ranking, friend_suggestions]
    analytics: [usage_patterns, feature_adoption]
    advertising: [ad_targeting, conversion_tracking]
    
  lifecycle:
    collection_point: OnboardingComposition
    expires_after: 365d
    renewal:
      type: prompt
      notice_before: 30d

erasureRequest:
  name: AccountDeletionRequest
  version: v1
  
  scope:
    subject_field: user_id
    
  strategy: hybrid
  
  hybrid:
    delete: [email, phone, bio, avatar, messages]
    anonymize: [posts, comments]
    retain: [user_id]
    
  cascade:
    behavior: automatic
    entities: [PostState, CommentState, MessageState]
    
  processing:
    sla: 720h
    
  completion:
    certificate: true
```

---

## Multi-Device

```yaml
experience:
  name: SocialApp
  version: v1
  
  entry_point: FeedScreen
  
  session:
    required: true
    
    devices:
      enabled: true
      max_active: 10
      
      state_scope:
        per_device:
          - feed.scroll_position
          - ui.theme
        shared:
          - draft_posts
          - notification_state
          
      handoff:
        enabled: true
        state_transfer: immediate
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: immediate
        field_strategies:
          - field: draft_posts
            strategy: union
          - field: likes
            strategy: union
            
      cache:
        views: [UserFeedView, ProfileView]
        max_size: 100MB
        ttl: 24h
```

---

## Key Patterns

### 1. Social Graph Queries

```yaml
pattern:
  type: path
  path:
    min_hops: 2
    max_hops: 2
```

### 2. ML Content Moderation

```yaml
toxicity_score:
  inference:
    model: content_moderator
    freshness:
      strategy: on_write
```

### 3. Trending Aggregation

```yaml
temporal:
  window:
    type: tumbling
    size: 1h
  aggregation:
    - field: hashtag
      function: count
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 10: Smart Factory / Industrial IoT

# Smart Factory / Industrial IoT

**Example ID**: 10  
**Industry**: Manufacturing  
**Complexity**: High  
**Key Features**: IoT, streaming, ML, safety, compliance

---

## Overview

A smart factory platform supporting:

- **Production Monitoring** - Real-time machine status
- **Predictive Maintenance** - ML-based failure prediction
- **Quality Control** - Sensor-based defect detection
- **Safety Systems** - Interlocks and emergency stops
- **Energy Management** - Usage optimization
- **Compliance** - Audit trails, data retention

---

## Data Model

### Machine

```yaml
dataType:
  name: Machine
  version: v1
  
  fields:
    machine_id:
      type: string
      
    name:
      type: string
      
    machine_type:
      type: enum
      values: [cnc, robot, conveyor, press, welder, assembler]
      
    location:
      type: object
      properties:
        building: { type: string }
        floor: { type: integer }
        zone: { type: string }
        coordinates: { type: point }
        
    status:
      type: enum
      values: [running, idle, maintenance, fault, emergency_stop]
      presentation:
        display_as: badge
        semantic_variants:
          running: success
          idle: neutral
          maintenance: warning
          fault: error
          emergency_stop: error
          
    production_rate:
      type: decimal
      presentation:
        display_as: gauge
        
    oee_score:
      type: decimal
      description: "Overall Equipment Effectiveness"
      presentation:
        display_as: percentage
        
    last_maintenance:
      type: timestamp
      
    next_scheduled_maintenance:
      type: timestamp
      
    failure_probability:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [vibration_data, temperature_data, runtime_hours]
        model:
          name: predictive_maintenance
          version: v2
        output:
          type: decimal
          range: { min: 0, max: 1 }
        freshness:
          strategy: periodic
          interval: 1h
```

---

## IoT Sensors

### Vibration Sensor

```yaml
worker_class:
  name: VibrationSensor
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_vibration
      - read_frequency_spectrum
      
    hardware:
      category: industrial
      certifications: [ATEX, IECEx]
      
    constraints:
      power:
        source: poe
        
      connectivity:
        type: always_on
        protocols: [ethernet]
        
      environment:
        temperature_range: { min: -20, max: 80 }
        ip_rating: IP67
        
    health:
      heartbeat_interval: 1m
      offline_threshold: 5m

deviceCapability:
  name: read_vibration
  version: v1
  type: sensor
  
  sensor:
    measures: vibration
    output:
      type: decimal
      unit: mm/s
      precision: 3
      range: { min: 0, max: 100 }
    sampling:
      min_interval: 100ms
      max_interval: 1s

deviceTelemetry:
  name: VibrationStream
  version: v1
  
  source:
    device_type: VibrationSensor
    capability: read_vibration
    
  collection:
    mode: continuous
    continuous:
      interval: 1s
      
  aggregation:
    enabled: true
    window: 1m
    functions: [min, max, avg, stddev]
    
  delivery:
    guarantee: at_least_once
    compression: lz4
    batching:
      enabled: true
      max_size: 100
```

### Temperature Sensor

```yaml
worker_class:
  name: IndustrialTempSensor
  runtime: embedded
  
  device:
    type: sensor
    
    capabilities:
      - read_temperature
      
    hardware:
      category: industrial
      certifications: [ATEX]
      
    constraints:
      environment:
        temperature_range: { min: -40, max: 200 }

deviceTelemetry:
  name: TemperatureStream
  version: v1
  
  source:
    device_type: IndustrialTempSensor
    capability: read_temperature
    
  collection:
    mode: threshold
    threshold:
      above: 80
      below: 5
      
  buffering:
    on_offline: buffer_local
    max_buffer_size: 10000
```

---

## Actuators with Safety

### Industrial Valve

```yaml
worker_class:
  name: IndustrialValve
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - set_valve_position
      - emergency_close
      - read_position
      
    hardware:
      category: industrial
      certifications: [SIL2, ATEX]
      
    health:
      heartbeat_interval: 1s
      offline_threshold: 5s

deviceCapability:
  name: set_valve_position
  version: v1
  type: actuator
  
  actuator:
    controls: valve
    input:
      type: integer
      range: { min: 0, max: 100 }
    feedback:
      confirmation: required
      timeout: 5s
      
  safety:
    constraints:
      - expression: "pressure_sensor.value < max_safe_pressure"
        on_violation: emergency_stop
      - expression: "rate_of_change(position) < 20"
        on_violation: reject
        
    interlocks:
      - condition: "upstream_valve.closed == true"
        action: prevent
      - condition: "emergency_mode == true"
        action: prevent

deviceCapability:
  name: emergency_close
  version: v1
  type: actuator
  
  actuator:
    controls: valve
    input:
      type: boolean
    feedback:
      confirmation: required
      timeout: 2s
      
  safety:
    constraints: []                    # No constraints - this IS safety

actuatorCommand:
  name: EmergencyShutdown
  version: v1
  
  target:
    device_type: IndustrialValve
    capability: emergency_close
    selector: "device.zone == request.zone"
    
  command:
    type: set_state
    set_state:
      value: true
      
  delivery:
    guarantee: exactly_once
    timeout: 2s
```

---

## Real-Time Analytics

```yaml
materialization:
  name: MachineHealthDashboard
  version: v1
  source: [VibrationStream, TemperatureStream, MachineState]
  targetState: MachineHealthMaterialized
  
  retrieval:
    mode: by_entity
    entity_key: machine_id
    
    temporal:
      window:
        type: sliding
        size: 5m
        slide: 30s
        
      aggregation:
        - field: vibration
          function: avg
          as: avg_vibration
        - field: vibration
          function: stddev
          as: vibration_variance
        - field: temperature
          function: max
          as: max_temperature
          
      timestamp_field: reading_time
      watermark: 10s
      
      overflow:
        strategy: sample
        sample_rate: 0.1
        
      emit:
        trigger: periodic
        periodic_interval: 30s
        
    freshness:
      max_staleness: 1m

materialization:
  name: ProductionMetricsView
  version: v1
  source: ProductionEventState
  targetState: ProductionMetricsMaterialized
  
  retrieval:
    mode: singleton
    
    temporal:
      window:
        type: tumbling
        size: 1h
        
      aggregation:
        - field: units_produced
          function: sum
          as: hourly_production
        - field: defects
          function: count
          as: defect_count
        - field: cycle_time
          function: avg
          as: avg_cycle_time
          
      emit:
        trigger: on_window_close
```

---

## Equipment Hierarchy

```yaml
relationship:
  name: Part_Of
  from: EquipmentState
  to: EquipmentState
  type: hierarchical
  cardinality: many-to-one
  
  recursion:
    enabled: true
    max_depth: 10
    direction: up

graphQuery:
  name: EquipmentLineageQuery
  version: v1
  description: "Find all parent equipment"
  
  traverses: Part_Of
  
  start:
    from: EquipmentState
    where: "equipment.id == request.equipment_id"
    
  pattern:
    type: path
    path:
      direction: up
      max_hops: 10
      
  result:
    includes: [id, name, equipment_type]
    depth_field: hierarchy_level
```

---

## Predictive Maintenance Workflow

```yaml
smsFlow:
  name: PredictiveMaintenanceFlow
  version: v1
  
  triggeredBy:
    signal: HighFailureProbabilitySignal
    when: "machine.failure_probability > 0.8"
    
  durability:
    enabled: true
    state: MaintenanceFlowState
    ttl: 7d
    
  steps:
    - work_unit: AnalyzeFailureMode
    
    - work_unit: CreateMaintenanceOrder
    
    - await:
        name: await_technician_assignment
        trigger:
          type: inputIntent
          intent: AcceptMaintenanceOrderIntent
          assignment:
            policy: skill_based
            pool: MaintenanceTechPool
        sla:
          target: 4h
        escalation:
          - after: 4h
            action: notify
            to: MaintenanceManager
            
    - await:
        name: await_completion
        trigger:
          type: inputIntent
          intent: CompleteMaintenanceIntent
        sla:
          target: 24h
          
    - work_unit: UpdateMachineRecord
    - work_unit: RecordMaintenanceHistory

taskPool:
  name: MaintenanceTechPool
  version: v1
  
  membership:
    type: role_based
    roles: [maintenance_tech, senior_tech]
    
  skills:
    - name: mechanical
    - name: electrical
    - name: plc_programming
    - name: robotics
```

---

## Compliance & Governance

```yaml
dataState:
  name: ProductionLogState
  type: ProductionLog
  lifecycle: persistent
  
  governance:
    classification: financial
    retention:
      policy: time_based
      duration: 3650d             # 10 years
      archive:
        enabled: true
        after: 365d
        tier: cold
      on_expiry: archive
    audit:
      access_logging: true
      modification_logging: true
      immutable: true
```

---

## Key Patterns

### 1. Safety Interlocks

```yaml
safety:
  constraints:
    - expression: "pressure < max_safe"
      on_violation: emergency_stop
  interlocks:
    - condition: "upstream.closed"
      action: prevent
```

### 2. Streaming Aggregation with Sampling

```yaml
overflow:
  strategy: sample
  sample_rate: 0.1
```

### 3. ML Predictive Maintenance

```yaml
failure_probability:
  inference:
    model: predictive_maintenance
    freshness:
      strategy: periodic
      interval: 1h
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

## EXAMPLE 11: Legal Case Management

# Legal Case Management

**Example ID**: 11  
**Industry**: Legal  
**Complexity**: High  
**Key Features**: Workflows, legal holds, delegation, search, compliance

---

## Overview

A legal case management platform supporting:

- **Matter Management** - Cases, clients, billing
- **Document Repository** - Storage, versioning, search
- **Workflow Automation** - Deadlines, approvals
- **E-Discovery** - Legal holds, collections
- **Time & Billing** - Time tracking, invoicing
- **Client Portal** - Secure document sharing

---

## Data Model

### Matter

```yaml
dataType:
  name: Matter
  version: v1
  
  fields:
    matter_id:
      type: string
      
    matter_number:
      type: string
      constraints:
        unique: true
        
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    client_id:
      type: string
      
    matter_type:
      type: enum
      values: [litigation, corporate, ip, real_estate, employment, regulatory]
      search:
        facet:
          enabled: true
          
    status:
      type: enum
      values: [active, pending, closed, on_hold]
      presentation:
        display_as: badge
        
    responsible_attorney:
      type: string
      
    practice_group:
      type: string
      search:
        facet:
          enabled: true
          
    opened_date:
      type: date
      
    closed_date:
      type: date
      
    billing_type:
      type: enum
      values: [hourly, fixed_fee, contingency, pro_bono]
      
  governance:
    classification: confidential
    retention:
      policy: time_based
      duration: 3650d             # 10 years
      archive:
        enabled: true
        after: 730d
        tier: cold
    audit:
      access_logging: true
      modification_logging: true
      immutable: true
```

### Legal Document

```yaml
dataType:
  name: LegalDocument
  version: v1
  
  fields:
    document_id:
      type: string
      
    matter_id:
      type: string
      
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        weight: 2.0
        
    document_type:
      type: enum
      values: [pleading, motion, brief, contract, correspondence, memo, evidence]
      search:
        facet:
          enabled: true
          
    content:
      type: structured_content
      structured_content:
        schema: legal_document
        blocks:
          allowed: [paragraph, heading, list, blockquote, table]
        marks:
          allowed: [bold, italic, underline, strikethrough, highlight]
          
    file:
      type: asset
      asset:
        category: document
        constraints:
          mime_types: [application/pdf, application/msword, application/vnd.openxmlformats-officedocument.wordprocessingml.document]
          max_size: 100MB
        lifecycle:
          type: permanent
        access:
          read: authenticated
          
    version:
      type: integer
      
    created_by:
      type: string
      
    created_at:
      type: timestamp
      
    tags:
      type: array
      items:
        type: string
      search:
        indexed: true
        
    privileged:
      type: boolean
      
    bates_start:
      type: string
      
    bates_end:
      type: string
```

---

## Attorney Delegation

```yaml
relationship:
  name: Delegates_Matter_Access
  from: AttorneyState
  to: AttorneyState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view_matter, edit_documents, bill_time, approve_expenses]
    constraints:
      transitive: false
      max_depth: 1
      same_realm: true
    requires:
      approval: grantor
    validity:
      type: temporal
      duration: 365d
    revocation:
      by: either

policy:
  name: MatterAccessPolicy
  version: v1
  
  appliesTo:
    type: dataState
    name: MatterState
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Delegates_Matter_Access
    max_chain: 1
    
  when: |
    (subject.id == matter.responsible_attorney) OR
    (subject.id IN matter.team_members) OR
    (delegation.active AND delegation.scope CONTAINS 'view_matter')
```

---

## Legal Holds

```yaml
legalHold:
  name: LitigationHold
  version: v1
  
  scope:
    entities: [LegalDocumentState, EmailState, MatterState]
    filter: "matter.id == hold.matter_id"
    
  reason: "Pending litigation - preservation required"
  reference: "hold.case_number"
  
  custodian: legal_department
  
  preserves:
    - dataState: LegalDocumentState
      fields: all
    - dataState: EmailState
      fields: all
    - dataState: MatterState
      fields: all
      
  overrides:
    retention: true
    erasure: true
    
  lifecycle:
    start_date: "hold.effective_date"
    review_interval: 90d
    
  release:
    requires_approval: true
    approvers: [partner, general_counsel]

inputIntent:
  name: InitiateLegalHoldIntent
  version: v1
  
  proposal:
    creates: LegalHoldState
    fields:
      matter_id:
        type: string
        required: true
      case_number:
        type: string
        required: true
      custodians:
        type: array
        required: true
      scope_description:
        type: string
        required: true
        
  notification:
    enabled: true
    trigger:
      on: intent_success
    channels:
      - type: email
        email:
          template: legal_hold_notice
          to_field: custodians
```

---

## Court Deadlines

```yaml
inputIntent:
  name: CourtDeadlineReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: deadline.due_date
        offset: 7d
    batch:
      enabled: true
      source:
        query: "deadline.status == 'pending'"
        
  notification:
    enabled: true
    channels:
      - type: email
        priority: 1
      - type: push
        priority: 2

smsFlow:
  name: DeadlineManagementFlow
  version: v1
  
  triggeredBy:
    inputIntent: CreateDeadlineIntent
    
  durability:
    enabled: true
    state: DeadlineFlowState
    ttl: 365d
    
  steps:
    - work_unit: ValidateDeadline
    
    - work_unit: CalculateInternalDeadlines
      output:
        internal_deadlines: "calculated_dates"
        
    - work_unit: AssignResponsibilities
    
    - for_each: "internal_deadlines"
      do:
        - work_unit: ScheduleReminder
        - await:
            name: await_task_completion
            trigger:
              type: inputIntent
              intent: CompleteDeadlineTaskIntent
              assignment:
                policy: direct
                subject: "task.assigned_to"
            sla:
              target: "task.due_date"
              warning_at: "task.due_date - 2d"
            escalation:
              - after: "task.due_date - 1d"
                action: notify
                to: "matter.responsible_attorney"
            timeout: "task.due_date + 1d"
            on_timeout: escalate
```

---

## Document Search

```yaml
searchIntent:
  name: DocumentSearchIntent
  version: v1
  
  searches: LegalDocumentState
  
  query:
    fields: [title, content, tags]
    strategy: full_text
    highlight:
      enabled: true
      fields: [title, content]
      
  filter:
    expression: "document.matter_id IN accessible_matters(subject.id)"
    
  facets:
    include: [document_type, practice_group, created_at]
    
  result:
    includes: [document_id, title, document_type, created_at, matter_id]
    max_results: 100
```

---

## Time Entry Workflow

```yaml
smsFlow:
  name: TimeEntryApprovalFlow
  version: v1
  
  triggeredBy:
    inputIntent: SubmitTimeEntryIntent
    
  durability:
    enabled: true
    state: TimeEntryFlowState
    ttl: 30d
    
  steps:
    - work_unit: ValidateTimeEntry
    
    - if: "time_entry.amount > 1000"
      then:
        - await:
            name: await_partner_approval
            trigger:
              type: inputIntent
              intent: ApproveTimeEntryIntent
              assignment:
                policy: direct
                subject: "matter.billing_partner"
            sla:
              target: 48h
            timeout: 7d
            on_timeout: continue_default
            default_result:
              decision: auto_approved
              
    - work_unit: UpdateBillingRecord
    
    - work_unit: UpdateMatterBalance
```

---

## Key Patterns

### 1. Legal Hold Preservation

```yaml
overrides:
  retention: true
  erasure: true
  
release:
  requires_approval: true
  approvers: [partner, general_counsel]
```

### 2. Court Deadline Reminders

```yaml
schedule:
  trigger:
    type: relative
    before_event: deadline.due_date
    offset: 7d
```

### 3. Matter Access Delegation

```yaml
delegation:
  scope: [view_matter, edit_documents, bill_time]
  same_realm: true
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---


## EXAMPLE 12: Voice-First Smart Home

**Example ID**: 12  
**Industry**: Consumer IoT  
**Complexity**: Medium  
**Key Features**: Voice, IoT, device groups, temporal access, scheduling

---

## Overview

A voice-controlled smart home platform supporting:

- **Voice Control** - Natural language commands
- **Device Management** - Lights, HVAC, locks, cameras
- **Routines** - Scheduled and triggered automations
- **Guest Access** - Time-bound permissions
- **Energy Monitoring** - Usage tracking

---

## Data Model

### Home

```yaml
dataType:
  name: Home
  version: v1
  
  fields:
    home_id:
      type: string
      
    name:
      type: string
      
    address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        state: { type: string }
        zip: { type: string }
        
    location:
      type: point
      point:
        coordinate_system: wgs84
        
    timezone:
      type: string
      
    owner_id:
      type: string
      
    members:
      type: array
      items:
        type: object
        properties:
          user_id: { type: string }
          role: { type: enum, values: [owner, admin, member, guest] }
```

### Room

```yaml
dataType:
  name: Room
  version: v1
  
  fields:
    room_id:
      type: string
      
    home_id:
      type: string
      
    name:
      type: string
      
    room_type:
      type: enum
      values: [living_room, bedroom, kitchen, bathroom, garage, office, outdoor]
      
    floor:
      type: integer
```

---

## IoT Devices

### Smart Light

```yaml
worker_class:
  name: SmartLight
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - set_power
      - set_brightness
      - set_color
      
    constraints:
      power:
        source: mains
        
      connectivity:
        type: always_on
        protocols: [wifi, zigbee]
        
    health:
      heartbeat_interval: 5m

deviceCapability:
  name: set_brightness
  version: v1
  type: actuator
  
  actuator:
    controls: dimmer
    input:
      type: integer
      range: { min: 0, max: 100 }
    feedback:
      confirmation: optional
      timeout: 5s
```

### Smart Thermostat

```yaml
worker_class:
  name: SmartThermostat
  runtime: edge
  
  device:
    type: hybrid
    
    capabilities:
      - read_temperature
      - read_humidity
      - set_target_temperature
      - set_mode
      
    health:
      heartbeat_interval: 5m

deviceCapability:
  name: set_target_temperature
  version: v1
  type: actuator
  
  actuator:
    controls: hvac
    input:
      type: decimal
      range: { min: 50, max: 85 }
    feedback:
      confirmation: required
      
  safety:
    constraints:
      - expression: "target >= 50 AND target <= 85"
        on_violation: clamp
```

### Smart Lock

```yaml
worker_class:
  name: SmartDoorLock
  runtime: embedded
  
  device:
    type: actuator
    
    capabilities:
      - lock
      - unlock
      - check_status
      - read_battery
      
    constraints:
      power:
        source: battery
        battery_capacity: 4000
        
      connectivity:
        type: always_on
        protocols: [ble, wifi]
        
    health:
      heartbeat_interval: 1h
      battery_low_threshold: 20

deviceCapability:
  name: unlock
  version: v1
  type: actuator
  
  actuator:
    controls: lock
    input:
      type: boolean
    feedback:
      confirmation: required
      timeout: 10s
      
  safety:
    constraints:
      - expression: "request.authorized == true"
        on_violation: reject
```

---

## Device Groups

```yaml
deviceGroup:
  name: LivingRoomLights
  version: v1
  
  membership:
    type: dynamic
    filter: "device.room_id == 'living_room' AND device.type == 'SmartLight'"
    
  properties:
    location:
      type: room
      room_id: living_room
      
  commands:
    broadcast: true
    sequential: false

deviceGroup:
  name: AllLights
  version: v1
  
  membership:
    type: dynamic
    filter: "device.type == 'SmartLight'"
    
  commands:
    broadcast: true

actuatorCommand:
  name: TurnOffAllLights
  version: v1
  
  target:
    device_type: SmartLight
    capability: set_power
    selector: "device IN group('AllLights')"
    
  command:
    type: set_state
    set_state:
      value: false
      
  delivery:
    guarantee: at_least_once
    timeout: 10s
```

---

## Voice Experience

```yaml
experience:
  name: SmartHomeVoice
  version: v1
  
  modality:
    type: voice
    voice:
      wake_word: "Hey Home"
      languages: [en-US]
      
  conversation:
    context:
      ttl: 5m
      carry_forward: [room_context, device_context]
      
    turn_taking:
      model: mixed
      timeout: 5s
      
    error_handling:
      no_match: reprompt
      no_input: close
      max_reprompts: 2
      
  entry_point: HomeAssistantDialog

presentationComposition:
  name: HomeAssistantDialog
  version: v1
  
  dialog:
    type: question
    
    prompts:
      initial: ReadyPrompt
      reprompt: DidntCatchPrompt
      help: HelpPrompt
      
    slots:
      - name: action
        maps_to: DeviceControlIntent.action
        required: true
        type: enum
        enum:
          values: [turn_on, turn_off, set, dim, brighten, lock, unlock]
        synonyms:
          turn_on: [switch on, enable, activate, power on]
          turn_off: [switch off, disable, deactivate, power off, kill]
          dim: [lower, reduce, decrease]
          brighten: [raise, increase]
          lock: [secure, close]
          unlock: [open]
          
      - name: device_target
        maps_to: DeviceControlIntent.target
        required: true
        type: entity
        entity:
          entity_type: DeviceEntity
        synonyms:
          living_room_lights: [living room, front room lights]
          bedroom_lights: [bedroom, master bedroom]
          all_lights: [all lights, every light, everywhere]
          thermostat: [temperature, heat, ac, air conditioning]
          front_door: [front lock, main door]
          
      - name: value
        maps_to: DeviceControlIntent.value
        required: false
        type: number
        elicitation:
          prompt: WhatValuePrompt
          
    confirmation:
      required: false
      
    fulfillment:
      intent: DeviceControlIntent
      response: ConfirmationPrompt
      
  transitions:
    on_complete: ReadyForMoreDialog
    on_help: HelpDialog

prompt:
  name: ReadyPrompt
  version: v1
  content:
    speech:
      ssml: |
        <speak>
          <audio src="listening_tone.mp3"/>
        </speak>

prompt:
  name: ConfirmationPrompt
  version: v1
  content:
    speech:
      ssml: "Done. The {{device_target}} is now {{action | past_tense}}."
  context_variables:
    - name: device_target
      from: slot
      field: device_target
    - name: action
      from: slot
      field: action
```

---

## Guest Access

```yaml
temporalAccess:
  name: GuestHomeAccess
  version: v1
  
  grants:
    policy: HomeDevicePolicy
    to: "request.guest_email"
    
  validity:
    type: time_bound
    starts_at: "request.access_start"
    expires_at: "request.access_end"
    
  constraints:
    device_bound: false
    
  revocation:
    on_breach: immediate

consentRecord:
  name: HomeDataConsent
  version: v1
  
  subject_field: user_id
  
  scopes:
    - name: voice_history
      description: "Store voice command history"
      required: false
      revocable: true
      
    - name: energy_analytics
      description: "Analyze energy usage patterns"
      required: false
      default: true
      revocable: true
```

---

## Scheduled Routines

```yaml
inputIntent:
  name: GoodNightRoutineIntent
  version: v1
  description: "Execute good night routine"
  
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 22 * * *"
        timezone: "home.timezone"
        
smsFlow:
  name: GoodNightRoutineFlow
  version: v1
  
  triggeredBy:
    inputIntent: GoodNightRoutineIntent
    
  steps:
    - work_unit: TurnOffAllLights
      target:
        group: AllLights
        
    - work_unit: LockAllDoors
      target:
        group: AllLocks
        
    - work_unit: SetThermostatNight
      params:
        temperature: 68
        
    - work_unit: ArmSecuritySystem
```

---

## Key Patterns

### 1. Voice Slot Synonyms

```yaml
synonyms:
  turn_on: [switch on, enable, activate]
  living_room_lights: [living room, front room]
```

### 2. Device Group Broadcast

```yaml
commands:
  broadcast: true
  sequential: false
```

### 3. Time-Bound Guest Access

```yaml
validity:
  type: time_bound
  starts_at: "request.access_start"
  expires_at: "request.access_end"
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---


## EXAMPLE 13: Subscription Commerce Platform

**Example ID**: 13  
**Industry**: SaaS / Subscription  
**Complexity**: Medium  
**Key Features**: External APIs, webhooks, scheduling, metering, streaming

---

## Overview

A subscription management platform supporting:

- **Subscription Lifecycle** - Trials, upgrades, cancellations
- **Billing Integration** - Payment processing, invoicing
- **Usage Metering** - Track and bill by consumption
- **Churn Prediction** - ML-based risk scoring
- **Renewal Management** - Automated renewals and reminders

---

## Data Model

### Subscription

```yaml
dataType:
  name: Subscription
  version: v1
  
  fields:
    subscription_id:
      type: string
      
    customer_id:
      type: string
      
    plan_id:
      type: string
      
    status:
      type: enum
      values: [trial, active, past_due, cancelled, expired]
      presentation:
        display_as: badge
        semantic_variants:
          trial: info
          active: success
          past_due: warning
          cancelled: error
          expired: neutral
          
    billing_cycle:
      type: enum
      values: [monthly, quarterly, annual]
      
    current_period_start:
      type: timestamp
      
    current_period_end:
      type: timestamp
      
    trial_end:
      type: timestamp
      
    cancelled_at:
      type: timestamp
      
    cancel_at_period_end:
      type: boolean
      
    mrr:
      type: decimal
      description: "Monthly Recurring Revenue"
      presentation:
        display_as: currency
        
    payment_method_id:
      type: string
      
    churn_risk_score:
      type: decimal
      inference:
        enabled: true
        source:
          fields: [customer_id, plan_id, usage_data, support_tickets]
        model:
          name: churn_predictor
          version: v2
        output:
          type: decimal
          range: { min: 0, max: 1 }
        freshness:
          strategy: periodic
          interval: 24h
        fallback:
          on_unavailable: default
          default_value: 0.5
```

### Usage Record

```yaml
dataType:
  name: UsageRecord
  version: v1
  
  fields:
    record_id:
      type: string
      
    subscription_id:
      type: string
      
    metric:
      type: string
      description: "e.g., api_calls, storage_gb, seats"
      
    quantity:
      type: decimal
      
    timestamp:
      type: timestamp
      
    idempotency_key:
      type: string
```

### Invoice

```yaml
dataType:
  name: Invoice
  version: v1
  
  fields:
    invoice_id:
      type: string
      
    subscription_id:
      type: string
      
    customer_id:
      type: string
      
    status:
      type: enum
      values: [draft, open, paid, void, uncollectible]
      
    amount_due:
      type: decimal
      presentation:
        display_as: currency
        
    amount_paid:
      type: decimal
      
    due_date:
      type: date
      
    paid_at:
      type: timestamp
      
    pdf:
      type: asset
      asset:
        category: document
        lifecycle:
          type: permanent
        access:
          read: authenticated
```

---

## Payment Integration

```yaml
externalDependency:
  name: StripeSubscriptions
  version: v1
  type: api
  
  capabilities:
    - create_subscription
    - update_subscription
    - cancel_subscription
    - create_invoice
    - charge_invoice
    - create_usage_record
    
  contract:
    create_subscription:
      input:
        schema:
          customer_id: { type: string, required: true }
          price_id: { type: string, required: true }
          trial_days: { type: integer }
      output:
        schema:
          subscription_id: { type: string }
          status: { type: string }
      errors:
        - code: payment_failed
          retryable: true
        - code: card_declined
          retryable: false
          
  sla:
    availability: 99.99%
    latency_p99: 2s
    
  resilience:
    timeout: 30s
    retry:
      enabled: true
      max_attempts: 3
      backoff:
        strategy: exponential
        initial: 500ms
    circuit_breaker:
      enabled: true
      threshold: 5
      reset_after: 60s
      
  fallback:
    on_unavailable: queue
    queue_ttl: 24h
    
  credentials:
    type: api_key
    secret_ref: stripe_secret_key

webhookReceiver:
  name: StripeSubscriptionWebhook
  version: v1
  
  source: StripeSubscriptions
  
  maps_to:
    inputIntent: SubscriptionEventIntent
    
  authentication:
    type: signature
    signature:
      algorithm: hmac_sha256
      header: Stripe-Signature
      secret_ref: stripe_webhook_secret
      timestamp_tolerance: 5m
      
  event_mapping:
    - external_event: customer.subscription.created
      intent_type: SubscriptionCreated
      field_mapping:
        subscription_id: "$.data.object.id"
        customer_id: "$.data.object.customer"
        status: "$.data.object.status"
        
    - external_event: customer.subscription.updated
      intent_type: SubscriptionUpdated
      field_mapping:
        subscription_id: "$.data.object.id"
        status: "$.data.object.status"
        cancel_at_period_end: "$.data.object.cancel_at_period_end"
        
    - external_event: invoice.payment_succeeded
      intent_type: PaymentSucceeded
      field_mapping:
        invoice_id: "$.data.object.id"
        subscription_id: "$.data.object.subscription"
        amount_paid: "$.data.object.amount_paid"
        
    - external_event: invoice.payment_failed
      intent_type: PaymentFailed
      field_mapping:
        invoice_id: "$.data.object.id"
        subscription_id: "$.data.object.subscription"
        
  idempotency:
    key_path: "$.id"
    ttl: 24h
```

---

## Usage Metering

```yaml
materialization:
  name: UsageAggregateView
  version: v1
  source: UsageRecordState
  targetState: UsageAggregateMaterialized
  
  retrieval:
    mode: by_query
    query_fields: [subscription_id, metric, period]
    
    temporal:
      window:
        type: tumbling
        size: 1h
        
      aggregation:
        - field: quantity
          function: sum
          as: total_quantity
        - field: record_id
          function: count
          as: record_count
          
      timestamp_field: timestamp
      
      emit:
        trigger: on_window_close
        
    freshness:
      max_staleness: 5m

inputIntent:
  name: ReportUsageIntent
  version: v1
  
  proposal:
    creates: UsageRecordState
    fields:
      subscription_id:
        type: string
        required: true
      metric:
        type: string
        required: true
      quantity:
        type: decimal
        required: true
      idempotency_key:
        type: string
        required: true
        
  delivery:
    guarantee: exactly_once
    deduplication:
      enabled: true
      key_field: idempotency_key
      ttl: 24h
```

---

## Scheduled Tasks

```yaml
inputIntent:
  name: RenewalReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: subscription.current_period_end
        offset: 7d
    batch:
      enabled: true
      source:
        query: "subscription.status == 'active' AND subscription.cancel_at_period_end == false"
        
  notification:
    enabled: true
    channels:
      - type: email
        email:
          template: renewal_reminder
          subject: "Your subscription renews in 7 days"

inputIntent:
  name: TrialEndingReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: subscription.trial_end
        offset: 3d
    batch:
      enabled: true
      source:
        query: "subscription.status == 'trial'"
        
  notification:
    enabled: true
    channels:
      - type: email
        priority: 1
      - type: push
        priority: 2

inputIntent:
  name: BillingCycleProcessIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: cron
      cron:
        expression: "0 0 * * *"         # Daily at midnight
        timezone: UTC
    batch:
      enabled: true
      source:
        query: "subscription.current_period_end <= now() AND subscription.status == 'active'"
      chunking:
        size: 1000
        parallel: 4
```

---

## Subscription Workflow

```yaml
smsFlow:
  name: SubscriptionCancellationFlow
  version: v1
  
  triggeredBy:
    inputIntent: CancelSubscriptionIntent
    
  steps:
    - work_unit: ValidateCancellation
    
    - work_unit: CalculateProration
      output:
        refund_amount: "calculation.refund"
        
    - if: "intent.immediate == true"
      then:
        - work_unit: ProcessImmediateCancellation
        - if: "refund_amount > 0"
          then:
            - work_unit: ProcessRefund
              external_dependency: StripeSubscriptions
      else:
        - work_unit: SchedulePeriodEndCancellation
        
    - work_unit: UpdateSubscriptionRecord
    
    - signal: SubscriptionCancelledSignal
    
  notification:
    enabled: true
    trigger:
      on: flow_success
    channels:
      - type: email
        email:
          template: cancellation_confirmation
```

---

## Churn Prevention

```yaml
smsFlow:
  name: ChurnPreventionFlow
  version: v1
  
  triggeredBy:
    signal: HighChurnRiskSignal
    when: "subscription.churn_risk_score > 0.7"
    
  steps:
    - work_unit: AnalyzeChurnFactors
    
    - work_unit: DetermineIntervention
      output:
        intervention_type: "analysis.recommendation"
        
    - if: "intervention_type == 'discount'"
      then:
        - work_unit: GenerateDiscountOffer
        - work_unit: SendDiscountEmail
    else:
      - if: "intervention_type == 'outreach'"
        then:
          - work_unit: CreateOutreachTask
          - await:
              name: await_outreach_completion
              trigger:
                type: inputIntent
                intent: CompleteOutreachIntent
                assignment:
                  policy: round_robin
                  pool: CustomerSuccessPool
              timeout: 7d
              
    - work_unit: RecordIntervention
```

---

## Key Patterns

### 1. Webhook Signature Verification

```yaml
authentication:
  type: signature
  signature:
    algorithm: hmac_sha256
    timestamp_tolerance: 5m
```

### 2. Usage Metering with Idempotency

```yaml
delivery:
  guarantee: exactly_once
  deduplication:
    key_field: idempotency_key
    ttl: 24h
```

### 3. Relative Scheduled Triggers

```yaml
trigger:
  type: relative
  before_event: subscription.trial_end
  offset: 3d
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---


## EXAMPLE 14: Collaborative Document Editor

**Example ID**: 14  
**Industry**: Productivity  
**Complexity**: High  
**Key Features**: Real-time collaboration, rich content, offline, multi-device

---

## Overview

A real-time collaborative document editor supporting:

- **Real-Time Editing** - Multiple cursors, live updates
- **Rich Content** - Formatting, tables, images, embeds
- **Version History** - Track changes, restore versions
- **Offline Editing** - Work without connection
- **Sharing & Permissions** - Granular access control
- **Comments & Discussions** - Threaded conversations

---

## Data Model

### Document

```yaml
dataType:
  name: Document
  version: v1
  
  fields:
    document_id:
      type: string
      
    title:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    content:
      type: structured_content
      structured_content:
        schema: rich_document
        blocks:
          allowed:
            - paragraph
            - heading
            - list
            - blockquote
            - code_block
            - table
            - image
            - divider
            - callout
            - embed
        marks:
          allowed:
            - bold
            - italic
            - underline
            - strikethrough
            - code
            - link
            - highlight
            - comment
        nesting:
          max_depth: 5
        links:
          internal:
            enabled: true
            targets: [Document]
          external:
            enabled: true
            validate: true
        constraints:
          max_length: 500000
        structure:
          outline: true
          
    owner_id:
      type: string
      
    folder_id:
      type: string
      
    created_at:
      type: timestamp
      
    updated_at:
      type: timestamp
      
    version:
      type: integer
      
    word_count:
      type: integer
      
    cover_image:
      type: asset
      asset:
        category: image
        variants:
          - name: thumbnail
            transform: resize
            params: { width: 400, height: 300 }
        access:
          read: authenticated

dataState:
  name: DocumentState
  type: Document
  lifecycle: persistent
  
  authority:
    scope: entity
    entity_key: document_id
```

### Document Version

```yaml
dataType:
  name: DocumentVersion
  version: v1
  
  fields:
    version_id:
      type: string
      
    document_id:
      type: string
      
    version_number:
      type: integer
      
    content_snapshot:
      type: structured_content
      
    created_by:
      type: string
      
    created_at:
      type: timestamp
      
    change_summary:
      type: string
```

### Comment

```yaml
dataType:
  name: Comment
  version: v1
  
  fields:
    comment_id:
      type: string
      
    document_id:
      type: string
      
    parent_comment_id:
      type: string
      description: "For threaded replies"
      
    author_id:
      type: string
      
    content:
      type: string
      
    anchor:
      type: object
      properties:
        type: { type: enum, values: [range, point] }
        start: { type: integer }
        end: { type: integer }
        
    status:
      type: enum
      values: [open, resolved]
      
    created_at:
      type: timestamp
```

---

## Real-Time Collaboration

```yaml
collaborativeSession:
  name: DocumentEditSession
  version: v1
  description: "Real-time collaborative document editing"
  
  binds_to:
    dataState: DocumentState
    entity_key: document_id
    
  presence:
    enabled: true
    fields:
      cursor:
        type: position
        scope: per_client
      selection:
        type: range
        scope: per_client
      user_color:
        type: string
        scope: per_session
      user_name:
        type: string
        scope: per_session
      is_typing:
        type: boolean
        scope: per_client
    ttl: 30s
    broadcast:
      to: same_entity
      throttle: 50ms
      
  operations:
    mode: stream
    ordering:
      guarantee: causal
    conflict:
      hint: merge
    types:
      - name: insert
        payload:
          position: integer
          content: string
      - name: delete
        payload:
          position: integer
          length: integer
      - name: format
        payload:
          range: object
          mark: string
          value: any
      - name: replace_block
        payload:
          block_id: string
          block_data: object
          
  subscription:
    to: all
    state:
      mode: delta
      buffer_size: 100
      
  access:
    max_participants: 50
    roles:
      owner: [edit, share, delete, manage_comments]
      editor: [edit, comment]
      commenter: [comment, view]
      viewer: [view]
      
  lifecycle:
    auto_close_after: 1h
    on_all_leave: keep_open
    inactivity_timeout: 30m
```

---

## Permission Delegation

```yaml
relationship:
  name: Shared_With
  from: DocumentState
  to: SubjectState
  type: delegation
  cardinality: one-to-many
  
  delegation:
    scope: [view, comment, edit]
    constraints:
      transitive: false
      max_depth: 1
    requires:
      approval: none
    validity:
      type: permanent
    revocation:
      by: grantor

policy:
  name: DocumentAccessPolicy
  version: v1
  
  appliesTo:
    type: dataState
    name: DocumentState
    
  effect: allow
  
  delegation:
    enabled: true
    via_relationship: Shared_With
    max_chain: 1
    
  when: |
    (subject.id == document.owner_id) OR
    (delegation.active AND delegation.scope CONTAINS 'view')
```

---

## Offline Support

```yaml
experience:
  name: DocumentEditor
  version: v1
  
  entry_point: DocumentList
  
  includes:
    - DocumentList
    - DocumentEditor
    - VersionHistory
    - Comments
    - Sharing
    
  session:
    required: true
    
    subject_binding:
      mode: authenticated
      
    lifetime:
      mode: sliding
      idle_timeout: 30d
      
    devices:
      enabled: true
      max_active: 10
      
      identification:
        by: device_id
        
      state_scope:
        per_device:
          - ui.sidebar_collapsed
          - ui.zoom_level
          - editor.cursor_position
        shared:
          - documents
          - folders
          - recent_documents
          
      handoff:
        enabled: true
        state_transfer: immediate
        transferable:
          - current_document_id
          - editor.cursor_position
          - editor.scroll_position
          
      concurrent_limits:
        on_new_device: allow
        
    offline:
      enabled: true
      
      capabilities:
        read: true
        write: queue
        
      sync:
        on_reconnect: background
        conflict_resolution: merge
        
        field_strategies:
          - field: document.content
            strategy: merge
          - field: document.title
            strategy: server_wins
          - field: comments
            strategy: union
            
      queue:
        max_size: 1000
        max_age: 30d
        on_overflow: reject_new
        
      cache:
        views: [RecentDocumentsView, FolderContentsView]
        max_size: 200MB
        ttl: 168h
```

---

## Version History

```yaml
inputIntent:
  name: CreateVersionSnapshotIntent
  version: v1
  
  proposal:
    creates: DocumentVersionState
    fields:
      document_id:
        type: string
        required: true
      change_summary:
        type: string
        required: false

smsFlow:
  name: AutoVersionFlow
  version: v1
  description: "Automatically create version snapshots"
  
  triggeredBy:
    schedule:
      type: interval
      interval: 1h
      
  steps:
    - work_unit: FindModifiedDocuments
      output:
        modified_docs: "query.results"
        
    - for_each: "modified_docs"
      do:
        - work_unit: CreateVersionSnapshot
        - work_unit: PruneOldVersions
          params:
            keep_count: 100
```

---

## Search

```yaml
searchIntent:
  name: DocumentSearchIntent
  version: v1
  
  searches: DocumentState
  
  query:
    fields: [title, content]
    strategy: full_text
    highlight:
      enabled: true
      fields: [title, content]
      
  filter:
    expression: "document.id IN accessible_documents(subject.id)"
    
  result:
    includes: [document_id, title, updated_at, owner_id, cover_image]
    max_results: 50
    
  suggestions:
    enabled: true
    field: title
```

---

## Embedded Assets

```yaml
inputIntent:
  name: UploadDocumentImageIntent
  version: v1
  
  proposal:
    creates: DocumentAssetState
    fields:
      document_id:
        type: string
        required: true
      image:
        type: asset
        asset:
          category: image
          constraints:
            max_size: 10MB
            mime_types: [image/jpeg, image/png, image/gif, image/webp]
          variants:
            - name: inline
              transform: resize
              params: { width: 800, max_height: 600 }
            - name: thumbnail
              transform: resize
              params: { width: 100, height: 100 }
          access:
            read: authenticated
```

---

## Key Patterns

### 1. Real-Time Presence with Throttling

```yaml
presence:
  fields:
    cursor:
      type: position
      scope: per_client
  broadcast:
    to: same_entity
    throttle: 50ms
```

### 2. Causal Operation Ordering

```yaml
operations:
  mode: stream
  ordering:
    guarantee: causal
  conflict:
    hint: merge
```

### 3. Offline Content Merge

```yaml
field_strategies:
  - field: document.content
    strategy: merge
  - field: comments
    strategy: union
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---


## EXAMPLE 15: Government Permitting System

**Example ID**: 15  
**Industry**: Government  
**Complexity**: High  
**Key Features**: Durable workflows, spatial, accessibility, compliance

---

## Overview

A government permitting system supporting:

- **Permit Applications** - Online submission with documents
- **Multi-Department Review** - Parallel and sequential approvals
- **Public Hearings** - Scheduling and notifications
- **Fee Collection** - Online payments
- **GIS Integration** - Property lookups, zoning
- **Accessibility** - WCAG 2.1 AA compliance

---

## Data Model

### Permit Application

```yaml
dataType:
  name: PermitApplication
  version: v1
  
  fields:
    application_id:
      type: string
      
    application_number:
      type: string
      constraints:
        unique: true
      presentation:
        display_as: identifier
        
    permit_type:
      type: enum
      values: [building, electrical, plumbing, mechanical, demolition, grading, special_use]
      search:
        facet:
          enabled: true
          
    status:
      type: enum
      values: [draft, submitted, under_review, pending_info, approved, denied, expired, revoked]
      presentation:
        display_as: badge
        semantic_variants:
          draft: neutral
          submitted: info
          under_review: warning
          pending_info: warning
          approved: success
          denied: error
          expired: neutral
          revoked: error
      accessibility:
        label:
          template: "Application status: {{value}}"
          
    applicant_id:
      type: string
      
    property_address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        state: { type: string }
        zip: { type: string }
        
    property_location:
      type: point
      point:
        coordinate_system: wgs84
        search:
          indexed: true
          strategy: spatial
          
    parcel_id:
      type: string
      
    project_description:
      type: string
      search:
        indexed: true
        strategy: full_text
        
    estimated_value:
      type: decimal
      presentation:
        display_as: currency
        
    documents:
      type: array
      items:
        type: asset
        asset:
          category: document
          constraints:
            mime_types: [application/pdf, image/jpeg, image/png]
            max_size: 50MB
          lifecycle:
            type: permanent
          access:
            read: authenticated
            
    submitted_at:
      type: timestamp
      
    decision_date:
      type: timestamp
      
  governance:
    classification: public
    retention:
      policy: time_based
      duration: 3650d             # 10 years
      archive:
        enabled: true
        after: 730d
        tier: cold
    audit:
      access_logging: true
      modification_logging: true
```

### Department Review

```yaml
dataType:
  name: DepartmentReview
  version: v1
  
  fields:
    review_id:
      type: string
      
    application_id:
      type: string
      
    department:
      type: enum
      values: [planning, building, fire, public_works, health, environmental]
      
    reviewer_id:
      type: string
      
    status:
      type: enum
      values: [pending, in_progress, approved, denied, needs_revision]
      
    comments:
      type: string
      
    conditions:
      type: array
      items:
        type: string
        
    reviewed_at:
      type: timestamp
```

### Zoning District

```yaml
dataType:
  name: ZoningDistrict
  version: v1
  
  fields:
    district_id:
      type: string
      
    name:
      type: string
      
    code:
      type: string
      
    boundary:
      type: polygon
      polygon:
        coordinate_system: wgs84
        presentation:
          display_as: region
          fill_opacity: 0.2
          stroke_color: "#0066CC"
          
    permitted_uses:
      type: array
      items:
        type: string
        
    max_height_feet:
      type: integer
      
    max_lot_coverage:
      type: decimal
```

---

## Spatial Queries

```yaml
searchIntent:
  name: ZoningLookupIntent
  version: v1
  description: "Find zoning district for a property"
  
  searches: ZoningDistrictState
  
  query:
    spatial:
      field: boundary
      type: contains_point
      contains_point:
        point: "request.property_location"
        
  result:
    includes: [district_id, name, code, permitted_uses, max_height_feet]

searchIntent:
  name: NearbyPermitsIntent
  version: v1
  description: "Find permits near a location"
  
  searches: PermitApplicationState
  
  query:
    spatial:
      field: property_location
      type: within_distance
      within_distance:
        from: "request.location"
        max_distance: 500m
    sort:
      by_distance_from: "request.location"
      
  filter:
    expression: "application.status IN ['submitted', 'under_review', 'approved']"
    
  result:
    includes: [application_id, application_number, permit_type, status, property_address]
```

---

## Multi-Department Workflow

```yaml
smsFlow:
  name: PermitReviewFlow
  version: v1
  description: "Multi-department permit review process"
  
  triggeredBy:
    inputIntent: SubmitPermitApplicationIntent
    
  durability:
    enabled: true
    state: PermitReviewFlowState
    ttl: 365d
    versioning:
      strategy: run_to_completion
      
  steps:
    - work_unit: ValidateApplication
    
    - work_unit: CalculateFees
      output:
        fee_amount: "calculation.total"
        
    - await:
        name: await_fee_payment
        trigger:
          type: inputIntent
          intent: PayPermitFeeIntent
          assignment:
            policy: direct
            subject: "application.applicant_id"
        sla:
          target: 30d
        timeout: 30d
        on_timeout: cancel
        
    - checkpoint: post_payment
    
    - work_unit: DetermineRequiredReviews
      output:
        required_departments: "routing.departments"
        
    # Parallel department reviews
    - parallel:
        - for_each: "required_departments"
          do:
            - await:
                name: await_department_review
                description: "Review by {{department.name}}"
                trigger:
                  type: inputIntent
                  intent: CompleteDepartmentReviewIntent
                  assignment:
                    policy: round_robin
                    pool: "{{department.code}}ReviewerPool"
                sla:
                  target: 14d
                  warning_at: 10d
                escalation:
                  - after: 14d
                    action: notify
                    to: "{{department.code}}Supervisor"
                  - after: 21d
                    action: escalate_pool
                    to: SeniorReviewerPool
                timeout: 30d
                result:
                  captures: [decision, conditions, comments]
                  
    - checkpoint: post_reviews
    
    - work_unit: ConsolidateReviews
      output:
        all_approved: "consolidation.all_approved"
        combined_conditions: "consolidation.conditions"
        
    - if: "all_approved"
      then:
        - if: "application.requires_hearing"
          then:
            - work_unit: SchedulePublicHearing
            - await:
                name: await_hearing_decision
                trigger:
                  type: inputIntent
                  intent: RecordHearingDecisionIntent
                sla:
                  target: 60d
                result:
                  captures: [hearing_decision, hearing_conditions]
                  
        - work_unit: IssuePermit
        - work_unit: NotifyApplicant
        - signal: PermitIssuedSignal
        
      else:
        - work_unit: CompileDenialReasons
        - work_unit: NotifyDenial
        - signal: PermitDeniedSignal
        
  checkpoints:
    - name: post_payment
      after_step: await_fee_payment
    - name: post_reviews
      after_step: parallel
      state_snapshot: full
      
  compensation:
    - from_checkpoint: post_payment
      do: RefundFees
    - from_checkpoint: post_reviews
      do: RevokeApprovals

taskPool:
  name: PlanningReviewerPool
  version: v1
  
  membership:
    type: role_based
    roles: [planning_reviewer, senior_planner]
    
  capacity:
    max_concurrent_per_member: 15
    
  availability:
    schedule: "0 8-17 * * 1-5"
    timezone: America/New_York
    
  skills:
    - name: residential
    - name: commercial
    - name: special_use
```

---

## Payment Integration

```yaml
externalDependency:
  name: GovPaymentProcessor
  version: v1
  type: api
  
  capabilities:
    - create_payment
    - check_status
    - process_refund
    
  resilience:
    timeout: 30s
    circuit_breaker:
      enabled: true
      threshold: 5
      
  credentials:
    type: api_key
    secret_ref: gov_pay_api_key
```

---

## Accessibility

```yaml
experience:
  name: PermitPortal
  version: v1
  description: "Public permit application portal"
  
  entry_point: PermitHome
  
  includes:
    - PermitHome
    - ApplicationForm
    - ApplicationStatus
    - DocumentUpload
    - PaymentForm
    - PermitSearch
    
  session:
    required: false
    
    subject_binding:
      mode: either
      
  accessibility:
    presets:
      - name: standard
        settings: {}
      - name: vision_impaired
        settings:
          font_scale: 1.5
          high_contrast: true
          motion: none
      - name: motor_impaired
        settings:
          focus_indicators: enhanced
          click_targets: large
          
  preferences:
    schema:
      language:
        type: enum
        values: [en, es, zh, vi, ko]
        default: en
      text_size:
        type: enum
        values: [normal, large, extra_large]
        default: normal

presentationView:
  name: ApplicationFormView
  version: v1
  
  accessibility:
    landmark: main
    skip_link:
      enabled: true
      label: "Skip to application form"
    keyboard_navigation:
      tab_order: sequential
      shortcuts:
        - key: "Alt+S"
          action: submit
          label: "Submit application"
        - key: "Alt+D"
          action: save_draft
          label: "Save as draft"
```

---

## Public Records Search

```yaml
searchIntent:
  name: PublicPermitSearchIntent
  version: v1
  
  searches: PermitApplicationState
  
  query:
    fields: [application_number, property_address, project_description]
    strategy: full_text
    
  filter:
    expression: "application.status NOT IN ['draft']"
    
  facets:
    include: [permit_type, status]
    
  result:
    includes: [application_id, application_number, permit_type, status, property_address, submitted_at]
```

---

## Scheduled Notifications

```yaml
inputIntent:
  name: PermitExpirationReminderIntent
  version: v1
  
  schedule:
    enabled: true
    trigger:
      type: relative
      relative:
        before_event: permit.expiration_date
        offset: 30d
    batch:
      enabled: true
      source:
        query: "permit.status == 'approved'"
        
  notification:
    enabled: true
    channels:
      - type: email
        email:
          template: permit_expiration_reminder
      - type: mail
        mail:
          template: permit_expiration_letter
```

---

## Key Patterns

### 1. Parallel Department Reviews

```yaml
parallel:
  - for_each: "required_departments"
    do:
      - await:
          assignment:
            pool: "{{department.code}}ReviewerPool"
```

### 2. Zoning Polygon Containment

```yaml
spatial:
  field: boundary
  type: contains_point
  contains_point:
    point: "request.property_location"
```

### 3. Accessibility-First Design

```yaml
accessibility:
  landmark: main
  skip_link:
    enabled: true
  keyboard_navigation:
    shortcuts:
      - key: "Alt+S"
        action: submit
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| Draft | 2026-01 | Initial example |

---

