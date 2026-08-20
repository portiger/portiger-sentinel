# Known limits

Listed so an install never surprises you. Each of these is a thing the product
does not do today, stated before you find it out yourself.

## No fleet view

Each cluster runs its own operator, database and console, and they are
independent by design — which is exactly why nothing has to cross a cluster
boundary. There is no single page summarising many installations.

Planned for the next version. We would rather say that than ship half of it.

## Scan verdicts are amd64

Scans and reports are recorded against `linux/amd64`. On a multi-architecture
image the finding describes the amd64 child of the index, not the variant an
arm64 node will actually run.

The operator image itself is built and signed for both architectures — this
limit is about what a verdict describes, not where Sentinel runs.

## Evidence is JSON

The package carries everything an auditor asks for and the console renders it
for printing, but there is no server-side PDF.

## The transparency log needs a route out

See [air-gapped.md](air-gapped.md#3-the-transparency-log).

## The operator does not sign on your behalf

`cosign sign` and `attest` are wired and work from the CLI, but nothing in the
operator signs for you: there is no key management, and the AttestationRenewal
CRD is defined and not yet reconciled.

## Framework and cluster counts are contractual, not enforced

The licence key carries both and the operator currently reads neither, so one
key activates on any number of clusters and generates every framework it has a
mapping for. The image band *is* enforced.

These remain the contractual limits of your agreement. We would rather write
this down than let you discover the gap and wonder what else is unstated.
