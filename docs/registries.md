# Private registries

Nothing to configure in the common case.

The operator resolves the same image pull secrets the pod itself references,
and those attached to its service account, and uses them for both the scan and
the signature check. A signature lives beside the image it signs, so it is
behind the same credential.

The credential is never written to the image record, and never appears in a
notification.

## Plain HTTP

A registry served over plain HTTP has to be named. Silently downgrading a
connection is not a decision a security tool should make for you:

```yaml
scan:
  insecureRegistries:
    - registry.internal:5000
```

## When a scan cannot reach a registry

Admission answers within its budget per your cold-image policy — it never
blocks on the network. The scan fails with the reason recorded on the job and
on the image, and stops after three attempts rather than retrying forever.
