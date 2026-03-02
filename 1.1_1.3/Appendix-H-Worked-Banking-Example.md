# Appendix H: Worked Banking Example (v1.3)

## Overview

This appendix provides a complete, worked example of modeling a banking system using SMS's invariant-oriented approach. The example is fully abstracted to illustrate principles rather than a specific implementation.

---

## Problem Statement

Design a system that supports:
1. Creating accounts
2. Processing transactions (debits and credits)
3. Maintaining accurate balances
4. Ensuring balances never go negative
5. Operating across multiple regions

### The Critical Invariant

> **Account balance must never be negative.**

This invariant spans multiple entities (Account + all Transactions) and is the central challenge.

---

## The Naive (Incorrect) Model

### Initial Attempt

```yaml
# ❌ INCORRECT - Do not use

dataType:
  name: Account
  version: v1
  fields:
    id: uuid
    owner_id: uuid
    balance: decimal  # ← PROBLEM: Derived state in entity

transition:
  name: ProcessTransaction
  precondition: account.balance >= transaction.amount  # ← PROBLEM: Coupling
  action:
    - account.balance -= transaction.amount  # ← PROBLEM: Coordination required
```

### Why This Fails

1. **Balance is derived**: It's calculated from transactions, not stored independently
2. **Precondition creates dependency**: Writing a transaction requires reading Account
3. **Coordination required**: Multiple concurrent transactions conflict
4. **Authority unclear**: Who updates balance during authority migration?

---

## The Correct Decomposition

### Step 1: Identify Entities

**Account** - Identity and configuration:
```yaml
dataType:
  name: Account
  version: v1
  fields:
    id: uuid
    owner_id: uuid
    account_type: enum[checking, savings]
    created_at: timestamp
    # NO balance field

model:
  name: Account
  mutability:
    scope: entity
    exclusive: true
  authority:
    scope: entity
    resolution: entity.home_region
    migration:
      allowed: false  # Financial data stays in region
```

**Transaction** - Immutable facts:
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
    reference: string

model:
  name: Transaction
  mutability:
    scope: append-only  # Transactions never change
```

### Step 2: Define Relationships

```yaml
relationship:
  name: debit
  from: Transaction
  to: Account
  type: affects
  cardinality: many-to-one
  semantics:
    causal: true   # Account must exist first
    ordered: true  # Order matters for balance

relationship:
  name: credit
  from: Transaction
  to: Account
  type: affects
  cardinality: many-to-one
  semantics:
    causal: true
    ordered: true
```

### Step 3: Define Views

**Account Balance View**:
```yaml
view:
  name: AccountBalance
  
  inputs:
    - Account
    - Transaction
  
  aggregation:
    type: compute
    expression: |
      FOR account IN Account:
        credits = SUM(t.amount FOR t IN Transaction WHERE t.credit_account_id == account.id)
        debits = SUM(t.amount FOR t IN Transaction WHERE t.debit_account_id == account.id)
        balance = credits - debits
        EMIT { account_id: account.id, balance: balance }
  
  invariant:
    expression: balance >= 0
    on_violation: flag  # Don't block, flag for review
  
  authority_agnostic: true
  epoch_tolerant: true
```

---

## Write Flow

### Creating a Transaction

```yaml
sms:
  flows:
    - name: ProcessTransaction
      triggeredBy:
        inputIntent: CreateTransaction
      
      steps:
        # Step 1: Validate input format
        - validate: transaction_format
        
        # Step 2: Check accounts exist (read-only)
        - work_unit: verify_accounts_exist
        
        # Step 3: Get CAS token for debit account
        - work_unit: get_authority_token
          for: debit_account
        
        # Step 4: Append transaction (immutable)
        - persist: Transaction
          idempotency:
            key: [transaction.id]
        
        # Step 5: Trigger view update (async)
        - emit: transaction_created
      
      failure:
        policy: abort  # Don't retry, let client handle
```

### What Happens to Balance

1. Transaction appended to stream
2. View worker receives transaction event
3. View recalculates balance for affected accounts
4. If invariant violated (balance < 0), view flags it
5. Flagged violations trigger alerts/review

---

## Authority Interaction

### Stable Authority for Accounts

```yaml
authority:
  scope: entity
  resolution: account.jurisdiction_region
  migration:
    allowed: false  # Regulatory requirement
  constraints:
    - type: governance
      rule: account.jurisdiction == authority_region.jurisdiction
```

### Transactions Follow Account Authority

Transactions affecting an account are written to the same region as the account's authority.

```go
func (w *TransactionWriter) Route(tx Transaction) string {
    // Route to debit account's authoritative region
    accountState := w.authorityStore.Get(tx.DebitAccountID)
    return accountState.AuthorityRegion
}
```

### View Spans Regions

The AccountBalance view consumes events from all regions:

```yaml
view:
  name: AccountBalance
  inputs:
    - Account      # From multiple regions
    - Transaction  # From multiple regions
  
  authority_agnostic: true  # Handles multi-region events
```

---

## Handling Invariant Violations

### What If Balance Goes Negative?

Since we don't block writes, negative balances are possible:

1. **Detection**: View calculates negative balance
2. **Flagging**: View marks account with violation
3. **Alert**: System emits signal for review
4. **Response**: Operations team reviews, may freeze account

### Violation Response Flow

```yaml
sms:
  flows:
    - name: HandleNegativeBalance
      triggeredBy:
        signal:
          type: invariant_violation
          view: AccountBalance
      
      steps:
        - work_unit: freeze_account
        - work_unit: notify_operations
        - work_unit: log_audit_event
```

### Why This Is Acceptable

1. **Rare occurrence**: Normal validation prevents most cases
2. **Detection guaranteed**: Every transaction updates view
3. **Clear remediation**: Freeze + review process
4. **No coordination tax**: Normal operations unaffected

---

## Multi-Region Behavior

### Region Layout

```
US-EAST (Primary for US accounts)
 ├─ STREAM transactions.v1.us-east
 ├─ KV account_balance.v1.us-east
 └─ Authority for US accounts

EU-WEST (Primary for EU accounts)
 ├─ STREAM transactions.v1.eu-west
 ├─ KV account_balance.v1.eu-west
 └─ Authority for EU accounts

GLOBAL (Read-only aggregation)
 └─ View replicates from both regions
```

### Cross-Region Transfer

When transferring between US and EU accounts:

```go
func ProcessCrossRegionTransfer(tx Transaction) error {
    // 1. Write transaction to source account's region
    sourceRegion := getAuthorityRegion(tx.DebitAccountID)
    writeToRegion(sourceRegion, tx)
    
    // 2. Views in both regions process the event
    //    (via event replication or multi-region stream)
    
    // 3. Balance views update independently
    //    Source region: balance decreases
    //    Target region: balance increases (via replicated event)
    
    return nil
}
```

### Regional Failure

If US-EAST fails:
- US account writes: **Unavailable**
- US account reads: **Stale** (from cached view)
- EU account writes: **Fully operational**
- EU account reads: **Fully operational**

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

### ✅ Consistency

- Eventual balance accuracy
- CAS tokens prevent conflicts
- Epoch ordering in views

### ✅ Auditability

- Complete transaction history
- Rebuildable balances
- Clear violation trail

---

## Complete Specification

### Entities

```yaml
# Account entity
dataType:
  name: Account
  version: v1
  fields:
    id: uuid
    owner_id: uuid
    account_type: enum[checking, savings, credit]
    created_at: timestamp
    jurisdiction: string

model:
  name: Account
  mutability:
    scope: entity
    exclusive: true
  authority:
    scope: entity
    resolution: entity.jurisdiction
    migration:
      allowed: false

# Transaction entity
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
    metadata: map

model:
  name: Transaction
  mutability:
    scope: append-only
```

### Relationships

```yaml
relationships:
  - name: debit
    from: Transaction
    to: Account
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
  
  - name: credit
    from: Transaction
    to: Account
    cardinality: many-to-one
    semantics:
      causal: true
      ordered: true
```

### Views

```yaml
view:
  name: AccountBalance
  inputs: [Account, Transaction]
  
  aggregation:
    credits: sum(t.amount WHERE t.credit_account_id == account.id)
    debits: sum(t.amount WHERE t.debit_account_id == account.id)
    balance: credits - debits
  
  invariant:
    expression: balance >= 0
    on_violation: flag
  
  authority_agnostic: true
  epoch_tolerant: true

view:
  name: TransactionHistory
  inputs: [Transaction]
  
  aggregation:
    type: list
    order_by: timestamp DESC
  
  authority_agnostic: true
```

### Policies

```yaml
policy:
  name: TransactionAmountValid
  appliesTo:
    type: inputIntent
    name: CreateTransaction
  effect: deny
  when: input.amount <= 0

policy:
  name: AccountMustExist
  appliesTo:
    type: transition
    name: ProcessTransaction
  effect: deny
  when: |
    NOT exists(Account, input.debit_account_id) 
    OR NOT exists(Account, input.credit_account_id)
```

---

## Comparison: Before and After

| Aspect | Traditional | SMS Invariant-Oriented |
|--------|-------------|----------------------|
| Balance storage | Mutable field | Derived view |
| Transaction write | Read → Check → Write | Append only |
| Coordination | Lock + Update | None |
| Failure isolation | Transaction rollback | Entity-scoped |
| Invariant enforcement | Blocking | Detecting |
| Multi-region | Complex 2PC | Regional isolation |

---

## Key Takeaways

1. **Separate facts from derivations**: Transactions are facts, balance is derived
2. **Views absorb complexity**: Balance calculation lives in view
3. **Accept eventual consistency**: Invariant violations detected, not prevented
4. **Authority at entity level**: Accounts independent, transactions follow
5. **Append-only for immutable facts**: Transactions never change

This model handles millions of transactions without coordination, scales horizontally, and survives regional failures gracefully.

---

**Version**: 1.3
**Status**: Normative Example
**Last Updated**: 2026-01-03
