# Appendix H: Worked Banking Example (v1.4)

## Overview

This appendix provides a complete, worked example of modeling a banking system using SMS's invariant-oriented approach and the new v1.4 UI composition grammar. The example demonstrates:

- **v1.3**: Entity decomposition, relationships, views, and authority
- **v1.4**: UI compositions, experiences, field permissions, and navigation

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

### ✅ Correctness
- Transactions are immutable facts
- Balance is derived, not stored
- Invariant violations are detected

### ✅ Availability
- Entity-scoped authority
- Regional failure isolation
- No global coordination

### ✅ UI Composition
- Semantic view grouping
- Policy-controlled visibility
- Field-level masking

### ✅ Navigation
- Hierarchical from parent relationships
- Purpose-based grouping
- Dynamic indicators

### ✅ Security
- Role-based access
- Field masking for PII
- Experience-level policies

---

**Version**: 1.4
**Status**: Normative Example
**Last Updated**: 2026-01-11
