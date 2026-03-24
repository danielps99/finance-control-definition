# Frontend Routes

## Authentication & User Management
- `/login` (POST)
- `/register` (POST)
- `/forgot-password` (POST)
- `/reset-password` (POST)
- `/profile` (GET)
- `/profile/edit` (PUT)

## Dashboard & Home
- `/dashboard` (GET)

## Core Financial Structure
- `/accounts` (GET)
- `/accounts/create` (POST)
- `/accounts/:id` (GET)
- `/accounts/:id/edit` (PUT)
- `/categories` (GET)
- `/categories/create` (POST)
- `/categories/:id` (GET)
- `/categories/:id/edit` (PUT)
- `/counterparties` (GET)
- `/counterparties/create` (POST)
- `/counterparties/:id` (GET)
- `/counterparties/:id/edit` (PUT)

## Income Management (Receivables)
- `/receivables` (GET)
- `/receivables/create` (POST)
- `/receivables/:id` (GET)
- `/receivables/:id/edit` (PUT)

## Expense Management (Payables)
- `/payables` (GET)
- `/payables/create` (POST)
- `/payables/:id` (GET)
- `/payables/:id/edit` (PUT)

## Credit Card Control
- `/credit-cards` (GET)
- `/credit-cards/create` (POST)
- `/credit-cards/:id` (GET)
- `/credit-cards/:id/edit` (PUT)
- `/credit-cards/:id/invoices` (GET)
- `/credit-cards/:id/invoices/:invoiceId` (GET)
- `/credit-cards/:id/invoices/:invoiceId/pay` (POST)

## Ledger & Cash Flow
- `/ledger` (GET)
- `/transfers/create` (POST)
- `/transfers/:id` (GET)
- `/transfers/:id/edit` (PUT)
- `/cash-flow` (GET)
