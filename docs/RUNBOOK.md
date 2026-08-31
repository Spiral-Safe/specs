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

The API token and pending ceremony bindings disappear when extension session
storage is cleared; re-enter the token and begin a fresh ceremony after a
browser restart if required. Ordinary Manifest V3 worker suspension does not
clear them. Rebuild and reload the unpacked extension after source changes.

From an allowlisted demo page, `spiralSafe:config`/`config.get` should expose
neither the API token nor any credential embedded in a backend or RPC URL path;
the public URLs are reduced to their origins. The full configured URLs remain
inside the trusted worker for requests.

## 5. Test the plugin, service, and SDK

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

## 6. Run bounded endpoint load tests

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

## 7. Render and run the local Kubernetes profile

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

## 8. Prepare the production Kubernetes profile

The production overlay is intentionally incomplete until the operator supplies:

1. a registry and immutable-image policy;
2. an encrypted, topology-aware StorageClass and snapshot policy;
3. a `vault-tls` Secret with `tls.crt`, `tls.key`, and `ca.crt`, covering
   `vault-active`, `vault-internal`, and StatefulSet peer DNS names;
4. a `vault-unseal` ConfigMap with `AWS_REGION` and
   `VAULT_AWSKMS_SEAL_KEY_ID`, plus a narrowly scoped workload identity;
5. a `spiral-service-auth` Secret containing `API_TOKEN_HASHES`, generated
   through the platform secret manager rather than committed plaintext. Each
   digest must map to `{"tenant":"...","users":["..."]}` with a non-empty
   username allowlist;
6. replacement of every `REPLACE_ME.invalid` WebAuthn/CORS value;
7. TLS ingress/Gateway and DNS resources;
8. reviewed namespace/pod labels for ingress and operator NetworkPolicy access;
9. backup, immutable audit export, metrics, alerts, and recovery ownership.

Do not add `VAULT_TOKEN` to the service Secret: production startup rejects
every static Vault token and authenticates through `VAULT_K8S_ROLE`. The
operator-only `VAULT_TOKEN` shown later is for an administrative CLI process,
not an environment variable on the HTTP adapter.

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
token rotation, rolling upgrades, audit-disk exhaustion, and leader failover.

Primary references:

- [Vault integrated Raft storage](https://developer.hashicorp.com/vault/docs/configuration/storage/raft)
- [Vault integrated-storage operations](https://developer.hashicorp.com/vault/docs/concepts/integrated-storage)
- [Vault HA with Raft and TLS](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/examples/ha-tls)
- [Skaffold configuration](https://skaffold.dev/docs/references/yaml/)

## 9. Build the Veil adapter

The safe workstation boundary is the ordinary container image and exported
tools:

```sh
cd services/nitro
VEIL_FQDN=vault.example.com \
WEBAUTHN_RP_ID=wallet.example.com \
WEBAUTHN_RP_ORIGINS=https://wallet.example.com \
make image
make tools
```

These values are measured image inputs. Use the intended immutable release
values before an EIF build. The Dockerfile verifies pinned Veil source, embeds
license files, builds the local Spiral Safe plugin, and includes
`veil-daemon`; `make tools` exports `veil-proxy` and `veil-verify`.

On suitable Linux with Nitro CLI:

```sh
make eif
```

On a dedicated enclave-enabled EC2 parent, after reviewing the script and
networking:

```sh
make run
```

The run path starts the privileged Veil host proxy and one named enclave. It
does not use debug mode, attach a console, terminate other enclaves, or stop the
new enclave when the proxy exits. Manage it explicitly by name.

Before accepting any real request, implement and test:

- durable, attested external persistence;
- nonce-bound independent verification with `veil-verify`;
- PCR/application-hash release policy;
- attestation-aware TLS pinning and Vault initialization;
- recovery-material delivery that the EC2 parent cannot observe;
- attestation-bound KMS/workload identity;
- plugin registration without embedded privileged credentials;
- full restart, upgrade, backup, restore, and disaster-recovery behavior.

Follow the [Veil repository](https://github.com/Amnesic-Systems/veil), its
[examples](https://github.com/Amnesic-Systems/veil-examples), and the
[AWS Nitro Enclaves guide](https://docs.aws.amazon.com/enclaves/latest/user/).
Review GPL-3.0 corresponding-source obligations before distributing a Veil
binary or combined image.

## 10. Other repositories

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
- **409 after `/init`:** the wallet is already registered or registration is
  incomplete; choose the intended recovery/re-registration action.
- **Expired/used ceremony:** begin a new ceremony; completion is deliberately
  one-shot.
- **Extension not discovered:** build the adapter and extension, reload the
  unpacked extension, then refresh the dApp.
- **`/readyz` fails:** check Vault seal/leader state, service-account
  authentication, CA configuration, and policy.
- **Raft pod will not join:** verify peer DNS/SANs, CA/client certificate, KMS
  identity, NetworkPolicy, initialization state, and existing quorum.
