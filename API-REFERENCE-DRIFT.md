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

## 5. `env_scope` 403 uses a non-standard error envelope

`internal/middleware/env_scope.go` returns `gin.H{"error": "environment_scope mismatch", "host_scope": ..., "actor_scope": ..., "actor": ...}` — **not** the canonical `respondError` shape `{code, error, details?}` mandated for partner-callable surfaces (see CLAUDE.md "Stable error codes on partner-callable handlers"). There is no stable `code` for this 403, so partners can't switch on it programmatically. Either route it through `respondError` with a new `ErrCode...` constant, or document the bespoke shape explicitly. (Docs now describe the literal body as-is.)

Also note the **enforcement asymmetry**: `ENV_SCOPE_ENFORCE=true` in `deploy-sandbox.yml` but unset in production `deploy.yml`, so prod is warn-only today. Docs intentionally describe the enforced contract (rule + 403) without advertising the prod gap.

---

## 6. `GET /config/coverage` is missing from `openapi.yaml`

The endpoint exists (`internal/rest/currency_config_controller.go:GetCoverage`, `cmd/main.go`) and has a docs page (`api-reference/config/coverage.mdx`), but it is **not** declared in `openapi.yaml` — so it's absent from partner SDK codegen and the served spec. Add it (unauthenticated; response = the `Coverage` struct: `{fiat: CurrencyCoverage[], wallet: WalletCoverage}`).

Also confirm the **public path**: the Go route is `api.GET("/config/coverage")` (internal `/api/config/coverage`), but every partner-facing path in the spec is served bare (no `/api`) on `api.teel.finance`. Docs now use `https://api.teel.finance/config/coverage` to match the quickstart and the rest of the surface — verify the ingress strips `/api` for this route too.

The ingress rewrite is idempotent (`^(/api)?(.*) → /api\2`), so both `/config/coverage` and `/api/config/coverage` reach the route.

---

## 7. Collections partner routes — spec vs handler (found writing the collections pages)

Docs follow the handler in each case.

### Open

- ~~**`fiat` links never reach `paid` in production**~~ Resolved: a credited deposit is now matched to its fiat link and marks it `paid` when the matched deposits cover its amount (`database.MatchCreditedDepositToInvoice`). Public pages describe the matching.
- **`CreatePaymentLink.settlementCurrency`**: listed with no caveat. `buildPaymentLink` refuses it on `fiat` and `stablecoin` (it is only for the sandbox-only `fiat_conversion` type, which the partner `collectionType` enum does not list).
- **`POST /collections/receiving-accounts/offramp/document`**: the spec says PDF, JPG or PNG. `utils.ValidateUpload` defaults (KYB set) also accept WEBP.
- **`PartnerPaymentLink.reference` example** `PL-7F3K2Q`: real references are `PL-` + 26 base32 characters (`generatePaymentLinkReference`).
- **`Error.code` description** still cites `go-server/rest/api_errors.go`.
- **Spec `servers[1]` and the `PartnerBearer` description** name `api-sandbox.teel.finance`, which has no DNS record. The sandbox API is served at `https://api.teel.finance/sandbox` (`deploy/apps/sandbox-webapi/overlays/prod/ingress/sandbox.yaml`). The docs pages now use that (fixed in `authentication.mdx`, `introduction.mdx`, `quickstart.mdx` and the collections guides); the spec still needs it.
- **Scoped settle signer is Tron-only**: `settleNeedsDashboardAuthorization` honours it on the Tron (USDT, `high`) rail only; any non-custodial Polygon/Ethereum (USDC, `low`) wallet is refused.
- **Payment link status on partner reads** uses `models.EffectiveStatus`, not `EffectiveStatusWithAllocation`, so a part-paid stablecoin invoice past `expiresAt` reads `expired` to a partner while it is still payable.

### Fixed in teel-backend#4289

- **Rate limits stacked**: the catch-all default limiter ran alongside each family limiter, so collections reads were effectively 60/min. Now each route is governed by exactly one limiter and the `X-RateLimit-*` headers match it.
- **Void of an expired link returned `200`**: voiding an expired or already-voided link now returns `409 PAYMENT_LINK_NOT_VOIDABLE` for all callers, as the spec said.
- **`test-fiat-payment` ignored expiry**: it now refuses links whose effective status isn't `open` with `409 CONFLICT`, like the stablecoin simulator.

---

_Add new rows below as later doc pages surface more drift._
