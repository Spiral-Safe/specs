# Security posture

## Bottom line

The workspace removes the original browser-held custody key, replaces the
public/root-token HTTP path, binds concurrent WebAuthn ceremonies, adds a
chain-specific signer, and supplies a production-intent Vault Raft baseline.
Those are meaningful architecture changes.

Spiral Safe is still an unaudited development system. Do not custody production
value until the remaining identity, transaction-policy, recovery, deployment,
hardware, and operational controls in this document are implemented and
independently reviewed.

## Controls now present

| Area | Current control |
| --- | --- |
| Browser key custody | Extension creates, receives, stores, and transmits no wallet private key. Vault/plugin owns the signing key. |
| Extension bridge | Request IDs; same-window/origin checks; top-level frame restriction; browser-reported tab/frame/origin binding; exact site allowlist; 2 MB bridge cap. |
| Extension credentials/state | API token and ephemeral ceremony bindings are held in trusted-context `chrome.storage.session`; pages can read non-secret settings but not the token/bindings. Page-facing configuration also strips backend/RPC URL paths that may contain API keys. Bindings survive worker suspension but not the browser session. Production backend/RPC URLs require HTTPS. |
| Wallet functions | Advertised connect/events, legacy transaction, batch, sign-and-send, message, and SIWS paths call the backend or configured RPC and have routing tests. |
| Wallet API authorization | Production account mode stores only HMAC-SHA-256 API-key digests and derives account, tenant, scopes, and non-empty username allowlist server-side; unauthorized scope/user receives 403 before Vault. Production billing rejects static API mappings. Billing-disabled production retains the static `{tenant, users:[...]}` digest map. Vault tokens are not client inputs, and production rejects every static `VAULT_TOKEN`. |
| Console authentication | Developer/admin roles and account bindings come from stored users and random server-side sessions. Passwords use scrypt; cookies are HttpOnly, SameSite=Strict, and Secure in production; state changes require exact Origin plus session-derived CSRF. Login work is equalized for unknown users and bounded per process/IP/email. |
| Account and quota state | PostgreSQL production storage has accounts, plans, one-time-reveal/revocable API keys, sessions, reservation/commit state, idempotency constraints, usage, outbox, and webhook claims. Readiness includes the billing store. Memory/demo mode is rejected in production. |
| Metering | Transaction completion reserves before signing, cancels a new reservation on failure or operation mismatch, and commits usage plus an outbox row only after Vault returns the same trusted stored operation. Message signatures consume no transaction unit. Period/wallet and transaction-ceremony idempotency plus reserved quota reduce duplicate charging and concurrent oversubscription. |
| Stripe/Metronome | Hosted Checkout/Portal keep payment collection off the service. Raw Stripe webhooks are signature-verified and claimed idempotently. Metronome export is asynchronous, uses stable transaction IDs, retries transient failures, and dead-letters permanent 4xx. Production requires both a global integration flag and per-account operator attestation before signing. |
| Request controls | Exact CORS origins, identifier/chain/operation/base64 validation, decoded-payload bound, request IDs, stable errors, timeouts, security headers, and bounded process-local rate limiting. |
| Local containment | Compose publishes only on `127.0.0.1`; service/client containers run as an unprivileged user. The self-contained demo uses text-only error insertion, security headers, and a same-host loopback API restriction. Vault dev mode and known credentials remain explicitly development-only. |
| Vault identity | Kubernetes deployments exchange a service-account JWT for a short-lived, engine-scoped Vault token and refresh it before lease expiry. |
| Tenant isolation | Vault storage keys are derived from server-authenticated tenant, username, and chain—not a caller field or Vault token string. |
| Ceremony isolation | Random two-minute, single-use IDs; typed registration/authentication state; up to eight pending ceremonies; signing payload bound server-side; extension cross-tab/origin binding. |
| WebAuthn | User verification is required; credential validation and sign-counter updates run through Vault framework routes; a nonzero counter regression is persisted as a disabled/clone-warned credential before signing; software-authenticator lifecycle and replay regressions exist. |
| Chain separation | Solana and Ethereum use a `ChainSigner` interface rather than a generic fallback. The Solana boundary converts upstream parser panics into validation errors and rejects non-legacy formats; Ethereum uses EIP-191 and restricts recovery IDs to 0/1. |
| Vault storage | Private data stays behind Vault's storage barrier; entries request seal wrapping where the configured seal supports it. |
| Kubernetes | Restricted pod contexts, probes/resources/PDBs, ingress NetworkPolicies, fail-closed Vault/DNS/labeled-billing egress, TLS Vault listener, retained Raft/audit PVCs, KMS auto-unseal inputs, anti-affinity, a billing migration init container, and controlled peer services. The production overlay excludes the local demo/client resources. |
| Dependencies | Direct versions, Node/Go/Vault images, and Veil source are pinned. Scanner/test evidence is dated and listed only in `VERIFICATION.md`; it does not cover provider configuration, container runtime behavior, or audit findings. |
| Enclave adapter | Nitriding is removed; Veil source is pinned; one strict manifest configures the same image as bootstrap or joiner; verification precedes the external enrollment hook; membership checks require exact voter/Autopilot health. The missing runtime delivery/storage/routing boundaries remain fail-closed. |

## Threat model boundary

The present design assumes:

- the Vault control plane, active plugin process, seal/KMS policy, and cluster
  administrators are trusted;
- API tokens are high entropy, mapped to the correct tenant and username
  allowlist/scopes, issued only to an authorized application, and transported
  over TLS;
- PostgreSQL, its TLS/backup operators, console administrators, and the
  API-key/session peppers are trusted for correct account authorization;
- Stripe webhook configuration and each externally provisioned
  Metronome-to-Stripe customer/contract mapping are correct, monitored, and
  reconciled with the local operator attestation;
- the configured WebAuthn RP ID/origins and extension site allowlist are exact;
- the authenticator and browser correctly implement WebAuthn;
- a user recognizes and intends the dApp request despite the current lack of
  trusted transaction display;
- the configured Solana RPC is trustworthy enough for broadcast/availability.

It does not protect against a malicious Vault root/operator, compromised plugin
binary, compromised allowlisted dApp, malicious browser/extension update,
stolen username-authorized token plus coerced/ambiguous WebAuthn approval, or a
broken production KMS/backup process. It also does not protect against a
malicious billing administrator, compromised PostgreSQL/operator credentials,
provider-account takeover, incorrect tax configuration, or an attested mapping
that has drifted from Metronome/Stripe reality.

Vault encryption is not the same as a non-exportable HSM key. The plugin must
load private-key bytes in memory to sign.

## Remaining critical and high-priority work

### End-user identity and authorization

Production account mode now has one-time-reveal/revocable API keys with stored
account, tenant, scopes, and username allowlists, plus password-authenticated
developer/admin console roles. A key cannot select an arbitrary username and
rejected names do not reach Vault. A developer can create/revoke keys only for
the session account; an admin can create accounts/developers and inspect usage.

This remains product-local application authorization rather than federated
proof of the current human. A password-file-only operator CLI can upsert an
account-independent admin and cannot promote a developer, but there is no
SSO/OIDC, MFA, email verification, forced password change/reset, wallet
ownership recovery, approval delegation, or service-account lifecycle API.
Login throttling and wallet request limiting are process-local. CORS is a
browser containment control, not authentication.

Production still needs:

- identity-aware edge authentication/SSO, MFA, secure operationalization of the
  admin bootstrap, forced credential rotation, and recovery;
- user-bound wallet ownership and approval policy beyond an API-key allowlist;
- API-key/session rotation, expiry, revocation, audit, and incident response;
- centralized/global rate limiting and abuse controls;
- least-privilege operator roles separated from Vault root/recovery authority.

In billing-disabled deployments, `API_TOKEN_HASHES` prevents storing cleartext
in the adapter configuration but does not make weak tokens safe. Store that
legacy hash map in an encrypted secret manager; Kubernetes Secret base64
encoding alone is not encryption. Production account mode rejects the map and
uses peppered database-backed API keys instead.

Production account mode uses `API_KEY_PEPPER` both for key verification and the
domain-separated opaque active-wallet identifier. The implementation has no
dual-pepper rotation window: changing it invalidates all existing API keys and
changes the wallet metering identifier, which can double-count a wallet inside
the same billing period. Rotation therefore needs coordinated key reissue,
period/reconciliation policy, and an incident runbook. Rotating
`CONSOLE_SESSION_SECRET` invalidates all console sessions and CSRF tokens; that
is safer operationally but still needs planned user communication.

### Human-readable signing intent

WebAuthn currently approves an opaque challenge. It does not show the trusted
chain, recipient, amount, program calls, nonce, fees, or decoded message.
An allowlisted but compromised dApp can choose the bytes that the user is asked
to authorize.

Before real assets:

- parse every supported transaction format;
- simulate and apply tenant/wallet policy;
- reject unexpected signers, programs/contracts, destinations, amounts, fees,
  nonces, and expired blockhashes;
- bind a canonical intent hash to the WebAuthn challenge and trusted display;
- produce an auditable approval record without logging sensitive credential
  material.

The service demo's Solana transaction is illustrative, not a policy engine.

### WebAuthn assurance and recovery

The automated lifecycle uses a software P-256 authenticator with `none`
attestation. Real Touch ID/Windows Hello/passkey/security-key testing remains
mandatory. There is no attestation allowlist, authenticator assurance policy,
multi-credential management UI, lost-authenticator recovery, step-up policy,
or administrative credential revocation.

The plugin requires WebAuthn user verification, so a user-presence-only
assertion cannot release a signature. It also persists a clone warning when a
previously nonzero authenticator signature counter regresses, rejects that
request before signing, and blocks later authentication starts. This is a safe
failure mode, but without credential replacement/recovery it can permanently
strand the wallet; multi-credential and counter-zero authenticators need an
explicit product policy.

RP ID and origin changes can strand credentials. Define domain migration,
credential recovery, backup-authenticator, and account-recovery rules before
launch.

### API and ceremony durability

One-time consumption is persisted before validating attacker-controlled
credentials, preventing replay but making completion non-retryable. There is no
idempotency/result-recovery protocol if a valid signature response is lost.
The completion request repeats `operation`, but Vault compares it with the
stored ceremony operation before signing and the adapter checks Vault's returned
operation before exposing output. Only transaction completion enters the
transaction-usage reservation path; messages rely on Vault's ceremony replay
protection.

The plugin uses process synchronization plus Vault logical storage. Run
multi-replica failover and contention tests to validate behavior during leader
changes, retries, partial writes, plugin restart, clock skew, and the maximum
pending-ceremony limit. Add versioned schemas and an explicit migration path.

`/init` creates a key-bearing but unusable record before attestation completes.
Add tenant quotas, expiry/garbage collection for abandoned registrations, and
an audited retry/recovery policy so an authorized abusive client cannot grow
storage indefinitely.

The HTTP rate limiter is process-local. Body-parser oversize/error behavior,
proxy IP trust, request timeouts, and upstream retry semantics need adversarial
tests behind the real ingress.

### Account, billing, and provider integrity

Production billing fails startup unless PostgreSQL, plans, Stripe, Metronome,
and the global invoicing verification flag are configured. An active account
still receives `402 billing_provisioning_required` until an administrator
records that account's exact Metronome-to-Stripe mapping. This prevents an
obvious unprovisioned account from generating unbilled signing traffic, but the
record is only an operator assertion. The service does not query Metronome to
prove that the customer alias, contract, rate card, billing-provider
configuration, product mapping, and Stripe customer still agree.

The local attestation atomically compares and snapshots the submitted Stripe
customer, Metronome ingest alias, local plan, and Metronome rate card. A
customer-alias/customer/plan/rate-card change or terminal subscription makes it
non-current and restores the 402 gate. That protects local state and closes the
admin-form/webhook race; it does not detect out-of-band provider drift.

Before paid use, automate reconciliation and alerts for mapping drift,
subscription status, pending/dead outbox records, provider delivery, invoice
finalization/collection, and tax failures. Complete a real sandbox invoice
before production. No live Stripe charge or Metronome-created invoice is part
of current evidence.

Checkout uses a stable account/intent-version idempotency key and refuses an
account with an existing subscription. An expired Checkout intent needs a
reviewed recovery procedure before incrementing the version; otherwise an
operator can create a parallel session. Webhooks verify the raw signature,
persist event claims, and fetch current subscription state, but key permissions,
endpoint routing, replay windows, event retention, provider outages, and
account-takeover behavior still need live tests. Checkout intentionally omits
automatic tax and payment-method selection. Tax registrations, product codes,
customer-location evidence, provider choice, and failure handling are launch
gates rather than guessed defaults.

Usage reservation is intentionally fail-safe toward customers. If Vault
succeeds but the PostgreSQL commit fails, the service withholds the result and
leaves the reservation instead of falsely cancelling it. A process crash in
that window leaves ambiguous state; the stale reservation expires and is not
charged. This can underbill. Never convert stale reservations into charges
without a separate audit-backed reconciliation process. A missing or expired
stored billing period also fails closed until Stripe refreshes it. Conversely,
monitor exporter lag/dead letters because signing does not wait for Metronome
and can continue while usage invoicing is delayed.

The PostgreSQL migration uses an advisory lock and idempotent DDL, but the
checked-in Kubernetes init container shares `DATABASE_URL` with the runtime.
Split the schema-owner/migration role from the service role before production.
Require database certificate verification, least privilege, encryption, HA,
PITR, tested restores, session cleanup, webhook/outbox retention, and capacity
limits. The memory store and seeded plans/users are development fixtures only.

API-key plaintext appears once on the create response and must never enter
recordings, traces, logs, analytics, tickets, or browser storage beyond the
intended client configuration. Console sessions are password-only and lack
MFA/SSO/reset; the login limiter is process-local and needs an ingress/global
control. The annotated recorder deliberately fills but does not submit the key
form and uses fake credentials.

### Chain capability

Solana support is limited to the legacy transaction format in the advertised
Wallet Standard surface. Versioned transactions are explicitly rejected, as
are malformed encodings and transactions requiring another signer; regression
tests cover truncated bytes that previously could panic the upstream parser.
Implement and test versioned decoding/signing before advertising it. Raw
message signatures require the application to define its own domain separation;
SIWS does so for sign-in.

Ethereum is an extensibility demonstration only: secp256k1 key/address and
EIP-191 personal-message signing. It has no transaction signing, EIP-712 typed
data, EIP-155 chain binding, nonce/fee management, simulation, or broadcast.

Adding a chain requires more than a `ChainSigner`: specify address/key rules,
canonical transaction parsing, domain separation, intent display, replay rules,
RPC trust, failure semantics, and independent test vectors.

### Vault and Raft operations

The production manifests are a baseline, not a complete service:

- Vault initialization and recovery-key custody are operator procedures;
- AWS KMS IAM/workload identity, TLS material, ingress, and StorageClass are
  external inputs;
- Raft snapshots, encrypted off-cluster retention, restore drills, quorum-loss
  recovery, and key rotation are not automated;
- the file audit device needs immutable centralized export, free-space alerts,
  retention, and failover validation;
- Vault egress remains cluster/provider-specific and must be restricted to
  exact Kubernetes/KMS/private endpoints;
- the service NetworkPolicy allows only Vault, DNS, and labeled in-cluster
  database/HTTPS egress targets; a target-specific PostgreSQL and
  Stripe/Metronome gateway/CIDR/FQDN policy is not supplied;
- production billing plan/URL values and all database/pepper/session/provider
  secrets are external, and the checked-in false invoicing flag intentionally
  prevents a zero-input launch;
- metrics collection, SLOs, dashboards, paging, and incident runbooks are not
  included;
- the shared Vault TLS Secret and workload identities need rotation and
  compromise procedures.

Integrated Raft makes multiple controlled nodes possible. It does not make a
permissionless network safe. A node that can join must be trusted to hold
replicated custody data. Independent operators require governance, certificate
issuance/revocation, attestation, quorum rules, monitoring, and removal before
they receive membership.

See Vault's [integrated-storage security model](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)
and [Autopilot documentation](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage/autopilot).

### Veil and Nitro Enclaves

The Veil path now has a strict same-image bootstrap/join manifest, renderer,
verification-before-enrollment sequence, and voter/Autopilot membership check.
Those controls constrain configuration and admission; they do not provide the
missing infrastructure needed for a real Nitro quorum:

- Veil's attestation listener on 8443 uses a shared `.invalid` sentinel in the
  measured image. The official verifier intentionally does not trust ordinary
  CA/hostname validation; it hashes the presented leaf certificate and requires
  that hash in the fresh nonce-bound Nitro attestation before comparing PCRs.
  Therefore the per-node `veilAddress` is only an operator-routable endpoint.
  Generic PKI clients must never use 8443, and Vault API traffic stays on its
  separate operator-CA mTLS endpoint;
- every EC2 parent/enclave pair reuses Veil's `10.0.0.1/10.0.0.2` TUN, so an
  external private per-node L4 overlay/forwarder must map unique advertised API
  and Raft addresses. Veil does not supply or authenticate that cross-host
  route;
- Nitro Enclaves have no persistent storage. The required
  `durableStorageBridge` is a manifest declaration, not code. A production
  voter needs encrypted, rollback-protected, fenced, per-node storage with
  correct ordering/fsync/crash and tested snapshot/restore semantics;
- Nitro CLI cannot inject the short-lived manifest or TLS/KMS material after
  launch. The external enrollment hook returns only a reference; the
  attestation-bound receiver/delivery service and KMS identity path are absent;
- no real PCR/nonce/TLS verification, KMS-bound auto-unseal, initialization,
  plugin registration, peer convergence, failure tolerance, altered-image
  rejection, or lifecycle has run on enclave-enabled EC2;
- the HTTP adapter, PostgreSQL/billing path, demos, and load suite remain
  outside the EIF; and `veil-proxy` needs root, TUN/NAT, and VSOCK privileges
  on each parent.

Do not admit a candidate as a voter until all three external boundaries—private
cross-host L4, rollback-protected durable storage, and attestation-bound
manifest/TLS/shared-KMS delivery—exist and have passed failure tests. The local
same-image simulation and mocked enrollment/membership test are not a live
quorum or hardware-attestation proof.

Veil is GPL-3.0-only. Preserve its exact source revision, license, reproducible
build instructions, notices, and required corresponding source/source offer.
Obtain legal review for distribution and compatibility with the combined image.
Vault's BUSL terms also require review.

### Supply chain and releases

Zero known npm advisories is a dated scanner result, not a guarantee. Keep the
narrow overrides covered by tests and remove them when upstream packages carry
the fixes. The service and client builder/runtime stages now pin the same Node
22 image digest, but there is still no organization-wide CI, SBOM, image
signing, provenance, secret scanning, or automated dependency gate.

The Go scan is path-sensitive and the npm result is advisory-database based;
neither covers container operating systems, Vault itself, Veil, browser
supply-chain compromise, or license risk. Enforce recurring `govulncheck`,
container/SBOM scanners, and release-time verification for every pinned image
and source revision in CI.

Pin release inputs, generate SBOMs, sign packages/images/EIF metadata, protect
branches, run tests in CI, record expected enclave measurements, and establish
an upgrade/rollback policy.

## Data migration warning

The old plugin used `<Vault client token>/<username>` storage keys and a
single Solana-shaped record. The new path is tenant/user/chain scoped and is not
backward-readable. Do not copy or reinterpret production records ad hoc.
Design an authenticated, audited migration with explicit source/destination
identity, key/address verification, rollback, and deletion policy. For
development-only data, re-registering is safer.

## Production release gates

Do not label the system production-ready until all are true:

1. threat model, custody promise, and trusted-computing base are approved;
2. SSO/MFA, secure administrator-bootstrap operations, forced password change,
   password/session/API-key lifecycle, wallet ownership, delegated approval,
   and recovery are complete;
3. PostgreSQL privilege/backup/recovery controls and live
   Stripe/Metronome/tax provisioning, per-account mapping reconciliation,
   sandbox invoice, collection, and failure alerts pass;
4. transaction decoding/policy/trusted display are complete for each advertised
   chain operation;
5. real browser/authenticator, concurrency, failure, and adversarial API suites
   pass;
6. target-cloud TLS, KMS, network, Raft, audit, monitoring, backup, and restore
   drills pass;
7. Veil/Nitro cross-host L4, rollback-protected durable storage, and
   attestation-bound manifest/TLS/KMS delivery exist, then a real quorum,
   initialization, attestation, failure, removal, and recovery matrix passes on
   enclave-enabled EC2 if that mode is offered;
8. dependency, container, license, SBOM, provenance, and secret-scanning gates
   are enforced;
9. an independent security review is complete and its findings are resolved;
10. incident response, key/token/certificate/provider rotation, disaster
   recovery, billing reconciliation, and
   operator ownership are documented and rehearsed.

Until then, use disposable devnet keys and test messages only.
