# Rollbar fork of weave-gitops

Upstream `weaveworks/weave-gitops` no longer publishes releases, and its last
published image (`ghcr.io/weaveworks/wego-app:v0.38.0`) only speaks Flux API
versions that Flux 2.7+ no longer serves. Rollbar runs the Flux UI
(flux-ui.rollbar.tools, cluster `compute`) from this fork, built by us from
upstream `main`.

## What differs from upstream

The diff is additive apart from one deletion, so syncing with upstream rarely
conflicts:

- `.github/workflows/rollbar-release.yaml` builds and publishes the release.
- `ROLLBAR.md` (this file).
- `.github/dependabot.yml` is deleted: Dependabot is off in this fork (version
  updates via that file, security updates via the repository setting;
  vulnerability alerts stay on). If an upstream merge brings the file back or
  conflicts on it, keep it deleted (`git rm .github/dependabot.yml`).
- Repository settings, not files: the upstream workflows `CI`, `ossf`,
  `Release`, `repo-stats` and `Upgrade flux` are disabled
  (`gh workflow disable`). They would fail or publish under the wrong names
  in the fork, and `CI` would also run its own release jobs on our `v*` tags.
  `Docs` stays enabled; its deploy job only runs in the weaveworks org.

The chart in `charts/gitops-server` is left untouched in git. The release
workflow rewrites `version`, `appVersion`, `image.repository` and `image.tag`
at packaging time (upstream hard-codes `image.tag` in `values.yaml`, so it has
to be overridden too).

## Release

Versions are `v<upstream base>-rollbar.<n>`, for example `v0.39.1-rollbar.1`.

```bash
gh workflow run rollbar-release.yaml --repo rollbar/weave-gitops -f version=v0.39.1-rollbar.2
# or push a tag:
git tag v0.39.1-rollbar.2 && git push origin v0.39.1-rollbar.2
```

The workflow publishes:

| Artifact | Reference |
|---|---|
| Image (linux/amd64 + linux/arm64) | `ghcr.io/rollbar/weave-gitops/gitops-server:v0.39.1-rollbar.2` |
| Helm chart `weave-gitops` | `oci://ghcr.io/rollbar/charts/weave-gitops`, version `0.39.1-rollbar.2`, appVersion `v0.39.1-rollbar.2` |

Both GHCR packages must stay **public**: the `compute` cluster pulls them
without credentials. A package created for the first time is private; change
it under the package's settings in the GitHub UI.

Then bump the pinned chart version in `rollbar/infra`:
`k8s/flux/infrastructure/base/flux-ui/flux-ui.yaml` (HelmRelease
`flux-system/ww-gitops`, `spec.chart.spec.version`). The HelmRepository is
`k8s/flux/infrastructure/base/sources/helm/flux-ui.yaml`.

## Sync with upstream

```bash
git remote add upstream https://github.com/weaveworks/weave-gitops.git  # once
git fetch upstream --tags
git checkout main
git merge upstream/main
git push origin main
```

Then release a new version, bumping the base when upstream bumped
`charts/gitops-server/Chart.yaml`. Before releasing, check whether upstream
changed `gitops-server.dockerfile`, the `LDFLAGS`/`echo-ldflags` targets in
`Makefile`, or the chart values that `rollbar/infra` sets.
