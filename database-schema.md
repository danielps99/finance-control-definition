# Database Schema

This document defines the table structure for the Finance Control system.

## Entities

### `users`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `email` | VARCHAR(255) | Unique, Not Null |
| `password_hash` | VARCHAR(255) | Encrypted password |
| `full_name` | VARCHAR(100) | |
| `is_verified` | BOOLEAN | Email verification status |
| `email_verification_token` | VARCHAR(255) | |
| `password_reset_token` | VARCHAR(255) | |
| `password_reset_expires_at`| TIMESTAMP | |
| `active` | BOOLEAN | Soft-delete flag |
| `version` | LONG | Optimistic locking |

### `workspaces`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `name` | VARCHAR(100) | Workspace name |
| `active` | BOOLEAN | Soft-delete flag |
| `version` | LONG | Optimistic locking |

### `accounts`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `name` | VARCHAR(100) | Account name |
| `should_show_in_dashboard` | BOOLEAN | Display preference flag |
| `type` | ENUM | `'CASH'`, `'BANK'`, `'WALLET'` |
| `balance` | DECIMAL(15,2) | Cached balance value |
| `active` | BOOLEAN | Soft-delete flag |
| `version` | LONG | Optimistic locking |

### `balances_history`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `account_id` | FK -> `accounts.id` | Associated Account (`ON DELETE RESTRICT`) |
| `date` | DATE | Snapshot date |
| `balance` | DECIMAL(15,2) | Balance at 23:59:59 of that date |

### `categories`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `parent_id` | FK -> `categories.id` | Subcategory parent (`ON DELETE RESTRICT`) |
| `name` | VARCHAR(100) | Category name |
| `description` | VARCHAR(255) | Description |
| `useable_in` | ENUM | `'RECEIVABLE'`, `'PAYABLE'` |
| `active` | BOOLEAN | Soft-delete flag |
| `version` | LONG | Optimistic locking |

### `counterparties`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `name` | VARCHAR(100) | Counterparty name |
| `email` | VARCHAR(255) | Contact email |
| `phone` | VARCHAR(20) | Contact phone |
| `type` | ENUM | `'CUSTOMER'`, `'SUPPLIER'`, `'BOTH'` |
| `active` | BOOLEAN | Soft-delete flag |
| `version` | LONG | Optimistic locking |

### `receivable_groups`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `categories_id` | FK -> `categories.id` | Associated Category (`ON DELETE RESTRICT`) |
| `default_account_id` | FK -> `accounts.id` | Default Account (`ON DELETE RESTRICT`) |
| `name` | VARCHAR(100) | Group name |
| `description` | VARCHAR(255) | Description |
| `sequence_counter` | INT | Number of installments |
| `original_total_amount` | DECIMAL(15,2) | Sum of base installments |
| `total_amount` | DECIMAL(15,2) | Sum with fines/interest |
| `received_amount` | DECIMAL(15,2) | Total received amount |
| `version` | LONG | Optimistic locking |

### `receivables` (Income)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `counterparty_id` | FK -> `counterparties.id` | Associated Counterparty (`ON DELETE RESTRICT`) |
| `receivable_group_id` | FK -> `receivable_groups.id` | Associated Group (`ON DELETE RESTRICT`) |
| `sequence` | INT | Installment index (e.g. 1, 2) |
| `original_amount` | DECIMAL(15,2) | Base amount |
| `fine_amount` | DECIMAL(15,2) | Fine amount |
| `interest_amount` | DECIMAL(15,2) | Interest amount |
| `discount_amount` | DECIMAL(15,2) | Discount amount |
| `total_amount` | DECIMAL(15,2) | Calculated total amount |
| `received_amount` | DECIMAL(15,2) | Amount received so far |
| `issue_date` | DATE | Issue date |
| `due_date` | DATE | Due date |
| `status` | ENUM | `'PENDING'`, `'PARTIALLY_RECEIVED'`, `'RECEIVED'`, `'OVERDUE'` |
| `payment_method` | ENUM | `'CASH'`, `'BANK_TRANSFER'`, `'CREDIT_CARD'`, `'PIX'`, `'BOLETO'` |
| `account_id` | FK -> `accounts.id` | Destination Account (`ON DELETE RESTRICT`) |
| `version` | LONG | Optimistic locking |

### `payable_groups`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `categories_id` | FK -> `categories.id` | Associated Category (`ON DELETE RESTRICT`) |
| `default_account_id` | FK -> `accounts.id` | Default Account (`ON DELETE RESTRICT`) |
| `name` | VARCHAR(100) | Group name |
| `description` | VARCHAR(255) | Description |
| `sequence_counter` | INT | Number of installments |
| `original_total_amount` | DECIMAL(15,2) | Sum of base installments |
| `total_amount` | DECIMAL(15,2) | Sum with fines/interest |
| `paid_amount` | DECIMAL(15,2) | Total paid amount |
| `version` | LONG | Optimistic locking |

### `payables` (Expenses)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `counterparty_id` | FK -> `counterparties.id` | Associated Counterparty (`ON DELETE RESTRICT`) |
| `payable_group_id` | FK -> `payable_groups.id` | Associated Group (`ON DELETE RESTRICT`) |
| `sequence` | INT | Installment index |
| `original_amount` | DECIMAL(15,2) | Base amount |
| `fine_amount` | DECIMAL(15,2) | Fine amount |
| `interest_amount` | DECIMAL(15,2) | Interest amount |
| `discount_amount` | DECIMAL(15,2) | Discount amount |
| `total_amount` | DECIMAL(15,2) | Calculated total amount |
| `paid_amount` | DECIMAL(15,2) | Amount paid so far |
| `issue_date` | DATE | Issue date |
| `due_date` | DATE | Due date |
| `status` | ENUM | `'PENDING'`, `'PARTIALLY_PAID'`, `'PAID'`, `'OVERDUE'` |
| `payment_method` | ENUM | `'CASH'`, `'BANK_TRANSFER'`, `'CREDIT_CARD'`, `'PIX'`, `'BOLETO'` |
| `account_id` | FK -> `accounts.id` | Payment Account (`ON DELETE RESTRICT`) |
| `version` | LONG | Optimistic locking |

### `credit_cards`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `name` | VARCHAR(100) | Credit card label |
| `limit_amount` | DECIMAL(15,2) | Total credit limit |
| `limit_available` | DECIMAL(15,2) | Available limit |
| `closing_day` | INT | Statement cutoff day (1-31) |
| `due_day` | INT | Payment due day (1-31) |
| `active` | BOOLEAN | Soft-delete flag |
| `should_show_in_dashboard` | BOOLEAN | Display preference flag |
| `version` | LONG | Optimistic locking |

### `credit_card_invoices`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `credit_card_id` | FK -> `credit_cards.id` | Associated Credit Card (`ON DELETE RESTRICT`) |
| `reference_month` | DATE | First day of reference month |
| `due_date` | DATE | Statement due date |
| `status` | ENUM | `'OPEN'`, `'CLOSED'`, `'PAID'` |
| `total_amount` | DECIMAL(15,2) | Sum of payables linked |
| `paid_amount` | DECIMAL(15,2) | Amount paid on invoice |
| `version` | LONG | Optimistic locking |

### `account_movements` (Ledger)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `reversed_movement_id` | FK -> `account_movements.id` | Optional reference to original movement (`ON DELETE RESTRICT`) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `account_id` | FK -> `accounts.id` | Associated Account (`ON DELETE RESTRICT`) |
| `type` | ENUM | `'CREDIT'`, `'DEBIT'` |
| `amount` | DECIMAL(15,2) | Movement amount |
| `date` | DATE | Movement date (Cash date) |
| `description` | VARCHAR(255) | Description / memo |
| `receivable_id` | FK -> `receivables.id` | Polymorphic FK (`ON DELETE RESTRICT`) |
| `payable_id` | FK -> `payables.id` | Polymorphic FK (`ON DELETE RESTRICT`) |
| `credit_card_invoice_id` | FK -> `credit_card_invoices.id` | Polymorphic FK (`ON DELETE RESTRICT`) |
| `transfer_id` | FK -> `transfers.id` | Polymorphic FK (`ON DELETE RESTRICT`) |

#### Polymorphic Foreign Keys Enforcement & ORM Strategy
In `account_movements`, exactly one of the four polymorphic foreign keys (`receivable_id`, `payable_id`, `credit_card_invoice_id`, `transfer_id`) must be non-null for any given entry.
- **PostgreSQL Database Constraint**:
  ```sql
  ALTER TABLE account_movements ADD CONSTRAINT chk_account_movements_polymorphic_fk
  CHECK (num_nonnulls(receivable_id, payable_id, credit_card_invoice_id, transfer_id) = 1);
  ```
- **ORM / Application Level**: Entities map these as standard `@ManyToOne(fetch = FetchType.LAZY)` optional associations, with service-layer validation ensuring only one link is set upon creation.

### `transfers`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace (`ON DELETE RESTRICT`) |
| `source_account_id` | FK -> `accounts.id` | Source Account (`ON DELETE RESTRICT`) |
| `destination_account_id`| FK -> `accounts.id` | Destination Account (`ON DELETE RESTRICT`) |
| `amount` | DECIMAL(15,2) | Transfer amount |
| `date` | DATE | Transfer execution date |
| `annotation` | VARCHAR(255) | Transfer description/memo |
| `status` | ENUM | `'COMPLETED'`, `'REVERSED'` |
| `version` | LONG | Optimistic locking |

---

## Auditing Strategy (Hibernate Envers)

1. **Main Table Philosophy**:
    - **ID Strategy**: UUID v7 (includes creation timestamp).
    - **Concurrency**: `@Version` column (`version` LONG) for optimistic locking.
2. **History/Audit Tables**:
    - Managed by Envers in `_AUD` tables.
    - **REVINFO**: Custom revision entity storing both `user_id` (from JWT `sub`) and `workspace_id` (from `WorkspaceContext`) populated via `UserRevisionListener` for fast tenant audit log filtering.
    - **[Table]_AUD**: History tables matching main tables plus `REV` and `REVTYPE` columns.
