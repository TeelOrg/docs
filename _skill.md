---
name: teel-api-docs
description: Generate Mintlify API documentation for the Teel platform — a cross-border payment and stablecoin transfer API
version: 1.0.0
---

# Teel API Documentation Skill

Generate comprehensive Mintlify-compatible API documentation for the Teel platform.

## Overview

Teel is a cross-border payment platform that supports fiat-to-fiat, fiat-to-stablecoin, and stablecoin-to-stablecoin transfers. The API is served from a Go backend (Gin framework) with Auth0 JWT authentication.

**Base URL:** `/api`

**Authentication:** Most endpoints require a valid Auth0 JWT Bearer token. Some endpoints also use custom headers (`X-User-ID`, `X-Customer-ID`, `X-Counterparty-ID`, `X-Control-Person-ID`, `X-Payment-Method-ID`) for resource scoping. Provider webhook endpoints are unauthenticated.

---

## API Groups

### Users

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/users/sync` | Sync user profile with backend after Auth0 login |
| GET | `/api/users/check-verification` | Check whether the user's email is verified |
| POST | `/api/users/link-accounts` | Link multiple authentication accounts for a user |
| POST | `/api/users` | Create a user |
| GET | `/api/users` | List users |
| GET | `/api/users/me` | Get current authenticated user profile |
| GET | `/api/users/me/settings` | Get current user settings |
| PUT | `/api/users/me/settings` | Update current user settings |
| GET | `/api/users/:id` | Get user by ID |
| PUT | `/api/users/:id` | Update user by ID |
| DELETE | `/api/users/:id` | Delete user by ID |
| GET | `/api/users/:id/business` | Get business associated with a user |
| POST | `/api/users/:id/business` | Associate a business with a user |

### Businesses

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/businesses` | Create a business |
| GET | `/api/businesses` | List businesses |
| GET | `/api/businesses/:id` | Get business by ID |
| PUT | `/api/businesses/:id` | Update business |
| DELETE | `/api/businesses/:id` | Delete business |
| GET | `/api/businesses/:id/users` | List users belonging to a business |

### Quotes

QuoteAudit middleware (JWT + user logging + audit trail).

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/quotes/onramp/all` | Get all available onramp quotes. Query: `source_currency`, `amount` |
| GET | `/api/quotes/onramp/best` | Get the best onramp quote. Query: `source_currency`, `amount` |
| GET | `/api/quotes/offramp/all` | Get all available offramp quotes. Query: `payout_currency`, `usdc_amount` |
| GET | `/api/quotes/offramp/best` | Get the best offramp quote. Query: `payout_currency`, `usdc_amount` |
| GET | `/api/quotes/fiat-to-fiat/best` | Get best fiat-to-fiat exchange quote. Query: `source_currency`, `target_currency`, `amount` |
| GET | `/api/quotes/fiat-to-fiat/all` | Get all fiat-to-fiat exchange quotes. Query: `source_currency`, `target_currency`, `amount` |
| GET | `/api/quotes/fiat-to-stablecoin/best` | Get best fiat-to-stablecoin quote. Query: `source_currency`, `amount`, `crypto_ticker` |
| GET | `/api/quotes/fiat-to-stablecoin/all` | Get all fiat-to-stablecoin quotes. Query: `source_currency`, `amount`, `crypto_ticker` |

### Crypto Payout

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/crypto-payout/quote` | Get crypto payout quote (DEX swap + optional bridge). Query: `sourceToken`, `destToken`, `sourceChain`, `destChain`, `amount`, `walletAddress`, `recipientAddress` |
| POST | `/api/crypto-payout/initiate` | Initiate a crypto payout execution |
| GET | `/api/crypto-payout/status/:id` | Get crypto payout status |

### Payouts

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/payouts` | Create a new payout |
| GET | `/api/payouts` | List all payouts. Query: `limit`, `offset` |
| GET | `/api/payouts/:id` | Get payout by ID |
| PUT | `/api/payouts/:id` | Update payout (status, metadata) |
| DELETE | `/api/payouts/:id` | Delete a payout |
| POST | `/api/payouts/:id/document` | Upload supporting document (multipart/form-data) |
| GET | `/api/payouts/:id/document/url` | Get signed download URL for payout document |
| GET | `/api/payouts/payment-purpose/:protocol` | Get payment purpose options for a provider protocol |
| GET | `/api/payouts/offramp-requirements/:protocol` | Get offramp requirements for a provider protocol |

### Recipients

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/recipients/wallet-networks` | Get available wallet networks for stablecoin recipients |
| POST | `/api/recipients` | Create a new recipient |
| GET | `/api/recipients` | List all recipients. Header: `X-User-ID` |
| GET | `/api/recipients/:id/with-counterparty` | Get recipient with counterparty payment method details |
| GET | `/api/recipients/:id` | Get recipient by ID |
| POST | `/api/recipients/:id/retry` | Retry failed recipient provisioning |
| PUT | `/api/recipients/:id` | Update a recipient |
| DELETE | `/api/recipients/:id` | Delete a recipient |

### Payment Methods

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/payment-methods/requirements` | Get payment method capability requirements |
| POST | `/api/payment-methods` | Create a new payment method |
| GET | `/api/payment-methods` | List user's payment methods |
| GET | `/api/payment-methods/:id` | Get payment method by ID |
| DELETE | `/api/payment-methods/:id` | Delete a payment method |

### KYB (Know Your Business)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/kyb/register` | Register for KYB |
| POST | `/api/kyb/register/:provider` | Register for KYB with a specific provider |
| GET | `/api/kyb/capabilities/currencies` | Get supported currencies |
| POST | `/api/kyb/capabilities/requirements` | Get form requirements for specific currencies |
| GET | `/api/kyb/capabilities/counterparty` | Get counterparty requirements. Query: `providers` |
| GET | `/api/kyb/capabilities/provider/:provider/requirements` | Get requirements for a specific provider |
| POST | `/api/kyb/capabilities/config/reload` | Reload KYB capabilities configuration |
| PUT | `/api/kyb/capabilities/config/currency` | Update currency configuration |
| PUT | `/api/kyb/capabilities/config/provider` | Update provider configuration |
| GET | `/api/kyb/onboarding/status` | Get provider onboarding status. Header: `X-User-ID` |
| POST | `/api/kyb/onboarding/create-accounts` | Create provider accounts with KYB data (multipart/form-data) |

### Conversion

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/conversion/onramp` | Execute onramp conversion (multipart/form-data, optional document) |
| GET | `/api/conversion/onramp/status` | Get onramp conversion status |

### Stablecoin Transfer

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/stablecoin-transfer/initiate` | Initiate a fiat-to-stablecoin transfer |
| GET | `/api/stablecoin-transfer/status/:id` | Poll transfer status |

### Bridge

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/bridge/quote` | Get a cross-chain bridge quote |
| POST | `/api/bridge/execute` | Execute a bridge transfer |
| GET | `/api/bridge/status/:id` | Get bridge transfer status |

### Tokens

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/tokens` | Get list of supported tokens |
| GET | `/api/tokens/:symbol` | Get token details by symbol |
| GET | `/api/tokens/:symbol/address` | Get token contract address by symbol |

### Banks

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/banks` | Get bank list by country. Query: `country` |

### Request for Quote

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/requestForQuote` | Get request for quote |

### Dashboard

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/dashboard/settings` | Save dashboard settings |
| GET | `/api/dashboard/settings` | Get dashboard settings |

### API Keys

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/apikeys/all` | List all API keys |
| GET | `/api/apikeys/:user_id` | Get API keys for a user |
| POST | `/api/apikeys/:user_id` | Create an API key for a user |
| DELETE | `/api/apikeys/:user_id` | Delete API keys for a user |

### Whitelist

JWT + UserLogger protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/whitelist` | List all whitelist entries |
| GET | `/api/whitelist/active` | List active whitelist entries |
| POST | `/api/whitelist` | Create a whitelist entry |
| PUT | `/api/whitelist/:id` | Update a whitelist entry |
| DELETE | `/api/whitelist/:id` | Delete a whitelist entry |
| PUT | `/api/whitelist/:id/active` | Toggle whitelist entry active status |

### Storage

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/storage/upload` | Upload a file to storage |
| GET | `/api/storage/presign` | Get a presigned upload URL |

### Admin

JWT protected.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/admin/config/reload` | Reload server configuration |

---

## Provider APIs

These are provider-specific endpoints used internally by the platform for executing transactions and managing provider integrations.

### Conduit

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/conduit/webhook` | Conduit webhook receiver |
| POST | `/api/conduit/customers` | Create Conduit customer. Header: `X-User-ID` |
| GET | `/api/conduit/customers` | List/get Conduit customers. Headers: `X-User-ID` or `X-Customer-ID` |
| POST | `/api/conduit/customers/entry` | Create customer entry |
| POST | `/api/conduit/control-persons` | Create control persons for KYB. Header: `X-Customer-ID` |
| POST | `/api/conduit/onboarding/documents` | Upload onboarding documents (multipart). Header: `X-Customer-ID` |
| POST | `/api/conduit/onboarding/submit` | Submit for onboarding. Header: `X-Customer-ID` |
| GET | `/api/conduit/onboarding/control-persons/liveness` | Get liveness check. Headers: `X-Customer-ID`, `X-Control-Person-ID` |
| POST | `/api/conduit/counterparties` | Create counterparty. Header: `X-Customer-ID` |
| GET | `/api/conduit/counterparties` | List counterparties. Header: `X-Customer-ID` |
| GET | `/api/conduit/counterparty` | Get specific counterparty. Headers: `X-Customer-ID`, `X-Counterparty-ID` |
| PATCH | `/api/conduit/payment-methods` | Update payment method. Header: `X-Customer-ID` |
| DELETE | `/api/conduit/counterparties/payment-methods` | Delete counterparty payment method. Headers: `X-Counterparty-ID`, `X-Payment-Method-ID` |
| POST | `/api/conduit/payment-methods/bank` | Create bank payment method. Header: `X-Customer-ID` |
| POST | `/api/conduit/payment-methods/wallet` | Create wallet payment method. Header: `X-Customer-ID` |
| GET | `/api/conduit/payment-methods` | List payment methods. Header: `X-Customer-ID` |
| DELETE | `/api/conduit/payment-methods` | Delete payment method |
| POST | `/api/conduit/quotes` | Create quote. Header: `X-Customer-ID` |
| POST | `/api/conduit/transaction` | Create transaction. Header: `X-Customer-ID` |
| GET | `/api/conduit/transaction` | Get transaction |
| POST | `/api/conduit/documents` | Upload documents |
| POST | `/api/conduit/customers/documents` | Upload customer documents |
| GET | `/api/conduit/customers/documents/download` | Download customer document |

### Airwallex

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/airwallex/webhook` | Airwallex webhook receiver |
| POST | `/api/airwallex/login` | Authenticate with Airwallex |
| GET | `/api/airwallex/global_accounts` | List global accounts |
| GET | `/api/airwallex/global_accounts/:ga_id` | Get global account by ID |
| GET | `/api/airwallex/global_accounts/transactions/:ga_id` | Get transactions for a global account |
| POST | `/api/airwallex/global_accounts/create` | Create a global account |
| GET | `/api/airwallex/current_balances` | Get current balances |
| GET | `/api/airwallex/balances_history` | Get balance history |
| GET | `/api/airwallex/linked_accounts` | List linked accounts |
| POST | `/api/airwallex/linked_accounts/create` | Create a linked account |
| POST | `/api/airwallex/quotes/create` | Create a quote |
| POST | `/api/airwallex/conversions/create_buy` | Create a buy conversion |
| POST | `/api/airwallex/conversions/create_sell` | Create a sell conversion |
| GET | `/api/airwallex/rates/current` | Get current exchange rates |
| POST | `/api/airwallex/transfer/create` | Create a transfer |
| POST | `/api/airwallex/transfer/create_id` | Create a transfer with ID |
| GET | `/api/airwallex/transfers` | List transfers |
| GET | `/api/airwallex/beneficiaries` | List beneficiaries |
| POST | `/api/airwallex/beneficiaries/create` | Create a beneficiary |
| GET | `/api/airwallex/payers` | List payers |
| POST | `/api/airwallex/payers/create` | Create a payer |
| POST | `/api/airwallex/payers/delete/:payer_id` | Delete a payer |
| POST | `/api/airwallex/payers/update/:payer_id` | Update a payer |
| GET | `/api/airwallex/payers/:payer_id` | Get payer by ID |
| POST | `/api/airwallex/simulate/deposit/create` | Simulate a deposit (sandbox) |
| GET | `/api/airwallex/deposits/list` | List deposits |
| POST | `/api/airwallex/deposits/create` | Create a deposit |

### TransFi

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/transfi/webhook` | TransFi webhook receiver |
| GET | `/api/transfi/supported-currencies` | Get supported currencies |
| GET | `/api/transfi/payment-methods-first` | Get payment methods (first) |
| GET | `/api/transfi/payment-methods` | Get payment methods |
| POST | `/api/transfi/users/individual` | Create individual user |
| GET | `/api/transfi/users/individual` | Get individual user |
| POST | `/api/transfi/users/business` | Create business user |
| GET | `/api/transfi/users/business` | Get business user |
| POST | `/api/transfi/onramp/invoice` | Create onramp invoice |
| POST | `/api/transfi/onramp/order` | Create onramp order |
| GET | `/api/transfi/onramp/order/:id` | Get onramp order |
| POST | `/api/transfi/kyc/submit` | Submit KYC |
| GET | `/api/transfi/kyc/user/:email` | Get KYC status by email |
| POST | `/api/transfi/offramp/prefund` | Create offramp prefund |
| POST | `/api/transfi/offramp/order` | Create offramp order |
| GET | `/api/transfi/offramp/order/:id` | Get offramp order |
| GET | `/api/transfi/quotes/offramp` | Get offramp quote |
| GET | `/api/transfi/quotes/onramp` | Get onramp quote |
| GET | `/api/transfi/quotes/two-way` | Get two-way quote |
| GET | `/api/transfi/quotes/deposit` | Get deposit quote |
| GET | `/api/transfi/quotes/withdraw` | Get withdraw quote |
| POST | `/api/transfi/payments/payin` | Create payin payment |
| POST | `/api/transfi/payments/payout` | Create payout payment |
| POST | `/api/transfi/payments/complete-payout` | Complete a payout |

### Nium

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/nium/get-client-details` | Get client details |
| GET | `/api/nium/get-client-fees` | Get client fees |
| GET | `/api/nium/get-client-limits` | Get client limits |
| GET | `/api/nium/get-client-transactions` | Get client transactions |
| POST | `/api/nium/client-prefund-request` | Create client prefund request |
| GET | `/api/nium/get-client-prefund-list` | List client prefunds |
| GET | `/api/nium/get-client-prefund-balances` | Get client prefund balances |
| POST | `/api/nium/add-customer` | Add a customer |
| POST | `/api/nium/update-customer/:customerID` | Update customer |
| GET | `/api/nium/get-customer-rfi/:customerID` | Get customer RFI (request for information) |
| POST | `/api/nium/respond-to-customer-rfi/:customerID` | Respond to customer RFI |
| POST | `/api/nium/upload-documents/:customerID` | Upload customer documents |
| POST | `/api/nium/add-corporate` | Add corporate entity |
| GET | `/api/nium/fetch-corporate` | Fetch corporate entity |
| GET | `/api/nium/fetch-corporate-rfi` | Fetch corporate RFI |
| POST | `/api/nium/respond-to-corporate-rfi` | Respond to corporate RFI |
| POST | `/api/nium/upload-corporate-documents/:customerID` | Upload corporate documents |
| POST | `/api/nium/regenerate-kyc-url/:customerID` | Regenerate KYC URL |
| GET | `/api/nium/get-corporate-constants` | Get corporate constants |
| POST | `/api/nium/update-corporate/:customerID` | Update corporate entity |
| GET | `/api/nium/get-wallet-balance` | Get wallet balance |
| POST | `/api/nium/wallet-transfer` | Create wallet transfer |
| POST | `/api/nium/add-wallet` | Add wallet |
| GET | `/api/nium/fetch-wallet` | Fetch wallet |
| POST | `/api/nium/add-funding-instrument` | Add funding instrument |
| POST | `/api/nium/assign-virtual-account-payment-id` | Assign virtual account payment ID |
| POST | `/api/nium/create-quote` | Create quote |
| GET | `/api/nium/fetch-quote/:quoteID` | Fetch quote by ID |
| POST | `/api/nium/create-conversion` | Create conversion |
| POST | `/api/nium/execute-conversion` | Execute conversion |
| GET | `/api/nium/fetch-exchange-rate` | Fetch exchange rate |

### PDAX

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/pdax/webhook` | PDAX webhook receiver |
| GET | `/api/pdax/asset-balance` | Get asset balance |
| GET | `/api/pdax/price-v1` | Get price (v1) |
| GET | `/api/pdax/price-v2` | Get price (v2) |
| POST | `/api/pdax/firm-quote-v1` | Get firm quote (v1) |
| POST | `/api/pdax/firm-quote-v2` | Get firm quote (v2) |
| POST | `/api/pdax/trade-v1` | Execute trade (v1) |
| GET | `/api/pdax/crypto-deposit-address` | Get crypto deposit address |
| POST | `/api/pdax/withdraw-fiat` | Withdraw fiat |
| POST | `/api/pdax/deposit-fiat` | Deposit fiat |
| POST | `/api/pdax/withdraw-crypto` | Withdraw crypto |
| POST | `/api/pdax/quote-trade-v1` | Quote trade (v1) |

### Changelly

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/changelly/get-status` | Get transaction status |
| GET | `/api/changelly/get-exchange-amount` | Get exchange amount |
| GET | `/api/changelly/get-fix-rate-for-amount` | Get fixed rate for amount |
| POST | `/api/changelly/create-transaction` | Create a transaction |
| POST | `/api/changelly/create-fix-transaction` | Create a fixed-rate transaction |

### Hata

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/hata/exchange_info` | Get exchange info |
| GET | `/api/hata/exchange_depth` | Get exchange depth (order book) |
| POST | `/api/hata/create_order` | Create order |
| POST | `/api/hata/cancel_order` | Cancel order |
| POST | `/api/hata/cancel_all_orders` | Cancel all orders |
| GET | `/api/hata/user_spot_balance` | Get user spot balance |
| GET | `/api/hata/user_orders_specific_pair` | Get user orders for pair |
| GET | `/api/hata/order_details` | Get order details |
| GET | `/api/hata/trade_history` | Get trade history |
| POST | `/api/hata/create_withdrawal` | Create withdrawal |
| GET | `/api/hata/user_withdrawal_history` | Get withdrawal history |
| GET | `/api/hata/user_deposit_history` | Get deposit history |
| GET | `/api/hata/internal_transfer_history` | Get internal transfer history |
| GET | `/api/hata/crypto_list` | Get crypto list |
| GET | `/api/hata/user_fiat_history` | Get fiat history |
| GET | `/api/hata/user_fiat_transaction_details` | Get fiat transaction details |

### Luno

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/luno/webhook` | Luno webhook receiver |
| GET | `/api/luno/alltickers` | Get all tickers |
| GET | `/api/luno/ticker` | Get ticker |
| GET | `/api/luno/orderbook` | Get order book |
| GET | `/api/luno/list-trades` | List trades |
| GET | `/api/luno/balances` | Get balances |
| GET | `/api/luno/list-transactions` | List transactions |
| GET | `/api/luno/list-pending-transactions` | List pending transactions |
| GET | `/api/luno/list-orders` | List orders |
| GET | `/api/luno/list-user-trades` | List user trades |
| GET | `/api/luno/fee-info` | Get fee info |
| GET | `/api/luno/funding-address` | Get funding address |
| GET | `/api/luno/list-withdrawals` | List withdrawals |
| GET | `/api/luno/get-withdrawal` | Get withdrawal |
| GET | `/api/luno/get-candles` | Get candles (OHLC) |
| GET | `/api/luno/get-move` | Get move |
| GET | `/api/luno/get-order` | Get order |
| GET | `/api/luno/get-order-v2` | Get order (v2) |
| GET | `/api/luno/get-order-v3` | Get order (v3) |
| GET | `/api/luno/get-markets` | Get markets |
| POST | `/api/luno/create-account` | Create account |
| POST | `/api/luno/create-beneficiary` | Create beneficiary |
| POST | `/api/luno/create-funding-address` | Create funding address |
| POST | `/api/luno/create-withdrawal` | Create withdrawal |
| POST | `/api/luno/create-market-order` | Create market order |
| POST | `/api/luno/create-limit-order` | Create limit order |
| POST | `/api/luno/cancel-withdrawal` | Cancel withdrawal |
| POST | `/api/luno/cancel-order` | Cancel order |
| POST | `/api/luno/send` | Send crypto |
| GET | `/api/luno/send-fee` | Get send fee |
| GET | `/api/luno/list-transfers` | List transfers |

### Borderless

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/borderless/webhook` | Borderless webhook receiver |
| GET | `/api/borderless/webhooks/:id` | Get webhook by ID |
| GET | `/api/borderless/webhooks` | List webhooks |
| POST | `/api/borderless/webhooks` | Create webhook |
| PATCH | `/api/borderless/webhooks/:id` | Update webhook |
| DELETE | `/api/borderless/webhooks/:id` | Delete webhook |

### OnrampMoney

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/onrampmoney/webhook` | OnrampMoney webhook receiver |

### 1inch Swap API (Proxy)

| Method | Endpoint | Description |
|--------|----------|-------------|
| ANY | `/1inch-swap-api/*path` | Proxy to 1inch swap API. Forwards all requests. |

---

## Other Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/metrics` | Prometheus metrics |
| GET | `/api/debug/headers` | Debug: show request headers |
| GET | `/ws` | WebSocket connection |
| GET | `/api/hello` | Health check |

---

## Mintlify Page Structure

When generating Mintlify documentation, organize pages as follows:

```
docs/
├── introduction.mdx
├── authentication.mdx
├── api-reference/
│   ├── users/
│   │   ├── sync.mdx
│   │   ├── link-accounts.mdx
│   │   ├── check-verification.mdx
│   │   ├── me.mdx
│   │   ├── me-settings.mdx
│   │   ├── get.mdx
│   │   ├── create.mdx
│   │   ├── update.mdx
│   │   ├── delete.mdx
│   │   └── business.mdx
│   ├── businesses/
│   │   ├── list.mdx
│   │   ├── get.mdx
│   │   ├── create.mdx
│   │   ├── update.mdx
│   │   ├── delete.mdx
│   │   └── users.mdx
│   ├── quotes/
│   │   ├── onramp-all.mdx
│   │   ├── onramp-best.mdx
│   │   ├── offramp-all.mdx
│   │   ├── offramp-best.mdx
│   │   ├── fiat-to-fiat-best.mdx
│   │   ├── fiat-to-fiat-all.mdx
│   │   ├── fiat-to-stablecoin-best.mdx
│   │   └── fiat-to-stablecoin-all.mdx
│   ├── crypto-payout/
│   │   ├── quote.mdx
│   │   ├── initiate.mdx
│   │   └── status.mdx
│   ├── payouts/
│   │   ├── list.mdx
│   │   ├── get.mdx
│   │   ├── create.mdx
│   │   ├── update.mdx
│   │   ├── delete.mdx
│   │   ├── upload-document.mdx
│   │   ├── get-document-url.mdx
│   │   ├── offramp-requirements.mdx
│   │   └── payment-purpose.mdx
│   ├── recipients/
│   │   ├── list.mdx
│   │   ├── get.mdx
│   │   ├── create.mdx
│   │   ├── update.mdx
│   │   ├── delete.mdx
│   │   ├── retry.mdx
│   │   ├── with-counterparty.mdx
│   │   └── wallet-networks.mdx
│   ├── kyb/
│   │   ├── register.mdx
│   │   ├── onboarding-status.mdx
│   │   ├── currencies.mdx
│   │   ├── counterparty-requirements.mdx
│   │   ├── form-requirements.mdx
│   │   ├── provider-requirements.mdx
│   │   └── create-accounts.mdx
│   ├── payment-methods/
│   │   ├── list.mdx
│   │   ├── get.mdx
│   │   ├── requirements.mdx
│   │   ├── create.mdx
│   │   └── delete.mdx
│   ├── stablecoin-transfer/
│   │   ├── initiate.mdx
│   │   └── status.mdx
│   ├── conversion/
│   │   ├── onramp.mdx
│   │   └── status.mdx
│   ├── bridge/
│   │   ├── quote.mdx
│   │   ├── execute.mdx
│   │   └── status.mdx
│   ├── tokens/
│   │   ├── list.mdx
│   │   ├── get.mdx
│   │   └── address.mdx
│   ├── banks/
│   │   └── list.mdx
│   ├── dashboard/
│   │   ├── get-settings.mdx
│   │   └── save-settings.mdx
│   ├── api-keys/
│   │   ├── list.mdx
│   │   ├── get.mdx
│   │   ├── create.mdx
│   │   └── delete.mdx
│   ├── whitelist/
│   │   ├── list.mdx
│   │   ├── create.mdx
│   │   ├── update.mdx
│   │   ├── delete.mdx
│   │   └── toggle-active.mdx
│   ├── storage/
│   │   ├── upload.mdx
│   │   └── presign.mdx
│   └── providers/
│       ├── conduit/
│       ├── airwallex/
│       ├── transfi/
│       ├── nium/
│       ├── pdax/
│       ├── changelly/
│       ├── hata/
│       ├── luno/
│       └── borderless/
└── guides/
    ├── fiat-to-fiat-payout.mdx
    ├── stablecoin-transfer.mdx
    ├── crypto-payout.mdx
    └── kyb-onboarding.mdx
```

## Mintlify MDX Template

Each API endpoint page should follow this template:

```mdx
---
title: "{Endpoint Title}"
api: "{METHOD} {path}"
description: "{Brief description}"
---

## Headers

<ParamField header="Authorization" type="string" required>
  Bearer token from Auth0
</ParamField>

## Path Parameters

<ParamField path="paramName" type="string" required>
  Description
</ParamField>

## Query Parameters

<ParamField query="paramName" type="string" required>
  Description
</ParamField>

## Body Parameters

<ParamField body="fieldName" type="string" required>
  Description
</ParamField>

## Response

<ResponseField name="fieldName" type="string">
  Description
</ResponseField>

<ResponseExample>
```json
{
  "example": "response"
}
```
</ResponseExample>
```

## Key Concepts

- **Onramp**: Converting fiat currency into USDC stablecoin
- **Offramp**: Converting USDC stablecoin into fiat currency
- **Fiat-to-Fiat**: Cross-border fiat payment (e.g., USD → MYR) routed through the best provider
- **Fiat-to-Stablecoin**: Converting fiat into a specific stablecoin (e.g., USD → MYRC)
- **Stablecoin-to-Stablecoin (S2S)**: Transferring stablecoins across chains via DEX swap + bridge
- **Quote**: A rate estimate from one or more providers before executing a transfer
- **KYB**: Know Your Business — compliance onboarding required before transacting
- **Recipient**: A counterparty (person or business) that receives payouts
- **Payment Method**: A bank account or wallet linked to the user for sending/receiving funds
- **Bridge**: Cross-chain transfer of tokens (e.g., USDT on Arbitrum → USDC on Base)
- **DEX Swap**: On-chain token swap via decentralized exchange (Uniswap, CowSwap)
- **Provider**: External service integrated for payments (Airwallex, TransFi, Nium, Conduit, etc.)
