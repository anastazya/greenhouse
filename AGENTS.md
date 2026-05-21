# Agents

This file provides guidance to Coding Agents when working with code in this repository.

## What this repo is

Greenhouse is a Kubebuilder-based Kubernetes operator (day-2 operations platform). It extends the Kubernetes API with CRDs (`Organization`, `Team`, `Cluster`, `Plugin`, `PluginDefinition`, `PluginPreset`, `TeamRole`, `TeamRoleBinding`, `Catalog`, `ClusterKubeconfig`, `ClusterPluginDefinition`) reconciled by controllers, and acts as an admission webhook. A separate Juno-based dashboard (not in this repo) talks to the kube-apiserver.

## Common commands

Tooling pinned in the `Makefile` is auto-installed into `./bin/` on first use — don't install globally.

| Task | Command |
| --- | --- |
| Generate deepcopy + CRDs + RBAC + webhooks + docs + license headers | `make generate-all` |
| Run unit/integration tests (uses `envtest`, sets up Flux CRDs) | `make test` |
| Run a single Go test | `KUBEBUILDER_ASSETS="$(./bin/setup-envtest use ENVTEST_K8S_VERSION -p path)" go test ./internal/controller/cluster/... -run TestX -v`. `ENVTEST_K8S_VERSION` is defined in the `Makefile`. |
| Format code (gofmt + goimports with local prefix) | `make fmt` |
| Lint | `make lint` (uses `./bin/golangci-lint` v2; config in `.golangci.yaml`) |
| `make fmt lint test` in one shot | `make check` |
| Build all binaries into `./bin/` | `make build` |
| Build only the operator | `make build-greenhouse` |
| Run operator locally against current kubeconfig | `make run` |
| Build greenhousectl CLI | `make cli` |
| Regenerate mocks (driven by `.mockery.yaml`) | `make mockery` |

## Local dev clusters

Two KinD clusters (`greenhouse-admin`, `greenhouse-remote`) are spun up by `greenhousectl dev setup`, wrapped in Make targets. Kubeconfigs end up under `./bin/*.kubeconfig`.

- `make setup` — full stack: operator + dashboard + cors-proxy + demo org with onboarded remote cluster.
- `make setup-controller-dev` — webhook server in-cluster, controllers run from your IDE. Set `CONTROLLERS_ONLY=true` in the run config.
- `make setup-webhook-dev` — controllers in-cluster, webhook server runs from your IDE.
- `make setup-e2e` then `make e2e-local SCENARIO=<name>` — run an e2e suite. Scenarios are folder names under `e2e/` containing an `e2e_test.go`. Each scenario uses a Go build tag of the form `<scenario>E2E` (e.g. `clusterE2E`).
- `make clean-e2e` — tear down the KinD clusters.

E2E tests against real clusters: set `EXECUTION_ENV=GARDENER`, `GREENHOUSE_ADMIN_KUBECONFIG`, `GREENHOUSE_REMOTE_KUBECONFIG`, then `make e2e SCENARIO=<name>`.

## Code architecture

**Binaries (`cmd/`)** — all published by `make build`:
- `greenhouse` — the controller manager + admission webhook server (the operator). Wiring lives in `cmd/greenhouse/controllers.go` and `cmd/greenhouse/webhooks.go`.
- `greenhousectl` — operator CLI; `dev setup` is what powers the local-dev Make targets.
- `idproxy`, `cors-proxy`, `service-proxy`, `authz` — supporting HTTP services deployed alongside the operator.

**API types (`api/`)** — CRD Go types, generated deepcopy, and the openapi/typescript generation pipeline.
- `api/v1alpha1` is the main version; `api/v1alpha2` currently only hosts `TeamRoleBinding`.
- `api/meta/v1alpha1` holds shared meta types.
- `zz_generated.deepcopy.go` files are generated — never hand-edit; run `make generate`.
- The local replace directive `github.com/cloudoperators/greenhouse/api => ./api` in `go.mod` is intentional and explicitly allow-listed in `.golangci.yaml` (`gomoddirectives.replace-local: true`).

**Controllers (`internal/controller/`)** — one subpackage per kind (`cluster/`, `organization/`, `plugin/`, `plugindefinition/`, `team/`, `teamrbac/`, `catalog/`). Most reconcilers go through `pkg/lifecycle.Reconcile`, which is the contract for `WaitUntilResourceReadyOrNotReady` in e2e helpers.

**Webhooks (`internal/webhook/`)** — versioned subdirs (`v1alpha1/`, `v1alpha2/`) plus a `greenhouse/` subpackage. `controller-gen` reads both `./internal/webhook/...` and `./internal/controller/...` to generate the webhook + RBAC manifests.

**Manifest pipeline** — `make manifests` runs `controller-gen` to produce CRDs in `hack/crd/bases`, then `hack/generate.go` converts them and writes to `charts/manager/crds/`; webhook + RBAC manifests are written to `charts/manager/templates/` and post-processed by `hack/helmify`. The `charts/manager/` Helm chart is the deployable artifact — never edit its CRDs/templates by hand.

**Other internal packages** worth knowing:
- `internal/dex` — vendored Dex types with their own deepcopy generation (`make generate` covers this too).
- `internal/flux`, `internal/helm`, `internal/ocimirror` — integrations powering plugin/catalog reconciliation.
- `internal/test` — shared test fixtures and envtest setup.
- `pkg/lifecycle`, `pkg/cel`, `pkg/mocks` — exported helpers.

**E2E (`e2e/`)** — Ginkgo suites guarded by build tags. Shared helpers in `e2e/shared/` (test env, client construction, cluster onboard/offboard, log collection). New suites must follow the `<dir>E2E` build-tag convention so the CI matrix can discover them via `make list-scenarios`.

## Conventions

- PR titles must follow Conventional Commits (`<type>(<scope>): <description>`); the allowed types/scopes are enforced by `.github/workflows/ci-pr-title.yaml`.
- License headers (`SPDX-FileCopyrightText` / `SPDX-License-Identifier: Apache-2.0`) are added by `make license` (uses the skywalking-eyes container).
- Always run `make generate-all` after touching anything under `api/`, controller/webhook annotations, or the docs templates — the generated files are committed.

## Coding Style

- Strictly adhere to [Effective Go](https://go.dev/doc/effective_go) and the [Google Go Style Guide](https://google.github.io/styleguide/go/guide).
- For code reviews follow the [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) guidelines.
- Always aim for the simplest solution that works, and avoid over-engineering. YAGNI (You Aren't Gonna Need It) is a good principle to keep in mind. Better write a boring implementation that is easy to understand and maintain, than a clever one that is hard to follow.

## Testing

- Use `gomega` and `ginkgo` for unit and integration tests. Follow the patterns in existing tests for consistency.
- Use test fixtures and helpers in `internal/test` to avoid duplication and keep tests focused on the behaviour being tested.
- You can run unit tests with `make test`. Running tests in an IDE may require setting the `KUBEBUILDER_ASSETS` environment variable to point to the `envtest` binaries, which can be installed with `./bin/setup-envtest use ENVTEST_K8S_VERSION -p path`. `ENVTEST_K8S_VERSION` is defined in the `Makefile`.
