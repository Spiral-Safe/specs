# HTTP and Vault API contract

This document describes the current workspace implementation. The HTTP API is
not versioned and has no generated OpenAPI schema yet; treat it as a
development contract.

## Service origin and authentication

The HTTP adapter listens on port 3000 by default. Browser and SDK clients talk
only to this adapter, never directly to Vault.

All wallet routes require:

```http
Authorization: Bearer <account-scoped-api-key>
Content-Type: application/json
```

### Account-backed production API keys

With production `BILLING_MODE=postgres`, API keys are issued from the developer
console with an `ssk_live_...` prefix and a cryptographically random secret.
The secret appears once. PostgreSQL stores only its non-secret prefix and an
HMAC-SHA-256 digest under `API_KEY_PEPPER`, along with the owning account,
tenant, username allowlist, scopes, last-use time, and revocation state.

Authentication derives all of these server-side. A caller cannot select an
account, tenant, role, scope, or additional username in the request body. The
service rejects an unauthorized username or missing scope with 403 before
Vault, an inactive account with `402 payment_required`, and a Metronome-enabled
account without a recorded per-account Stripe mapping with
`402 billing_provisioning_required`.

Available scopes are:

| Scope | Routes |
| --- | --- |
| `wallets:read` | `/check` |
| `wallets:write` | `/init`, `/create` |
| `signatures:create` | `/signin`, `/complete` |

Production billing rejects `API_TOKEN_HASHES`, `API_TOKENS`, and `API_TOKEN`
rather than mixing database-issued and static credentials.

### Billing-disabled and development tokens

When the account layer is disabled, `API_TOKEN_HASHES` maps 64-character
SHA-256 token digests to static principals:

```json
{
  "<sha256-of-token>": {
    "tenant": "tenant-a",
    "users": ["alice", "bob"]
  }
}
```

Production requires a non-empty `users` array. `API_TOKENS` uses the same
principal value shape with plaintext tokens as keys; a single `API_TOKEN` uses
`API_TENANT` and a comma-separated `API_USERS`. Outside
`SERVICE_DEV_MODE=true`, plaintext API tokens must be at least 24 characters.
Development mode alone accepts the legacy string tenant mapping or a missing
`API_USERS` list as an explicit tenant-wide wildcard.

The adapter injects the authenticated tenant into every Vault request; a body
field named `tenant` has no effect. It rejects a username outside the
credential's allowlist with `403 user_forbidden` before making a Vault request.

### Console sessions

Developer and admin pages do not use wallet API keys as browser sessions.
`POST /developer/login` and `POST /admin/login` verify the stored role and
scrypt password hash, then set a random opaque session cookie. Only a keyed
token hash is stored server-side. The cookie is HttpOnly, SameSite=Strict, and
Secure outside memory development mode. Mutating console forms additionally
require a session-derived CSRF token and exact `CONSOLE_ORIGIN`.

Role and account identity come from the stored session, not form fields. A
developer session is bound to one account; an admin session has no account and
can create/inspect accounts. These sessions are not accepted in
`Authorization` for wallet routes.

API tokens are application credentials, not Vault tokens. The service uses
either:

- `VAULT_K8S_ROLE` and its projected service-account JWT to obtain and refresh
  a short-lived Vault token in production; or
- `VAULT_TOKEN` only in explicit development mode.

Every static `VAULT_TOKEN` value is rejected when `SERVICE_DEV_MODE` is false,
whether or not the token happens to be a root token. Production therefore
requires Kubernetes authentication rather than a long-lived Vault credential.

The SDK and extension never accept or expose a Vault token.

## Common identity and encoding

Every wallet operation has:

| Field | Type | Rules |
| --- | --- | --- |
| `username` | string | 1–64 safe identifier characters; first character alphanumeric |
| `chain` | string | `solana` (default) or `ethereum` |

Signing payloads use standard base64. WebAuthn challenge, user ID, credential
ID, and response buffers use base64url in their JSON representation.

The plugin's WebAuthn user handle is a deterministic 43-byte base64url
SHA-256 digest scoped to tenant, username, and chain. The clear username is not
used as the protocol user handle.

## Routes

### Public probes

| Method and path | Success | Meaning |
| --- | --- | --- |
| `GET /healthz` | `200 {"status":"ok"}` | The Node process can serve HTTP. It does not prove Vault is usable. |
| `GET /readyz` | `200 {"status":"ready"}` | Vault is usable and, when billing is enabled, the account store is ready. Returns 503 otherwise. |

No bearer token is required for probes.

Wallet authorization and metering are:

| Route | Required scope | Successful-use metric with billing enabled |
| --- | --- | --- |
| `/init` | `wallets:write` | none; it may create an incomplete key-bearing record |
| `/create` | `wallets:write` | first `active_wallet` for tenant/user/chain in the account period |
| `/check` | `wallets:read` | first `active_wallet` for tenant/user/chain in the account period |
| `/signin` | `signatures:create` | first `active_wallet` for tenant/user/chain in the account period |
| `/complete` | `signatures:create` | one `transaction_signed` only when `operation=transaction`, keyed by the one-time ceremony ID; message completion has no transaction unit |

For transaction completion, the adapter reserves quota before the Vault call,
commits only after Vault succeeds and returns the same trusted stored
operation, and cancels a newly created reservation when Vault fails or the
operation differs. Message completion creates no `transaction_signed`
reservation. Commit also creates the durable Metronome outbox record. If Vault
succeeds but commit fails, the response fails and the reservation remains for
reconciliation/expiry; the service does not falsely cancel it. A duplicate
reservation still in progress returns 409 before a second Vault call; a
committed `/complete` retry returns `usage_already_committed` and cannot create
a second outbox record. Reservations also require the current time to fall
inside the stored account billing period; an expired/missing provider period
returns 402 until a verified Stripe webhook refreshes it.

### `POST /init`

Requires `wallets:write`. It starts registration and creates a chain key if the
wallet record does not exist. `/init` itself is not a committed usage unit.

```json
{"username":"alice","chain":"solana"}
```

```json
{
  "chain": "solana",
  "address": "<base58-public-key>",
  "pubKey": "<base58-public-key>",
  "ceremonyId": "<opaque-base64url-id>",
  "options": {"publicKey": {}}
}
```

Ethereum returns an EIP-55 `address` and no `pubKey` compatibility alias.
A completed wallet returns 409. Calling `/init` again for an incomplete
wallet starts another bounded registration ceremony.

### `POST /create`

Requires `wallets:write`. It completes exactly one registration ceremony and,
on success, commits the period's active-wallet unit if this is the first
successful action for that wallet.

```json
{
  "username": "alice",
  "chain": "solana",
  "ceremonyId": "<id returned by /init>",
  "credential": {
    "id": "<base64url>",
    "rawId": "<base64url>",
    "type": "public-key",
    "response": {
      "clientDataJSON": "<base64url>",
      "attestationObject": "<base64url>"
    }
  }
}
```

Success returns:

```json
{"chain":"solana","address":"<base58>","pubKey":"<base58>"}
```

The ceremony ID is consumed before credential validation. A cancelled or
malformed completion must restart at `/init`.

WebAuthn user verification is required; registration options request it and
the plugin validates the authenticator response.

The adapter requires non-empty `id`, `rawId`, `type: "public-key"`,
`response.clientDataJSON`, and `response.attestationObject` fields before it
forwards a registration credential to Vault.

### `POST /check`

Requires `wallets:read`. It looks up a completed tenant/user/chain wallet and,
on success, commits the period's active-wallet unit if needed:

```json
{"username":"alice","chain":"ethereum"}
```

```json
{"chain":"ethereum","address":"0x..."}
```

An unknown wallet returns 404; an initialized but unregistered wallet returns
409.

### `POST /signin`

Requires `signatures:create`. It begins WebAuthn authorization, binds the exact
signing request to the new ceremony, and on success commits the period's
active-wallet unit if needed:

```json
{
  "username": "alice",
  "chain": "solana",
  "operation": "transaction",
  "payload": "<standard-base64>"
}
```

`operation` is `transaction` (default) or `message`. `rawTx` remains a
legacy alias for `payload`. The decoded payload must be 1–131072 bytes with
the default `MAX_SIGNING_PAYLOAD_BYTES`.

Success returns the wallet identity, WebAuthn assertion options, and a new
`ceremonyId`. Ethereum rejects `transaction`; its only implemented
operation is `message`. When a Solana transaction reaches `/complete`,
malformed/truncated encodings and versioned formats are rejected as validation
errors rather than returning a signature; only legacy transactions are signed.

### `POST /complete`

Requires `signatures:create`. It reserves a signed-transaction unit, completes
the signing ceremony, and commits that unit only for `operation=transaction`
after Vault returns the signature and its stored operation matches the request.
Message completion uses the same one-time Vault ceremony protection but does
not consume a transaction unit:

```json
{
  "username": "alice",
  "chain": "solana",
  "operation": "transaction",
  "ceremonyId": "<id returned by /signin>",
  "credential": {
    "id": "<base64url>",
    "rawId": "<base64url>",
    "type": "public-key",
    "response": {
      "clientDataJSON": "<base64url>",
      "authenticatorData": "<base64url>",
      "signature": "<base64url>",
      "userHandle": null
    }
  }
}
```

`operation` defaults to `transaction` for backward compatibility. The client
must send `message` for a message ceremony. Vault recovers the operation and
payload from its stored ceremony, consumes the ceremony, rejects a requested
operation mismatch before signing, and returns the stored operation on success.
The adapter independently compares that returned value before exposing output
or committing transaction usage. The client cannot replace the stored
operation or payload during completion.

The adapter requires the common credential fields plus
`response.authenticatorData` and `response.signature` before forwarding an
assertion.

Output by operation:

| Chain/operation | Result |
| --- | --- |
| Solana transaction | `encodedTX`: standard-base64 serialized signed legacy transaction |
| Solana message | `signature`: standard-base64 64-byte Ed25519 signature |
| Ethereum message | `signature`: standard-base64 65-byte `R || S || V`; `V` is 0 or 1 and the digest uses EIP-191's personal-message prefix |
| Ethereum transaction | unsupported |

The ID is consumed before assertion validation and cannot be replayed.
The assertion must include WebAuthn user verification. The plugin persists the
updated authenticator signature counter. If a previously nonzero counter
regresses, it stores the credential's clone-warning/disabled state, returns a
rejection without signing, and rejects later authentication starts for that
wallet until a recovery path exists.

## Account console and billing routes

These routes exist only when the billing runtime is enabled. Console HTML forms
use `application/x-www-form-urlencoded`; authenticated mutations require the
session cookie, exact configured origin, and CSRF field.

| Method and path | Authorization | Purpose |
| --- | --- | --- |
| `GET /developer/login`, `POST /developer/login` | public form / developer credentials | Start a developer role session. |
| `POST /developer/logout` | developer session + CSRF | Delete the server-side session. |
| `GET /developer` | developer session | Show the session account, plan/status, current usage/limits/estimates, API-key metadata, and subscription actions. |
| `GET /developer/api/usage` | developer session | Return the current account's period and daily usage JSON. |
| `POST /developer/api-keys` | developer session + CSRF | Create a name/users/scopes-constrained key. Returns its plaintext exactly once. |
| `POST /developer/api-keys/:id/revoke` | developer session + CSRF | Revoke a key owned by the session account. |
| `POST /developer/billing/checkout` | developer session + CSRF | Redirect to hosted subscription Checkout; refuses a parallel subscription. |
| `POST /developer/billing/portal` | developer session + CSRF | Redirect an existing Stripe customer to the hosted Customer Portal. |
| `GET /admin/login`, `POST /admin/login` | public form / admin credentials | Start an admin role session. |
| `POST /admin/logout` | admin session + CSRF | Delete the server-side session. |
| `GET /admin` | admin session | Show aggregate usage/delivery state, account list/create form, and optional redacted account detail; email and raw Stripe/Metronome identifiers are omitted. |
| `GET /admin/api/analytics` | admin session | Return aggregate and per-account usage without account email fields. |
| `POST /admin/accounts` | admin session + CSRF | Atomically create a past-due account and its first developer. |
| `POST /admin/accounts/:id/metronome-verified` | admin session + CSRF | Atomically compare and record the submitted Stripe customer, Metronome ingest alias, local plan, and rate card; requires a configured Metronome runtime. |

There is no administrator bootstrap HTTP endpoint. The operator-only
`billing:bootstrap-admin` command upserts an account-independent admin from a
bounded regular password file, rejects password arguments/symlinks, and cannot
promote a developer. There is still no SSO, MFA, forced password change/reset,
or credential-recovery flow. Seeded developer/admin credentials exist only in
explicit development memory mode and are rejected in production.

### Stripe boundary

`POST /billing/stripe/webhook` is outside browser-session and bearer auth. It
accepts at most 1 MiB of raw `application/json`, verifies `Stripe-Signature`
against `STRIPE_WEBHOOK_SECRET`, and atomically claims event IDs. It processes
`checkout.session.completed` and subscription created/updated/deleted events,
retrieves current subscription state rather than trusting event order, and
requires exactly one subscription item whose current Stripe Price maps uniquely
to one local plan before updating account status, customer, subscription, plan,
and billing period.
Invalid signatures return 400; an in-progress duplicate returns 503 and other
processing failures return 500 so Stripe can retry.

The Stripe 22.4.0 client pins API version `2026-07-29.dahlia`. Checkout sends
one operator-configured base Price, an account/plan metadata binding, a stable
account/intent-version idempotency key, and
`spiral_safe_<eight-lowercase-letters>` as `integration_identifier`. It
intentionally sets neither `payment_method_types` nor `automatic_tax`.
Distinct Stripe Products/Prices and all tax decisions are operator inputs.

### Metronome outbox boundary

The service exporter posts arrays of 1–100 records to
`https://api.metronome.com/v1/ingest` (or the configured HTTPS endpoint). Each
record has a deterministic `transaction_id`, account ingest alias as
`customer_id` (using the attested alias snapshot once present),
`spiral_active_wallet` or `spiral_transaction_signed` event type, RFC3339
timestamp, and string-valued properties. Network errors, 429, and
5xx retry with bounded exponential backoff; other 4xx responses become
dead-letter records visible to administrators. Export runs asynchronously and
never delays a successful signing response.

Metronome ingest does not by itself prove or create a Stripe charge. The
operator must provision and test each customer's Metronome contract/rate-card
and Stripe provider mapping, then record the in-service attestation. The store
atomically compares and snapshots the submitted Stripe customer, Metronome
ingest alias, local plan, and Metronome rate card; changing any of them or
clearing a terminal subscription makes it non-current. No live Stripe charge is
part of the current verification record.

## Ceremony guarantees

Registration and authentication ceremonies:

- use a cryptographically random 32-byte ID encoded as unpadded base64url;
- expire two minutes after creation;
- are single-use;
- are typed as registration or authentication;
- bind the server-side WebAuthn session;
- require WebAuthn user verification;
- for signing, also bind chain, operation, and payload;
- are limited to eight pending entries per wallet.

The extension adds its own binding of the backend-issued ID to browser tab,
top-level frame, exact origin, username, chain, operation, and the same
two-minute lifetime. This non-key metadata is stored in trusted
`chrome.storage.session` so Manifest V3 worker suspension during a passkey
prompt does not lose the binding; it is removed on use or expiry. The plugin
remains authoritative if extension state is cleared or the worker is bypassed.

## Error contract

Non-success adapter responses have a stable JSON shape:

```json
{
  "error": {
    "code": "wallet_not_found",
    "message": "..."
  },
  "requestId": "..."
}
```

The same request ID is returned in `X-Request-ID`. A caller may provide an ID
matching `[A-Za-z0-9._-]{8,128}`; otherwise the service generates one.

Common statuses:

| Status | Typical code/reason |
| ---: | --- |
| 400 | invalid username, chain, operation, payload, ceremony ID, credential shape, or JSON |
| 401 | missing or invalid bearer token |
| 402 | inactive subscription (`payment_required`) or missing per-account usage-invoicing attestation (`billing_provisioning_required`) |
| 403 | browser origin outside the exact CORS allowlist, username not authorized, or API key missing the route scope |
| 404 | route or wallet not found |
| 409 | wallet already registered, registration incomplete, matching usage still reserved (`usage_in_progress`), or signed ceremony usage already committed (`usage_already_committed`) |
| 422 | Vault/WebAuthn rejected credential, operation, or ceremony |
| 429 | local per-tenant/IP rate limit or plan quota exceeded; request limiting sets `Retry-After` |
| 502 | Vault failure not safely attributable to the client |
| 503 | readiness failure |

Responses set `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`,
and a restrictive Content Security Policy. Wallet API responses use
`Referrer-Policy: no-referrer`; console/login/CSS responses use `same-origin`
so native same-origin form submissions retain an Origin boundary. Production
also sets HSTS.

## Limits and CORS

| Setting | Default | Scope |
| --- | ---: | --- |
| JSON body parser | 256 KiB | entire HTTP request |
| `MAX_SIGNING_PAYLOAD_BYTES` | 131072 | decoded signing payload |
| `RATE_LIMIT_WINDOW_MS` | 60000 | process-local window |
| `RATE_LIMIT_MAX` | 60 production / 300 dev | requests per tenant and client IP |
| `RATE_LIMIT_BUCKETS` | 10000 | memory bound for limiter keys |
| console form parser | 32 KiB | `/developer` and `/admin` URL-encoded forms |
| `CONSOLE_SESSION_TTL_MS` | 28800000 (8 hours) | stored role session and cookie |
| `BILLING_RESERVATION_TTL_MS` | 300000 (5 minutes); accepted range 60000–3600000 | ambiguous uncommitted usage reservation |
| Stripe webhook raw body | 1 MiB | signature-verified webhook only |
| `METRONOME_BATCH_SIZE` | 100, capped at 100 | asynchronous outbox claim/ingest batch |

`CORS_ALLOWED_ORIGINS` is required. Production accepts exact HTTPS and
`chrome-extension://` origins. Development additionally permits extension
origins so unpacked extension IDs do not have to be committed. Requests without
an `Origin` header, such as server-side SDK calls, are allowed.

`CONSOLE_ORIGIN` is a separate exact HTTPS origin in production and is added to
CORS for console-adjacent requests. Native login and mutating forms require
that exact `Origin`; `Origin: null` is rejected.

The process-local limiter is a protective backstop, not a global quota across
replicas. A production ingress needs a centralized limit.

## Direct Vault plugin interface

The adapter calls a secrets engine mounted at `spiral-safe/`:

| Vault path | Logical operations | Fields |
| --- | --- | --- |
| `spiral-safe/users` | create/update/read | identity, optional `ceremonyId` and registration `credential` |
| `spiral-safe/check` | create/update | identity |
| `spiral-safe/auth` | create/update | identity; begin uses `operation` and `payload`; complete uses requested `operation`, `ceremonyId`, and assertion `credential`, while Vault enforces its stored operation |

Both create and update are deliberately registered for write-style paths so
the callback is reachable through Vault framework routing regardless of the
logical operation selected by the client. The software-authenticator lifecycle
test exercises registration, check, authentication, and completion through
`Backend.HandleRequest` with update operations instead of calling only the
handlers directly; a separate framework regression exercises the create route.

Here, identity is `tenant`, `username`, and `chain`. Direct Vault access
bypasses the adapter's tenant derivation and validation boundary and should be
reserved for controlled operator/testing use. The Kubernetes service policy is
scoped to the Spiral Safe engine and does not give the adapter administrative
Vault privileges.

## Client contracts

The SDK constructor requires `{baseUrl, apiToken, chain?}`; only loopback
development may use HTTP. It exposes `init`, `create`, `check`, `signin`,
and `complete`, and throws `SpiralSafeAPIError` with status, code, and
request ID.

The extension implements WebAuthn binary conversion and the full Solana Wallet
Standard flow. Its custom `spiralSafe:` feature exposes backend registration
and non-secret configuration inspection. `config.get` omits the bearer token
and returns only the origin portion of backend/RPC URLs so path-embedded API
keys cannot leak to a page. It supports Solana only; use the SDK or service demo
for the Ethereum example.

The `services/public` demo is a local acceptance client, not a production
application. Its scripts and styles are bundled without CDN dependencies,
errors are inserted as text, and its static server emits CSP, framing,
referrer, MIME-sniffing, opener, and WebAuthn permissions headers. The demo
accepts only a same-host, plain-HTTP loopback API origin and is deleted by the
production Kubernetes overlay.
