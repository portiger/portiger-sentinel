# Configuration

Everything this operator can be told, and where each thing is told to it.

Written because it was scattered: some settings are chart values, some are
environment variables the chart renders, some live only in the console, and
until this page existed the only way to know which was which was to read
`values.yaml`, `config.go` and the templates side by side. Somebody installing
from a chart repository has none of those open.

## How configuration arrives

Three surfaces, in order of precedence.

| | Surface | Wins over | Survives a reinstall |
|---|---|---|---|
| 1 | **Chart values** — `values.yaml`, rendered into env and Secrets | everything below | yes, it is your manifest |
| 2 | **The database** — set through the console | nothing above it | yes, unless you drop the volume |
| 3 | **Defaults** compiled into the binary | — | — |

Where a setting exists on more than one surface, **the chart wins and is
reconciled at every start.** That is deliberate and it is the same rule for the
licence key and for single sign-on: an installation that describes something in
its manifests should not have that description quietly overwritten by a click,
and a click that would be undone by the next restart is refused rather than
accepted.

Anything the chart does not supply stays editable in the console exactly as
before. Adding a declarative path never removes the interactive one.

## Install

```sh
helm repo add portiger https://charts.portiger.com
helm repo update

helm install sentinel portiger/portiger-sentinel \
  --namespace portiger-sentinel --create-namespace \
  --set clusterName=production-eu \
  --wait
```

`clusterName` is the one value worth setting on the first install even in a
trial. It is the label every notification and every evidence document uses to
say which installation it came from, and it cannot be inferred from the
cluster.

---

## Identity and scale

| Value | Default | What it does |
|---|---|---|
| `clusterName` | `""` | Names this installation in notifications and evidence. Never a hostname — nothing topological leaves the cluster. |
| `replicaCount` | `2` | Webhook replicas. Spread across nodes by anti-affinity; one is enough to serve, two to survive a node. |
| `scanner.replicas` | `1` | Scan workers. Raising this with a `ReadWriteOnce` cache is refused at install — the volume binds to one node. |
| `image.repository` | `docker.io/portiger/sentinel-k8s` | Point at your mirror for an air-gapped install. |
| `image.tag` | chart appVersion | |

## Database

The schema builds itself. Migrations are compiled into the binary and applied
on first connection, each recorded in `schema_migrations` so it runs exactly
once. There is no SQL for you to run and no migration step to forget on
upgrade.

| Value | Default | What it does |
|---|---|---|
| `postgresql.enabled` | `true` | Install a PostgreSQL alongside. Turn off to use your own. |
| `postgresql.persistence.size` | `20Gi` | |
| `postgresql.auth.password` | `""` | Generated when empty. |
| `externalDsnExistingSecret.name` | `""` | **Preferred** for an external database. |
| `externalDsnExistingSecret.key` | `dsn` | |
| `externalDsn` | `""` | A DSN inline. Avoid: it lands in your Helm release history. |

```sh
kubectl -n portiger-sentinel create secret generic sentinel-db \
  --from-literal=dsn='postgres://user:pass@db.internal:5432/sentinel?sslmode=require'

helm upgrade sentinel portiger/portiger-sentinel \
  --set postgresql.enabled=false \
  --set externalDsnExistingSecret.name=sentinel-db
```

## Admission

| Value | Default | What it does |
|---|---|---|
| `webhook.failurePolicy` | `Ignore` | With `Ignore`, an operator outage means unguarded, never blocked. `Fail` denies until the operator answers — Enterprise, and a deliberate posture. |
| `webhook.coldImagePolicy` | `allowAndScan` | What happens to a digest never seen before. The alternative denies it, which is `Fail`'s companion. |
| `webhook.timeoutSeconds` | `15` | The API server's patience. Admission decides from cache, so this is a ceiling, not a budget. |
| `webhook.bypassNamespaces` | `[]` | Namespaces admission does not review. The operator's own is always exempt. |
| `webhook.tls.certManager.enabled` | `false` | Off by default: the operator issues and rotates its own pair, and every replica picks up a new one without a restart. |

## Scanning and the supply chain

| Value | Default | What it does |
|---|---|---|
| `scan.workers` | `2` | Concurrent scans per scanner pod. |
| `scan.timeout` | `10m` | Per scan. |
| `scan.rescanInterval` | `24h` | How often a known digest is re-evaluated against a refreshed database. |
| `scan.insecureRegistries` | `[]` | Registries served over plain HTTP. Must be named — silently downgrading a connection is not a decision a security tool makes for you. |
| `scan.cosignOffline` | `false` | Verify signatures against the key and skip the transparency-log check. A real reduction in what a signature proves; see [air-gapped](#air-gapped-clusters). |

Private registries need nothing configured: the operator resolves the same
image pull secrets the pod references, and those on its service account, and
uses them for both the scan and the signature check.

## Air-gapped clusters

| Value | Default | What it does |
|---|---|---|
| `trivy.dbRepository` | `""` | OCI reference for the vulnerability database inside your perimeter. |
| `trivy.javaDbRepository` | `""` | The Java artifact database — only reached when scanning Java images, and unreachable at exactly the wrong moment if unset. |
| `trivy.dbRegistrySecret` | `""` | A `dockerconfigjson` Secret holding the credential for that mirror. Almost every registry inside a perimeter asks for one. |
| `trivy.updateIntervalHours` | `6` | |
| `trivy.cache.persistence.*` | `5Gi`, RWO | Without a volume, every scanner restart re-downloads the database. |

```sh
crane copy ghcr.io/aquasecurity/trivy-db:2      registry.internal/trivy-db:2
crane copy ghcr.io/aquasecurity/trivy-java-db:1 registry.internal/trivy-java-db:1

kubectl -n portiger-sentinel create secret docker-registry trivy-db-auth \
  --docker-server=registry.internal --docker-username=... --docker-password=...
```

```yaml
trivy:
  dbRepository:      registry.internal/trivy-db:2
  javaDbRepository:  registry.internal/trivy-java-db:1
  dbRegistrySecret:  trivy-db-auth
```

The database is measured in gigabytes, which is why it travels as an OCI
artifact — the same way every other image in such a cluster arrives — and not
as a mounted bundle.

**The transparency log is the one part not yet solved.** cosign fetches Rekor's
public keys from the Sigstore TUF repository over the internet, so an
air-gapped cluster sets `scan.cosignOffline: true` and the log check is
skipped. Transparency is what makes a signature auditable by someone other than
its signer, so this is stated as a setting rather than taken as a fallback.

## Single sign-on (Enterprise)

OAuth 2.0 and OpenID Connect are one surface here, not two: the client id,
secret, scopes and redirect are the authorization-code flow, and the issuer
with the `openid` scope is what makes it OIDC. A provider is described once.

| Value | Default | What it does |
|---|---|---|
| `sso.enabled` | `false` | |
| `sso.kind` | `""` | `keycloak`, `entra`, `okta`, `google`, `github`, `gitlab`, or empty for a generic provider. |
| `sso.name` | `Single sign-on` | The label on the sign-in button. |
| `sso.baseUrl` | `""` | The directory's own address. |
| `sso.tenant` | `""` | Keycloak realm, or Entra directory id. |
| `sso.issuer` | `""` | Only for `kind: ""`. Derived for the named kinds, so the two cannot disagree. |
| `sso.clientId` | `""` | |
| `sso.scopes` | `openid profile email` | |
| `sso.redirectUri` | `""` | Where the directory sends the browser back. An address a browser can reach, which the cluster cannot infer. |
| `sso.existingSecret.name` | `""` | Holds the client secret. **Required** — the chart refuses to render without it. |

`kind` decides how the issuer is derived, so most installations name a server
rather than an issuer URL:

| kind | derived from | issuer |
|---|---|---|
| `keycloak` | `baseUrl` + `tenant` | `{baseUrl}/realms/{tenant}` |
| `entra` | `tenant` | `login.microsoftonline.com/{tenant}/v2.0` |
| `okta` | `baseUrl` | the organisation URL |
| `google` | — | fixed |
| `github`, `gitlab` | `baseUrl`, or the public host | |
| `""` | `issuer`, stated | any other OIDC provider |

```sh
kubectl -n portiger-sentinel create secret generic sentinel-sso \
  --from-literal=client-secret='...'
```

```yaml
sso:
  enabled: true
  kind: keycloak
  name: "Corporate directory"
  baseUrl: https://id.example.com
  tenant: corporate
  clientId: portiger-sentinel
  redirectUri: https://sentinel.example.com/api/v1/auth/sso/callback
  existingSecret:
    name: sentinel-sso
```

A provider supplied this way is **owned by the chart**: reconciled at every
start, shown in the console but not editable there, and removed when you remove
the block and upgrade. An installation that leaves `sso.enabled: false`
configures providers in the console exactly as before.

**Single sign-on authenticates; it does not provision.** Somebody who signs in
through the directory must already have an account here or they are refused. A
directory able to mint accounts would make everyone in it an operator of this
cluster's admission control.

## Licence

| Value | Default | What it does |
|---|---|---|
| `license.existingSecret.name` | `""` | **Preferred.** |
| `license.existingSecret.key` | `license-key` | |
| `license.key` | `""` | Inline. Avoid: it lands in your release history. |
| `defaultImageQuota` | `15` | The free allowance. A key raises it. |

The key is an Ed25519-signed token verified locally against a public key
compiled into the binary. Activation makes no network call, which is what lets
it run in an air-gapped cluster; because the term is annual, that one local
verification is all it ever performs.

A key supplied by the chart is pinned: a stored key cannot win it back, so a
chart upgrade stays the thing that changes the entitlement. When the term ends
the installation falls back to Community Edition — scanning and admission keep
running under the free limits, and nothing is denied over a billing date.

## Default policy

Seeded once by a post-install hook and owned by you afterwards — the console,
`kubectl`, or whatever GitOps controller runs the cluster. It is deliberately
not a chart-managed resource: that would revert your edits on every upgrade.

| Value | Default |
|---|---|
| `defaultPolicy.create` | `true` |
| `defaultPolicy.enforcement` | `Enforce` |
| `defaultPolicy.blockLatestTag` | `true` |
| `defaultPolicy.thresholds.blockCritical` | `true` |
| `defaultPolicy.thresholds.blockHigh` | `true` |
| `defaultPolicy.digestEnforcement` | `AllowTags` |
| `defaultPolicy.requireSignature` | `false` |
| `defaultPolicy.requireSBOM` | `false` |
| `defaultPolicy.slaWindows.critical` | `24h` |
| `defaultPolicy.slaWindows.high` | `168h` |
| `defaultPolicy.triageDefaults.requireJustification` | `true` |

Without it admission still enforces: the operator falls back to a built-in
policy rather than admitting everything.

## Observability

| Value | Default |
|---|---|
| `observability.logLevel` | `info` |
| `observability.logFormat` | `json` |
| `observability.logComponentLevels` | `""` — e.g. `admission=debug,scan=warn` |
| `observability.otlp.endpoint` | `""` — empty means nothing is exported |
| `observability.otlp.protocol` | `grpc` |
| `observability.traces.sampleRatio` | `0.1` |
| `metrics.serviceMonitor.enabled` | `false` |

## Updates

| Value | Default | What it does |
|---|---|---|
| `updates.endpoint` | `""` | A release feed to poll. **There is deliberately no default host**: an installation that was not told where to look never reaches out. |
| `updates.interval` | `24h` | |

---

## What is still console-only

Stated here rather than discovered during an install. These are configured
after the operator is running, and an installation rebuilt from its manifests
does not get them back:

| | Where it lives | Why it matters |
|---|---|---|
| **Notification channels** | `notification_channels` | Slack, Teams, e-mail, webhooks. |
| **SMTP** | `settings` | The mail relay for e-mail notifications. |
| **Organisation profile** | `settings` | Legal name, address, security contact, disclosure policy — the manufacturer identification that appears **in the evidence document**. A fresh installation produces evidence that cannot name who operates the control until somebody fills this in. |
| **Audit retention** | `settings` | Days of audit log kept. |

Single sign-on was on this list until it was moved to the chart, and the same
pattern applies to the rest. They are named here so an estate that expects
every configuration change to arrive through a reviewed one knows what it is
taking on before it installs, not afterwards.

## Environment variables

The chart renders these; they are listed because an operator debugging a pod
sees the variable, not the value that produced it.

`API_PORT` · `BYPASS_NAMESPACES` · `CERT_SECRET_NAME` · `CLUSTER_NAME` ·
`COLD_IMAGE_POLICY` · `COSIGN_BINARY` · `COSIGN_OFFLINE` · `DATABASE_DSN` ·
`DEFAULT_IMAGE_QUOTA` · `DEFAULT_POLICY_PATH` · `DOCKER_CONFIG` ·
`INSECURE_REGISTRIES` · `LICENSE_KEY` · `LOG_COMPONENT_LEVELS` · `LOG_FORMAT` ·
`LOG_LEVEL` · `METRICS_PORT` · `NODE_NAME` · `OPERATOR_ROLE` · `OTEL_*` ·
`POD_NAME` · `POD_NAMESPACE` · `RESCAN_INTERVAL` · `SCAN_TIMEOUT` ·
`SCAN_WORKERS` · `SECURE_COOKIES` · `SESSION_SECRET` · `SESSION_TTL` ·
`SSO_BASE_URL` · `SSO_CLIENT_ID` · `SSO_CLIENT_SECRET` · `SSO_ISSUER` ·
`SSO_KIND` · `SSO_NAME` · `SSO_REDIRECT_URI` · `SSO_SCOPES` · `SSO_TENANT` ·
`TLS_CERT_PATH` · `TLS_KEY_PATH` · `TRIVY_BINARY` · `TRIVY_CACHE_DIR` ·
`TRIVY_DB_REPOSITORY` · `TRIVY_JAVA_DB_REPOSITORY` · `UPDATE_CHECK_INTERVAL` ·
`UPDATE_CHECK_URL` · `WEBHOOK_CONFIG_NAME` · `WEBHOOK_PORT` ·
`WEBHOOK_SERVICE_DNS`

`OPERATOR_ROLE` is `all`, `webhook` or `scanner`, and a misspelling stops the
process at start rather than producing a pod that runs and does nothing.

`TRIVY_DB_REPOSITORY`, `TRIVY_JAVA_DB_REPOSITORY` and `DOCKER_CONFIG` are read
by Trivy itself, not by the operator.
