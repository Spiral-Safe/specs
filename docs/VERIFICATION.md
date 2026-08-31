# Verification record

Verification date: **2026-08-30**.

This record distinguishes automated checks that actually ran from source review
and from environment-dependent acceptance work. A pass means only what the
listed command can establish.

## Evidence levels

- **Executed:** command completed against the current workspace.
- **Reviewed/rendered:** code or generated configuration was inspected, but the
  target runtime was not created.
- **Not run:** no claim is made for the behavior.

The workspace contains uncommitted changes across repositories. Results apply
to this local tree, not to the remote `main` branches until the changes are
committed.

## Executed automated checks

The remediated Go module declares Go 1.25.7. Container builders use
`golang:1.26@sha256:e30143be198ab04cf7ba25fba83ab3a692ca584c994aad0bf131fa0eb32dd8c1`;
the direct Vault API/SDK dependencies are v1.23.0 and v0.25.1.

| Area | Command/check | Result and meaning |
| --- | --- | --- |
| Extension | `npm test`; `npm run build` | **Pass: 15/15 and production bundle built.** Covers protocol codecs, settings/origin validation, bearer isolation, redaction of path-embedded credentials from page-facing backend/RPC URLs, authoritative ceremony IDs, expiry/replay/cross-tab rejection, browser-session ceremony persistence across controller restart, backend registration/signing requests, RPC submission, provider events, every advertised signing method, SIWS binding, and Wallet Standard routing with test doubles. Webpack reports only its non-blocking size/performance warning for the 462 KiB `inject.js`; optimize that bundle before store release. The checks do not install Chrome or use a physical authenticator. |
| Wallet adapter | `npm test` | **Pass: 2/2.** Builds CommonJS/ESM/types and verifies feature registration/routing. It does not validate a live backend. |
| SDK | `npm test` | **Pass: 5/5.** Builds the bundle and verifies bearer headers, identity/ceremony bodies, all five routes, URL constraints, binary payload conversion, and structured errors against mocks. |
| Service adapter/config/Vault client | `npm test` | **Pass: 12/12.** Covers public health/readiness, bearer rejection, server-side tenant injection, production username allowlists and pre-Vault 403 rejection, allowed-origin CORS, signing payload/legacy alias forwarding, chain/credential validation, missing-registration and Vault error mapping, rejection of every static production Vault token, local config constraints, Kubernetes-token lease caching, and one retry after a Vault 403. The process-local rate limiter still needs dedicated behavior tests. |
| Vault plugin and chain signers | pinned Go 1.26 container, `go test ./...` against the exact final source | **Pass.** The plugin package passed; the command package has no tests. Coverage is detailed below. This is a software-authenticator/framework test, not a platform-authenticator or live Vault process. |
| Go vulnerability scan | same pinned container, `go run golang.org/x/vuln/cmd/govulncheck@v1.7.0 ./...` against the current database | **No vulnerabilities found: 0 affected and 0 in imported packages.** One required module contains a vulnerable symbol that the application does not call. This path-sensitive result must be rerun when code or dependencies change. |
| Load harness safety/parser | `npm run load:test` / `node --test load-test/suite.test.mjs` | **Pass: 7/7.** Confirms all endpoint scenarios, mutation opt-in, loopback/remote safeguards, token requirement, URL rejection, smoke bounds, and expected-status overrides. For `/create`, `/signin`, and `/complete`, the harness deliberately measures only invalid or missing-user paths; it does not establish successful WebAuthn/signing capacity. |
| npm dependency audits | full `npm audit --json` in `services`, `extension`, `wallet-adapter`, and `sdk` | **Pass: zero known advisories in all four full dependency trees** (195, 181, 124, and 114 audited dependencies respectively). This is a dated registry/scanner result, not a security audit. Narrow overrides remain subject to compatibility testing and future removal. |
| Skaffold image build | exact-final-source `skaffold build --cache-artifacts=false --file-output=...` | **Pass:** plugin `sha256:e75c47eaac944816d99bcebd16cba5648dd571803622dda7d65b6e13a24f5168`, service `sha256:8b33ac9525d2105767273dfdb97db1293a48c45f9e77e9ccd9e484a1e40c8691`, and local client `sha256:21ccdb6b7a66e917ade288f86f693bbfa956d3698064d59a8f9817655dd8261c`. Images were built, not pushed or deployed. |
| Skaffold/Kubernetes configuration | local/production diagnose, direct Kustomize and offline Skaffold render, kubeconform v0.7.0 strict | **Pass:** local 12/12 and production 19/19 resources, with zero invalid, errors, or skipped. Production resolves exactly two artifacts (plugin/service), no client and no port forwarding; local resolves all three. No objects were applied to a cluster. |
| Deployment scripts/config | `bash -n`, ShellCheck, Make dry-runs, diff checks, and production Vault HCL parse | **Pass.** Vault parsed the production configuration through to the expected unavailable external AWS-KMS boundary. No privileged, cluster, KMS, or other cloud operation was invoked. |
| Local service demo browser smoke | self-contained client opened at `http://127.0.0.1:9082` | **Pass for the static/browser boundary.** The UI rendered with zero browser logs; `?api=https://evil.example` was rejected; clicking Register reached the fetch path, confirming the bundled `browser.js`/`index.js` wiring. No backend success response, extension, or real authenticator was involved. |
| Exact-source Compose/plugin/API smoke | isolated loopback Compose stack with Vault 1.21.4, then the all-route suite | **Pass, then cleanly removed.** Vault catalogued and enabled the Spiral Safe plugin. The endpoint matrix completed all 8 scenarios with 0 unexpected: health 200, readiness 200, unauthenticated check 401, missing check 404, init 200, and negative missing-state create/signin/complete 404. A second same-user init returned the same address with a distinct 43-character ceremony ID, exercising the framework update route; incomplete signin returned the expected `wallet_not_found` 404. This is one bounded request per path class, not successful-ceremony capacity. Containers, volumes, networks, and all task temporary artifacts were removed. |
| Veil image | exact-final-source ARM64 `docker build -f nitro/Dockerfile -t spiral-safe/veil:validation .` plus layout/metadata smoke | **Pass:** image `sha256:ce474d54d2d7f520ef9e15a498a62865bb0271a26deefcc3465d7c11c76ff7d4`, with pinned Veil `2b8c06ca651e09b21832f6fc4ae2605371386f76`, Go 1.26, Vault 1.21.4, executable layout, source label, and license files. No EIF or enclave was started. |
| Veil host/verifier tools | `docker buildx build --platform linux/arm64 --target veil-tools --output type=local` | **Pass.** Exported executable, stripped, static ARM64 ELF `veil-proxy`/`veil-verify` and the GPL license. The privileged proxy was not run. |

## Executed Go lifecycle coverage

The final-source Go suite contains:

- tenant and chain storage isolation;
- deterministic, tenant/user/chain-scoped WebAuthn user handles that remain
  within the 64-byte protocol limit;
- Solana message and legacy-transaction verification, wrong-signer rejection,
  malformed/truncated parser containment, and versioned-transaction rejection;
- Ethereum key generation, EIP-55 address derivation, EIP-191 signature
  generation, address recovery, recovery-ID bounds, and transaction rejection;
- a complete cryptographic software-authenticator registration and assertion
  lifecycle routed through Vault `Backend.HandleRequest`, including logical
  create/update route coverage;
- required WebAuthn user-verification rejection, one-time ceremony replay
  rejection, sign-counter persistence, and permanent clone-warning disablement
  after a nonzero counter regression.

The service and extension suites jointly test the contract boundary: the
server—not the client—injects the tenant, and the backend-issued ceremony ID is
the same ID the extension binds to the requesting tab/frame/origin.

These are necessary unit/integration properties. They do not prove that every
browser/authenticator or distributed failure mode works.

## Reviewed or rendered, not deployed

| Area | What was inspected | What remains unproven |
| --- | --- | --- |
| Production Vault Raft/API | Three-node StatefulSet, headless/active services, two API replicas, TLS configuration, retained data/audit PVCs, KMS seal inputs, anti-affinity, PDB, RBAC, Kubernetes auth, NetworkPolicies, username-scoped API principal inputs, and deletion of all local client/demo resources | initialization, auto-unseal, TLS SAN correctness, real joins/elections, leader routing, snapshots/restores, upgrades, quorum loss, KMS outage, audit export, and an external relying-party/ingress |
| Local Skaffold | Default dev overlay and port-forward configuration | successful apply on kind/minikube/Docker Desktop and live rebuild loop |
| Extension runtime | Manifest, privileged/background/content/page boundaries, popup configuration, demo source, automated mocks, and controller recreation against a shared session-store test double | unpacked Chrome installation, optional-host permission UX, an actual MV3 suspension/resume cycle during WebAuthn, and live Wallet Standard dApp interoperability |
| Local service demo | Self-contained bundle inputs, text-only DOM insertion, CSP/security headers, same-host loopback API validation, production deletion patch, and browser smoke | a platform-authenticator ceremony and cross-browser matrix |
| Container inputs | Pinned Node 22, Go 1.26, Vault 1.21.4, and Veil source references plus exact built digests | registry provenance, SBOMs, signatures, and production runtime behavior |
| Veil adapter | Built daemon/proxy/verify roles, pinned source, loopback Vault config, readiness wiring, non-debug named launch, and licenses | EIF behavior, attestation document validation, persistent state, secure bootstrap, KMS binding, EC2 networking, and recovery |

## Explicitly not run

The following remain acceptance requirements, not completed verification:

- real platform/roaming-authenticator `/init → /create → /signin → /complete`
  on Chrome, Safari, Firefox, and Edge targets;
- an unpacked-extension end-to-end sign transaction/message/batch/send/SIWS
  session with the real backend and authenticator;
- high-volume or sustained load, soak, fault-injection, and multi-replica
  ceremony tests;
- successful `/create → /signin → /complete` load or capacity testing; the
  endpoint harness uses negative/missing-user cases for those three routes;
- Solana devnet broadcast through the extension with a funded disposable key;
- a real Kubernetes apply, IRSA/workload identity, ingress, public TLS/DNS,
  StorageClass, KMS auto-unseal, Vault initialization/unseal, Raft failover,
  node replacement, snapshot/restore, upgrade, backup, and disaster-recovery
  exercise;
- production OIDC/session/token issuance and revocation;
- EIF creation or launch/termination on an enclave-enabled EC2 parent; no
  enclave lifecycle command was run;
- Veil/NSM nonce-bound independent attestation, PCR/application-hash comparison,
  altered-image rejection, attestation-aware Vault bootstrap, persistent state,
  or KMS policy enforcement;
- penetration testing, external cryptographic/security audit, SBOM/provenance
  verification, and license counsel review.

## Non-runtime repository baseline

Earlier workspace checks, unaffected by the signing changes, established:

- the Hugo website can initialize its pinned Terminal-theme submodule and build
  and serve its primary pages; its visible `/showcase` target was missing;
- the token-list TypeScript build and main PR validator pass, while its partner
  snapshots/tooling need separate data-maintenance review;
- all eight repositories are full clones; `services` is the only repository
  with an additional remote branch and the organization’s only Git tag.

These are inventory/onboarding facts, not evidence for the custody runtime.

## How to close the remaining evidence gaps

1. Run the Go, Node, Compose, safe load, and browser manual steps from
   [RUNBOOK.md](RUNBOOK.md) from a clean checkout in CI and retain logs plus
   artifact hashes.
2. Add an actual-browser suite using WebAuthn virtual authenticators, then run
   the manual platform/roaming-authenticator matrix separately.
3. Deploy the production overlay into a disposable but cloud-real cluster and
   execute documented TLS/KMS/Raft/backup/failure drills.
4. Build the Veil image and EIF twice, record image/EIF hashes and measurements,
   launch without debug mode, and verify nonce-bound attestation from an
   independent relying party.
5. Complete a threat-model review and external audit before any mainnet or
   production secret.
