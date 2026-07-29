# Self-Healing Demo

A deliberately broken Kubernetes workload that an AI agent diagnoses and fixes via
pull request, with Argo CD syncing the merged change back to the cluster.

## The loop

1. `oom-demo` runs with a 64Mi memory limit while trying to allocate ~150Mi.
2. The kernel OOM-kills it; Kubernetes restarts it; it crash-loops.
3. Dynatrace raises a problem on the restart loop.
4. Davis CoPilot reads the live environment, determines the limit is too low, and
   opens a PR against this repo on a `selfheal/oom-demo-<timestamp>` branch.
5. The PR is merged; Argo CD syncs; the pod recovers.

## Repo layout

    manifests/oom-demo.yaml    The broken Deployment. One field matters:
                               resources.limits.memory

## Prerequisites

- A Kubernetes cluster (this demo runs on k3s)
- Dynatrace Operator deployed, with a DynaKube monitoring the cluster
- Argo CD installed in the `argocd` namespace
- TODO: Dynatrace Workflow / CoPilot configuration — see "Outside this repo" below

## Setup

Create the namespace:

```bash
kubectl create namespace selfhealing-demo
```

Register the Argo CD application:

```yaml
# argocd-selfhealing-demo-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: selfhealing-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/jib94/selfhealing-demo.git
    targetRevision: main
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: selfhealing-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd-selfhealing-demo-application.yaml
```

## Running the demo

**To trigger the problem:** ensure `resources.limits.memory` in
`manifests/oom-demo.yaml` is set to `64Mi`, then commit and push. Argo syncs and the
pod begins crash-looping within a minute or two.

**To reset after a demo:** set the limit back to `64Mi` — the agent will have raised
it as part of the previous run.

Watch it happen:

```bash
kubectl -n selfhealing-demo get pods -w
```

Look for `OOMKilled` in the pod status and a climbing restart count.

## Outside this repo

These pieces are not version-controlled here, but the loop does not close without
them:

- TODO: the Dynatrace Workflow (or equivalent) that watches for the problem and
  invokes CoPilot — record its name and which environment it lives in
- TODO: the GitHub credential CoPilot uses to open PRs, and where it is stored
- TODO: whether PR merge is manual or automated

## Troubleshooting

**Pod is `Running` instead of crash-looping.** The memory limit was probably left
raised from a previous run. Check the live value:

```bash
kubectl -n selfhealing-demo get deploy oom-demo \
  -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}'
```

**Argo shows `Unknown` or `ComparisonError`.** The repo path or URL is wrong.
Confirm `path: manifests` matches the folder name exactly:

```bash
kubectl -n argocd get application selfhealing-demo -o jsonpath='{.spec.source}'
```

Force a re-poll:

```bash
kubectl -n argocd annotate application selfhealing-demo \
  argocd.argoproj.io/refresh=hard --overwrite
```

**Changes pushed to git are not reaching the cluster.** Argo polls roughly every
three minutes. Force a sync:

```bash
kubectl -n argocd patch application selfhealing-demo --type merge \
  -p '{"operation":{"sync":{"revision":"main"}}}'
```

**Agent opened a PR but nothing changed.** The PR still needs merging. Also note
that `selfHeal: true` means Argo reverts any manual `kubectl edit` on the
deployment — change the manifest in git, not the cluster.

**Stale `selfheal/*` branches piling up.** The agent creates one per run. List the
ones already merged:

```bash
git branch -r --merged origin/main | grep selfheal
```

Delete them (this is irreversible on the remote — check the list first):

```bash
git branch -r --merged origin/main | grep selfheal | sed 's|origin/||' \
  | xargs -n1 git push origin --delete
```

## Notes

- `selfHeal: true` and `prune: true` are both enabled. Anything removed from
  `manifests/` is deleted from the cluster on the next sync.
- The demo is namespaced to `selfhealing-demo` and removes cleanly:
  `kubectl delete namespace selfhealing-demo`.
- The wealth demo previously lived in this repo. It now has its own home at
  https://github.com/jib94/wealth-demo.
