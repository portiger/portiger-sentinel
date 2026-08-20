# Security policy

## Reporting a vulnerability

Write to **security@portiger.com**. Please do not open a public issue for a
security defect in Sentinel — the point of a private channel is that a fix can
reach installations before the details do.

Tell us what you found, how to reproduce it, and what you think the impact is.
You will get an acknowledgement within three working days and an assessment
within ten.

## What we will do

- Confirm or refute the finding, and say which.
- Agree a disclosure date with you. We will not sit on a confirmed defect
  waiting for a convenient release.
- Credit you in the advisory unless you would rather we did not.

## Scope

In scope: the operator, its API, its console, the Helm chart, and the release
artefacts and their signatures.

Out of scope: findings in Trivy, cosign or PostgreSQL themselves — report those
upstream. We do want to hear about it if a version we pin carries a known
defect and we have not moved.

## What Sentinel sends

Nothing. There is no usage reporting, no crash reporting, no update check and
no licence call-back. If you find network egress from the operator that is not
a registry you configured, a database you configured, or a notification channel
you configured, treat that as a security report — because it would be one.
