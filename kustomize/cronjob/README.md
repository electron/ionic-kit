# cronjob

A `CronJob` that runs the app's own image with a different `command`. Each one
replaces a single Heroku Scheduler entry. It uses the same security context,
ServiceAccount and `server-secrets` as `node-web-app`, so the job sees the
same env vars as the web process.

## Overlay contract

Kustomize won't accept two resources with the same name, so each schedule gets
its own small kustomization that renames the job:

```
deploy/base/
  kustomization.yaml          # resources: [ ../../node-web-app URL, cron-sync ]
  cron-sync/kustomization.yaml
```

```yaml
# deploy/base/cron-sync/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/electron/ionic-kit//kustomize/cronjob?ref=<sha>
nameSuffix: -sync            # CronJob/cron -> CronJob/cron-sync
patches:
  - target: { kind: CronJob }
    patch: |-
      - op: replace
        path: /spec/schedule
        value: "*/20 * * * *"
      - op: replace
        path: /spec/jobTemplate/spec/template/spec/containers/0/command
        value: ["node", "lib/permissions/run.js"]
```

The parent `deploy/base` still handles the image rename (`images:`), the
namespace and the `part-of` label, and those apply to the CronJob too.

The base sets `timeZone` to UTC. Heroku Scheduler also uses UTC, so schedules
carry over unchanged. Dead-man-snitch style `&& curl …` suffixes become
`command: ["sh", "-c", "node lib/x.js && curl …"]`; the Node image includes
`sh`.
