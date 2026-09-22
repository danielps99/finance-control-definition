# RESTful API Endpoints

**Base URL:** `/api/v1`  
**Authentication:** Bearer Token (JWT) — All business endpoints are secured with `@Authenticated`. Endpoint access does not differentiate permissions by roles.  
**Context:** Tenant isolation is enforced automatically via the `workspace_id` claim in the authenticated JWT token.

---

## 1. Authentication & Users (`/auth`, `/users`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| POST | `/auth/register` | Register new user and create workspace | `@PermitAll` |
| POST | `/auth/login` | Login and retrieve JWT access & refresh tokens | `@PermitAll` |
| POST | `/auth/refresh` | Refresh expired access token | `@PermitAll` *(Requires valid Refresh Token in body/cookie)* |
| POST | `/auth/logout` | Logout & revoke refresh token | `@Authenticated` |
| POST | `/auth/forgot-password` | Request password reset | `@PermitAll` |
| POST | `/auth/reset-password` | Perform password reset | `@PermitAll` |
| GET | `/users/validate-email` | Validate email verification token | `@PermitAll` |
| GET | `/users/me` | Get current user profile | `@Authenticated` |
| PUT | `/users/me` | Update profile | `@Authenticated` |

### Sample Request/Response

**POST `/auth/login` Request:**
```json
{
  "email": "user@example.com",
  "password": "SecretPassword123!"
}
```

**POST `/auth/login` Response (200 OK):**
```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 86400,
  "workspaceId": "01912a3b-4c5d-7e8f-9a0b-1c2d3e4f5a6b",
  "user": {
    "id": "01912a3b-1111-7e8f-9a0b-1c2d3e4f5a6b",
    "email": "user@example.com",
    "fullName": "Maria Silva"
  }
}
```

---

## 2. Dashboard (`/dashboard`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| GET | `/dashboard` | Dashboard summary | `@Authenticated` |
| GET | `/dashboard/alerts` | List of pending alerts | `@Authenticated` |

### Query Parameters

- **GET `/dashboard`**:
  - `referenceDate` (*optional*, `DATE` `YYYY-MM-DD`, default: current date): Date anchor for calculating account balances and overdue counters.

### Sample Response

**GET `/dashboard?referenceDate=2026-08-03` Response (200 OK):**
```json
{
  "referenceDate": "2026-08-03",
  "netWorth": 15450.00,
  "overdueReceivables": 1200.00,
  "overduePayables": 450.00,
  "availableCreditLimit": 5000.00,
  "accounts": [
    {
      "id": "01912a3b-2222-7e8f-9a0b-1c2d3e4f5a6b",
      "name": "Checking Account - Bank A",
      "type": "BANK",
      "balance": 10450.00,
      "shouldShowInDashboard": true
    }
  ]
}
```

---

## 3. Financial Accounts (`/accounts`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| GET | `/accounts` | List all accounts | `@Authenticated` |
| POST | `/accounts` | Create new account | `@Authenticated` |
| GET | `/accounts/{id}` | Get account details | `@Authenticated` |
| PUT | `/accounts/{id}` | Update account | `@Authenticated` |
| DELETE | `/accounts/{id}` | Soft delete account | `@Authenticated` |

### Sample Request/Response

**POST `/accounts` Request:**
```json
{
  "name": "Main Bank Account",
  "type": "BANK",
  "initialBalance": 5000.00,
  "shouldShowInDashboard": true
}
```

---

## 4. Categories (`/categories`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| GET | `/categories` | List categories (Tree structure) | `@Authenticated` |
| POST | `/categories` | Create category | `@Authenticated` |
| PUT | `/categories/{id}` | Update category | `@Authenticated` |
| DELETE | `/categories/{id}` | Soft delete category | `@Authenticated` |

### Query Parameters

- **GET `/categories`**:
  - `useableIn` (*optional*, `ENUM`: `'RECEIVABLE'`, `'PAYABLE'`): Filter categories by usage type.

---

## 5. Counterparties (`/customers`, `/suppliers`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| GET | `/customers` | List customers | `@Authenticated` |
| POST | `/customers` | Create customer | `@Authenticated` |
| PUT | `/customers/{id}` | Update customer | `@Authenticated` |
| DELETE | `/customers/{id}` | Soft delete customer | `@Authenticated` |
| GET | `/suppliers` | List suppliers | `@Authenticated` |
| POST | `/suppliers` | Create supplier | `@Authenticated` |
| PUT | `/suppliers/{id}` | Update supplier | `@Authenticated` |
| DELETE | `/suppliers/{id}` | Soft delete supplier | `@Authenticated` |

---

## 6. Receivables (Income) (`/receivables`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| POST | `/receivables/list` | List receivables | `@Authenticated` |
| POST | `/receivables` | Create new receivable | `@Authenticated` |
| GET | `/receivables/{id}` | Get single receivable | `@Authenticated` |
| PUT | `/receivables/{id}` | Update receivable | `@Authenticated` |
| DELETE | `/receivables/{id}` | Delete (if pending) | `@Authenticated` |
| POST | `/receivables/{id}/receive` | Record receipt | `@Authenticated` |
| POST | `/receivables/{id}/reverse` | Reverse transaction | `@Authenticated` |
| POST | `/receivables/{id}/attachments`| Upload proof | `@Authenticated` |
| DELETE | `/receivables/{id}/attachments/{attachmentId}`| Delete attachment | `@Authenticated` |

### Request Body

- **POST `/receivables/list` Request Body**:
  - `startDate` (*optional*, `YYYY-MM-DD`): Filter due dates from this date.
  - `endDate` (*optional*, `YYYY-MM-DD`): Filter due dates up to this date.
  - `status` (*optional*, `ENUM`: `'PENDING'`, `'PARTIALLY_RECEIVED'`, `'RECEIVED'`, `'OVERDUE'`).
  - `counterpartyId` (*optional*, `UUID`).
  - `categoryIds` (*optional*, `List<UUID>`): Filter by leaf category IDs.

**Sample Request Body:**
```json
{
  "startDate": "2026-08-01",
  "endDate": "2026-08-31",
  "status": "PENDING",
  "counterpartyId": "01912a3b-3333-7e8f-9a0b-1c2d3e4f5a6b",
  "categoryIds": [
    "01912a3b-4444-7e8f-9a0b-1c2d3e4f5a6b"
  ]
}
```

### Sample Request/Response

**POST `/receivables/{id}/receive` Request:**
```json
{
  "accountId": "01912a3b-2222-7e8f-9a0b-1c2d3e4f5a6b",
  "amount": 500.00,
  "paymentMethod": "PIX",
  "paymentDate": "2026-08-03",
  "fineAmount": 0.00,
  "interestAmount": 0.00,
  "discountAmount": 0.00
}
```

---

## 7. Payables (Expenses) (`/payables`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| POST | `/payables/list` | List payables | `@Authenticated` |
| POST | `/payables` | Create new payable | `@Authenticated` |
| GET | `/payables/{id}` | Get single payable | `@Authenticated` |
| PUT | `/payables/{id}` | Update payable | `@Authenticated` |
| DELETE | `/payables/{id}` | Delete (if pending) | `@Authenticated` |
| POST | `/payables/{id}/pay` | Record payment | `@Authenticated` |
| POST | `/payables/{id}/reverse` | Reverse transaction | `@Authenticated` |
| POST | `/payables/{id}/attachments`| Upload proof | `@Authenticated` |
| DELETE | `/payables/{id}/attachments/{attachmentId}`| Delete attachment | `@Authenticated` |

### Request Body

- **POST `/payables/list` Request Body**:
  - `startDate` (*optional*, `YYYY-MM-DD`).
  - `endDate` (*optional*, `YYYY-MM-DD`).
  - `status` (*optional*, `ENUM`: `'PENDING'`, `'PARTIALLY_PAID'`, `'PAID'`, `'OVERDUE'`).
  - `counterpartyId` (*optional*, `UUID`).
  - `categoryIds` (*optional*, `List<UUID>`): Filter by leaf category IDs.

**Sample Request Body:**
```json
{
  "startDate": "2026-08-01",
  "endDate": "2026-08-31",
  "status": "PENDING",
  "counterpartyId": "01912a3b-3333-7e8f-9a0b-1c2d3e4f5a6b",
  "categoryIds": [
    "01912a3b-4444-7e8f-9a0b-1c2d3e4f5a6b"
  ]
}
```

---

## 8. Credit Cards (`/credit-cards`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| GET | `/credit-cards` | List credit cards | `@Authenticated` |
| POST | `/credit-cards` | Add new card | `@Authenticated` |
| PUT | `/credit-cards/{id}` | Update card | `@Authenticated` |
| DELETE | `/credit-cards/{id}` | Soft delete card | `@Authenticated` |
| GET | `/credit-cards/{id}/invoices` | List invoices | `@Authenticated` |
| GET | `/credit-cards/{id}/invoices/current` | Current open invoice | `@Authenticated` |
| GET | `/credit-cards/invoices/{invoiceId}` | Invoice details | `@Authenticated` |
| POST | `/payables/credit-card-invoice-payment`| Pay an invoice | `@Authenticated` |

---

## 9. Internal Transfers (`/transfers`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| GET | `/transfers` | List transfers history | `@Authenticated` |
| POST | `/transfers` | Create transfer | `@Authenticated` |
| POST | `/transfers/{id}/reverse` | Reverse transfer | `@Authenticated` |

### Request Body

- **POST `/transfers` Request Body**:
  - `sourceAccountId` (*required*, `UUID`): Account to transfer funds from.
  - `destinationAccountId` (*required*, `UUID`): Account to receive funds.
  - `amount` (*required*, `DECIMAL(15,2)`): Transfer amount (must be > 0).
  - `date` (*required*, `YYYY-MM-DD`): Execution date of the transfer.
  - `annotation` (*optional*, `VARCHAR(255)`): Description or memo.

**Sample Request Body:**
```json
{
  "sourceAccountId": "01912a3b-2222-7e8f-9a0b-1c2d3e4f5a6b",
  "destinationAccountId": "01912a3b-3333-7e8f-9a0b-1c2d3e4f5a6b",
  "amount": 1000.00,
  "date": "2026-08-09",
  "annotation": "Monthly savings allocation"
}
```

---

## 10. Reports (`/reports`)

| Method | Endpoint | Description | Security |
| :--- | :--- | :--- | :--- |
| POST | `/reports/cash-flow` | Detailed cash flow report | `@Authenticated` |
| POST | `/reports/receivables/export` | CSV/Excel export | `@Authenticated` |
| POST | `/reports/payables/export` | CSV/Excel export | `@Authenticated` |
| POST | `/reports/moviments/export` | CSV/Excel export | `@Authenticated` |

### Request Body / Query Parameters

- **POST `/reports/cash-flow` Request Body**:
  - `startDate` (*required*, `YYYY-MM-DD`): Report range start date.
  - `endDate` (*required*, `YYYY-MM-DD`): Report range end date.
  - `viewType` (*required*, `ENUM`: `'DAILY'`, `'MONTHLY'`).
  - `mode` (*required*, `ENUM`: `'SYNTHETIC'`, `'ANALYTIC'`).
  - `openingBalance` (*optional*, `ENUM`: `'ZERO'`, `'PREVIOUS_DAY'`, default: `'PREVIOUS_DAY'`).
  - `categoryIds` (*optional*, `List<UUID>`): Filter by leaf category IDs.

- **POST `/reports/receivables/export` & `/reports/payables/export` & `/reports/moviments/export` Request Body**:
  - `startDate` (*optional*, `YYYY-MM-DD`).
  - `endDate` (*optional*, `YYYY-MM-DD`).
  - `status` (*optional*, `ENUM`).
  - `counterpartyId` (*optional*, `UUID`).
  - `categoryIds` (*optional*, `List<UUID>`): Filter export by leaf category IDs.

**Sample Request Body:**
```json
{
  "startDate": "2026-08-01",
  "endDate": "2026-08-31",
  "viewType": "DAILY",
  "mode": "SYNTHETIC",
  "openingBalance": "PREVIOUS_DAY",
  "categoryIds": [
    "01912a3b-4444-7e8f-9a0b-1c2d3e4f5a6b"
  ]
}
```

**Sample Response (200 OK - SYNTHETIC Mode):**
```json
[
  {
    "date": "2026-08-01",
    "openingBalance": 10000.00,
    "inflows": {
      "realized": 1500.00,
      "projected": 500.00,
      "total": 2000.00
    },
    "outflows": {
      "realized": 300.00,
      "projected": 200.00,
      "total": 500.00
    },
    "netChange": 1500.00,
    "otherMovementsNet": 0.00,
    "closingBalance": 11500.00
  }
]
```

**Sample Response (200 OK - ANALYTIC Mode):**
```json
[
  {
    "date": "2026-08-01",
    "accounts": [
      {
        "accountId": "01912a3b-2222-7e8f-9a0b-1c2d3e4f5a6b",
        "accountName": "Checking Account - Bank A",
        "openingBalance": 10000.00,
        "inflows": {
          "realized": 1500.00,
          "projected": 500.00,
          "total": 2000.00
        },
        "outflows": {
          "realized": 300.00,
          "projected": 200.00,
          "total": 500.00
        },
        "netChange": 1500.00,
        "otherMovementsNet": 0.00,
        "closingBalance": 11500.00
      }
    ]
  }
]
```

---

## 11. Error Responses Blueprint (`ErrorResponseDTO`)

All error responses across all endpoints follow the uniform payload structure described in [REST API Error Handling Guidelines](error-handling-guidelines.md).

**Sample Error Response (422 Unprocessable Entity / Business Rule Violation):**
```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-08-03T14:30:00Z",
  "message": "BUSINESS_RULE_VIOLATION",
  "path": "/api/v1/payables/01912a3b-3333-7e8f-9a0b-1c2d3e4f5a6b/pay",
  "params": {
    "reason": "Payment date cannot be in the future"
  }
}
```


