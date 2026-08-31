# Spiral Safe system documentation

This repository is the canonical map of the eight repositories in the
[Spiral Safe organization](https://github.com/orgs/Spiral-Safe/repositories).
It describes the implementation in this workspace, including changes that are
not part of the remote default branches until they are reviewed and committed.

Spiral Safe is currently a development-stage, WebAuthn-authorized signing and
account platform:

- a browser extension exposes a real Solana Wallet Standard provider;
- the extension and SDK call an HTTP service using account-scoped API keys with
  explicit wallet scopes and username allowlists;
- a Vault secrets-engine plugin owns the wallet keys and WebAuthn credentials;
- Solana legacy transactions/messages and Ethereum EIP-191 messages have
  chain-specific signers;
- PostgreSQL stores accounts, hashed API keys, console sessions, plan/quota
  state, committed usage, and an asynchronous Metronome outbox;
- developer/admin dashboards expose account operations, hosted Stripe Checkout
  and Customer Portal entry points, usage, and delivery audit state;
- local Compose, bounded endpoint load tests, a Skaffold/Kubernetes baseline,
  annotated Playwright walkthroughs, and an AWS Nitro Enclaves adapter based on
  Veil are included.

This is not an audited custody or billing product. The production Kubernetes,
Stripe/Metronome, and Veil paths still require deployment-specific identity,
TLS, KMS, database, egress, tax, backup, monitoring, recovery, and real-hardware
verification. No live Stripe charge or physical-authenticator proof is claimed.

## Start here

- [Architecture, flows, and trust boundaries](docs/ARCHITECTURE.md)
- [HTTP and Vault API contract](docs/API.md)
- [Repository and branch inventory](docs/REPOSITORY-INVENTORY.md)
- [Local, extension, load, Kubernetes, and Veil runbooks](docs/RUNBOOK.md)
- [Security posture and remaining release blockers](docs/SECURITY.md)
- [Verification record](docs/VERIFICATION.md)

The shortest onboarding path is:

1. read [ARCHITECTURE.md](docs/ARCHITECTURE.md) and
   [SECURITY.md](docs/SECURITY.md);
2. start the disposable Compose stack from `services`;
3. complete the browser demo and extension demo runbooks;
4. run the service, SDK, adapter, extension, and load-test suites;
5. render—but do not blindly deploy—the production Kubernetes overlay;
6. treat the Veil/Nitro path as a separate, hardware-dependent security
   project.

## Capability boundary

Implemented and automated in source:

- tenant/user/chain-isolated Vault records;
- one-time, expiring WebAuthn ceremony IDs;
- full registration/authentication with a cryptographic software authenticator;
- Solana legacy transaction and Ed25519 message signing;
- Ethereum address derivation and EIP-191 message signing;
- extension-side Wallet Standard routing without browser-held custody keys;
- authenticated HTTP routes, validation, rate limiting, health probes, and
  username/scope authorization before Vault plus Kubernetes Vault
  authentication;
- one-time-reveal, peppered API-key hashes; developer/admin sessions; PostgreSQL
  migrations; subscription/quota gates; and reserve/commit/cancel usage
  accounting with a durable Metronome outbox;
- a password-file-only operator command to create or rotate the first
  account-independent production administrator;
- hosted subscription Checkout and Customer Portal helpers, signature-verified
  idempotent Stripe webhooks, and per-account Metronome-to-Stripe provisioning
  attestation;
- required WebAuthn user verification and persistent disablement after a
  nonzero authenticator-counter regression;
- a self-contained, loopback-constrained local demo that is removed from the
  production overlay;
- developer/admin console pages and four annotated fixture-mode Playwright
  recordings;
- three-node Vault integrated-Raft manifests and a same-image Veil
  bootstrap/join admission scaffold.

Still requiring real environments or additional product work:

- platform and roaming-authenticator coverage on supported browsers;
- Ethereum transaction signing;
- production SSO/OIDC, forced password rotation/reset and MFA lifecycle, wallet
  recovery, and delegated approval administration beyond the current operator
  bootstrap, console roles, and API-key create/revoke flow;
- live Stripe Checkout/webhook/collection and Metronome rating/invoicing proof,
  including independently verified per-account provider mappings and tax
  configuration;
- transaction decoding, policy/simulation, and human-readable intent binding;
- a cloud-tested Raft backup/restore, KMS, monitoring, and disaster-recovery
  program;
- a real Nitro quorum: cross-host private L4 routing, rollback-protected durable
  Raft storage, and attestation-bound manifest/TLS/KMS delivery, followed by
  enclave lifecycle and membership validation on enclave-enabled EC2.

The historical roadmap issues remain useful planning inputs, but they are not
release evidence: [extension](https://github.com/Spiral-Safe/specs/issues/1),
[SaaS/API](https://github.com/Spiral-Safe/specs/issues/2),
[consumer passkey wallet](https://github.com/Spiral-Safe/specs/issues/3), and
[production deployment](https://github.com/Spiral-Safe/specs/issues/4).
