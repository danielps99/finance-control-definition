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
- `IF (paid_amount == 0)` -> **PENDING**
- `IF (paid_amount < total_amount AND paid_amount > 0)` -> **PARTIALLY_PAID**
- `IF (paid_amount >= total_amount)` -> **PAID**
- *Check Due Date*: If Pending/Partial and `due_date < NOW` -> **OVERDUE**
 Riverside logic.
