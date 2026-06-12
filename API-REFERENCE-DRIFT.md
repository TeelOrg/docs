# API-reference / OpenAPI drift

Tracking sheet for the **API-reference owner**. These items surfaced while cleaning the
**Documentation** pages (Getting Started + Guides) on branch `docs/cleanup-documentation-pages`.

Each row is a place where `go-server/rest/openapi/openapi.yaml` (and the `api-reference/*.mdx`
pages generated/mirrored from it) disagrees with what the live Go handler actually accepts or
returns. The handler is the source of truth — the spec drives partner SDK codegen, so a wrong
spec means partners generate a client the server silently rejects.

Source-of-truth files cited as `teel-backend/go-server/...`.

---

## 1. `CreateRecipient` request schema — wrong casing, type, and a phantom required field

**Spec** (`openapi.yaml` `components.schemas.CreateRecipient`): snake_case, `is_business: boolean`,
`currency` listed as required.

**Handler** (`models/recipient.go` `CreateRecipientDTO`, bound via `ShouldBindJSON` in
`rest/recipient_controller.go:432`):

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

**Handler** (`rest/rfq_proxy_controller.go:64-76`, `RFQExecuteRequest`):

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

`middleware/env_scope.go:235` returns `gin.H{"error": "environment_scope mismatch", "host_scope": ..., "actor_scope": ..., "actor": ...}` — **not** the canonical `respondError` shape `{code, error, details?}` mandated for partner-callable surfaces (see CLAUDE.md "Stable error codes on partner-callable handlers"). There is no stable `code` for this 403, so partners can't switch on it programmatically. Either route it through `respondError` with a new `ErrCode...` constant, or document the bespoke shape explicitly. (Docs now describe the literal body as-is.)

Also note the **enforcement asymmetry**: `ENV_SCOPE_ENFORCE=true` in `deploy-sandbox.yml` but unset in production `deploy.yml`, so prod is warn-only today. Docs intentionally describe the enforced contract (rule + 403) without advertising the prod gap.

---

## 6. `GET /config/coverage` is missing from `openapi.yaml`

The endpoint exists (`rest/currency_config_controller.go:GetCoverage`, `main.go:340`) and has a docs page (`api-reference/config/coverage.mdx`), but it is **not** declared in `openapi.yaml` — so it's absent from partner SDK codegen and the served spec. Add it (unauthenticated; response = the `Coverage` struct: `{fiat: CurrencyCoverage[], wallet: WalletCoverage}`).

Also confirm the **public path**: the Go route is `api.GET("/config/coverage")` (internal `/api/config/coverage`), but every partner-facing path in the spec is served bare (no `/api`) on `api.teel.finance`. Docs now use `https://api.teel.finance/config/coverage` to match the quickstart and the rest of the surface — verify the ingress strips `/api` for this route too.

---

_Add new rows below as later doc pages surface more drift._
