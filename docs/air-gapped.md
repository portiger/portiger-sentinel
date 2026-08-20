# Air-gapped clusters

Three things normally reach the internet. All three can be pointed inward.

## 1. The operator image

Mirror it into your registry and set `image.repository`. The image is
multi-architecture and signed; verify it before mirroring.

## 2. The vulnerability database

Trivy can pull its database from any OCI registry. Mirror the repositories once
with whatever already moves images inside your perimeter:

```sh
crane copy ghcr.io/aquasecurity/trivy-db:2      registry.internal/trivy-db:2
crane copy ghcr.io/aquasecurity/trivy-java-db:1 registry.internal/trivy-java-db:1
```

Then name them:

```yaml
trivy:
  dbRepository:     registry.internal/trivy-db:2
  javaDbRepository: registry.internal/trivy-java-db:1
```

The unpacked database is measured in gigabytes, which is why it travels as an
OCI artifact — the same way every other image in such a cluster arrives —
rather than as a mounted bundle. Refresh it on whatever schedule you refresh
your mirror; the operator reports the database's age and version on the About
page and in every evidence document, so staleness is visible rather than
silent.

## 3. The transparency log

This is the one that is not fully solved yet, and it is stated plainly rather
than hidden in a footnote.

cosign checks that a signature was recorded in Rekor, and it fetches Rekor's
public keys from the Sigstore TUF repository over the internet. In a cluster
with no route out, that turns every verification into a DNS failure — which
would otherwise be recorded as "unsigned", which is worse than useless.

The current answer:

```yaml
scan:
  cosignOffline: true
```

This verifies the signature against the key and skips the log check. It is a
real reduction in what a signature proves — transparency is what makes a
signature auditable by someone other than its signer — so it is a deliberate
setting, not a fallback the operator takes on its own.

**The better answer, on the roadmap:** the pinned cosign accepts a Sigstore
trust root read from disk. Fetched once where there is a route out and carried
in like any other artefact, it lets an air-gapped cluster verify the
transparency log properly instead of skipping it.

## Licensing

The licence is an offline signed token. Activation makes no network call, so an
air-gapped installation activates and renews exactly like a connected one.

## What is left

With the above in place, block all egress from the namespace and nothing stops
working. The operator makes no call to Portiger at any point.
