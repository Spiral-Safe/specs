# HTTP and Vault API contract

This document describes the current workspace implementation. The HTTP API is
not versioned and has no generated OpenAPI schema yet; treat it as a
development contract.

## Service origin and authentication

The HTTP adapter listens on port 3000 by default. Browser and SDK clients talk
only to this adapter, never directly to Vault.

All wallet routes require:

```http
Authorization: Bearer <tenant-and-username-scoped-api-token>
Content-Type: application/json
```

The adapter hashes the supplied token with SHA-256 and maps it to a principal
configured by the operator. It injects that principal's tenant into every
Vault request; a body field named `tenant` has no effect. It also rejects a
username outside the principal's allowlist with `403 user_forbidden` before
making any Vault request.

The preferred production configuration is `API_TOKEN_HASHES`, a JSON object
whose keys are 64-character SHA-256 token digests and whose values have this
shape:

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
| `GET /readyz` | `200 {"status":"ready"}` | The service can obtain its Vault credential and Vault reports an active or acceptable standby state. Returns 503 otherwise. |

No bearer token is required for probes.

### `POST /init`

Starts registration and creates a chain key if the wallet record does not
exist.

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

Completes exactly one registration ceremony.

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

Looks up a completed tenant/user/chain wallet:

```json
{"username":"alice","chain":"ethereum"}
```

```json
{"chain":"ethereum","address":"0x..."}
```

An unknown wallet returns 404; an initialized but unregistered wallet returns
409.

### `POST /signin`

Begins WebAuthn authorization and binds the exact signing request to the new
ceremony:

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

Completes the signing ceremony:

```json
{
  "username": "alice",
  "chain": "solana",
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

The operation and payload are recovered from server-side ceremony state; the
client cannot replace them during completion.

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
| 403 | browser origin outside the exact CORS allowlist, or token not authorized for the requested username |
| 404 | route or wallet not found |
| 409 | wallet already registered or registration incomplete |
| 422 | Vault/WebAuthn rejected credential, operation, or ceremony |
| 429 | local per-tenant/IP rate limit exceeded; see `Retry-After` |
| 502 | Vault failure not safely attributable to the client |
| 503 | readiness failure |

Responses set `Cache-Control: no-store`, `X-Content-Type-Options: nosniff`,
and `Referrer-Policy: no-referrer`.

## Limits and CORS

| Setting | Default | Scope |
| --- | ---: | --- |
| JSON body parser | 256 KiB | entire HTTP request |
| `MAX_SIGNING_PAYLOAD_BYTES` | 131072 | decoded signing payload |
| `RATE_LIMIT_WINDOW_MS` | 60000 | process-local window |
| `RATE_LIMIT_MAX` | 60 production / 300 dev | requests per tenant and client IP |
| `RATE_LIMIT_BUCKETS` | 10000 | memory bound for limiter keys |

`CORS_ALLOWED_ORIGINS` is required. Production accepts exact HTTPS and
`chrome-extension://` origins. Development additionally permits extension
origins so unpacked extension IDs do not have to be committed. Requests without
an `Origin` header, such as server-side SDK calls, are allowed.

The process-local limiter is a protective backstop, not a global quota across
replicas. A production ingress needs a centralized limit.

## Direct Vault plugin interface

The adapter calls a secrets engine mounted at `spiral-safe/`:

| Vault path | Logical operations | Fields |
| --- | --- | --- |
| `spiral-safe/users` | create/update/read | identity, optional `ceremonyId` and registration `credential` |
| `spiral-safe/check` | create/update | identity |
| `spiral-safe/auth` | create/update | identity; begin uses `operation` and `payload`; complete uses `ceremonyId` and assertion `credential` |

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
