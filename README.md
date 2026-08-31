# Spiral Safe system documentation

This repository is the canonical map of the eight repositories in the
[Spiral Safe organization](https://github.com/orgs/Spiral-Safe/repositories).
It describes the implementation in this workspace, including changes that are
not part of the remote default branches until they are reviewed and committed.

Spiral Safe is currently a development-stage, WebAuthn-authorized signing
system:

- a browser extension exposes a real Solana Wallet Standard provider;
- the extension and SDK call an HTTP service whose bearer credentials are
  scoped to an operator-configured tenant and username allowlist;
- a Vault secrets-engine plugin owns the wallet keys and WebAuthn credentials;
- Solana legacy transactions/messages and Ethereum EIP-191 messages have
  chain-specific signers;
- local Compose, bounded endpoint load tests, a Skaffold/Kubernetes baseline,
  and an AWS Nitro Enclaves adapter based on Veil are included.

This is not an audited custody product. The production Kubernetes and Veil
paths are build/render baselines that still require deployment-specific TLS,
identity, KMS, backup, monitoring, recovery, and real-hardware verification.

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
  username authorization before Vault plus Kubernetes Vault authentication;
- required WebAuthn user verification and persistent disablement after a
  nonzero authenticator-counter regression;
- a self-contained, loopback-constrained local demo that is removed from the
  production overlay;
- three-node Vault integrated-Raft manifests and a Veil build adapter.

Still requiring real environments or additional product work:

- platform and roaming-authenticator coverage on supported browsers;
- Ethereum transaction signing;
- production OIDC/session issuance, token rotation/revocation, wallet recovery,
  and richer user/role administration;
- transaction decoding, policy/simulation, and human-readable intent binding;
- a cloud-tested Raft backup/restore, KMS, monitoring, and disaster-recovery
  program;
- Veil attestation, persistence, bootstrap, KMS policy, and enclave lifecycle
  validation on an enclave-enabled EC2 parent.

The historical roadmap issues remain useful planning inputs, but they are not
release evidence: [extension](https://github.com/Spiral-Safe/specs/issues/1),
[SaaS/API](https://github.com/Spiral-Safe/specs/issues/2),
[consumer passkey wallet](https://github.com/Spiral-Safe/specs/issues/3), and
[production deployment](https://github.com/Spiral-Safe/specs/issues/4).
