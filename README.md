# gha-runners-controller

## Purpose

This values-only wrapper installs the upstream Actions Runner Controller chart
`gha-runner-scale-set-controller` at version `0.14.2`. It keeps the live controller
settings without a copy of the upstream templates. CI publishes it to GHCR.

## Before first install

1. Use Helm at `/opt/homebrew/bin/helm`.
2. Make sure `arc-systems` and `arc-runners` exist. This chart does not create them.
3. Use release name `gha-runners-controller` and namespace `arc-systems`.
   The required anti-affinity selector and the gha-runners
   `controllerServiceAccount` (`gha-runners-controller-gha-rs-controller`) use this name.
4. Build dependencies with `/opt/homebrew/bin/helm dependency build charts`.
5. After publication, replace `PENDING_CI_VERSION` in
   `cluster-config/argo/dev/gha-runners-controller.yaml` with the first CI chart version.
6. Review the Argo CD diff before a manual sync.

## Values reference

All controller values are below `gha-runner-scale-set-controller`.
The upstream values object stays open; unknown wrapper-level keys are rejected.

| Key below the dependency | Wrapper default | Reason |
| --- | --- | --- |
| `replicaCount` | `2` | Keep leader election and a standby replica. |
| `image.tag` | `0.14.2` | Keep the live image. |
| `resources.requests` | CPU `10m`, memory `128Mi` | Keep live requests. |
| `resources.limits` | CPU `100m`, memory `256Mi` | Keep live limits. |
| `affinity` | Required pod anti-affinity by hostname | Keep replicas on different nodes. |
| `flags.logLevel` / `flags.logFormat` | `info` / `json` | Keep live logging. |
| `flags.runnerMaxConcurrentReconciles` | Upstream `2` | No override is needed. |
| `flags.watchSingleNamespace` | Unset | Set to `arc-runners` in apps/dev. |

The namespace controls both the controller argument and upstream watch RBAC.
The CI values use `fake-runners` to test this path. There are no wrapper ingress,
secret, or other optional objects.

## Upgrades

Update the dependency version and image tag together. Generate `Chart.lock` with
Helm, run the local checks, then render with the environment values and compare
against the existing Application. Include CRDs in the adoption render:

```sh
export TMPDIR=/tmp/ulw-wrappers/gha-runners-controller
mkdir -p "$TMPDIR"
../.omo/wrapper-charts/tools/checks.sh "$PWD" "$TMPDIR"
/opt/homebrew/bin/helm template gha-runners-controller charts \
  -n arc-systems -f ../cluster-config/apps/dev/gha-runners-controller.yaml --include-crds
```

## Adoption notes

- The old Application `gha-runner-scale-set-controller` was deleted and the
  controller reinstalled as Application and release `gha-runners-controller`. Keep the
  namespace, all four CRDs, and `ServerSideApply=true`. Argo CD has `skipCrds: false`.
  Local `helm template` needs `--include-crds` to match Argo CD's CRD render.
- The Application moves from project `default` and the kustomize-envsubst plugin
  to project `platform` and two Helm/value sources. Sync stays manual.
  Live has no ignoreDifferences rules to carry.
- Upstream 0.14.2 hardcodes `terminationGracePeriodSeconds: 10` in its Deployment.
  The old values file set `35`, but upstream ignores the value and live also uses
  `10`, so the wrapper does not carry it. A 35-second grace period needs a future
  upstream change.
- The environment runner namespace stays in apps/dev. Replica count, resource
  settings, image, affinity, and logging stay in shared defaults.
- Parity results and local check output are in
  `../.omo/evidence/wrappers/arc-controller/`.
- The environment render matches all 14 live tracked objects, including the four
  CRDs: `SAME=14 CHANGED=0 NEW=0 PRUNE=0 HOOK=0 UNKNOWN-KIND=0`.
  There are no object differences to accept.
