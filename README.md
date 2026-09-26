# ionic-kit

Shared building blocks for apps on the ionic platform (Electron's AKS cluster):
a reusable build-and-deploy workflow, kustomize bases and a Node Dockerfile.
Moving an app off Heroku should mean copying three files, not writing thirty.

This repo is public on purpose. Public app repos can only call reusable
workflows that live in public repos, and ArgoCD can only fetch a remote
kustomize base without credentials if the repo is public. Nothing in here is
secret.

## Adopting the kit

1. Copy `templates/Dockerfile.node` to `Dockerfile`. It runs as non-root on
   `PORT=8080` with a read-only root filesystem.
2. Copy `templates/build-image.yml` to `.github/workflows/build-image.yml`. Set
   `app` and pin `uses:` to a commit SHA of this repo.
3. Copy `templates/deploy/` to `deploy/`. Set the app name, the hostname and
   the base's `?ref=<sha>`.

Then the app gets an entry in `electron/infra`, which creates the namespace,
the ExternalSecret, the DNS record and the repo secrets the workflow needs.
The full migration runbook lives in `electron/infra` at
`terraform/global/ionic-platform-apps/migrating-from-heroku.md`.

## Pin by SHA

Apps pin this repo by commit SHA in both places (`uses:` and `?ref=`), with the
tag in a comment next to it. A tag can be moved; a SHA can't. That way every
kit upgrade goes through the app repo's normal review.

## What lives where

| Here | `electron/infra` (wg-infra) | App repo |
| --- | --- | --- |
| Reusable deploy workflow | Namespace, ServiceAccount, ExternalSecret `server-secrets`, `origin-tls` | `Dockerfile` |
| kustomize bases (`node-web-app`, `cronjob`) | Key Vault + secrets, CI-push identity, ArgoCD project/app | `deploy/` overlays |
| Templates | Repo secrets/variables, Cloudflare DNS, migration runbook | Caller workflow |

## Security notes

* **No secrets in the kit.** The workflow receives four repo secrets from the
  caller, each passed by name (never `secrets: inherit`). The only
  infrastructure detail in here is the ACR registry name, and pushing to or
  pulling from it still needs Azure credentials. The ArgoCD hostname and the
  shared vault name come in through the caller's repo variables.
* **Callers pin SHAs.** A compromised tag in this repo can't reach an app that
  pins a SHA. Every action the kit itself uses is pinned to a SHA too.
* **Narrowly scoped credentials.** CI authenticates to Azure with OIDC only, so
  there is no Azure client secret to leak. The ArgoCD Cloudflare Access token
  is read from Key Vault at run time, the ArgoCD token can only `get`, `update`
  and `sync` one app, and ACR push is limited by ABAC to one repository. A
  leaked `ARGOCD_TOKEN` can redeploy that one app and nothing else.
* **The pod security context is not optional.** Azure Policy on the cluster
  rejects root containers and privilege escalation. The base sets compliant
  defaults, and overlays shouldn't loosen them.

## Repo layout

```
.github/workflows/deploy.yml   reusable workflow (workflow_call)
.github/workflows/ci.yml       this repo's own checks
kustomize/node-web-app/        remote base: Deployment + Service + Ingress
kustomize/cronjob/             remote base: Heroku Scheduler replacement
kustomize/examples/            local-path overlay used by CI
templates/                     files an app copies
```

License: MIT.
