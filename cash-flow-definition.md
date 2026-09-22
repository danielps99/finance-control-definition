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
  "other_movements_net": 0.00,
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
         "other_movements_net": ...,
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
> **Reactive Application Strategy:** `balances_history` is updated **reactively on every financial movement**, rather than waiting for a delayed batch job or relying on database triggers:
> 1. **Immediate Transactional Reaction**: Whenever an `account_movement` (payment, receipt, transfer, or reversal) is created or modified, the Java `AccountMovementService` reactively updates `balances_history` for that date (and propagates to future dates) within the exact same `@Transactional` boundary.
> 2. **Real-time & Explicit**: Provides real-time history availability while keeping all calculation logic explicitly testable and maintainable in Java application code.

### B. Interval Calculations
1. **Historical Logic (T < Today)**:
    - **Inflows**: Realized from credits in `account_movements`.
    - **Outflows**: Realized from debits in `account_movements`.
    - **Internal Transfer Handling**:
      - **SYNTHETIC Mode (Consolidated)**: Internal transfers are excluded from `inflows` and `outflows` because net workspace liquidity is unchanged.
      - **ANALYTIC Mode (Per-Account)**: Internal transfers are evaluated per account (inflow for destination account, outflow for source account).
2. **Future Logic (T > Today)**:
    - **Inflows**: Projected from `receivables`.
    - **Outflows**: Projected from `payables` and `credit_card_invoices`.
3. **Today Logic (T == Today)**:
    - Combined Realized (movements today) and Projected (due today).

---

## 4. Category Filtering Rules & Impact Example

### Filtering Rules
- **Hierarchy**: Only leaf categories are allowed.
- **Transfers & Unselected Movements**: Internal transfers carry no categories, and movements outside the selected categories are excluded from `inflows` and `outflows`. When category filtering is applied (`categoryIds` present), these non-matching movements are aggregated into `other_movements_net`.
- **Opening & Closing Balance Reconciliation**: **Opening balance is NOT filtered by categories** and represents actual total account liquidity. `net_change` reflects strictly category movements (`inflows.total - outflows.total`). `closing_balance` reconciles actual ending bank liquidity via the formula: `closing_balance = opening_balance + net_change + other_movements_net`.

### Category Filter Impact Example

Assuming account start balance is **$10,000.00**.
On 2026-08-03:
- Income (Category: "Software Sales", UUID `cat-1`): +$2,000.00
- Expense (Category: "Office Supplies", UUID `cat-2`): -$500.00

**1. Unfiltered Query (`categoryIds = []`):**
```json
{
  "date": "2026-08-03",
  "opening_balance": 10000.00,
  "inflows": { "realized": 2000.00, "projected": 0.00, "total": 2000.00 },
  "outflows": { "realized": 500.00, "projected": 0.00, "total": 500.00 },
  "net_change": 1500.00,
  "other_movements_net": 0.00,
  "closing_balance": 11500.00
}
```

**2. Filtered Query (`categoryIds = ["cat-1"]`):**
```json
{
  "date": "2026-08-03",
  "opening_balance": 10000.00,
  "inflows": { "realized": 2000.00, "projected": 0.00, "total": 2000.00 },
  "outflows": { "realized": 0.00, "projected": 0.00, "total": 0.00 },
  "net_change": 2000.00,
  "other_movements_net": -500.00,
  "closing_balance": 11500.00
}
```
*(Notice `inflows` and `outflows` strictly measure `cat-1` activity (+2000.00), unselected expenses (-500.00) move to `other_movements_net`, and `closing_balance` perfectly reconciles to actual ending bank liquidity of $11,500.00).*

