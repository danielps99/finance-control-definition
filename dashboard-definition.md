# Dashboard Calculation & Logic

**Endpoint:** `GET /dashboard?referenceDate={YYYY-MM-DD}`  
**Default referenceDate:** Today  
**Constraint:** `referenceDate` >= Today (Historical view via Reports only)

## 1. Concept: "Snapshot & Forecasting"
The dashboard focuses on current financial health and future projections.
- **Current View**: If `referenceDate == Today`, show the live state of finances.
- **Future View**: If `referenceDate > Today`, show the projected state based on scheduled items.

## 2. KPI Cards (Snapshot at referenceDate)

### A. Total Balance (Net Worth)
- **Definition**: Sum of money in all accounts (Cash + Bank + Wallet).
- **Formula**: Sum of cached `balance` of each account.
- **Note**: Strictly *realized* balance. Does not include projections.

### B. Total Receivable (Outstanding)
- **Definition**: Money expected but not yet received.
- **Logic Separation**:
  1. **Overdue**: `due_date < referenceDate`
  2. **Projected**: `due_date >= referenceDate` AND `due_date <= (referenceDate + X days)`
- **Formula**: Sum of (`total_amount - received_amount`) where:
  - `due_date <= referenceDate`
  - `status` IN (`PENDING`, `PARTIALLY_RECEIVED`, `OVERDUE`)
  - `active = TRUE`

### C. Total Payable (Liabilities)
- **Definition**: Money expected to be paid.
- **Logic Separation**:
  1. **Overdue**: `due_date < referenceDate`
  2. **Projected**: `due_date >= referenceDate` AND `due_date <= (referenceDate + X days)`
- **Formula**: Sum of (`total_amount - paid_amount`) where:
  - `due_date <= referenceDate` (If credit card, use invoice due date)
  - `status` IN (`PENDING`, `PARTIALLY_PAID`, `OVERDUE`)
  - `active = TRUE`

### D. Credit Card Available Limit
- **Definition**: Sum of `limit_available` from all cards.
- **Formula**: Get cached `limit_available` of each credit card.

## 3. JSON Response Example

```json
{
  "current": {
    "referenceDate": "2026-02-04",
    "netWorth": {
      "totalBalance": 3000.00,
      "accounts": [
        { "name": "Account 1", "balance": 1000.00 },
        { "name": "Account 2", "balance": 2000.00 }
      ]
    },
    "receivable": {
      "overdue": 500.00,
      "dueToday": 1000.00
    },
    "payable": {
      "overdue": 200.00,
      "dueToday": 300.00
    },
    "creditCard": {
      "totalLimitAvailable": 8000.00,
      "cards": [
        { "name": "Credit Card 1", "limitAvailable": 5000.00 },
        { "name": "Credit Card 2", "limitAvailable": 3000.00 }
      ]
    }
  },
  "projection": {
    "referenceDate": "2026-02-15",
    "receivable": 500.00,
    "payable": 200.00,
    "receivableNextDays": 500.00,
    "payableNextDays": 200.00
  }
}
```
