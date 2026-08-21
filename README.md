<h1>Portiger Sentinel</h1>

**Container image security for Kubernetes that never leaves your cluster.**

Sentinel is an operator that scans image vulnerabilities, verifies cosign
signatures, decides what admission accepts, and keeps the record an auditor
asks for. All of it runs inside your cluster. It opens no connection to
Portiger — not at install, not at activation, not at renewal.

[sentinel.portiger.com](https://sentinel.portiger.com) · [Documentation](docs/) · [Every setting](docs/configuration.md) · [Report a bug](../../issues/new/choose)

> **This repository is documentation and the issue tracker.** The product
> source is not here. If something is broken, or the docs are wrong, open an
> issue — that is exactly what this repo is for.

## Install

```sh
helm repo add portiger https://charts.portiger.com
helm repo update

helm install sentinel portiger/portiger-sentinel \
  --namespace portiger-sentinel --create-namespace \
  --set clusterName=production-eu \
  --wait
```

The schema builds itself. Migrations are compiled into the binary and applied
on first connection, each recorded once in a `schema_migrations` table, so
there is no SQL to run and no migration step to forget on upgrade.

Full instructions: [docs/install.md](docs/install.md).

## What it does

| | |
|---|---|
| **Scanning** | Trivy, pinned into the image. Findings by severity, with the fixed version where one exists. |
| **Signatures** | cosign verification, by key or keyless, plus in-toto attestations with a maximum age. |
| **Admission** | Decides from a warm verdict cache — it never scans inline, so it never times out a deployment. |
| **Evidence** | Period reports mapped to a control set, in four honest states: `met`, `partial`, `gap`, `outside_scope`. |
| **Triage** | Accept a risk with an expiry and a reason; break-glass when a deployment cannot wait. Both recorded. |
| **Notifications** | Slack, Teams, Discord, Telegram, e-mail, webhooks — naming the installation, never its topology. |

## Editions

Community Edition is free and unlimited in time. Every security capability is
in it — a community edition with the security removed would deserve to lose to
the free tools. What a key buys is scale: more clusters, more frameworks, a
larger image allowance, plus single sign-on and strict admission.

See [pricing](https://sentinel.portiger.com/#pricing).

## Known limits

These are stated here for the same reason they are on the site: an install
should never surprise you.

- **No fleet view.** Each cluster is independent, with its own database and
  console. A page summarising many installations is planned for the next
  version.
- **Scan verdicts are amd64.** On a multi-architecture image the finding
  describes the `linux/amd64` child of the index, not the variant an arm64 node
  runs. The operator image itself is built and signed for both.
- **Evidence is JSON.** The console renders it for printing; there is no
  server-side PDF.
- **The transparency log needs a route out.** An air-gapped cluster sets
  `scan.cosignOffline: true`, which verifies the signature against the key and
  skips the Rekor check. That is a real reduction, so it is a deliberate
  setting rather than a silent fallback.
- **The operator does not sign for you.** cosign signing and attestation work
  from the CLI, but there is no key management inside the operator.

## Verify what you run

A product that tells you to verify signatures should be verifiable itself.

```sh
cosign verify \
  --key https://sentinel.portiger.com/portiger.pub \
  portiger/portiger-sentinel:1.0.0
```

## Security

Do not open a public issue for a vulnerability in Sentinel. See
[SECURITY.md](SECURITY.md).
