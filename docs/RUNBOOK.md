# Build, test, and deployment runbook

All commands assume the eight repositories are sibling directories under one
workspace. Use disposable credentials and networks unless a section explicitly
describes a reviewed production procedure.

## Prerequisites

Core development:

- Git;
- Docker Engine and Docker Compose;
- Node.js 22 or newer and npm;
- Go 1.25.7 or newer, or a Go container for direct plugin tests;
- a current Chromium-family browser for the unpacked extension;
- a platform or roaming authenticator for the manual WebAuthn check.

Kubernetes:

- a local cluster such as kind, minikube, or Docker Desktop;
- Skaffold v2, `kubectl`, and Kustomize;
- a target registry for the production profile.

Veil/Nitro:

- Docker Buildx for the ordinary image;
- Linux Nitro CLI for EIF construction;
- an enclave-enabled EC2 parent, allocated enclave CPU/memory, VSOCK and TUN
  access, and reviewed AWS networking/IAM for launch and attestation.

A normal workstation can validate source, packages, manifests, and ordinary
containers. It cannot prove AWS Nitro isolation or attestation.

## 1. Inspect the workspace

```sh
git -C services status --short
git -C services branch -a
git -C services log --oneline --graph --decorate --all
```

Repeat `git status --short` for each repository before editing. The workspace
may contain intentional cross-repository changes; do not clean or reset them.

## 2. Run the disposable core stack

From `services`:

```sh
docker compose build
docker compose up -d
```

Expected host listeners:

| Port | Component |
| ---: | --- |
| 8200 | Vault dev API and plugin |
| 3000 | authenticated HTTP adapter |
| 9080 | service browser demo |

Open <http://localhost:9080>. The checked-in development values are:

- API token: `spiral-safe-local-development-only`;
- API tenant: `local`;
- Vault dev root token: `root`;
- WebAuthn RP ID: `localhost`;
- WebAuthn origin: `http://localhost:9080`.

These values and Vault's in-memory data are intentionally disposable. Compose
binds all three published ports to `127.0.0.1`, and the service/client images
run as the unprivileged Node user. Those safeguards reduce accidental exposure;
the known credentials and Vault dev mode still make this a local-only profile.

The demo ships its JavaScript and CSS in the client image; it does not load a
UI or Solana library from a CDN. Its static server sends a restrictive CSP and
other browser security headers, and UI errors are rendered as text. The `api`
query parameter, when used for an alternate port, must still resolve to a
plain-HTTP loopback origin with the same hostname as the demo page. The sample
Solana transaction preparation does contact the configured devnet RPC.

Use alternate service/Vault ports while preserving the WebAuthn demo origin:

```sh
VAULT_PORT=18200 SERVICE_PORT=13000 CLIENT_PORT=9080 docker compose up -d
```

Then open <http://localhost:9080/?api=http://localhost:13000>.

Smoke the process, Vault readiness, HTTP authorization, and plugin lookup:

```sh
curl -fsS http://127.0.0.1:3000/healthz
curl -fsS http://127.0.0.1:3000/readyz
curl -i -X POST http://127.0.0.1:3000/check \
  -H 'content-type: application/json' \
  --data '{"username":"missing-user","chain":"solana"}'
curl -i -X POST http://127.0.0.1:3000/check \
  -H 'authorization: Bearer spiral-safe-local-development-only' \
  -H 'content-type: application/json' \
  --data '{"username":"missing-user","chain":"solana"}'
```

The unauthenticated request should return 401; the authenticated lookup should
reach Vault and return 404.

### Local developer and admin consoles

Compose also enables the account layer in explicit in-memory demo mode. Open:

- <http://localhost:3000/developer/login> with
  `developer@example.test` / `demo-developer-only`;
- <http://localhost:3000/admin/login> with
  `admin@example.test` / `demo-admin-only`.

The developer can inspect fixture plan/usage data, create and revoke a scoped
`ssk_test_...` key, and view the secret on its one-time response. The admin can
inspect aggregate/account state and create another fixture account/developer.
Do not paste even a production-like key into screenshots, traces, shell history,
or support logs.

This mode uses process memory, zero-price fixture plans, a known session secret,
and known passwords. Stripe and Metronome are disabled, so Checkout/Portal and
real invoicing are not exercised. Restarting the service loses the entire
account/session/usage fixture. Production rejects memory mode and demo seeding.

Stop the disposable stack:

```sh
docker compose down --remove-orphans
```

Restarting Vault dev mode loses every wallet and key.

## 3. Exercise the service demo and WebAuthn

The service demo supports:

- Solana registration and legacy transaction signing;
- Ethereum registration and EIP-191 message signing.

For a real-authenticator acceptance check:

1. Start Compose and visit the exact `http://localhost:9080` origin.
2. Register a unique Solana username and approve `navigator.credentials.create`
   with Touch ID, Windows Hello, a passkey, or a hardware security key.
3. Generate and sign the sample Solana transaction. Deserialize `encodedTX`
   and verify the expected signer before any optional devnet submission.
4. Register a different username or select the Ethereum chain, sign a known
   message, decode the 65-byte signature, and recover the displayed EIP-55
   address using the EIP-191 digest.
5. Test cancellation, malformed completion, replay, expiry, and two concurrent
   tabs. No failed path may return a signature.
6. Confirm an authenticator assertion without user verification is rejected.
   A nonzero signature-counter regression is a security event that permanently
   disables that credential in the current implementation; use a disposable
   identity because credential recovery is not implemented.
7. Repeat on every browser/authenticator class the product intends to support.

The automated Go test uses a real cryptographic software authenticator, but it
does not replace this platform/hardware matrix.

## 4. Build and run the extension demo

Build the adapter first because the extension consumes it as a sibling package:

```sh
cd wallet-adapter
npm ci
npm test

cd ../extension
npm ci
npm test
npm run build
npm run build:demo
```

Load `extension/dist` in Chrome/Chromium:

1. open `chrome://extensions`;
2. enable Developer mode;
3. choose **Load unpacked** and select `extension/dist`;
4. pin Spiral Safe and open its popup;
5. set backend `http://localhost:3000`, a unique username, API token
   `spiral-safe-local-development-only`, devnet RPC
   `https://api.devnet.solana.com`, cluster `solana:devnet`, and allowed
   website origin `http://localhost:9080`;
6. save and grant only the requested RPC/backend host access.

The extension demo and service demo both default to port 9080. For the
extension demo, start only Vault and the adapter:

```sh
cd services
docker compose up -d --build vault service

cd ../extension
npm run demo
```

Open <http://localhost:9080>, refresh after the extension is loaded, then
exercise:

- register passkey;
- connect/disconnect and account-change events;
- sign message;
- sign one and two legacy transactions;
- Sign In With Solana;
- sign and send after funding the disposable devnet address.

Each sign operation should prompt WebAuthn. Batch signing intentionally uses one
approval per transaction. Sign-and-send submits the already signed transaction
through the RPC configured in the extension.

The completion request repeats the operation selected at sign-in. Vault rejects
any mismatch against its stored ceremony before signing, and the adapter also
requires Vault to return that same operation. Successful transaction completion
consumes one transaction unit; Solana and Ethereum message signatures do not.

The API token and pending ceremony bindings disappear when extension session
storage is cleared; re-enter the token and begin a fresh ceremony after a
browser restart if required. Ordinary Manifest V3 worker suspension does not
clear them. Rebuild and reload the unpacked extension after source changes.

From an allowlisted demo page, `spiralSafe:config`/`config.get` should expose
neither the API token nor any credential embedded in a backend or RPC URL path;
the public URLs are reduced to their origins. The full configured URLs remain
inside the trusted worker for requests.

## 5. Generate annotated product recordings

The recorder lives at `services/recording` and writes ignored native WebM,
Playwright trace, screenshot, timeline, and root-manifest artifacts. Install
both Node projects first, then test or capture all four flows:

```sh
cd services
npm ci
cd ../extension
npm ci
cd ../services
npx playwright install chromium
npm run recording:test
npm run recording:all
```

The browser install is needed only when neither the pinned Playwright Chromium
nor a supported local Chromium-family browser is already available; it requires
network access. The extension flow is headed because ordinary headless Chromium
does not load unpacked extensions.

For the reproducible named walkthrough currently recorded in this workspace:

```sh
node recording/record.mjs --flow all --run-id product-walkthrough-final
```

The exact root artifact is
`services/recording/output/product-walkthrough-final/manifest.json`. Under it,
each
of `extension-demo`, `standalone-wallet`, `developer-dashboard`, and
`admin-dashboard` contains `recording.webm`, `trace.zip`, `timeline.json`, and
numbered screenshots. Open a trace with, for example:

```sh
npx playwright show-trace \
  recording/output/product-walkthrough-final/developer-dashboard/trace.zip
```

The extension flow loads the actual built unpacked extension in a persistent
headed Chromium context and drives its real Wallet Standard demo. The wallet
flow drives the actual standalone page. The dashboard flows compile the
service, serve the real `/developer` and `/admin` routes, and log in through the
real role/session forms. Every action receives a numbered title/description
callout and target highlight before capture.

Every capture is visibly labeled **FIXTURE MODE · SYNTHETIC LOCAL DATA**. A
loopback fixture replaces Vault, Stripe, Metronome, cloud infrastructure,
Solana RPC, production credentials, and physical passkeys. Dashboard capture
uses the development in-memory billing seed and a fake Vault client that
refuses wallet calls; Stripe/Metronome stay disabled. Extension/wallet capture
uses a CDP virtual WebAuthn authenticator and a fake marker token. The developer
flow fills but does not submit the key form, so no key plaintext enters an
artifact.

`bypassCSP` is enabled only in the capture context so the annotation overlay
can be injected. Before doing that, the recorder asserts the real raw response
CSP, `X-Content-Type-Options: nosniff`, and `Referrer-Policy: same-origin`, and
checks the authenticated document response again. This is a recorder safety
check, not proof of browser CSP enforcement. Generated traces can contain DOM
and source snapshots; review them before sharing. Native WebM output needs no
FFmpeg conversion. See `services/recording/README.md` for individual flow
commands and the selector contract.

These recordings prove deterministic product wiring and presentation only.
They do not prove a live Vault signing ceremony, Stripe charge, Metronome
invoice, production database, physical authenticator, or capacity.

## 6. Test the plugin, service, and SDK

From `services`:

```sh
go test ./...
npm ci
npm run lint
npm test
npm run build:client
npm audit
```

If Go is not installed:

```sh
docker run --rm -v "$PWD:/src" -w /src \
  golang:1.26@sha256:e30143be198ab04cf7ba25fba83ab3a692ca584c994aad0bf131fa0eb32dd8c1 \
  go test ./...
```

The Go suite drives the Vault framework through `Backend.HandleRequest` for a
complete registration → check → authentication → Ethereum EIP-191 signature
flow with a P-256 software authenticator. It covers both create/update routing,
required user verification, persistent counter-regression disablement, address
recovery, tenant/chain isolation, bounded user handles, ceremony replay
rejection, and the Solana/Ethereum signer boundaries.

From `sdk`:

```sh
npm ci
npm test
npm audit
```

Minimal SDK shape:

```ts
import { SpiralSafeSDK } from "@spiralsafe/sdk";

const client = new SpiralSafeSDK({
  baseUrl: "http://localhost:3000",
  apiToken: "spiral-safe-local-development-only",
  chain: "solana",
});

const begin = await client.init("alice");
// The browser calls navigator.credentials.create() with begin.options.
// Serialize that credential, then:
await client.create("alice", begin.ceremonyId, credential);
```

The SDK deliberately leaves `navigator.credentials` and credential
serialization to the relying-party page.

## 7. Run bounded endpoint load tests

The dependency-free suite covers both probes and every wallet route. It refuses
remote targets, remote plaintext tokens, and mutating scenarios unless the
corresponding opt-ins are present.

Safe one-request-per-scenario smoke:

```sh
cd services
SPIRAL_SAFE_API_TOKEN=spiral-safe-local-development-only \
  node load-test/suite.mjs --smoke
```

This runs `/healthz`, `/readyz`, unauthenticated rejection, and authenticated
missing-wallet `/check`.

Small all-route validation load:

```sh
SPIRAL_SAFE_API_TOKEN=spiral-safe-local-development-only \
  node load-test/suite.mjs \
    --scenario all \
    --requests 10 \
    --concurrency 2 \
    --allow-mutations
```

Use `--chain ethereum` for the Ethereum entry paths. The successful `/init`
case creates a new incomplete record. The `/create`, `/signin`, and `/complete`
scenarios each use fresh missing usernames; the completion cases also carry
placeholder credentials/ceremony IDs. They exercise only negative validation
or missing-wallet paths, never a successful WebAuthn ceremony or signing
operation. Successful assertions cannot be safely replayed across concurrent
workers, so this suite does **not** measure successful ceremony/signing
capacity. Use the lifecycle test for cryptographic correctness and a
purpose-built virtual-authenticator runner for future success-path capacity
work.

Run the harness parser/safety tests:

```sh
node --test load-test/suite.test.mjs
```

Results are environment-specific smoke evidence, not capacity claims. Load only
systems you own or are explicitly authorized to test.

## 8. Render and run the local Kubernetes profile

From `services`, inspect without mutating a cluster:

```sh
kustomize build deploy/overlays/local
kustomize build deploy/overlays/production
skaffold render --offline=true --digest-source=none
```

With a disposable local cluster selected:

```sh
skaffold dev
```

The default profile builds the plugin/Vault, adapter, and client images, applies
the local Kustomize overlay, tails logs, and forwards ports 8200, 3000, and
9080. It still uses single-node Vault dev mode and the known local credentials;
it is a Kubernetes-shaped development environment, not HA.

The service and local-demo Dockerfiles pin
`node:22@sha256:8a34c4ab3ea2c5cd194f07e317b2a8f09461d3c8b05c4e34c8ccd56d56024c4d`
for their builder and runtime stages. Keep that digest review/refresh in the
image release process rather than silently reverting to a mutable tag.

## 9. Prepare the production Kubernetes profile

The production overlay is intentionally incomplete until the operator supplies:

1. a registry and immutable-image policy;
2. an encrypted, topology-aware StorageClass and snapshot policy;
3. a `vault-tls` Secret with `tls.crt`, `tls.key`, and `ca.crt`, covering
   `vault-active`, `vault-internal`, and StatefulSet peer DNS names;
4. a `vault-unseal` ConfigMap with `AWS_REGION` and
   `VAULT_AWSKMS_SEAL_KEY_ID`, plus a narrowly scoped workload identity;
5. production values for `BILLING_PLANS_JSON`, `CONSOLE_ORIGIN`, Checkout
   success/cancel URLs, Portal return URL, and the reviewed Metronome ingest
   endpoint/interval/batch size. Start from `services/.env.billing.example`;
6. a `spiral-service-billing` Secret delivered by an external-secret/GitOps
   mechanism, containing `DATABASE_URL`, independent random
   `API_KEY_PEPPER` and `CONSOLE_SESSION_SECRET`, a least-privilege restricted
   `STRIPE_API_KEY`, `STRIPE_WEBHOOK_SECRET`, and `METRONOME_API_TOKEN`;
7. replacement of every `REPLACE_ME.invalid` WebAuthn/CORS/console value;
8. exact database/provider egress: the checked-in policy permits only a
   same-namespace pod labeled `spiral-safe.io/billing-database=allowed` on
   5432 and a separate `spiral-safe.io/billing-gateway=allowed` pod on 443;
9. TLS ingress/Gateway and DNS resources, including the Stripe webhook route;
10. a controlled password-file Secret/mount and operator job for the provided
    production administrator bootstrap, plus an external forced-change/MFA/
    rotation procedure;
11. reviewed namespace/pod labels for ingress and operator NetworkPolicy
    access; and
12. PostgreSQL/Vault backups, immutable audit export, metrics, alerts,
    reconciliation, and recovery ownership.

The production service input groups are:

| Source | Values |
| --- | --- |
| Checked-in production posture | `SERVICE_DEV_MODE=false`, `BILLING_MODE=postgres`, `BILLING_DEMO_SEED=false`, `DATABASE_SSL=true`, Kubernetes Vault auth, and `METRONOME_STRIPE_INVOICING_VERIFIED=false` until deliberately patched after proof |
| Operator ConfigMap values | exact CORS/WebAuthn/console origins; `BILLING_PLANS_JSON`; Stripe Checkout success/cancel, Portal return, and intent-version values; Metronome ingest URL/interval/batch; reservation/session TTLs; and the verified-invoicing gate |
| External Secret values | `DATABASE_URL`, `API_KEY_PEPPER`, `CONSOLE_SESSION_SECRET`, restricted `STRIPE_API_KEY`, `STRIPE_WEBHOOK_SECRET`, and `METRONOME_API_TOKEN` |

Keep the database, pepper, session, and provider secrets independent. Demo
credentials are optional base-manifest inputs for local memory mode and must
not be supplied to the production overlay.

Do not create `spiral-service-auth` for production billing and do not add
`API_TOKEN_HASHES`, `API_TOKENS`, or `API_TOKEN`: production account mode
rejects static API credentials and uses developer-issued, database-backed keys.
Do not add `VAULT_TOKEN` either; production rejects every static Vault token
and authenticates through `VAULT_K8S_ROLE`. The operator-only `VAULT_TOKEN`
shown later belongs to an administrative CLI process, not the HTTP adapter.

The checked-in production ConfigMap deliberately sets
`METRONOME_STRIPE_INVOICING_VERIFIED=false` and omits production plan/provider
values. Supplying the secrets without completing those values does not produce
a partially working biller: configuration fails closed at startup. Flip the
global flag only after the environment-level provider integration and sandbox
invoice are verified; wallet API calls still return 402 until each account's
mapping attestation is recorded.

### Database migration and privilege boundary

The production patch adds a `billing-migrate` init container that runs
`node dist/billing/migrate.js`. The migration is idempotent and takes a
PostgreSQL advisory transaction lock, so concurrent replica init containers do
not race. The equivalent operator command is:

```sh
cd services
DATABASE_URL='postgresql://...' DATABASE_SSL=true npm run billing:migrate
```

The checked-in pod uses the same `DATABASE_URL` for migration and runtime.
Before production, split schema-owner/migration credentials from the narrower
service data role; otherwise every API replica retains unnecessary DDL
authority. Configure enforced TLS/CA trust, HA, encrypted backups,
point-in-time recovery, connection limits, restore drills, and migration
rollback ownership. No real production PostgreSQL migration has been claimed.

After the schema exists, create or rotate the first account-independent admin
from a protected operator shell or one-shot job:

```sh
DATABASE_URL='postgresql://...' DATABASE_SSL=true \
  npm run billing:bootstrap-admin -- \
  --email admin@example.com \
  --password-file /run/secrets/spiral_admin_password
```

The CLI refuses `--password`, symlinks, non-regular files, and files larger
than 4 KiB; it hashes the password with the console scrypt parameters and logs
no secret. It can only insert/update an account-independent admin row and never
promotes a developer. Use operator database credentials, mount the file from a
secret manager, and delete or unmount it immediately. This bootstrap does not
provide SSO, MFA, forced first-login change, password reset, or recovery.

### Stripe and Metronome provisioning gate

For every production plan, create a distinct Stripe Product and base Price and
distinct Metronome Product/rate-card IDs. Test the Stripe restricted-key
permissions and configure `POST /billing/stripe/webhook` with the exact signing
secret. Checkout intentionally does not choose payment methods or automatic
tax; tax registrations, product codes, evidence, provider, and failure alerts
must be decided and tested before paid launch.

Then, for every account:

1. let the verified Stripe webhook record the Stripe customer/subscription;
2. create the Metronome customer using the account's exact ingest alias;
3. attach the intended rate-card contract and select the same Stripe customer
   as its billing-provider configuration;
4. map invoice-line products, ingest sandbox events, finalize an invoice, and
   verify collection/error alerts; and
5. only then use the admin account view to record the per-account attestation.

The store atomically compares and snapshots the submitted Stripe customer,
Metronome ingest alias, visible local plan, and Metronome rate card.
Customer-alias/customer/plan/rate-card changes and terminal subscription
clearing make it non-current and restore the 402 gate. It is still an operator
assertion, not a live Metronome query. Reconcile it continuously and fail closed
if provider state drifts. Current repository verification includes mocked
integrations only, not a live Stripe charge or Metronome-created invoice.

The production overlay explicitly deletes the local `spiral-client`
Deployment, Service, and disruption budget, and the Skaffold production profile
does not build the client artifact. Only the API and Vault topology belong in
this baseline; provide a separately reviewed relying-party application and
TLS ingress.

Deploy only after rendering and review:

```sh
skaffold run -p production --default-repo REGISTRY/PROJECT
```

Vault starts uninitialized. Through an authenticated operator channel,
initialize exactly once and place recovery keys and the initial root token into
an approved offline workflow. Never emit them through CI, Git, pod logs, or a
shared shell.

After Vault is active, use a short-lived operator token and the trusted CA:

```sh
VAULT_ADDR=https://127.0.0.1:8200 \
VAULT_CACERT=/secure/path/ca.crt \
VAULT_TLS_SERVER_NAME=vault-active \
VAULT_TOKEN=REDACTED \
./deploy/scripts/bootstrap-vault.sh
```

The script registers/enables the plugin, enables the file audit device, creates
an engine-scoped service policy, enables/configures Kubernetes auth, and binds
the `spiral-service` ServiceAccount. It does not initialize/unseal Vault or
save a root token.

Verify cluster state:

```sh
vault operator raft list-peers
vault status
kubectl -n spiral-safe get pods,pvc,pdb,networkpolicy
```

Scale only through a reviewed StatefulSet change. Preserve an odd voter count
and admit nodes with trusted TLS, KMS workload identity, Kubernetes identity,
and network policy. Do not allow anonymous node joins.

Schedule Vault Raft snapshots to encrypted off-cluster storage and perform a
documented restore drill before calling the deployment durable. Test quorum
loss, pod/node/zone failure, KMS outage/recovery, certificate rotation, service
token rotation, API-key pepper/key reissue, console-session-secret rotation,
provider-key/webhook-secret rotation, rolling upgrades, audit-disk exhaustion,
and leader failover. A pepper change also changes the opaque active-wallet
metering ID, so coordinate it with the billing period and reconciliation rather
than rotating it blindly.

Primary references:

- [Vault integrated Raft storage](https://developer.hashicorp.com/vault/docs/configuration/storage/raft)
- [Vault integrated-storage operations](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)
- [Vault HA with Raft and TLS](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-tls)
- [Skaffold configuration](https://skaffold.dev/docs/references/yaml/)

## 10. Build the Veil adapter

The safe workstation boundary is the ordinary Linux/amd64 image, exported
tools, strict manifest tests, and configuration-only topology simulation:

```sh
cd services/nitro
make image
make tools
make test
make simulate
```

The Dockerfile verifies pinned Veil source, embeds license files, builds the
current Spiral Safe plugin and strict `spiral-node-config` renderer, and
includes `veil-daemon`. `make tools` exports `veil-proxy`, `veil-verify`, and
the renderer. `make test` runs renderer tests, shell syntax/ShellCheck, the
same-image simulation, and a mocked enrollment/membership test.

The checked-in image deliberately uses the shared
`spiral-safe-attestation.invalid` Veil FQDN. Do not replace it with per-node
image builds: official `veil-verify`
does not trust ordinary CA/hostname validation for this attestation endpoint.
It hashes the leaf certificate actually presented by the routed node and
requires that hash in a fresh nonce-bound Nitro attestation before PCR
comparison. Each manifest's `veilAddress` is therefore only the operator's
routable endpoint to that node. Port 8443 is attestation-only; never point a
generic PKI client or the Vault API at it.

Copy `same-image-manifest.template.json` to a protected path outside the source
tree and fill every placeholder. Bootstrap and joiner manifests must carry the
same operator-declared image digest, EIF SHA-384, source/Veil revisions, and AWS
KMS key ID, but distinct node IDs and durable storage. The renderer only checks
the OCI digest's syntax and equality in the simulation; signed release
provenance must bind it to the EIF before it can be an admission claim. A
bootstrap manifest has no leaders or join-client TLS; a joiner has at least one
HTTPS leader and mTLS join files.
Every manifest needs a window of at most 30 minutes, odd minimum voter count of
at least three, exact 18200/18201 tunnel binds, operator mTLS material paths,
WebAuthn RP values, and explicit external storage/route implementation names.

Validate the schema before a hardware launch:

```sh
NODE_MANIFEST=/secure/node-a.json CHECK_FILES=false make validate-config
```

`CHECK_FILES=false` is only for pre-provisioning schema review. Admission must
later validate the real trust/key files and their restrictive permissions.

On suitable Linux with Nitro CLI:

```sh
make image eif tools
```

On a dedicated enclave-enabled EC2 parent, after providing the private
cross-host L4 route and reviewing privileged TUN/VSOCK networking:

```sh
NODE_MANIFEST=/secure/node-a.json make run
```

The run path verifies the manifest/EIF hash, starts the privileged Veil host
proxy, and launches one manifest-named enclave. It does not use debug mode,
attach a console, terminate other enclaves, or stop the new enclave when the
proxy exits. Manage it explicitly by name. Vault waits inside for a manifest;
the repository intentionally supplies no generic upload endpoint.

From a clean checkout at the exact manifest source revision, perform the Veil
source/PCR/nonce/TLS proof before any identity is released:

```sh
NODE_MANIFEST=/secure/node-a.json \
SOURCE_DIR=/secure/clean-services-checkout \
make verify-admission
```

After that succeeds, an operator-owned admission service may return one
short-lived, attestation-bound bundle reference:

```sh
SPIRAL_ENROLLMENT_HOOK=/opt/spiral/bin/issue-enrollment \
NODE_MANIFEST=/secure/node-a.json \
SOURCE_DIR=/secure/clean-services-checkout \
make admit
```

The hook must be an absolute executable and its stdout must be exactly one
bounded `file:/...` or `secret-ref://...` reference. The repository does not
implement the receiver that resolves that reference inside the EIF or delivers
the manifest, TLS keys, and KMS identity after attestation. Until such a channel
exists, Vault cannot safely proceed past its wait boundary.

After an externally initialized/unsealed joiner has converged, use a short-lived
operator token and separate client certificate to prove exact voter identity,
minimum quorum, and healthy Autopilot:

```sh
VAULT_CACERT=/secure/operator-ca.crt \
VAULT_CLIENT_CERT=/secure/operator.crt \
VAULT_CLIENT_KEY=/secure/operator.key \
VAULT_TLS_SERVER_NAME=vault.internal \
VAULT_TOKEN=REDACTED \
NODE_MANIFEST=/secure/node-a.json \
make check-membership
```

Before accepting any real request, implement and test all three missing runtime
boundaries:

- a private per-node L4 forwarder/overlay from unique advertised API and Raft
  addresses to each host's reused `10.0.0.2` tunnel endpoint;
- durable, encrypted, rollback-protected, fenced storage with Raft-compatible
  ordering/fsync/crash semantics and tested snapshot/restore; and
- the attestation-bound manifest/TLS/shared-KMS identity delivery service,
  including recovery material and plugin bootstrap without parent-visible
  privileged credentials.

Then run a real three-node bootstrap/join/failover/removal/replacement cycle,
KMS outage/recovery, rollback attempt, backup/restore, altered-image rejection,
and nonce-bound independent attestation on enclave-enabled EC2. Neither
`make simulate` nor the mocked enrollment test is a live Vault quorum.

Follow the [Veil repository](https://github.com/Amnesic-Systems/veil), its
[examples](https://github.com/Amnesic-Systems/veil-examples), and the
[AWS Nitro Enclaves guide](https://docs.aws.amazon.com/enclaves/latest/user/).
Review GPL-3.0 corresponding-source obligations before distributing a Veil
binary or combined image.

## 11. Other repositories

Website:

```sh
cd website
git submodule update --init --recursive
docker build -t spiral-safe-website .
docker run --rm -p 127.0.0.1:8080:80 spiral-safe-website
```

Token list:

```sh
cd token-list
yarn install --frozen-lockfile
yarn build
yarn validate-PR
```

The token list is not a signer dependency. Its live partner-refresh commands
contact external data sources and should be treated as a separate data
maintenance workflow.

## Common troubleshooting

- **WebAuthn origin error:** use the exact origin configured in
  `WEBAUTHN_RP_ORIGINS`; `localhost` and `127.0.0.1` are different.
- **401 from the API:** re-enter the extension session token or use the
  configured tenant bearer token.
- **403 CORS/origin error:** align service CORS, extension site allowlist, and
  the real browser origin. Do not add wildcards.
- **403 `user_forbidden`:** add the exact username to that token's production
  `users` mapping or issue a different authorized credential. The request is
  rejected before Vault, so changing wallet storage cannot fix it.
- **403 `scope_forbidden`:** issue a key with the route's required scope; the
  client cannot elevate the stored key record.
- **402 `payment_required`:** the stored account is absent/inactive; reconcile
  the verified Stripe subscription webhook state before retrying.
- **402 `billing_provisioning_required`:** the Metronome runtime is enabled but
  the account's exact Stripe-provider mapping has not been operator-attested.
  Provision and sandbox-test it; do not bypass the gate.
- **429 `quota_exceeded`:** the plan's active-wallet or transaction limit is
  exhausted for the current stored billing period. Do not delete usage rows to
  evade accounting.
- **Console login loops or 403 form submission:** use the exact
  `CONSOLE_ORIGIN`; native forms require the same Origin plus the session CSRF
  token, and production cookies require HTTPS.
- **Billing pod will not start:** inspect missing PostgreSQL/plan/Stripe/
  Metronome configuration, the intentionally false global invoicing gate,
  migration permissions, database TLS, and NetworkPolicy/egress gateway.
- **409 after `/init`:** the wallet is already registered or registration is
  incomplete; choose the intended recovery/re-registration action.
- **Expired/used ceremony:** begin a new ceremony; completion is deliberately
  one-shot.
- **Extension not discovered:** build the adapter and extension, reload the
  unpacked extension, then refresh the dApp.
- **`/readyz` fails:** check Vault seal/leader state, service-account
  authentication, CA configuration and policy, plus PostgreSQL readiness when
  billing is enabled.
- **Raft pod will not join:** verify peer DNS/SANs, CA/client certificate, KMS
  identity, NetworkPolicy, initialization state, and existing quorum.
