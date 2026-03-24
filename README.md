# Finance Control System MVP

The purpose of this project is to create an MVP of a finance control system that can be used to manage the financial transactions of a company or a person.

The system will be a web application built with the following technologies:
- **Backend:** Spring Boot for RESTful API
- **Database:** PostgreSQL
- **Frontend:** Angular for the web application. It should easily responsive to mobile devices.
- **Authentication:** JWT
- **Identity of tables:** UUID v7 as primary key
- **Auditing:** `org.hibernate.envers.Audited`

## Features

### 1. Multi-Tenancy & Auditing
- **Workspace Management**: Support for multiple isolated environments for each user.
- **Audit Trails**: Automatic tracking of `created_by` and `updated_by` user and timestamps for every single record (Users, Accounts, Transactions, etc.).
- **User Management**: Secure email/password authentication with full name profiles.

### 2. Core Financial Structure
- **Multi-Account Types**: Manage various money repositories:
  - **Cash**: Physical money on hand.
  - **Bank**: Checking/Savings accounts.
  - **Wallet**: Digital wallets (PayPal, Wise, etc.).
- **Account Balance Caching**: Optimized balance retrieval via stored fields (updated by system logic).
- **Hierarchical Categories**: Categorize transactions with 2 levels of depth (Parent -> Child).
  - **Usage Constraints**: Categories strictly defined as for `RECEIVABLE` (Income) or `PAYABLE` (Expense).
- **Unified Counterparties**: Centralized directory for:
  - **Customers**: Entities you receive money from.
  - **Suppliers**: Entities you pay.
  - **Both**: Entities acting in both roles.

### 3. Income Management (Receivables)
- **Income Grouping**: Organize income streams (e.g., "Web Project A", "Consulting Contract").
- **Installment Tracking**: Split large receivables into multiple records linked via `receivable_groups`.
- **Granular Statuses**: Track lifecycle via `PENDING`, `PARTIALLY_RECEIVED`, `RECEIVED`, or `OVERDUE`.
- **Payment Methods**: Support for Cash, Bank Transfer, Credit Card, Pix, and Boleto.
- **Rich Attachments**: Attach proofs via:
  - **Files**: Physical file uploads.
  - **Links**: External URLs (Drive, Dropbox).
  - **Types**: Invoices, Receipts, or General proofs.

### 4. Expense Management (Payables)
- **Mandatory Grouping**: All expenses belong to a `payable_group`, enabling powerful tracking of:
  - **Recurring Bills**: Internet, Rent, Subscriptions.
  - **Installments**: "Laptop purchase (Spread over 12 months)".
- **Expense Lifecycle**: Manage `PENDING`, `PARTIALLY_PAID`, `PAID`, and `OVERDUE` states.
- **Payment Methods**: Cash, Transfer, Credit Card, Pix, Boleto.
- **Rich Attachments**: Link Bills, Invoices, and Receipts (Files/Links) to specific expenses.

### 5. Credit Card Control
- **Card Configuration**: Set rigid Limits, Closing Days, and Due Days for multiple cards.
- **Invoice Lifecycle**:
  - **Automatic Buckets**: Expenses automatically fall into Monthly Invoices based on dates.
  - **Invoice Status**: Track `OPEN` (current), `CLOSED` (awaiting payment), and `PAID`.
- **Expense Linking**: Direct link between a Payable (Expense) and a specific Credit Card Invoice.
- **Invoice Payment**: Paying a credit card invoice is treated as a Payable event, updating the ledger and closing the invoice.
- **Invoice Attachments**: Attach generic receipts/statements to the monthly invoice itself.

### 6. Ledger & Cash Flow
- **Unified Ledger (Account Movements)**: A single, immutable timeline of *all* financial events.
  - **Polymorphic Associations**: Every movement links back to its source: Receivable, Payable, Transfer, or Credit Card Invoice.
- **Transfers**: Record neutral movements (Concept of "moving money", not spending it) between accounts.
- **Cash vs. Competence**:
  - **Competence**: Via Issue/Due dates in Receivables/Payables.
  - **Cash**: Via actual date in Account Movements.

### 7. Reporting & Logic
- **Sequence Counters**: Automatic sequencing for items within groups (e.g., "Installment 1/12", "Installment 2/12").
- **Active Flags**: Soft-delete capability for Accounts, Categories, and Counterparties (`active = boolean`).

### 8. Dashboard & Home View
- **Accounts Overview**: Real-time summary cards showing current balance for each account.
- **Consolidated Totals**: Application-wide "Net Worth" or "Total Available" display.
- **Quick Actions**: "Add Transaction" buttons directly from the dashboard.
- **Pending Alerts**: Widget showing overdue bills, open credit card invoices, or budget warnings.

---

## Documentation Links

- [User Stories](user-stories.md)
- [Database Schema](database-schema.md)
- [Reversal Logic](reversal-logic.md)
- [API Endpoints](api-endpoints.md)
- [Dashboard Definition](dashboard-definition.md)
- [Cash Flow Definition](cash-flow-definition.md)
- [Frontend Routes](frontend-routes.md)
