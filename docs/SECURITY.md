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
| HTTP authorization | Bearer tokens are hashed and mapped server-side to `{tenant, users:[...]}` principals. Production requires a non-empty username allowlist; an unauthorized username receives 403 before Vault. Vault tokens are not client inputs, and production rejects every static `VAULT_TOKEN`. |
| Request controls | Exact CORS origins, identifier/chain/operation/base64 validation, decoded-payload bound, request IDs, stable errors, timeouts, security headers, and bounded process-local rate limiting. |
| Local containment | Compose publishes only on `127.0.0.1`; service/client containers run as an unprivileged user. The self-contained demo uses text-only error insertion, security headers, and a same-host loopback API restriction. Vault dev mode and known credentials remain explicitly development-only. |
| Vault identity | Kubernetes deployments exchange a service-account JWT for a short-lived, engine-scoped Vault token and refresh it before lease expiry. |
| Tenant isolation | Vault storage keys are derived from server-authenticated tenant, username, and chain—not a caller field or Vault token string. |
| Ceremony isolation | Random two-minute, single-use IDs; typed registration/authentication state; up to eight pending ceremonies; signing payload bound server-side; extension cross-tab/origin binding. |
| WebAuthn | User verification is required; credential validation and sign-counter updates run through Vault framework routes; a nonzero counter regression is persisted as a disabled/clone-warned credential before signing; software-authenticator lifecycle and replay regressions exist. |
| Chain separation | Solana and Ethereum use a `ChainSigner` interface rather than a generic fallback. The Solana boundary converts upstream parser panics into validation errors and rejects non-legacy formats; Ethereum uses EIP-191 and restricts recovery IDs to 0/1. |
| Vault storage | Private data stays behind Vault's storage barrier; entries request seal wrapping where the configured seal supports it. |
| Kubernetes | Restricted pod contexts, probes/resources/PDBs, ingress NetworkPolicies, service egress restriction, TLS Vault listener, retained Raft/audit PVCs, KMS auto-unseal inputs, anti-affinity, and controlled peer services. The production overlay excludes the local demo/client resources. |
| Dependencies | On 2026-08-30, final full npm audits for services, extension, wallet-adapter, and SDK reported zero known advisories. Final-source `govulncheck` reported no vulnerabilities, zero affected and zero in imported packages; one required module contains vulnerable symbols that the application does not call. |
| Enclave adapter | Nitriding is removed from the workspace build path; Veil source is pinned and verified, debug/console/terminate-all behavior is removed, and license material is embedded. |

## Threat model boundary

The present design assumes:

- the Vault control plane, active plugin process, seal/KMS policy, and cluster
  administrators are trusted;
- API tokens are high entropy, mapped to the correct tenant and username
  allowlist, issued only to an authorized application, and transported over
  TLS;
- the configured WebAuthn RP ID/origins and extension site allowlist are exact;
- the authenticator and browser correctly implement WebAuthn;
- a user recognizes and intends the dApp request despite the current lack of
  trusted transaction display;
- the configured Solana RPC is trustworthy enough for broadcast/availability.

It does not protect against a malicious Vault root/operator, compromised plugin
binary, compromised allowlisted dApp, malicious browser/extension update,
stolen username-authorized token plus coerced/ambiguous WebAuthn approval, or a
broken production KMS/backup process.

Vault encryption is not the same as a non-exportable HSM key. The plugin must
load private-key bytes in memory to sign.

## Remaining critical and high-priority work

### End-user identity and authorization

The production bearer mapping now establishes both a tenant and an explicit
username allowlist, so a token cannot select an arbitrary username and rejected
names do not reach Vault. This is still static, operator-managed application
authorization rather than proof of the current human user. There is no
OIDC/session exchange, role model, wallet ownership administration, token
issuance/revocation API, end-user recovery, or approval delegation. CORS is a
browser containment control, not authentication.

Production needs:

- identity-aware edge authentication and short-lived application sessions;
- user-bound authorization policy beyond the static application-token
  allowlist;
- high-entropy token issuance, rotation, revocation, and incident response;
- centralized/global rate limiting and abuse controls;
- least-privilege operator roles separated from Vault root/recovery authority.

`API_TOKEN_HASHES` prevents storing cleartext in the adapter configuration but
does not make weak tokens safe. Store the hash map in an encrypted secret
manager; Kubernetes Secret base64 encoding alone is not encryption.

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

The Veil adapter is build scaffolding, not an attested release:

- Nitro Enclaves have no persistent storage, so the current `/vault/data`
  directory disappears with the enclave;
- no attestation-aware client initializes Vault or delivers recovery material
  without exposing it to the EC2 parent;
- no PCR/application-hash-bound KMS or workload-identity policy is implemented;
- plugin registration still needs an authorized post-initialization operator;
- nonce-bound attestation and altered-image rejection have not run on EC2;
- the HTTP adapter, tenant authorization, demos, and load suite are outside the
  EIF;
- `veil-proxy` needs root, TUN/NAT, and VSOCK privileges on the parent.

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
2. end-user session identity, authorization administration, token lifecycle,
   and recovery are implemented beyond the static tenant/username mapping;
3. transaction decoding/policy/trusted display are complete for each advertised
   chain operation;
4. real browser/authenticator, concurrency, failure, and adversarial API suites
   pass;
5. target-cloud TLS, KMS, network, Raft, audit, monitoring, backup, and restore
   drills pass;
6. Veil/Nitro persistence, initialization, attestation, and KMS binding pass on
   enclave-enabled EC2 if that mode is offered;
7. dependency, container, license, SBOM, provenance, and secret-scanning gates
   are enforced;
8. an independent security review is complete and its findings are resolved;
9. incident response, key/token/certificate rotation, disaster recovery, and
   operator ownership are documented and rehearsed.

Until then, use disposable devnet keys and test messages only.
