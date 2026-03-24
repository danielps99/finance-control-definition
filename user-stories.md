# User Stories

## 1. Authentication & Access
**As a user**, I want to log into the system securely and access my specific environments, **so that** I can protect my financial data and manage different contexts.

**Acceptance Criteria:**
- User can log in with email and password.
- User can log out to end the session.
- Access to dashboard and internal pages is restricted to authenticated users.
- Session persists during active use.
- **User can switch between Workspaces (e.g., Personal, Business) if they have access to multiple.**

## 2. Dashboard Overview
**As a user**, I want to see a high-level overview of my finances immediately upon login, **so that** I can quickly assess my financial health.

**Acceptance Criteria:**
- Dashboard displays "Total Receivable" for the current month.
- Dashboard displays "Total Payable" for the current month.
- Dashboard highlights overdue invoices/bills count and total value.
- Dashboard shows a "Net Worth" summary (Consolidated total of all asset accounts).
- Dashboard shows a cash flow chart (projection) for the next 30-60 days.
- Dashboard provides "Quick Action" buttons to immediately register a new Receivable or Payable.
- Clicking on summary cards navigates to the respective filtered list.

## 3. Customer Management
**As a user**, I want to manage a list of customers, **so that** I can easily associate them with invoices and track who owes me.

**Acceptance Criteria:**
- Create, edit, and delete (soft-delete/archive) customer records.
- Required fields: Name/Company.
- Optional fields: Email, Phone/WhatsApp, Notes.
- Search and filter functionality for the customer list.

## 4. Supplier Management
**As a user**, I want to manage a list of suppliers, **so that** I can track my expenses and know who I need to pay.

**Acceptance Criteria:**
- Create, edit, and delete (soft-delete/archive) supplier records.
- Required fields: Name/Company.
- Optional fields: Email, Phone, Notes.
- Search and filter functionality for the supplier list.

## 5. Accounts Receivable (Invoices)
**As a user**, I want to register receivables, **so that** I can track incoming payments and manage my cash flow.

**Acceptance Criteria:**
- Register a new receivable with: Customer, Description, Value, Issue Date, Due Date.
- **Assign to a Receivable Group (e.g., "Consulting Project A") for better tracking.**
- Status tracking: Pending, Paid, Overdue.
- **Option to attach proofs via File upload (PDF/Image) or External Link.**
- Ability to mark as Paid manually.
- System automatically flags as Overdue if the due date passes without payment.

## 6. Accounts Payable (Expenses)
**As a user**, I want to register expenses, **so that** I can ensure bills are paid on time and track my spending.

**Acceptance Criteria:**
- Register a new expense with: Supplier, Description, Value, Due Date, Payment Method.
- **Assign to a Payable Group (e.g., "Recurring Bills").**
- Status tracking: Pending, Paid, Overdue.
- **Option to attach proofs via File upload (PDF/Image) or External Link.**
- Ability to mark as Paid manually.
- System automatically flags as Overdue if the due date passes without payment.

## 7. Search & Filtering
**As a user**, I want to filter and search through my transactions, **so that** I can quickly locate specific invoices or expenses.

**Acceptance Criteria:**
- Filter by Status (Pending, Paid, Overdue).
- Filter by Entity (Customer, Supplier).
- Filter by Date Range (Issue Date, Due Date).
- Sort results by Value, Date, or Status.
- Fast and responsive search by description or entity name.

## 8. Notifications & Reminders
**As a user**, I want to receive alerts about upcoming and overdue payments, **so that** I avoid late fees and missed collections.

**Acceptance Criteria:**
- Email reminder sent 2-3 days before a due date.
- Immediate notification when a transaction becomes overdue.
- User option to enable/disable specific notification types.

## 9. Export & Reporting
**As a user**, I want to export my financial data, **so that** I can share it with my accountant or perform offline analysis.

**Acceptance Criteria:**
- Export Receivables list to CSV/Excel.
- Export Payables list to CSV/Excel.
- Exports respect currently applied filters.
- Monthly summary report of Total Income vs. Total Expenses.

## 10. Financial Account Management
**As a user**, I want to manage multiple financial accounts (e.g., Cash, Bank), **so that** I can track balances separately.

**Acceptance Criteria:**
- Create, edit, and delete (soft-delete/archive) financial accounts.
- Define account type (Cash, Bank, Wallet) and initial balance.
- View current balance for each account on the dashboard.

## 11. Balance Automation & Integrity
**As a user**, I want my account balances to update automatically when I record transactions, **so that** my records always reflect reality without manual math.

**Acceptance Criteria:**
- Marking a receivable as "Paid" increases the selected account's balance.
- Marking a payable as "Paid" decreases the selected account's balance.
- **System ensures all balance changes are recorded as visible "Movements" in the ledger.**

## 12. Internal Transfers
**As a user**, I want to record transfers between my accounts, **so that** I can track money movement without affecting my profit/loss.

**Acceptance Criteria:**
- Record a transfer from Account A to Account B.
- Specify Date, Value, and Notes.
- Balance decreases in Account A and increases in Account B.
- Transfers do not count as Income or Expense in reports.

## 13. Balance History
**As a user**, I want to see the history of balance changes for an account, **so that** I can audit my funds.

**Acceptance Criteria:**
- View a list of all movements (Credits, Debits, Transfers) for a specific account.
- Filter history by date range.

## 14. Manage Credit Cards
**As a user**, I want to register my credit cards, **so that** I can control expenses made with them.

**Acceptance Criteria:**
- Create, edit, and delete (soft-delete/archive) credit cards.
- Required fields: Name, Limit, Closing Day, Due Day.
- View current used limit and available limit.
- **User understands that expenses made after the 'Closing Day' will automatically appear in the next month's invoice.**

## 15. Register Credit Card Expense
**As a user**, I want to record a purchase made with a credit card, **so that** it appears on the correct monthly invoice.

**Acceptance Criteria:**
- Select "Credit Card" as payment method when registering an expense.
- Select which card was used.
- Enter purchase date (system auto-assigns to correct invoice month).
- Support for installments (e.g., divide value by N months).

## 16. Pay Credit Card Invoice
**As a user**, I want to pay my credit card invoice, **so that** I can settle my debt and free up my limit.

**Acceptance Criteria:**
- View list of monthly invoices (Open, Closed, Paid).
- "Pay Invoice" action creates a standard Payable (Expense) linked to a bank/cash account.
- Paying the invoice resets the used limit for that period.

## 17. User Registration & Setup
**As a new user**, I want to register in the system, **so that** I can create a personal workspace and access the application.

**Acceptance Criteria:**
- User provides registration details (Name, Email, Password).
- System creates a new User entity and a Workspace.
- System sends a validation link to the registered email address.
- User must validate email to activate the account.

## 18. Forgot Password
**As a user**, I want to request a password redefinition, **so that** I can regain access if I forget my credentials.

**Acceptance Criteria:**
- "Forgot Password" link available on the login screen.
- User enters registered email to request reset.
- System sends a password redefinition link to the email.

## 19. Category Management
**As a user**, I want to organize my finances into Categories, **so that** I can analyze where my money comes from and goes.

**Acceptance Criteria:**
- Create, edit, and delete (soft-delete) categories.
- Organize categories hierarchically (Parent -> Child).
- Define constraints: Income (Receivable) or Expense (Payable).
- Select a category when creating any Receivable or Payable.

## 20. Transaction Reversal to Correct Mistakes
**As a user**, I want to undo a mistake, **so that** my balance is corrected without deleting the history of the error.

**Acceptance Criteria:**
- User uses a "Reverse" action on a completed transaction.
- **System creates a new "Compensating Transaction" (Reversal) in the ledger.**
- The original transaction remains in history but is linked to its reversal.
- The status of the original Payable/Receivable entity is reverted (e.g., back to "Pending").
