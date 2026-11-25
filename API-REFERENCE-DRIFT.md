# API-reference / OpenAPI drift

Tracking sheet for the **API-reference owner**. These items surfaced while cleaning the
**Documentation** pages (Getting Started + Guides) on branch `docs/cleanup-documentation-pages`.

Each row is a place where `payments-orchestrator/internal/rest/openapi/openapi.yaml` (and the `api-reference/*.mdx`
pages generated/mirrored from it) disagrees with what the live Go handler actually accepts or
returns. The handler is the source of truth — the spec drives partner SDK codegen, so a wrong
spec means partners generate a client the server silently rejects.

Source-of-truth files are cited relative to `teel-backend/go-apps/backend/payments-orchestrator/`
(`cmd/main.go`, `internal/rest/...`, `internal/middleware/...`), except `models/...`, which lives in
`teel-backend/go-apps/backend/shared/models/`. The backend moved there from `go-server/`; line
numbers below are from the old tree and may have shifted.

---

## 1. `CreateRecipient` request schema — wrong casing, type, and a phantom required field

**Spec** (`openapi.yaml` `components.schemas.CreateRecipient`): snake_case, `is_business: boolean`,
`currency` listed as required.

**Handler** (`models/recipient.go` `CreateRecipientDTO`, bound via `ShouldBindJSON` in
`internal/rest/recipient_controller.go`):

- Fields are **camelCase**: `isBusiness`, `businessName`, `transferType`, `paymentMethods`.
- `isBusiness` is a **string** (`"business"`; empty defaults to business via `IsBusinessRecipient()`), not a boolean.
- There is **no top-level `currency`** field — currency is per-payment-method.

Go JSON binding matches tags exactly, so a snake_case body has every multi-word field **silently dropped**
(no business name, no transfer type, no payment methods). Confirmed convention in `.claude/agents/docs-reviewer.md`:
"Request DTOs are camelCase; responses are snake_case."

## 2. `RecipientPaymentMethod` schema — wrong `type` enum + casing

**Spec**: `type` enum `[bank_account, wallet]`, snake_case fields (`bank_name`, `account_number`).

**Handler** (`models/recipient.go` `CounterpartyPaymentMethodDTO`, `RecipientType` const):

- `type` is `"bank"` or `"wallet"` (`RecipientTypeBank = "bank"`), **not** `bank_account`.
- Fields are camelCase: `bankName`, `accountNumber`, `routingNumber`, `swiftCode`, etc.
- **Wallet PMs**: on create, the chain is the `rail` field (`dtoToRecipientPaymentMethod` does `LookupChainID(pm.Rail)`), with `walletAddress` + `currency` (token). The spec's `wallet_network` doesn't match the create DTO (`CounterpartyPaymentMethodDTO` has no `walletNetwork` on the create side; the read model `RecipientPaymentMethod` exposes both `rail` and `walletNetwork`, camelCase). Reconcile the create-side wallet field set in the spec.
- `GET /recipients/wallet-networks` returns a **top-level JSON array** of `{name, displayName, chainId}` (no `tokens`, no `{networks:[…]}` wrapper) — confirm the spec matches.

## 3. `/rfq/execute` request schema — wrong required field, missing corridor fields

**Spec**: `required: [quoteId]`; properties `quoteId`, `userPaymentMethodId`, `recipientId`,
`recipientPaymentMethodId`, `metadata`.

**Handler** (`internal/rest/rfq_proxy_controller.go`, `RFQExecuteRequest`):

- Required: `fromCurrency`, `toCurrency`, `amount` (`gt=0`), `recipientId`.
- Pins the route via `routeProtocol` (the opaque token from the quote response).
- **No `quoteId` field** is read on execute. `quoteId` is only used to poll `GET /rfq/status/{quoteId}`.
- No `metadata` field; instead `flowType`, `paymentPurpose`, `reference`, `walletAddress`, `targetAmount`, `supportingDocumentKey`.

## 4. `Quote` response schema — missing the `protocol` route token

**Spec** `Quote`: `quoteId, provider, fromAmount, toAmount, rate, fee, expiresAt`.

**Reality**: the quote response also carries an opaque **`protocol`** field (masked by
`internal/routealias`), which the partner must send back as `routeProtocol` on `/rfq/execute`.
Without it documented, partners can't pin a route. See `internal/routealias/routealias.go`
(`MaskResponseBody`, key `"protocol"`).

---

## 5. ~~`env_scope` 403 uses a non-standard error envelope~~ (resolved)

The 403 now carries `code: "ENV_SCOPE_MISMATCH"` (`internal/middleware/env_scope.go`), and `ENV_SCOPE_ENFORCE` is set on the prod orchestrator overlay. `authentication.mdx` and `errors.mdx` name the code.

---

## 6. ~~`GET /config/coverage` is missing from `openapi.yaml`~~ (resolved)

`/config/coverage` is declared in the served spec.

---

## 7. Collections partner routes — spec vs handler (found writing the collections pages)

Docs follow the handler in each case.

### Open

- ~~**`fiat` links never reach `paid` in production**~~ Resolved: a credited deposit is now matched to its fiat link and marks it `paid` when the matched deposits cover its amount (`database.MatchCreditedDepositToInvoice`). Public pages describe the matching.
- **`CreatePaymentLink.settlementCurrency`**: listed with no caveat. `buildPaymentLink` refuses it on `fiat` and `stablecoin` (it is only for the sandbox-only `fiat_conversion` type, which the partner `collectionType` enum does not list).
- **`POST /collections/receiving-accounts/offramp/document`**: the spec says PDF, JPG or PNG. `utils.ValidateUpload` defaults (KYB set) also accept WEBP.
- **`PartnerPaymentLink.reference` example** `PL-7F3K2Q`: real references are `PL-` + 26 base32 characters (`generatePaymentLinkReference`).
- ~~**`Error.code` description** still cites `go-server/rest/api_errors.go`.~~ Resolved: the spec no longer names `go-server`.
- ~~**Spec `servers[1]` and the `PartnerBearer` description** name `api-sandbox.teel.finance`.~~ Resolved: `servers[1]` is `https://api.teel.finance/sandbox`.
- **Scoped settle signer is Tron-only**: `settleNeedsDashboardAuthorization` honours it on the Tron (USDT, `high`) rail only; any non-custodial Polygon/Ethereum (USDC, `low`) wallet is refused.
- **Payment link status on partner reads** uses `models.EffectiveStatus`, not `EffectiveStatusWithAllocation`, so a part-paid stablecoin invoice past `expiresAt` reads `expired` to a partner while it is still payable.

### Fixed in teel-backend#4289

- **Rate limits stacked**: the catch-all default limiter ran alongside each family limiter, so collections reads were effectively 60/min. Now each route is governed by exactly one limiter and the `X-RateLimit-*` headers match it.
- **Void of an expired link returned `200`**: voiding an expired or already-voided link now returns `409 PAYMENT_LINK_NOT_VOIDABLE` for all callers, as the spec said.
- **`test-fiat-payment` ignored expiry**: it now refuses links whose effective status isn't `open` with `409 CONFLICT`, like the stablecoin simulator.

### Surfaced fixing stale payout, recipient and error pages (`docs/fix-stale-pages`)

- **`GET /payouts/{id}/retryable` `data.action`** is described as "provider-specific" in `openapi.yaml`. A partner response must not vary by, or reveal, the provider; the description should name the actual action values.
- **`GET /metrics/earnings` `from` / `to`** are typed `format: date`; the handler (quote-engine `internal/api/internal_router.go` `handleGetEarningsMetrics`) also accepts RFC 3339 timestamps.
- **`POST /rfq/execute-batch` per-row key** is `items[].idempotency_key` (snake_case) in the quote-engine `batchItem` struct and the spec, against the camelCase convention. The docs page now says `idempotency_key`; it used to say `idempotencyKey`, which is silently dropped.
- **`POST /rfq/execute-batch` 202 `rows[]`**: the spec lists `transactionId`, `quoteId`, `errorCode`, `retryable` on the 202 rows; the handler serves only `idx` and `status: pending` (they appear on `GET /jobs/{jobId}`). The spec's 202 also omits `batchId`, `collectionId`, `totalSourceAmount` and `escrowTxHash` from the schema while its prose names `batchId`. The handler comment says `items` is served empty; it is not served at all.
- **`POST /rfq/execute-batch` refusals without a code**: more than 49 items (`400`) and the not-ready `503`s are `{"success": false, "error": …}` with no `code` (quote-engine `rfq_router.go` `handleLockAndExecuteBatch`). The spec lists `422` only as `FundingRailUnsupported` and has no `403 PAYOUTS_NOT_ENABLED` / `422 DIRECT_BANK_BATCH_UNSUPPORTED` / `502` entry.
- **`POST /rfq/confirm-escrow-deposit` `400`**: the spec's body is `{success: false, error: "invalid request"}`; the Core proxy now refuses first with `{code: INVALID_JSON, error}`.
- **`Error.code` enum lists codes nothing partner-reachable emits**: `UPSTREAM_TIMEOUT`, `SANDBOX_PAYOUTS_UNAVAILABLE` and `QUOTE_NOT_FOUND` have no emitter; `WALLET_NOT_SET_UP` is documented in `api_errors.go` as never sent to a partner key; `SOURCE_NOT_SERVICEABLE` is sent only by `POST /payment-methods`, which is `denyPartner`. `errors.mdx` leaves them out.
- **`COLLECTIONS_WALLET_SOURCE_NOT_SUPPORTED`** is `422` everywhere except `collections_convert_controller.go`, which sends it as `403`; a code should map to one status.

---

### Surfaced fixing the auth, webhooks and collections pages (`docs/audit-auth-webhooks-collections`)

- **`PATCH /webhooks/subscriptions/{id}` claims idempotency in the spec**: the route mounts no `idempotency` middleware (`cmd/main.go` webhooks group). The docs page does not document `Idempotency-Key`. Spec fix: TeelOrg/teel-backend#6145.
- **Partner WebSocket auth removed from the docs**: `internal/websocket/handler.go` accepts only a dashboard JWT (`token`); the `{type: "auth", apiKey}` frame is no longer read. Decision: partners are pointed at webhooks and polling, and the dashboard JWT path is not documented. The `wss://…/sandbox/ws` host line in `AGENTS.md` is now unused by public pages.
- **Item 2 above assumes `GET /recipients/wallet-networks` is partner-reachable**: it is mounted with `denyPartner`, so a partner key cannot call it.
- **Webhook replay's per-user `429` has no `code`**: `middleware.PerUserRateLimiter` answers `{"success": false, "error": "rate limit exceeded"}`, outside the `respondError` shape. The replay page documents the bare body.
- **Webhook subscription and delivery responses serve `userId`**: the handlers serialize `models.WebhookSubscription` / `models.WebhookDelivery` directly rather than through a partner DTO, so `userId` is on the wire. The pages document it as served.

_Add new rows below as later doc pages surface more drift._
