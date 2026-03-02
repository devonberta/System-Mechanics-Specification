# Appendix H: Worked Banking Example (v1.5)

## Overview

This appendix provides a complete, worked example of modeling a banking system using SMS's invariant-oriented approach with comprehensive v1.5 features. The example demonstrates:

- **v1.3**: Entity decomposition, relationships, views, and authority
- **v1.4**: UI compositions, experiences, field permissions, and navigation
- **v1.5**: Assets, notifications, scheduled triggers, external integrations, data governance, accessibility, and conversational UI

---

## Problem Statement

Design a system that supports:
1. Creating accounts
2. Processing transactions (debits and credits)
3. Maintaining accurate balances
4. Ensuring balances never go negative
5. Operating across multiple regions
6. **Customer-facing UI** for account holders
7. **Administrator UI** for bank staff

### The Critical Invariant

> **Account balance must never be negative.**

This invariant spans multiple entities (Account + all Transactions) and is the central challenge.

---

## Data Layer (v1.3)

### Entity: Account

```yaml
dataType:
  name: Account
  version: v1
  fields:
    id: uuid
    owner_id: uuid
    account_number: string
    account_type: enum[checking, savings, credit]
    status: enum[active, frozen, closed, pending_approval]
    currency: string
    jurisdiction: string
    created_at: timestamp

model:
  name: Account
  mutability:
    scope: entity
    exclusive: true
  authority:
    scope: entity
    resolution: entity.jurisdiction_region
    migration:
      allowed: false  # Financial data stays in region
```

### Entity: Customer

```yaml
dataType:
  name: Customer
  version: v1
  fields:
    id: uuid
    email: string
    full_name: string
    phone: string
    ssn: string
    date_of_birth: timestamp
    kyc_status: enum[pending, verified, rejected]
    risk_tier: enum[low, medium, high]
    created_at: timestamp
```

### Entity: Transaction

```yaml
dataType:
  name: Transaction
  version: v1
  fields:
    id: uuid
    timestamp: timestamp
    debit_account_id: uuid
    credit_account_id: uuid
    amount: decimal
    currency: string
    reference: string
    category: enum[transfer, payment, deposit, withdrawal, fee, interest]
    status: enum[pending, completed, failed, reversed]
    metadata: map

model:
  name: Transaction
  mutability:
    scope: append-only  # Transactions never change
```

### Entity: Alert

```yaml
dataType:
  name: Alert
  version: v1
  fields:
    id: uuid
    type: enum[fraud_suspected, large_transaction, negative_balance, kyc_expired]
    severity: enum[low, medium, high, critical]
    entity_type: string
    entity_id: uuid
    message: string
    status: enum[open, acknowledged, resolved, escalated]
    created_at: timestamp
    resolved_at: timestamp
    resolved_by: uuid
```

### Entity: AuditLog

```yaml
dataType:
  name: AuditLog
  version: v1
  fields:
    id: uuid
    actor_id: uuid
    actor_type: enum[customer, admin, system]
    action: string
    entity_type: string
    entity_id: uuid
    changes: map
    ip_address: string
    timestamp: timestamp
```

---

## Relationships (v1.3)

```yaml
relationships:
  - name: customer_owns_account
    from: Account
    to: Customer
    cardinality: many-to-one
    semantics:
      causal: true
      
  - name: transaction_debits_account
    from: Transaction
    to: Account
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
      
  - name: transaction_credits_account
    from: Transaction
    to: Account
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
      
  - name: alert_references_entity
    from: Alert
    to: Account
    cardinality: many-to-one
    
  - name: audit_references_actor
    from: AuditLog
    to: Customer
    cardinality: many-to-one
```

---

## Views (v1.3)

### AccountBalance View

```yaml
view:
  name: AccountBalance
  inputs: [Account, Transaction]
  aggregation:
    credits: sum(t.amount WHERE t.credit_account_id == account.id AND t.status == "completed")
    debits: sum(t.amount WHERE t.debit_account_id == account.id AND t.status == "completed")
    pending: sum(t.amount WHERE t.status == "pending")
    balance: credits - debits
    available: balance - pending
  invariant:
    expression: balance >= 0 OR account.account_type == "credit"
    on_violation: flag
  authority_agnostic: true
```

### CustomerSummary View

```yaml
view:
  name: CustomerSummary
  inputs: [Customer, Account, AccountBalance]
  aggregation:
    account_count: count(accounts)
    total_balance: sum(balances)
    has_alerts: exists(alerts WHERE status == "open")
```

### SystemMetrics View

```yaml
view:
  name: SystemMetrics
  inputs: [Customer, Account, Transaction, Alert]
  aggregation:
    total_customers: count(customers)
    active_accounts: count(accounts WHERE status == "active")
    pending_transactions: count(transactions WHERE status == "pending")
    open_alerts: count(alerts WHERE status == "open")
    critical_alerts: count(alerts WHERE severity == "critical" AND status == "open")
```

---

## Policy Layer

### Subjects and Roles

```yaml
policy:
  subjects:
    - name: customer
      type: user
      attributes:
        id: uuid
        email: string
        kyc_status: string
        
    - name: admin
      type: user
      attributes:
        id: uuid
        email: string
        department: string
        clearance_level: integer
        
  roles:
    - name: customer
      description: "Regular banking customer"
      grants:
        - view_own_accounts
        - view_own_transactions
        - create_transfers
        - update_own_profile
        
    - name: customer_service
      description: "Customer service representative"
      grants:
        - view_customer_accounts
        - view_transactions
        - acknowledge_alerts
        
    - name: compliance_officer
      description: "Compliance and KYC officer"
      grants:
        - view_all_customers
        - view_kyc_details
        - approve_kyc
        - reject_kyc
        - view_audit_logs
      inherits: [customer_service]
      
    - name: fraud_analyst
      description: "Fraud detection analyst"
      grants:
        - view_all_transactions
        - view_alerts
        - resolve_alerts
        - escalate_alerts
        - freeze_accounts
      inherits: [customer_service]
      
    - name: operations_manager
      description: "Operations manager"
      grants:
        - view_system_metrics
        - view_all_accounts
        - unfreeze_accounts
        - reverse_transactions
      inherits: [fraud_analyst, compliance_officer]
```

### Policies

```yaml
policies:
  - name: CustomerViewOwnData
    version: v1
    effect: allow
    when: |
      subject.role == "customer" AND
      subject.id == data.owner_id
      
  - name: AdminViewSensitiveData
    version: v1
    effect: allow
    when: |
      subject.role in ["compliance_officer", "operations_manager", "system_admin"] AND
      subject.clearance_level >= 3
      
  - name: AdminFreezeAccount
    version: v1
    effect: allow
    when: |
      subject.role in ["fraud_analyst", "operations_manager", "system_admin"]

policy_sets:
  - name: CustomerAccess
    resolution: deny_overrides
    policies: [RequireAuthentication, CustomerViewOwnData]
    
  - name: AdminBasicAccess
    resolution: deny_overrides
    policies: [RequireAuthentication, AdminViewCustomers]
    
  - name: AdminSensitiveAccess
    resolution: deny_overrides
    policies: [RequireAuthentication, AdminViewSensitiveData]
```

---

## UI Layer (v1.4)

### Presentation Views

#### Customer Views

```yaml
ui:
  views:
    - name: CustomerDashboardView
      version: v1
      consumes:
        dataState: CustomerSummary
        version: v1
      guarantees:
        - show_account_overview
        - show_recent_activity
        - show_quick_actions
      interactionModel:
        badges:
          - if: has_alerts
            label: "Action Required"
            severity: warning
        actions:
          - if: true
            action: new_transfer
            label: "New Transfer"
            
    - name: CustomerAccountListView
      version: v1
      consumes:
        dataState: AccountList
        version: v1
      guarantees:
        - show_all_accounts
        - show_balances
      interactionModel:
        filters:
          - field: account_type
            type: enum
          - field: status
            type: enum
        sorts:
          - field: created_at
            default: desc
          - field: balance
            
    - name: CustomerAccountDetailsView
      version: v1
      consumes:
        dataState: AccountRecord
        version: v1
      guarantees:
        - show_account_info
        - show_account_status
      interactionModel:
        badges:
          - if: status == "frozen"
            label: "Account Frozen"
            severity: error
          - if: status == "active"
            label: "Active"
            severity: success
        actions:
          - if: status == "active"
            action: transfer_funds
            label: "Transfer"
          - if: true
            action: download_statement
            label: "Download Statement"
      fieldPermissions:
        - field: account_number
          visible: true
        - field: balance
          visible: true
        - field: available_balance
          visible: true
          
    - name: CustomerBalanceView
      version: v1
      consumes:
        dataState: AccountBalance
        version: v1
      guarantees:
        - show_current_balance
        - show_available_balance
        - show_pending
      interactionModel:
        badges:
          - if: balance < 100
            label: "Low Balance"
            severity: warning
          - if: balance < 0
            label: "Overdrawn"
            severity: error
            
    - name: CustomerTransactionListView
      version: v1
      consumes:
        dataState: TransactionHistory
        version: v1
      guarantees:
        - show_transactions
        - show_running_balance
      interactionModel:
        filters:
          - field: timestamp
            type: date_range
          - field: category
            type: enum
          - field: amount
            type: range
        sorts:
          - field: timestamp
            default: desc
          - field: amount
            
    - name: CustomerTransferFormView
      version: v1
      consumes:
        dataState: TransferDraft
        version: v1
      guarantees:
        - capture_transfer_details
        - show_validation
      interactionModel:
        actions:
          - if: form.valid
            action: submit_transfer
            label: "Transfer Funds"
            intent: CreateTransfer
          - if: true
            action: cancel
            label: "Cancel"
            
    - name: CustomerProfileView
      version: v1
      consumes:
        dataState: CustomerRecord
        version: v1
      guarantees:
        - show_profile_info
        - show_security_settings
      interactionModel:
        actions:
          - if: true
            action: edit_profile
            label: "Edit Profile"
          - if: true
            action: change_password
            label: "Change Password"
      fieldPermissions:
        - field: email
          visible: true
        - field: phone
          visible: true
        - field: ssn
          visible: false  # Customers can't see their full SSN
        - field: kyc_status
          visible: true
```

#### Admin Views

```yaml
    - name: AdminDashboardView
      version: v1
      consumes:
        dataState: SystemMetrics
        version: v1
      guarantees:
        - show_system_health
        - show_key_metrics
        - show_alert_summary
      interactionModel:
        badges:
          - if: open_alerts > 0
            label: "{{open_alerts}} Open Alerts"
            severity: warning
          - if: critical_alerts > 0
            label: "{{critical_alerts}} Critical"
            severity: error
        actions:
          - if: open_alerts > 0
            action: view_alerts
            label: "View Alerts"
            
    - name: AdminCustomerDetailsView
      version: v1
      consumes:
        dataState: CustomerRecord
        version: v1
      guarantees:
        - show_full_customer_info
        - show_kyc_details
        - show_risk_assessment
      interactionModel:
        badges:
          - if: kyc_status == "pending"
            label: "KYC Pending"
            severity: warning
          - if: kyc_status == "verified"
            label: "Verified"
            severity: success
          - if: kyc_status == "rejected"
            label: "Rejected"
            severity: error
          - if: risk_tier == "high"
            label: "High Risk"
            severity: error
        actions:
          - if: kyc_status == "pending"
            action: approve_kyc
            label: "Approve KYC"
            intent: ApproveKYC
          - if: kyc_status == "pending"
            action: reject_kyc
            label: "Reject KYC"
            intent: RejectKYC
      fieldPermissions:
        - field: full_name
          visible: true
        - field: email
          visible: true
        - field: phone
          visible: true
        - field: ssn
          visible:
            policy: AdminViewSensitiveData
          mask:
            unless: subject.clearance_level >= 4
            type: partial
            reveal: last4
        - field: date_of_birth
          visible:
            policy: AdminViewSensitiveData
        - field: kyc_status
          visible: true
        - field: risk_tier
          visible: true
          
    - name: AdminAccountDetailsView
      version: v1
      consumes:
        dataState: AccountRecord
        version: v1
      guarantees:
        - show_full_account_info
        - show_owner_link
        - show_activity_summary
      interactionModel:
        badges:
          - if: status == "frozen"
            label: "Frozen"
            severity: error
          - if: status == "pending_approval"
            label: "Pending Approval"
            severity: warning
        actions:
          - if: status == "active"
            action: freeze_account
            label: "Freeze Account"
            intent: FreezeAccount
          - if: status == "frozen"
            action: unfreeze_account
            label: "Unfreeze Account"
            intent: UnfreezeAccount
            
    - name: AdminAlertDetailsView
      version: v1
      consumes:
        dataState: AlertRecord
        version: v1
      guarantees:
        - show_alert_details
        - show_entity_context
        - show_resolution_history
      interactionModel:
        badges:
          - if: severity == "critical"
            label: "Critical"
            severity: critical
          - if: severity == "high"
            label: "High Priority"
            severity: warning
        actions:
          - if: status == "open"
            action: acknowledge_alert
            label: "Acknowledge"
            intent: AcknowledgeAlert
          - if: status in ["open", "acknowledged"]
            action: resolve_alert
            label: "Resolve"
            intent: ResolveAlert
          - if: status in ["open", "acknowledged"]
            action: escalate_alert
            label: "Escalate"
            intent: EscalateAlert
```

---

## UI Compositions (v1.4)

### Customer Compositions

```yaml
compositions:
  - name: CustomerDashboard
    version: v1
    description: "Customer home with account overview"
    primary: CustomerDashboardView
    navigation:
      label: "Dashboard"
      purpose: home
      policy_set: CustomerAccess
    related:
      - view: CustomerAccountListView
        relationship: detail
        cardinality: many
        navigation:
          label: "My Accounts"
          scope: within_composition
          
  - name: CustomerAccountWorkspace
    version: v1
    description: "Single account with balance and transactions"
    primary: CustomerAccountDetailsView
    navigation:
      label: "{{account.account_type}} ****{{account.account_number_last4}}"
      purpose: accounts
      parent: CustomerDashboard
      param: account_id
      policy_set: CustomerAccess
      indicator:
        when: account.status == "frozen"
        severity: error
    related:
      - view: CustomerBalanceView
        relationship: derived
        required: true
        navigation:
          label: "Balance"
          scope: within_composition
          
      - view: CustomerTransactionListView
        relationship: detail
        cardinality: many
        basis: [transaction_debits_account, transaction_credits_account]
        navigation:
          label: "Transactions"
          scope: within_composition
          
      - view: CustomerTransferFormView
        relationship: action
        trigger: transfer_funds
        policy_set: CustomerTransferAccess
        navigation:
          label: "Transfer"
          scope: on_action
          
  - name: CustomerProfileWorkspace
    version: v1
    description: "Customer profile management"
    primary: CustomerProfileView
    navigation:
      label: "Profile"
      purpose: settings
      policy_set: CustomerAccess
```

### Admin Compositions

```yaml
  - name: AdminDashboard
    version: v1
    description: "Admin home with system metrics"
    primary: AdminDashboardView
    navigation:
      label: "Dashboard"
      purpose: home
      policy_set: AdminBasicAccess
      indicator:
        when: critical_alerts > 0
        severity: critical
    related:
      - view: AdminAlertListView
        relationship: supplementary
        navigation:
          label: "Recent Alerts"
          scope: within_composition
        data_scope:
          status: "open"
          limit: 5
          
  - name: AdminCustomerWorkspace
    version: v1
    description: "Customer details with accounts and history"
    primary: AdminCustomerDetailsView
    navigation:
      label: "{{customer.full_name}}"
      purpose: customers
      parent: AdminCustomerListWorkspace
      param: customer_id
      policy_set: AdminBasicAccess
      indicator:
        when: customer.kyc_status == "pending"
        severity: warning
    related:
      - view: AdminAccountListView
        relationship: detail
        cardinality: many
        basis: customer_owns_account
        navigation:
          label: "Accounts"
          scope: within_composition
        data_scope:
          owner_id: "{{customer.id}}"
          
      - view: AdminAuditLogListView
        relationship: historical
        cardinality: many
        navigation:
          label: "Activity Log"
          scope: within_composition
        data_scope:
          entity_id: "{{customer.id}}"
          
  - name: AdminAccountWorkspace
    version: v1
    description: "Account details with owner and transactions"
    primary: AdminAccountDetailsView
    navigation:
      label: "{{account.account_number}}"
      purpose: accounts
      parent: AdminAccountListWorkspace
      param: account_id
      policy_set: AdminAccountManagement
      indicator:
        when: account.status == "frozen"
        severity: error
    related:
      - view: CustomerBalanceView
        relationship: derived
        required: true
        navigation:
          label: "Balance"
          scope: within_composition
          
      - view: AdminTransactionListView
        relationship: detail
        cardinality: many
        navigation:
          label: "Transactions"
          scope: within_composition
          
      - view: AdminCustomerDetailsView
        relationship: contextual
        basis: customer_owns_account
        navigation:
          label: "Owner"
          scope: within_composition
          
  - name: AdminAlertWorkspace
    version: v1
    description: "Alert details with context"
    primary: AdminAlertDetailsView
    navigation:
      label: "Alert #{{alert.id_short}}"
      purpose: alerts
      parent: AdminAlertListWorkspace
      param: alert_id
      policy_set: AdminAlertManagement
    related:
      - view: AdminAccountDetailsView
        relationship: contextual
        basis: alert_references_entity
        navigation:
          label: "Related Account"
          scope: within_composition
```

---

## Experiences (v1.4)

### Customer Experience

```yaml
experiences:
  - name: CustomerBanking
    version: v1
    description: "Customer-facing banking experience"
    entry_point: CustomerDashboard
    
    includes:
      - CustomerDashboard
      - CustomerAccountWorkspace
      - CustomerProfileWorkspace
      - HelpWorkspace
      
    policy_set: CustomerAccess
    
    unauthenticated:
      redirect_to: LoginWorkspace
      
    on_unauthorized: conceal
```

### Administrator Experience

```yaml
  - name: BankingAdministration
    version: v1
    description: "Administrative banking experience"
    entry_point: AdminDashboard
    
    includes:
      - AdminDashboard
      - AdminCustomerListWorkspace
      - AdminCustomerWorkspace
      - AdminAccountListWorkspace
      - AdminAccountWorkspace
      - AdminTransactionListWorkspace
      - AdminAlertListWorkspace
      - AdminAlertWorkspace
      - AdminKYCQueueWorkspace
      - AdminAuditLogWorkspace
      - AdminReportsWorkspace
      - HelpWorkspace
      
    policy_set: AdminBasicAccess
    
    unauthenticated:
      redirect_to: LoginWorkspace
      
    on_unauthorized: indicate
```

---

## Navigation (v1.4)

```yaml
navigation:
  purposes:
    - name: home
      description: "Primary entry point / dashboard"
    - name: accounts
      description: "Account-related destinations"
    - name: transactions
      description: "Transaction-related destinations"
    - name: customers
      description: "Customer management destinations"
    - name: alerts
      description: "Alert and notification destinations"
    - name: compliance
      description: "Compliance and KYC destinations"
    - name: audit
      description: "Audit and logging destinations"
    - name: reports
      description: "Reporting destinations"
    - name: settings
      description: "User settings and preferences"
    - name: help
      description: "Help and support destinations"
    - name: authentication
      description: "Login and authentication"
      
  scopes:
    - name: within_composition
      description: "Visible when parent composition is active"
    - name: always_visible
      description: "Visible in primary navigation"
    - name: on_action
      description: "Visible when triggered by action"
```

---

## Complete Navigation Hierarchy

### Customer Navigation

```
CustomerBanking Experience
│
├── Dashboard (home) [root]
│   └── My Accounts [within_composition]
│
├── Account Details (accounts)
│   ├── Balance [within_composition]
│   ├── Transactions [within_composition]
│   └── Transfer [on_action]
│
├── Profile (settings)
│
└── Help (help)
```

### Admin Navigation

```
BankingAdministration Experience
│
├── Dashboard (home) [root]
│   └── Recent Alerts [within_composition]
│
├── Customers (customers)
│   └── Customer Details
│       ├── Accounts [within_composition]
│       └── Activity Log [within_composition]
│
├── Accounts (accounts)
│   └── Account Details
│       ├── Balance [within_composition]
│       ├── Transactions [within_composition]
│       └── Owner [within_composition]
│
├── Transactions (transactions)
│
├── Alerts (alerts) [indicator: open_alerts > 0]
│   └── Alert Details
│       └── Related Account [within_composition]
│
├── KYC Queue (compliance) [indicator: pending_kyc > 0]
│
├── Audit Logs (audit)
│
├── Reports (reports)
│
└── Help (help)
```

---

## v1.5 Extensions

### Assets: Check Images (Addendum AK)

Banks need to store check images for regulatory compliance:

```yaml
dataType:
  name: Transaction
  version: v2
  fields:
    id: uuid
    # ... existing fields ...
    check_image: asset  # NEW in v1.5
    
asset:
  name: check_image
  types: [image/jpeg, image/png, image/tiff]
  max_size: 10MB
  transformations:
    thumbnail: resize(width: 300, height: 150)
    full: compress(quality: 85)
  retention:
    duration: 7_years  # Banking regulation
    delete_after: true
  access:
    policy: ViewCheckImage
```

**Cross-reference**: Addendum AK (Asset Semantics)

---

### Notifications: Transaction Alerts (Addendum AN)

Send multi-channel notifications for significant transactions:

```yaml
notification:
  name: LargeTransactionAlert
  trigger:
    event: Transaction.created
    condition: amount > account.alert_threshold
  channels:
    - type: email
      template: large_transaction_email
      to: customer.email
    - type: sms
      template: large_transaction_sms
      to: customer.phone
      condition: customer.sms_alerts_enabled
    - type: push
      template: large_transaction_push
      condition: customer.push_enabled
  delivery:
    retry: 3
    timeout: 30s
  tracking:
    log_delivery: true
    log_read: true
```

**Cross-reference**: Addendum AN (Notification Channel Semantics)

---

### Scheduled Triggers: Monthly Statements (Addendum AM)

Generate account statements on a schedule:

```yaml
scheduledTrigger:
  name: GenerateMonthlyStatement
  schedule: "0 0 1 * *"  # First day of month at midnight
  timezone: account.timezone
  intent: GenerateStatement
  scope: per_entity
  entity_filter: account.status == active AND account.statement_delivery == monthly
  idempotency:
    key: [account.id, date.month]
  retry:
    max_attempts: 3
    backoff: exponential
```

**Cross-reference**: Addendum AM (Scheduled Trigger Semantics)

---

### External Integration: Credit Bureau (Addendum AW)

Integrate with external credit scoring service:

```yaml
externalIntegration:
  name: CreditBureauAPI
  endpoint: https://api.creditbureau.example
  authentication:
    type: api_key
    credential_ref: credit_bureau_key
  operations:
    - name: getCreditScore
      method: POST
      path: /v1/credit-score
      timeout: 5s
  resilience:
    circuit_breaker:
      threshold: 5
      timeout: 30s
      reset_after: 60s
    retry:
      max_attempts: 3
      backoff: exponential
    fallback:
      strategy: use_cached
      cache_ttl: 24h
  observability:
    track_latency: true
    track_errors: true
    alert_on_failure_rate: 0.1
```

**Cross-reference**: Addendum AW (External Integration Semantics)

---

### Data Governance: PII Classification (Addendum AX)

Classify and manage customer data for GDPR compliance:

```yaml
dataGovernance:
  model: Customer
  classification:
    level: pii
    categories: [personal_identifiable, financial, contact]
  retention:
    duration: 7_years  # After account closure
    delete_after: true
    archive_after: 1_year
  right_to_erasure:
    supported: true
    cascade: [AuditLog, Transaction]
    exceptions: [Transaction]  # Keep for audit
    pseudonymize: [Transaction.metadata.customer_name]
  data_lineage:
    track: true
    sources: [kyc_verification, manual_entry]
  audit:
    log_access: true
    log_modification: true
    retention: 10_years
```

**Cross-reference**: Addendum AX (Data Governance Semantics)

---

### Search: Transaction Search (Addendum AL)

Enable customers to search their transaction history:

```yaml
search:
  name: TransactionSearch
  target: TransactionView
  scope: account.id == session.account_id  # User can only search their own
  fields:
    - name: reference
      weight: 2.0
      analyzers: [keyword, edge_ngram]
    - name: category
      weight: 1.5
      facet: true
    - name: amount
      weight: 1.0
      range_facet: true
    - name: timestamp
      weight: 0.5
      range_facet: true
  ranking:
    function: bm25
    boost: recency
  pagination:
    default_size: 25
    max_size: 100
```

**Cross-reference**: Addendum AL (Search Semantics)

---

### Form Binding: Transfer Form (Addendum X)

Bind transfer form to input intent:

```yaml
presentationView:
  name: TransferForm
  version: v1
  consumes:
    dataState: AccountBalance
  triggers:
    intent: InitiateTransfer
    binding: form
    field_mapping:
      - view_field: from_account_id
        supplies: from_account
        source: context
        required: true
      - view_field: to_account_number
        supplies: to_account
        source: input
        required: true
        validation:
          constraint: matches_account_format(to_account_number)
          message: "Invalid account number format"
      - view_field: amount
        supplies: amount
        source: input
        required: true
        validation:
          constraint: amount > 0 AND amount <= balance
          message: "Amount must be positive and not exceed balance"
      - view_field: reference
        supplies: reference
        source: input
        required: false
    confirmation:
      required: true
      message: "Transfer {{amount}} from {{from_account}} to {{to_account}}?"
    on_success:
      action: navigate_back
      notification:
        type: toast
        message: "Transfer initiated successfully"
    on_error:
      action: display_inline
      show_field_errors: true
```

**Cross-reference**: Addendum X (Form Intent Binding)

---

### Accessibility Preferences (Addendum AI)

Support customer accessibility needs:

```yaml
accessibilityPreferences:
  name: BankingAccessibility
  scope: session
  preferences:
    - name: high_contrast
      type: boolean
      default: false
      applies_to: [all_views]
    - name: font_scale
      type: enum
      values: [small, medium, large, xlarge]
      default: medium
      applies_to: [all_views]
    - name: reduce_motion
      type: boolean
      default: false
      applies_to: [animations, transitions]
    - name: screen_reader_hints
      type: boolean
      default: false
      applies_to: [all_views]
  keyboard_navigation:
    enabled: true
    shortcuts:
      - key: "Alt+H"
        action: navigate_home
      - key: "Alt+T"
        action: navigate_transactions
      - key: "Alt+N"
        action: new_transfer
```

**Cross-reference**: Addendum AI (Accessibility and User Preferences)

---

### Conversational UI: Banking Assistant (Addendum AV)

Voice/chat interface for common banking tasks:

```yaml
conversationalExperience:
  name: BankingAssistant
  channels: [chat, voice]
  intents:
    - intent: CheckBalance
      utterances:
        - "What's my balance"
        - "How much money do I have"
        - "Show my account balance"
      response:
        type: template
        template: "Your {{account_type}} account balance is {{balance}}"
        data_source: AccountBalance
    - intent: InitiateTransfer
      utterances:
        - "Transfer money"
        - "Send money to [account]"
        - "Pay [amount] to [account]"
      multi_turn: true
      slots:
        - name: to_account
          type: account_number
          prompt: "Which account would you like to transfer to?"
          validation: account_exists(to_account)
        - name: amount
          type: decimal
          prompt: "How much would you like to transfer?"
          validation: amount > 0 AND amount <= balance
      confirmation:
        required: true
        template: "Transfer {{amount}} to {{to_account}}?"
      action: submit_intent
    - intent: TransactionHistory
      utterances:
        - "Show my transactions"
        - "Recent activity"
        - "What did I spend on [category]"
      response:
        type: list
        data_source: TransactionView
        limit: 10
  fallback:
    strategy: human_handoff
    message: "Let me connect you with a representative"
```

**Cross-reference**: Addendum AV (Conversational Experience Semantics)

---

### Durable Workflow: Loan Approval (Addendum AT)

Multi-day loan approval process with human steps:

```yaml
durableWorkflow:
  name: LoanApproval
  trigger:
    intent: SubmitLoanApplication
  steps:
    - name: creditCheck
      action: callExternalIntegration
      integration: CreditBureauAPI
      timeout: 5m
      retry:
        max_attempts: 3
      compensation: logCreditCheckFailure
    
    - name: automaticDecision
      action: evaluatePolicy
      policy: LoanAutoApprovalPolicy
      branches:
        auto_approved:
          condition: credit_score > 750 AND amount < 50000
          next: approved
        auto_rejected:
          condition: credit_score < 600
          next: rejected
        manual_review:
          condition: true
          next: manualReview
    
    - name: manualReview
      action: human_task
      assigned_to: underwriting_team
      timeout: 3_days
      escalation:
        after: 2_days
        to: senior_underwriter
      ui: LoanReviewComposition
      branches:
        approved: { next: approved }
        rejected: { next: rejected }
        request_more_info: { next: waitForCustomer }
    
    - name: waitForCustomer
      action: wait_for_event
      event: CustomerDocumentsUploaded
      timeout: 7_days
      on_timeout: rejected
      on_event: manualReview
    
    - name: approved
      action: submit_intent
      intent: ApproveLoan
      notification: LoanApprovedNotification
      end: true
    
    - name: rejected
      action: submit_intent
      intent: RejectLoan
      notification: LoanRejectedNotification
      end: true
  
  state_persistence:
    duration: 90_days
  observability:
    track_step_duration: true
    alert_on_timeout: true
```

**Cross-reference**: Addendum AT (Durable Workflow Semantics)

---

### Delegation: Authorized User (Addendum AU)

Allow account holder to delegate access to another person:

```yaml
delegation:
  name: AuthorizedUser
  delegator: account.owner_id
  delegate: authorized_user.id
  permissions:
    - ViewBalance
    - ViewTransactions
    - InitiateTransfer
  scope:
    accounts: [delegator.accounts]
    max_transfer_amount: 1000
  temporal:
    starts_at: delegation.created_at
    expires_at: delegation.created_at + 1_year
    auto_revoke: true
  consent:
    required: true
    captured_at: delegation.created_at
    method: explicit
    evidence: delegation.consent_signature
  audit:
    log_all_actions: true
    notify_delegator: true
```

**Cross-reference**: Addendum AU (Delegation and Consent Semantics)

---

## Key Design Decisions

### Intent Over Implementation

| Declared | NOT Declared |
|----------|--------------|
| View relationships | Layout/rendering |
| Navigation purposes | Nav bar styles |
| Field visibility | CSS/colors |
| Indicator conditions | Badge UI |
| On_unauthorized behavior | Error page design |

### Policy Integration

| Level | Policy Source |
|-------|---------------|
| Experience | `experience.policy_set` |
| Composition | `composition.navigation.policy_set` |
| Related View | `related.policy` or `related.policy_set` |
| Field | `fieldPermissions.visible.policy` |

### Hierarchy Rules

1. Roots have no `parent`
2. Hierarchy must be acyclic
3. `purpose` groups for rendering hints
4. `indicator` for dynamic state

---

## Guarantees Achieved

### ✅ Correctness (v1.3)
- Transactions are immutable facts
- Balance is derived, not stored
- Invariant violations are detected

### ✅ Availability (v1.3)
- Entity-scoped authority
- Regional failure isolation
- No global coordination

### ✅ UI Composition (v1.4)
- Semantic view grouping
- Policy-controlled visibility
- Field-level masking

### ✅ Navigation (v1.4)
- Hierarchical from parent relationships
- Purpose-based grouping
- Dynamic indicators

### ✅ Security (v1.4)
- Role-based access
- Field masking for PII
- Experience-level policies

### ✅ Assets (v1.5)
- Check images with lifecycle management
- Transformation hints for thumbnails
- 7-year retention for compliance

### ✅ Notifications (v1.5)
- Multi-channel alerts (email, SMS, push)
- Delivery tracking and retry
- User preference respect

### ✅ Scheduled Automation (v1.5)
- Monthly statement generation
- Timezone-aware scheduling
- Idempotent execution

### ✅ External Integration (v1.5)
- Credit bureau with circuit breaker
- Fallback to cached data
- Observability and alerting

### ✅ Data Governance (v1.5)
- PII classification and tracking
- GDPR right to erasure
- 7-year audit retention

### ✅ Search (v1.5)
- Transaction search with facets
- User-scoped security
- Ranking and pagination

### ✅ Accessibility (v1.5)
- High contrast and font scaling
- Screen reader support
- Keyboard navigation

### ✅ Conversational UI (v1.5)
- Voice and chat interfaces
- Multi-turn conversations
- Human handoff fallback

### ✅ Long-Running Processes (v1.5)
- Loan approval workflow
- Human task integration
- State persistence and compensation

### ✅ Delegation (v1.5)
- Authorized user access
- Time-bounded permissions
- Consent capture and audit

---

**Version**: 1.5
**Status**: Normative Example
**Last Updated**: 2026-01-18
