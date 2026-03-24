# Database Schema

This document defines the table structure for the Finance Control system.

## Entities

### `users`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | Primary Key (UUID v7) |
| `workspace_id` | FK -> `workspaces.id` | Associated Workspace |
| `email` | VARCHAR(255) | Unique, Not Null |
| `password_hash` | VARCHAR(255) | Encrypted password |
| `full_name` | VARCHAR(100) | |
| `is_verified` | BOOLEAN | Email verification status |
| `email_verification_token` | VARCHAR(255) | |
| `password_reset_token` | VARCHAR(255) | |
| `password_reset_expires_at`| TIMESTAMP | |
| `active` | BOOLEAN | |
| `version` | LONG | Optimistic locking |

### `workspaces`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `name` | VARCHAR(100) | |
| `active` | BOOLEAN | |
| `version` | LONG | |

### `accounts`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `name` | VARCHAR(100) | |
| `should_show_in_dashboard` | BOOLEAN | |
| `type` | ENUM | 'CASH', 'BANK', 'WALLET' |
| `balance` | DECIMAL(15,2) | Cached value |
| `active` | BOOLEAN | |
| `version` | LONG | |

### `balances_history`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `account_id` | FK -> `accounts.id` | |
| `date` | DATE | |
| `balance` | DECIMAL(15,2) | Balance at 23:59:59 of that date |

### `categories`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `parent_id` | FK -> `categories.id` | Subcategory parent |
| `name` | VARCHAR(100) | |
| `description` | VARCHAR(255) | |
| `useable_in` | ENUM | 'RECEIVABLE', 'PAYABLE' |
| `active` | BOOLEAN | |
| `version` | LONG | |

### `counterparties`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `name` | VARCHAR(100) | |
| `email` | VARCHAR(255) | |
| `phone` | VARCHAR(20) | |
| `type` | ENUM | 'CUSTOMER', 'SUPPLIER', 'BOTH' |
| `active` | BOOLEAN | |
| `version` | LONG | |

### `receivable_groups`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `categories_id` | FK -> `categories.id` | |
| `default_account_id` | FK -> `accounts.id` | |
| `name` | VARCHAR(100) | |
| `description` | VARCHAR(255) | |
| `sequence_counter` | INT | Number of installments |
| `original_total_amount` | DECIMAL(15,2) | Sum of base installments |
| `total_amount` | DECIMAL(15,2) | Sum with fines/interest |
| `received_amount` | DECIMAL(15,2) | |
| `version` | LONG | |

### `receivables` (Income)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `counterparty_id` | FK -> `counterparties.id` | |
| `receivable_group_id` | FK -> `receivable_groups.id` | |
| `sequence` | INT | Installment index |
| `original_amount` | DECIMAL(15,2) | |
| `fine_amount` | DECIMAL(15,2) | |
| `interest_amount` | DECIMAL(15,2) | |
| `discount_amount` | DECIMAL(15,2) | |
| `total_amount` | DECIMAL(15,2) | Calculated |
| `received_amount` | DECIMAL(15,2) | |
| `issue_date` | DATE | |
| `due_date` | DATE | |
| `status` | ENUM | 'PENDING', 'PARTIALLY_RECEIVED', 'RECEIVED', 'OVERDUE' |
| `payment_method` | ENUM | 'CASH', 'BANK_TRANSFER', 'PIX', etc. |
| `account_id` | FK -> `accounts.id` | |
| `version` | LONG | |

### `payable_groups`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `categories_id` | FK -> `categories.id` | |
| `default_account_id` | FK -> `accounts.id` | |
| `name` | VARCHAR(100) | |
| `description` | VARCHAR(255) | |
| `sequence_counter` | INT | |
| `original_total_amount` | DECIMAL(15,2) | |
| `total_amount` | DECIMAL(15,2) | |
| `paid_amount` | DECIMAL(15,2) | |
| `version` | LONG | |

### `payables` (Expenses)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `counterparty_id` | FK -> `counterparties.id` | |
| `payable_group_id` | FK -> `payable_groups.id` | |
| `sequence` | INT | |
| `original_amount` | DECIMAL(15,2) | |
| `fine_amount` | DECIMAL(15,2) | |
| `interest_amount` | DECIMAL(15,2) | |
| `discount_amount` | DECIMAL(15,2) | |
| `total_amount` | DECIMAL(15,2) | |
| `paid_amount` | DECIMAL(15,2) | |
| `issue_date` | DATE | |
| `due_date` | DATE | |
| `status` | ENUM | 'PENDING', 'PARTIALLY_PAID', 'PAID', 'OVERDUE' |
| `payment_method` | ENUM | |
| `account_id` | FK -> `accounts.id` | |
| `version` | LONG | |

### `credit_cards`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `name` | VARCHAR(100) | |
| `limit_amount` | DECIMAL(15,2) | |
| `limit_available` | DECIMAL(15,2) | |
| `closing_day` | INT | 1-31 |
| `due_day` | INT | 1-31 |
| `active` | BOOLEAN | |
| `should_show_in_dashboard` | BOOLEAN | |
| `version` | LONG | |

### `credit_card_invoices`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `credit_card_id` | FK -> `credit_cards.id` | |
| `reference_month` | DATE | First day of month |
| `due_date` | DATE | |
| `status` | ENUM | 'OPEN', 'CLOSED', 'PAID' |
| `total_amount` | DECIMAL(15,2) | Sum of payables |
| `paid_amount` | DECIMAL(15,2) | Sum of paid/partially paid |
| `version` | LONG | |

### `account_movements` (Ledger)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `reversed_movement_id` | FK -> `account_movements.id` | |
| `workspace_id` | FK -> `workspaces.id` | |
| `account_id` | FK -> `accounts.id` | |
| `type` | ENUM | 'CREDIT', 'DEBIT' |
| `amount` | DECIMAL(15,2) | |
| `date` | DATE | |
| `description` | VARCHAR(255) | |
| `receivable_id` | FK -> `receivables.id` | Polymorphic |
| `payable_id` | FK -> `payables.id` | Polymorphic |
| `credit_card_invoice_id` | FK -> `credit_card_invoices.id` | Polymorphic |
| `transfer_id` | FK -> `transfers.id` | Polymorphic |

### `transfers`
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | PK UUID | |
| `workspace_id` | FK -> `workspaces.id` | |
| `annotation` | VARCHAR(255) | |
| `source_account_id` | FK -> `accounts.id` | |
| `destination_account_id`| FK -> `accounts.id` | |
| `amount` | DECIMAL(15,2) | |

---

## Auditing Strategy (Hibernate Envers)

1. **Main Table Philosophy**:
    - **ID Strategy**: UUID v7 (includes creation timestamp).
    - **Concurrency**: `@Version` column (`version` LONG) for optimistic locking.
2. **History/Audit Tables**:
    - Managed by Envers in `_AUD` tables.
    - **REVINFO**: Global revision info with `user_id`.
    - **[Table]_AUD**: History tables matching main tables plus `REV` and `REVTYPE` columns.
