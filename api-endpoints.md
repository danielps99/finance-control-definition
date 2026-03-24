# RESTful API Endpoints

**Base URL:** `/api/v1`  
**Authentication:** Bearer Token (JWT)  
**Context:** `X-Workspace-ID` header required for all business endpoints.

## 1. Authentication & Users
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| POST | `/auth/register` | Register new user and create workspace |
| POST | `/auth/login` | Login and retrieve JWT |
| POST | `/auth/refresh` | Refresh JWT |
| POST | `/auth/logout` | Logout |
| POST | `/auth/forgot-password` | Request password reset |
| POST | `/auth/reset-password` | Perform password reset |
| GET | `/users/validate-email` | Validate email |
| GET | `/users/me` | Get current user profile |
| PUT | `/users/me` | Update profile |

## 2. Dashboard
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/dashboard?referenceDate=...` | Dashboard summary |
| GET | `/dashboard/alerts` | List of pending alerts |

## 3. Financial Accounts
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/accounts` | List all accounts |
| POST | `/accounts` | Create new account |
| GET | `/accounts/{id}` | Get account details |
| PUT | `/accounts/{id}` | Update account |
| DELETE | `/accounts/{id}` | Soft delete account |

## 4. Categories
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/categories` | List categories (Tree structure) |
| POST | `/categories` | Create category |
| PUT | `/categories/{id}` | Update category |
| DELETE | `/categories/{id}` | Soft delete category |

## 5. Counterparties
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/customers` | List customers |
| POST | `/customers` | Create customer |
| PUT | `/customers/{id}` | Update customer |
| DELETE | `/customers/{id}` | Soft delete customer |
| GET | `/suppliers` | List suppliers |
| POST | `/suppliers` | Create supplier |
| PUT | `/suppliers/{id}` | Update supplier |
| DELETE | `/suppliers/{id}` | Soft delete supplier |

## 6. Receivables (Income)
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/receivables` | List receivables |
| POST | `/receivables` | Create new receivable |
| GET | `/receivables/{id}` | Get single receivable |
| PUT | `/receivables/{id}` | Update receivable |
| DELETE | `/receivables/{id}` | Delete (if pending) |
| POST | `/receivables/{id}/receive` | Record receipt |
| POST | `/receivables/{id}/reverse` | Reverse transaction |
| POST | `/receivables/{id}/attachments`| Upload proof |
| DELETE | `/receivables/.../attachments/{id}`| Delete attachment |

## 7. Payables (Expenses)
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/payables` | List payables |
| POST | `/payables` | Create new payable |
| GET | `/payables/{id}` | Get single payable |
| PUT | `/payables/{id}` | Update payable |
| DELETE | `/payables/{id}` | Delete (if pending) |
| POST | `/payables/{id}/pay` | Record payment |
| POST | `/payables/{id}/reverse` | Reverse transaction |
| POST | `/payables/{id}/attachments`| Upload proof |
| DELETE | `/payables/.../attachments/{id}`| Delete attachment |

## 8. Credit Cards
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/credit-cards` | List credit cards |
| POST | `/credit-cards` | Add new card |
| PUT | `/credit-cards/{id}` | Update card |
| DELETE | `/credit-cards/{id}` | Soft delete card |
| GET | `/credit-cards/{id}/invoices` | List invoices |
| GET | `/credit-cards/.../invoices/current` | Current open invoice |
| GET | `/credit-cards/invoices/{id}` | Invoice details |
| POST | `/payables/credit-card-invoice-payment`| Pay an invoice |

## 9. Internal Transfers
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/transfers` | List transfers history |
| POST | `/transfers` | Create transfer |
| POST | `/transfers/{id}/reverse` | Reverse transfer |

## 10. Reports
| Method | Endpoint | Description |
| :--- | :--- | :--- |
| GET | `/reports/cash-flow` | Detailed cash flow report |
| GET | `/reports/receivables/export` | CSV/Excel export |
| GET | `/reports/payables/export` | CSV/Excel export |
| GET | `/reports/moviments/export` | CSV/Excel export |
