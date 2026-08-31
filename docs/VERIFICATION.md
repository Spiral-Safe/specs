# Verification record

Verification date: **2026-08-30**.

This record separates commands that actually ran, source/configuration review,
fixture demonstrations, and environment-dependent work that has not run. A
pass establishes only the property named in its row. The account/billing,
recording, deployment, and same-image Nitro changes are still uncommitted, so
results apply to this workspace rather than remote `main`.

## Evidence levels

- **Executed:** the command completed against the identified source state.
- **Fixture capture:** actual product code ran, but named external/security
  boundaries were deterministic substitutes.
- **Reviewed/rendered:** source or generated configuration was inspected; the
  target runtime was not created.
- **Not run:** no behavior claim is made.

## Executed automated checks

| Area | Command/check | Result and scope |
| --- | --- | --- |
| Service TypeScript | `npm run build`; `npm run lint`; `TEST_DATABASE_URL=<ephemeral-postgresql-url> npm test` against the exact final source | **Pass: build and formatting; 46 tests passed, 0 failed, 0 skipped.** Covers adapter/config/Vault-client behavior plus API-key hashing/revocation, scrypt, admin bootstrap input safety, session/role/Origin/CSRF behavior, console redaction and atomic full-tuple provider-mapping confirmation, current-period analytics, memory-store quotas/idempotency/expiry, transaction-only completion metering, trusted Vault-operation reconciliation, commit ambiguity, provider-period fail-closed behavior, mapping invalidation, Stripe Checkout/Portal/webhook mapping, Metronome delivery/retry/dead-letter logic, account-aware wallet calls, and the real PostgreSQL integration. The PostgreSQL run caught and validated fixes for explicit reservation-expiry timestamp/integer casts and the daily-usage grouping alias. It is one test database, not HA/failover/PITR evidence; provider calls remain mocked. |
| Service dependency audit | full and runtime-only `npm audit` after the final billing dependency graph | **Pass: zero known advisories.** This is a dated registry/scanner result, not a security audit. |
| Extension | `npm test`; `npm run build` | **Pass: 15/15 and production bundle built.** Covers bridge/config/origin rules, secret and URL-path redaction, ceremony binding/persistence/replay/cross-tab rejection, backend/RPC routing, provider events, advertised Wallet Standard methods, and SIWS with test doubles. The build retains a non-blocking size warning for the large injected bundle. |
| Wallet adapter | `npm test` | **Pass: 2/2.** Builds CommonJS/ESM/types and verifies feature registration/routing with test doubles. |
| SDK | `npm test` | **Pass: 5/5.** Builds and verifies bearer headers, identities/ceremonies, all five wallet routes, URL constraints, payload conversion, and structured errors against mocks. |
| Other npm dependency audits | full `npm audit` in extension, wallet-adapter, and SDK after their final dependency graphs | **Pass: zero known advisories in each recorded run.** No recording dependency was added to these trees. |
| Vault plugin and chain signers | pinned Go 1.26 container, `go test ./...` | **Pass.** Covers the cryptographic software-authenticator/framework lifecycle and chain boundaries summarized below. It is not a physical-authenticator or live Vault-process test. |
| Go vulnerability scan | pinned Go container, `govulncheck@v1.7.0 ./...` | **No reachable or imported-package vulnerability in the recorded current core graph.** One required module contained an uncalled vulnerable symbol. Rerun when Go code or dependencies change. |
| Load harness safety/parser | `npm run load:test` | **Pass: 9/9.** Covers endpoint scenario selection, mutation/remote-target/token safeguards, status parsing, exact-Origin rejection, manual redirect observation, and per-request ceremony-ID uniqueness. It does not run successful WebAuthn ceremonies or establish capacity. |
| Live all-route fixture load | disposable Compose stack; `node load-test/suite.mjs --scenario all --allow-mutations --requests 10 --concurrency 2 --chain ethereum` with only local credentials | **Pass: 260 requests across 26 method/path scenarios, 0 unexpected statuses or client errors.** Earlier runs exposed and led to fixes for automatic redirect following, shared probe ceremony IDs, and treating the console's correct 403 exact-Origin rejection as an expected login-probe outcome; the final repeated run passed. Wallet mutation cases use successful `/init` plus negative missing-wallet/placeholder-credential paths, so this is endpoint/control-plane smoke evidence, not successful signing throughput or production capacity. Containers, network, and uniquely named images were removed afterward. |
| Recording runner | `npm run recording:test` with permission to bind its loopback listener | **Pass: 8/8.** Verifies manifest/selector/fixture/output safety contracts, collision-aware annotation placement, first-presented-frame timeline timing, and the no-video path; it does not launch the four product flows. |
| Annotated product capture | `node recording/record.mjs --flow all --run-id product-walkthrough-final` against the exact final source | **Fixture capture pass for all four flows with zero recorded warnings:** four native WebM files, four trace ZIPs, 20 annotated PNGs, and four five-step timelines under `services/recording/output/product-walkthrough-final/`. It loaded the actual unpacked extension and its Wallet Standard demo, the actual `services/public` page, and the actual compiled `/developer` and `/admin` console routes. |
| Nitro manifest renderer | pinned Go 1.26, `go test ./...` in `services/nitro/config-renderer` | **Pass.** Includes strict JSON/schema, placeholder/expiry/mode/bind/leader checks, manifest symlink/mutability rejection, trust-file group/other-write rejection, private-key permissions, and bootstrap/join `retry_join` rendering. |
| Nitro scripts and topology | ShellCheck 0.11, `bash -n`, `make -C nitro simulate`, `tests/enroll-node-test.sh` | **Pass.** Simulation proves consistent declared image/KMS inputs, a shared measured attestation sentinel, distinct node IDs/directories/routed addresses, and join mTLS configuration. It does not bind the declared OCI digest to an EIF; that needs signed external release provenance. The mocked admission test proves Veil verification precedes the hook, failure blocks it, only a short-lived reference is emitted, and unhealthy/insufficient Autopilot membership is rejected. It is not a live quorum. |
| Same-image ordinary container | exact-source Linux/amd64 OCI build and inspection | **Pass:** manifest `sha256:4b149d3ac8ffbdc41d9bbe9d5a3f7e2b5677d552583d2337c7e45d2d24beb0ce`, config `sha256:b929465267ba089060a51403bef822cc4ca196fe97d8ba1b8192774d1ced2bda`. Labels/environment contain the shared `spiral-safe-attestation.invalid` verifier-only sentinel, and the build statically checks pinned Veil's leaf-certificate-hash/attestation binding. No EIF was built or launched; an OCI archive checksum is intentionally not treated as identity because its creation annotation is not stable. |

### Recording trust boundary

Every captured page is labeled **FIXTURE MODE · SYNTHETIC LOCAL DATA**. The
extension/wallet flows use a loopback fake signing/RPC backend, fake token, and
CDP virtual WebAuthn authenticator. Dashboard flows use the actual compiled
routes and session forms but an in-memory demo account store, fake Vault client,
and disabled Stripe/Metronome. The key form is not submitted. A raw preflight
and authenticated response assert CSP/nosniff/same-origin-referrer headers
before the capture-only CSP bypass injects annotations. None of this proves a
physical authenticator, production CSP enforcement, real signing backend,
database, provider, billing, or capacity.

## Current deployment/configuration checks

| Area | Command/check | Result and scope |
| --- | --- | --- |
| Kustomize/Skaffold | direct Kustomize and offline Skaffold local/production render; Skaffold diagnose | **Pass:** local 13 resources and production 19; render methods agree and diagnose succeeds. No resources were applied to a cluster. |
| Compose configuration/runtime | Compose configuration plus isolated build/boot on alternate loopback ports | **Pass for the local fixture:** `/healthz` and `/readyz` succeeded, client returned 200, and developer/admin login redirects resolved to 200. Stack and temporary image were removed. This did not complete WebAuthn signing or external billing. |
| Production fail-closed gates | migration/process and configuration fixtures | **Pass:** migration entrypoint runs as UID 10001 with a read-only root filesystem and fails without `DATABASE_URL`; configuration rejects missing plans, missing Stripe plan IDs, and unverified Metronome-to-Stripe invoicing. |
| Production egress shape | rendered NetworkPolicies | **Pass for manifest intent:** same-namespace PostgreSQL port 5432 and HTTPS gateway port 443 use separate labels. External database/provider routing remains operator-owned and was not exercised. |

## Executed Go lifecycle coverage

The recorded final core Go suite contains:

- tenant and chain storage isolation;
- deterministic tenant/user/chain WebAuthn user handles within the protocol
  limit;
- Solana message and legacy-transaction verification, wrong-signer rejection,
  malformed/truncated parser containment, and versioned-transaction rejection;
- Ethereum EIP-55 key/address derivation, EIP-191 signing/recovery, recovery-ID
  bounds, and transaction rejection;
- trusted completion-operation recovery and mismatch rejection before signing;
- complete P-256 software-authenticator registration/assertion through Vault
  `Backend.HandleRequest`, including create/update routing;
- required user-verification rejection, one-time ceremony replay rejection,
  sign-counter persistence, and permanent clone-warning disablement after a
  nonzero counter regression.

These properties do not prove every browser/authenticator or distributed
failure mode.

## Reviewed source/configuration boundaries

| Area | Present in source | Still unproven |
| --- | --- | --- |
| Account API | PostgreSQL store, one-time-reveal hashed scoped keys, sessions/roles, admin bootstrap, quotas, period checks, durable usage/outbox/webhook claims | real PostgreSQL migration/HA/TLS/PITR/restore, split migration/runtime roles, multi-replica contention, SSO/MFA/password recovery |
| Stripe/Metronome | hosted Checkout/Portal helpers, raw signature verification, current-Price plan mapping, provider-state mapping invalidation, asynchronous stable outbox events | live restricted-key permissions, Checkout, webhook delivery, Metronome customer/contract/provider mapping, invoice, collection, tax, reconciliation and alerts |
| Production Kubernetes | three-node Vault Raft intent, two API replicas, migration init container, external billing inputs, client removal, and restrictive egress design | real apply, TLS/KMS/storage/ingress/database/provider routing, joins/failover/restore/upgrades |
| Extension runtime | actual unpacked extension ran against the fixture recorder | real backend + physical authenticator, optional-host permission UX, true MV3 suspension during a passkey prompt, and live third-party dApp interoperability |
| Veil admission | same-image strict renderer, official-verifier wrapper, enrollment hook boundary, membership checks, ordinary OCI image and tests | EIF, EC2/NSM proof, cross-host L4, durable anti-rollback storage, attestation-bound bundle/TLS/KMS delivery, initialization and live quorum |

## Explicitly not run

The following remain acceptance requirements, not completed verification:

- a physical platform/roaming-authenticator
  `/init → /create → /signin → /complete` matrix on supported Chrome, Safari,
  Firefox, and Edge targets;
- an unpacked-extension session with the real Vault backend and physical
  authenticator covering transaction/message/batch/send/SIWS;
- successful ceremony/signing capacity, sustained load, soak, fault injection,
  and multi-replica contention. The endpoint load harness uses negative or
  missing-user `/create`, `/signin`, and `/complete` paths;
- a funded Solana devnet broadcast through the extension;
- a real production PostgreSQL migration, narrowed runtime role, HA/failover,
  PITR/restore, retention, and secret rotation exercise;
- live Stripe Checkout/Portal/webhook, a Metronome ingest-to-Stripe invoice,
  charge/collection/refund, mapping reconciliation, tax flow, or provider
  outage/recovery;
- a real Kubernetes apply, registry push, workload identity, public TLS/DNS,
  ingress, StorageClass, KMS auto-unseal, Vault initialization, Raft failover,
  snapshots/restores, upgrades, backup, and disaster recovery;
- EIF creation or launch/termination, independent nonce/PCR/TLS attestation on
  enclave-enabled EC2, cross-host routing, durable enclave storage,
  attestation-bound runtime delivery/KMS identity, or a live Vault quorum;
- penetration testing, external custody/billing/security audit, SBOM/provenance
  verification, and license/tax counsel review.

## Non-runtime repository baseline

Earlier onboarding checks established that the Hugo website builds with its
pinned theme but has a missing visible `/showcase` target, and that the token
list's build/main validator pass while partner-refresh tooling needs separate
data-maintenance review. These are inventory facts, not custody-runtime
evidence.

## Closing the gaps

1. Retain exact test logs, recording manifests, image manifests/configs, and
   dependency lockfiles with each release candidate.
2. Run the separate physical-authenticator/browser matrix against the real
   backend.
3. Complete Stripe/Metronome sandbox-to-invoice proof and PostgreSQL
   migration/restore/HA exercises before enabling billing.
4. Deploy the production overlay to a disposable cloud-real cluster and run
   the TLS/KMS/Raft/database/egress/backup/failure drills.
5. Implement the three missing Nitro boundaries, then run a real same-image
   quorum and independent hardware-attestation lifecycle.
6. Complete threat-model and independent audits before any mainnet asset or
   paid production account.
