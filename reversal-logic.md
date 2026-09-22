# Reversal Transaction Strategy

## Core Principle: Immutability
In financial systems, you should **NEVER DELETE** a record from the `account_movements` (Ledger) table once committed. Instead, create a **Compensating Transaction**.

## Scenario 1: Reversing a Payment (Payable)

### Initial State
- **Payable #10**: Total: $100.00, Paid: $100.00, Status: `PAID`
- **Movement #50**: Type: `DEBIT`, Amount: $100.00, Account: Bank A

### The Fix (Reversal)
1. **Create a Compensating "Credit" Movement**:
   - `type`: `CREDIT`
   - `amount`: $100.00
   - `account_id`: Same account (Bank A)
   - `payable_id`: #10
   - `description`: "Reversal of payment #50 - Error correction"
2. **Update the Payable Entity**:
   - `paid_amount`: $0
   - `status`: `PENDING`

## Scenario 2: Reversing a Receivable (Income)

### Initial State
- **Receivable #20**: Total: $500.00, Received: $500.00, Status: `RECEIVED`
- **Movement #51**: Type: `CREDIT`, Amount: $500.00

### The Fix (Reversal)
1. **Create Compensating "Debit" Movement**:
   - `type`: `DEBIT`
   - `amount`: $500.00
   - `receivable_id`: #20
   - `description`: "Reversal of receipt #51"
2. **Update Receivable Entity**:
   - `received_amount`: $0
   - `status`: `PENDING`

## Scenario 3: Reversing a Credit Card Invoice Payment

### Initial State
- **Credit Card Invoice #30**: Total: $1,200.00, Paid: $1,200.00, Status: `PAID`
- **Movement #52**: Type: `DEBIT`, Amount: $1,200.00, Account: Bank A, `credit_card_invoice_id`: #30

### The Fix (Reversal)
1. **Create Compensating "Credit" Movement**:
   - `type`: `CREDIT`
   - `amount`: $1,200.00
   - `account_id`: Same account (Bank A)
   - `credit_card_invoice_id`: #30
   - `reversed_movement_id`: #52
   - `description`: "Reversal of credit card invoice payment #52"
2. **Update the Credit Card Invoice Entity**:
   - `paid_amount`: $0.00
   - `status`: Rollback status to `CLOSED` (or `OPEN` if cutoff date has not passed).
3. **Update Account Balance**:
   - `balance`: `balance + $1,200.00`

## Scenario 4: Reversing an Internal Transfer

### Initial State
- **Transfer #40**: Amount: $1,000.00, Source: Bank A (Debit #60), Destination: Bank B (Credit #61)
- **Movement #60**: Type: `DEBIT`, Amount: $1,000.00, Account: Bank A, `transfer_id`: #40
- **Movement #61**: Type: `CREDIT`, Amount: $1,000.00, Account: Bank B, `transfer_id`: #40

### The Fix (Reversal)
1. **Create Dual Compensating Movements**:
   - **Compensating Credit for Source Account (Bank A)**:
     - `type`: `CREDIT`
     - `amount`: $1,000.00
     - `account_id`: Bank A
     - `transfer_id`: #40
     - `reversed_movement_id`: #60
     - `description`: "Reversal of transfer #40 (Source refund)"
   - **Compensating Debit for Destination Account (Bank B)**:
     - `type`: `DEBIT`
     - `amount`: $1,000.00
     - `account_id`: Bank B
     - `transfer_id`: #40
     - `reversed_movement_id`: #61
     - `description`: "Reversal of transfer #40 (Destination debit)"
2. **Update Transfer Entity**:
   - `status`: `REVERSED`
3. **Update Account Balances**:
   - `Bank A balance`: `balance + $1,000.00`
   - `Bank B balance`: `balance - $1,000.00`

---

## Implementation Details

### Cache vs. Ledger Consistency
Application logic **MUST** be transactional.

```sql
BEGIN TRANSACTION;
  -- Record the correction
  INSERT INTO account_movements (type, amount, ...) VALUES ('CREDIT', 100, ...);
  -- Update the cached source
  UPDATE payables SET paid_amount = paid_amount - 100 WHERE id = 10;
  -- Update the account balance
  UPDATE accounts SET balance = balance + 100 WHERE id = ...; 
COMMIT;
```

### Recommended Schema Improvements
- **`reversed_movement_id`**: FK -> `account_movements.id` to link the correction explicitly.

### Status Transition Logic
Recalculate status after reversal:
- `IF (paid_amount == 0)` -> **PENDING** (or **CLOSED** for credit card invoices)
- `IF (paid_amount < total_amount AND paid_amount > 0)` -> **PARTIALLY_PAID**
- `IF (paid_amount >= total_amount)` -> **PAID**
- *Check Due Date*: If Pending/Partial and `due_date < NOW` -> **OVERDUE**

Reversal logic rules ensure ledger integrity and auditability across all entity types.

