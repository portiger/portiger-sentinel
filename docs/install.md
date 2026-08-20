# Install

## Requirements

- Kubernetes 1.27 or later, any conformant distribution. Tested on k3s,
  minikube and kubeadm clusters.
- Helm 3.
- PostgreSQL 14 or later, or let the chart install one.
- A storage class, if you keep the vulnerability database cache on a volume.
  Recommended: without it every scanner restart re-downloads the database.
- Node architecture `linux/amd64` or `linux/arm64`. The release image carries
  both.

## Install

```sh
helm repo add portiger https://charts.portiger.com
helm repo update

helm install sentinel portiger/portiger-sentinel \
  --namespace portiger-sentinel --create-namespace \
  --set clusterName=production-eu \
  --wait
```

Set `clusterName`. It is the label every notification and evidence document
uses to say which installation it came from, and it cannot be inferred from the
cluster.

Then reach the console:

```sh
kubectl -n portiger-sentinel port-forward svc/sentinel 8080:8080
helm -n portiger-sentinel get notes sentinel   # first-run credentials
```

There is no ingress by default. An admission controller's console is not
something to expose without deciding to.

## Database

**The schema builds itself.** Migrations are compiled into the binary and
applied on first connection, each recorded in a `schema_migrations` table so it
runs exactly once. An upgrade applies whatever is new and skips the rest. There
is no SQL for you to run and no migration step to forget.

To use a PostgreSQL you already operate, hand over a DSN in a Secret rather
than in values — a DSN in a values file ends up in your Helm release history.

```sh
kubectl -n portiger-sentinel create secret generic sentinel-db \
  --from-literal=dsn='postgres://user:pass@db.internal:5432/sentinel?sslmode=require'

helm upgrade sentinel portiger/portiger-sentinel \
  --namespace portiger-sentinel \
  --set postgresql.enabled=false \
  --set externalDsnExistingSecret.name=sentinel-db \
  --set externalDsnExistingSecret.key=dsn
```

## Licence key

Community Edition needs no key.

An Enterprise key is a signed token carrying your claims — edition, cluster
count, framework count, image band, expiry. The operator verifies it against a
public key compiled into the binary, locally, on every start, with no network
call. Because the term is annual, that one local verification is all it ever
performs.

```sh
kubectl -n portiger-sentinel create secret generic sentinel-licence \
  --from-file=key=./licence.txt

helm upgrade sentinel portiger/portiger-sentinel \
  --namespace portiger-sentinel \
  --set license.existingSecret.name=sentinel-licence
```

When the term ends the installation falls back to Community Edition. Scanning
and admission keep running under the free limits. Nothing is locked and no
deployment is denied because of a billing date — a security control that
switches itself off over an invoice is a worse outcome than an unpaid one.

## Upgrade and uninstall

Upgrade with `helm upgrade`. New migrations apply on start. Admission keeps
answering throughout: replicas roll one at a time, and the webhook's
certificate is picked up without a restart.

Uninstalling removes the operator and its webhook registration. Your database
is not dropped — the record of what was decided outlives the tool that decided
it, which is the point of keeping it in your own PostgreSQL.

## Verify the release

```sh
cosign verify \
  --key https://sentinel.portiger.com/portiger.pub \
  portiger/portiger-sentinel:1.0.0
```

Both architectures are signed. Verify before you mirror the image inward.
