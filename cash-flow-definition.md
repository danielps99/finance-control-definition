# Cash Flow Report Definition

**Endpoint:** `POST /reports/cash-flow`  
**Parameters:**
- `startDate`: YYYY-MM-DD
- `endDate`: YYYY-MM-DD
- `viewType`: 'DAILY' or 'MONTHLY'
- `opening_balance`: 'ZERO' or 'PREVIOUS_DAY'
- `mode`: 'SYNTHETIC' or 'ANALYTIC'
- `categoryIds`: List of UUIDs (Optional)

## 1. Concept
The Cash Flow report aggregates historical and projected financial data to provide a view of liquidity. It operates in two modes:
- **Realized (Historical)**: Based on `account_movements`.
- **Projected (Future)**: Based on `receivables` and `payables`.

### Aggregation Modes
- **SYNTHETIC**: Sum of all accounts (Consolidated).
- **ANALYTIC**: Breakdown by each account.

## 2. Report Row Structure

### SYNTHETIC Mode
```json
{
  "date": "YYYY-MM-DD",
  "opening_balance": 1000.00,
  "inflows": { "realized": 500.00, "projected": 200.00, "total": 700.00 },
  "outflows": { "realized": 100.00, "projected": 50.00, "total": 150.00 },
  "net_change": 550.00,
  "closing_balance": 1550.00
}
```

### ANALYTIC Mode
```json
[
  {
    "date": "YYYY-MM-DD",
    "accounts": [
       {
         "account_id": "UUID",
         "account_name": "Bank Name",
         "opening_balance": 1000.00,
         "inflows": { ... },
         "outflows": { ... },
         "net_change": ...,
         "closing_balance": ...
       }
    ]
  }
]
```

## 3. Calculation Logic

### A. Opening Balance Calculation (The Anchor)
- **Constraint**: `startDate <= Today`.
- **Strategy**:
  1. Find `balances_history` for `startDate - 1 day`.
  2. Fallback to last entry before `startDate`.
  3. Default to 0 if no history exists.

> [!NOTE]
> **Reactive Strategy:** Instead of daily jobs, DB triggers update `balances_history` on every movement (SET balance = balance + Delta WHERE date >= MovementDate).

### B. Interval Calculations
1. **Historical Logic (T < Today)**:
    - **Inflows**: Realized from credits in `account_movements`.
    - **Outflows**: Realized from debits in `account_movements`.
2. **Future Logic (T > Today)**:
    - **Inflows**: Projected from `receivables`.
    - **Outflows**: Projected from `payables` and `credit_card_invoices`.
3. **Today Logic (T == Today)**:
    - Combined Realized (movements today) and Projected (due today).

## 4. Category Filtering Rules
- **Hierarchy**: Only leaf categories are allowed.
- **Transfers**: Transfers have no categories; they are grouped as "Transfers" category if filtering is active.
- **Opening Balance**: NOT affected by category filters to maintain total funds context.
