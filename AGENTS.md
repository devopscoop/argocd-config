# AGENTS.md

Instructions for AI coding agents working in this repo.

## What this repo is

A single-purpose configuration repo: no application code, no build step, no test
suite. The deliverable is `values.yaml`, a values file for the upstream
`argo/argo-cd` Helm chart that registers a Config Management Plugin (CMP) named
`argocd-gomplate`.

It is one of three repos in the pattern:

- [argocd-applicationset-pattern](https://github.com/arturo-builds-infra/argocd-applicationset-pattern) — the app/ApplicationSet layout and the `.tpl` files the plugin renders
- [argocd-gomplate](https://github.com/arturo-builds-infra/argocd-gomplate) — the sidecar image supplying `gomplate` and `helm`
- this repo — the ArgoCD-side config that wires the plugin in

## Commands

```shell
# Install the local CLI tools (helm, kubectl)
brew bundle                                              # macOS
grep -vE '^(#|$)' pkglist.txt | sudo pacman -S --needed - # Arch

helm repo add argo https://argoproj.github.io/argo-helm && helm repo update

# Check a values.yaml change renders before applying it
helm template argocd argo/argo-cd --namespace argocd -f values.yaml

helm install argocd argo/argo-cd --namespace argocd --create-namespace -f values.yaml
helm upgrade argocd argo/argo-cd --namespace argocd -f values.yaml

# Plugin sidecar logs — the only place rendering errors surface
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server -c argocd-gomplate

# Confirm the plugin ConfigMap actually landed in the sidecar
kubectl exec -n argocd -it <argocd-repo-server-pod> -- ls /home/argocd/cmp-server/config/
```

## How the plugin is wired (spans several parts of values.yaml)

1. `configs.cmp.create: true` plus `configs.cmp.plugins.argocd-gomplate` makes
   the chart render a ConfigMap named `argocd-cmp-cm` with the key
   `argocd-gomplate.yaml`.
2. `repoServer.volumes` mounts that ConfigMap, and the `argocd-gomplate` entry in
   `repoServer.extraContainers` mounts the single key over
   `/home/argocd/cmp-server/config/plugin.yaml` (`subPath`), then runs
   `/var/run/argocd/argocd-cmp-server` out of the shared `var-files` volume.
3. So the plugin's entire implementation is the embedded shell script under
   `configs.cmp.plugins.argocd-gomplate.generate.args` — that block is the only
   real code in this repo. Changing it needs a `helm upgrade` and a repo-server
   restart to take effect.
4. `gomplate` and `helm` come from the sidecar image. The script must not assume
   any workstation tooling.

## The generate script's contract

Variables reach the script from an Application's `spec.source.plugin.env`, which
ArgoCD exposes prefixed with `ARGOCD_ENV_`. Currently consumed:
`HELM_CHART_URL`, `HELM_CHART_NAME`, `HELM_CHART_VERSION`,
`APPLICATION_NAMESPACE`, `ENVIRONMENT`, `ALLOWED_ENVIRONMENTS`, plus
ArgoCD's own `ARGOCD_APP_NAME`.

Two mutually exclusive modes, decided by which `.tpl` files the app directory has:

- **ApplicationSet mode** — `application.yaml.tpl` exists: render it, print it,
  `exit 0`.
- **Helm mode** — render `values.yaml.tpl`, then emit `pre.yaml.tpl` (if
  present), `helm template` output, and `post.yaml.tpl` (if present) as one
  `---`-separated stream on stdout.

Chart source is chosen by prefix: `oci://` URLs are concatenated into the chart
ref, anything else is passed via `--repo`. Every `gomplate` call passes
`-d env=overrides.yaml`; templates reach that data with `{{ ds "env" }}`, and
plain environment variables with `{{ .Env.VARIABLE_NAME }}`.

Cluster labels under `configs.clusterCredentials` (e.g. `alias`, `environment`)
are what templates read as environment variables, so label keys have to match
what the pattern repo's templates expect. The `in-cluster` block's values are
examples.

## Gotchas in the current values.yaml

- `ALLOWED` is computed from `ARGOCD_ENV_ALLOWED_ENVIRONMENTS` and never read
  (`values.yaml:67-76`) — the environment gate is a no-op today. Decide whether to
  enforce or drop it; don't silently "tidy" it either way.
- `command: [sh, -c]`, but the script uses `[[ ... ]]` (`values.yaml:83`), which
  is not POSIX. It only works because the image's `sh` accepts it. Keep new
  script code POSIX-compatible, or switch the command to `bash` deliberately.
- The `docker-config` volume references secret `ghcr-docker-config` with no
  `optional: true` (`values.yaml:40-42`), so the repo-server pod will not start
  until that secret exists in the `argocd` namespace — even though README.md
  frames registry credentials as needed only for private charts.
- `HELM_CONFIG_HOME` / `HELM_CACHE_HOME` point into `/tmp` (the `cmp-tmp`
  emptyDir) because the sidecar runs non-root; helm needs writable dirs. Any new
  helm step must stay inside those paths. `HELM_REGISTRY_CONFIG` instead points
  at the mounted dockerconfig, so registry auth and helm's scratch dirs are
  separate concerns.
- `runAsUser: 999` and `podSecurityContext.fsGroup: 999` must stay aligned or the
  mounted dockerconfig becomes unreadable. The sidecar image is pinned to
  `:latest`.

Every key in `values.yaml` has to exist in the upstream argo-cd chart's schema —
verify against the chart, not by analogy, when adding one.

## Package manifests

This repo ships a `Brewfile` (macOS: `brew bundle`) and a `pkglist.txt` (Arch Linux) that install every local CLI tool the repo uses. Keep them in sync with the code:

- When you add a tool, script, or a new external command inside an existing script, add the package to BOTH files, with a comment noting what uses it.
- When a tool stops being used, remove it from both files.
- Verify package names before adding them: `brew info <formula>` for Homebrew, and the official repos/AUR for Arch (e.g. kubectl is Homebrew `kubernetes-cli` but Arch `kubectl`). If a package is AUR-only, note that in pkglist.txt's header instructions.
- Only workstation tools belong in the manifests — software that runs inside the cluster (e.g. gomplate and helm inside the argocd-gomplate sidecar) does not.
- Update the "Install required packages" section in README.md if the tool list changes.

## CI

The only workflows are the two Claude Code ones in `.github/workflows/`. Both
pin action SHAs and `--model claude-opus-5 --effort xhigh`, and both carry long
comments explaining exactly why their `permissions` blocks and tool allowlists
are scoped as they are (notably: the review workflow's `--allowed-tools`
inline-comment entry is load-bearing, not optional). Read those comments before
changing either file, and keep the pins.

Commit messages use conventional prefixes (`feat:`, `docs:`, `ci:`).
