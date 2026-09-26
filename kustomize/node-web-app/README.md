# node-web-app

A `Deployment`, `Service` and `Ingress` for an HTTP service listening on
`$PORT` (8080).

Everything is named `server`: the Deployment, Service, Ingress, ServiceAccount
and the `server-secrets` Secret. That matches the `components = { server = … }`
entry in `electron/infra`, which provisions the ServiceAccount and the
ExternalSecret under those names.

## Overlay contract

Reference it by URL, pinned to a commit:

```yaml
resources:
  - https://github.com/electron/ionic-kit//kustomize/node-web-app?ref=<sha>
```

Your overlays must set:

| What | How |
| --- | --- |
| Namespace | `namespace: <app>` |
| Image | `images: [{ name: electronionicplatform.azurecr.io/PLACEHOLDER_APP, newName: electronionicplatform.azurecr.io/<app>, newTag: main }]` |
| Hostname | JSON patch on `Ingress/server`: replace `/spec/rules/0/host` |
| `part-of` label | `labels: [{ pairs: { app.kubernetes.io/part-of: <app> }, includeTemplates: true }]` |

Optional settings. Apart from replicas, these are JSON patches on
`Deployment/server`:

| What | Path |
| --- | --- |
| Probe path (default `/healthz`) | `/spec/template/spec/containers/0/livenessProbe/httpGet/path` and `…/readinessProbe/httpGet/path` |
| Plain env vars | add to `/spec/template/spec/containers/0/env/-` |
| Resources | `/spec/template/spec/containers/0/resources` |
| Replicas | `replicas: [{ name: server, count: N }]` |
| More scratch space | replace `/spec/template/spec/volumes/0` with `emptyDir: { sizeLimit: 10Gi }` or a PVC (with a PVC, also set `strategy: { type: Recreate }`) |

`templates/deploy/` in this repo is a filled-in example of all of the above.

## What the pod gets

* `PORT=8080`, plus every key of `server-secrets` as an env var.
* A read-only root filesystem with a writable `emptyDir` at `/tmp`.
* No service-account token, no capabilities, uid/gid 1000, `RuntimeDefault` seccomp.
  The cluster enforces these through Azure Policy, so an image that needs root
  won't run.
