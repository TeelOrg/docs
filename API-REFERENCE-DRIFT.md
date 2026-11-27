# API-reference / OpenAPI drift

Tracking sheet for places where `payments-orchestrator/internal/rest/openapi/openapi.yaml`, the
`api-reference/*.mdx` pages, and the live Go handler disagree. The handler is the source of truth:
the spec drives partner SDK codegen, so a wrong spec means partners generate a client the server
silently rejects.

Source-of-truth files are cited relative to `teel-backend/go-apps/backend/payments-orchestrator/`,
except `models/...` (`teel-backend/go-apps/backend/shared/models/`) and quote-engine paths
(`teel-backend/go-apps/quote-engine/`).

## Open

These are not spec typos. The spec and pages describe the handler as it behaves; each item is a
behaviour that needs an engineering decision.

### Partner responses

- **Webhook subscription and delivery responses serve `userId`**: the handlers serialize
  `models.WebhookSubscription` / `models.WebhookDelivery` directly rather than through a partner
  DTO. The pages document it as served.
- **`COLLECTIONS_WALLET_SOURCE_NOT_SUPPORTED`** is `422` everywhere except
  `internal/rest/collections_convert_controller.go`, which sends it as `403`. A code should map to
  one status.
- **Payment link status on partner reads** uses `models.EffectiveStatus`, not
  `EffectiveStatusWithAllocation`, so a part-paid stablecoin invoice past `expiresAt` reads
  `expired` to a partner while it is still payable.
- **Scoped settle signer is Tron-only**: `settleNeedsDashboardAuthorization` honours it on the
  Tron (USDT, `high`) rail only; any non-custodial Polygon/Ethereum (USDC, `low`) wallet is refused.

### Errors without a `code`

- **`POST /rfq/execute-batch`**: more than 49 items (`400`) and the not-ready `503`s are
  `{"success": false, "error": …}` with no `code` (quote-engine `internal/api/rfq_router.go`
  `handleLockAndExecuteBatch`).
- **Webhook replay's per-user `429`**: `middleware.PerUserRateLimiter` answers
  `{"success": false, "error": "rate limit exceeded"}`, outside the `respondError` shape.

### Spec shape

- **`POST /rfq/execute-batch` per-row key** is `items[].idempotency_key` (snake_case) in the
  quote-engine `batchItem` struct and the spec, against the camelCase convention. Renaming it
  breaks partners who send it today.
- **`Error.code` enum lists codes nothing partner-reachable emits**: `UPSTREAM_TIMEOUT`,
  `SANDBOX_PAYOUTS_UNAVAILABLE` and `QUOTE_NOT_FOUND` have no emitter; `WALLET_NOT_SET_UP` is never
  sent to a partner key; `SOURCE_NOT_SERVICEABLE` is sent only by `POST /payment-methods`, which is
  dashboard-only. `errors.mdx` leaves them out. Removing values from a published enum changes
  generated clients.

## Decisions

- **Partner WebSocket auth is not documented**: `internal/websocket/handler.go` accepts only a
  dashboard JWT (`token`); the `{type: "auth", apiKey}` frame is no longer read. Partners are
  pointed at webhooks and polling.

## Resolved

- `CreateRecipient` and `RecipientPaymentMethod`: the spec is camelCase, `isBusiness` is a
  boolean (as `models.CreateRecipientDTO` binds it), `currency` is optional, and `type` is
  `bank | wallet`.
- `/rfq/execute`: the spec pins the route with `routeProtocol` and lists the corridor fields;
  `supportingDocumentKey` is refused and documented as removed.
- Quote responses carry the `protocol` route token in the spec and pages.
- `env_scope` 403 carries `code: "ENV_SCOPE_MISMATCH"`.
- `GET /config/coverage` is in the spec.
- `CreatePaymentLink.settlementCurrency`: the spec lists it only on the response, for dashboard
  conversion links.
- Offramp document upload: the spec accepts WEBP.
- `PartnerPaymentLink.reference` example is `PL-` + 26 base32 characters.
- `servers[1]` is `https://api.teel.finance/sandbox`; the spec no longer names `go-server`.
- `fiat` links reach `paid` when matched deposits cover the amount.
- `PATCH /webhooks/subscriptions/{id}` no longer claims idempotency (teel-backend#6145).
- `GET /recipients/wallet-networks` is dashboard-only; guides use `GET /config/coverage`.
- Collections rate limits, void of an expired link, and `test-fiat-payment` expiry (teel-backend#4289).
- `POST /rfq/execute-batch` 202 body, item cap and error statuses; `POST /rfq/confirm-escrow-deposit`
  `400 INVALID_JSON`; `GET /payouts/{id}/retryable` `action` values; `GET /metrics/earnings`
  `from`/`to` accept RFC 3339 (teel-backend#6191).

_Add new rows under **Open** as later doc pages surface more drift._
