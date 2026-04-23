# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Rook is a Kubernetes operator that deploys and manages Ceph storage clusters. The codebase is Go. Everything here is a single `rook` binary (see `cmd/rook/main.go`) that dispatches to one of many subcommands — the same binary runs as the operator, as per-daemon init/sidecar processes (mon, osd, mgr, etc.), and as user-facing CLI tools — driven by Kubernetes CRDs defined under `pkg/apis/ceph.rook.io/v1`.

## Common commands

All day-to-day tasks go through `make`. Invoke from the repo root.

- `make build` — build the `rook` binary and container image for the host platform (linux only). Use `make -j4 build`.
- `make build.all` — cross-compile for all platforms (amd64 host only).
- `make test` — run all unit tests (`go test ./...` under `cmd` and `pkg`).
- `make test-integration` — run integration tests from `tests/integration` (requires a live cluster; prefer CI).
- `make lint` — runs every linter: `yamllint`, `pylint`, `shellcheck`, `checkmake`, `vet`, `markdownlint`, `golangci-lint`, `helm.lint`.
- `make vet` / `make golangci-lint` / `make fmt` / `make fmt-fix` — individual Go checks/formatters (`fmt-fix` rewrites files).
- `make generate` — regenerate all derived files (`gen.codegen`, `gen.crds`, `gen.rbac`). **Run after any change to `pkg/apis/ceph.rook.io/v1/`** or to helm charts that feed RBAC generation.
- `make crds.manifests` / `make crds.docs` / `make gen-rbac` — individual pieces of the above.
- `make mod.check` / `make mod.update` — vet or refresh go modules.
- `make help` — dump all targets and available options.

### Running a single test

Unit tests are plain `go test`; run them directly for tight loops:

```
go test -tags ceph_preview ./pkg/operator/ceph/cluster/osd/...
go test -tags ceph_preview -run TestFooBar ./pkg/operator/ceph/cluster
```

The `ceph_preview` build tag is **required** — without it, code calling the go-ceph rgw/admin account API won't compile. The Makefile sets this via `TAGS ?= ceph_preview`.

For Makefile-driven filtering of the unit-test target, `SUITE=<regex>` becomes `-run <regex>` and `TESTFILTER=<regex>` becomes `-testify.m <regex>`.

Integration tests take a suite name: `make test-integration SUITE=CephSmokeSuite` (see `tests/integration/*_test.go`).

## Build constraints worth knowing

- Go `1.25` or `1.26` only (checked in `build/makelib/golang.mk`; older Go fails `go.check`).
- `CGO_ENABLED=0` — the operator binary is statically linked. Don't import packages that require cgo.
- Two go modules: the repo root (`github.com/rook/rook`) and `pkg/apis` (`github.com/rook/rook/pkg/apis`), which is a **separate module** so that external consumers of the CRD types don't pull in the whole operator tree. Changes to types under `pkg/apis/ceph.rook.io/v1/` must keep this module self-contained.

## Architecture big picture

### Operator = controller-runtime manager with many reconcilers

`cmd/rook ceph operator` boots `pkg/operator/ceph/operator.go` → `Operator.Run`, which starts a `controller-runtime` manager registered in `pkg/operator/ceph/cr_manager.go`. That file is the authoritative list of all CRD reconcilers — adding a new CRD means wiring its `Add` function into `AddToManagerFuncs` (or `AddToManagerFuncsMaintenance` for the machine-disruption manager).

Each subsystem under `pkg/operator/ceph/` is a reconciler for one or more CRDs:

- `cluster/` — the root `CephCluster` reconciler, plus `mon/`, `mgr/`, `osd/`, `rbd/` (mirroring), `nodedaemon/` (crash collector, exporter).
- `pool/`, `file/`, `object/`, `nfs/`, `nvmeof/`, `client/` — each Ceph storage-type CRD and its sub-resources (e.g. `object/user`, `object/zone`, `file/mirror`, `pool/radosnamespace`).
- `csi/` — CSI driver deployment.
- `disruption/` — PDB management and machine-disruption awareness.
- `controller/` — shared helpers (OperatorConfig, predicates, common reconcile plumbing). The `opcontroller` import alias you'll see everywhere points here.

A reconcile typically: reads the CR → talks to the Kubernetes API via `clusterd.Context` (clientset + dynamic + controller-runtime client) → executes Ceph admin commands by exec'ing into the operator/toolbox pod via `pkg/daemon/ceph/client`.

### Daemon-side code

`pkg/daemon/ceph/` contains code that runs **inside Ceph container images**, not in the operator: OSD prepare/activate (`osd/`), cleanup jobs (`cleanup/`), the `discover` daemon, and the shared `client/` library the operator uses to issue `ceph` CLI commands. `cmd/rook/ceph/*.go` maps CLI subcommands (`ceph osd`, `ceph mgr`, `ceph config`, `ceph cleanup`) to these packages.

### CRD types and generated code

- Types live under `pkg/apis/ceph.rook.io/v1/` (one file per CRD kind: `cluster.go`, `pool.go`, `nfs.go`, …).
- `pkg/client/` is 100% auto-generated (typed clientsets, informers, listers). **Never edit `pkg/client/` by hand** — change the types in `pkg/apis` and run `make generate`.
- CRD YAML manifests in `deploy/examples/crds.yaml` and the helm charts are also generated from the types via `controller-gen` (see `build/crds/build-crds.sh`, driven by `make crds.manifests`).
- RBAC under `deploy/charts/rook-ceph/templates/` is the source of truth; `deploy/examples/common.yaml` is generated from it via `build/rbac/gen-common.sh` (`make gen-rbac`).

### Deployment surface

- `deploy/charts/` — Helm charts (`rook-ceph` for the operator, `rook-ceph-cluster` for an actual cluster). These are primary; manifests derive from them.
- `deploy/examples/` — rendered YAMLs for users who don't use Helm, plus sample CRs (`cluster.yaml`, `object.yaml`, `filesystem.yaml`, …).
- `images/ceph/` — the Dockerfile that produces the `rook/ceph` image, which bundles the Rook binary on top of upstream Ceph.

### Test layout

- Unit tests: colocated with source (`*_test.go`). `pkg/operator/test/` and `pkg/operator/ceph/test/` hold reusable fakes/scaffolding (fake ceph client, fake k8s objects).
- Integration tests: `tests/integration/` (the `*_test.go` entry points) use the framework in `tests/framework/` (`installer/` deploys operator+cluster, `clients/` wraps kube/rook API calls, `utils/` are shared helpers). These compile to a test binary (`TEST_PACKAGES = $(GO_PROJECT)/tests/integration`) and expect a real cluster.

## Repository conventions

- **DCO is enforced.** Every commit must have `Signed-off-by:` — use `git commit -s`. PRs without it are blocked by a bot.
- **Commit message format:** `<subsystem>: <what changed>` on the subject (≤70 chars), blank line, why in the body. Allowed subsystem prefixes are enforced by `commitlint` — see `.commitlintrc.json` before inventing a new one.
- PRs target `master`. Backports to release branches happen automatically via `mergify` when a `backport-release-x.y` label is set on the master PR; don't open PRs directly against release branches.
- Never edit auto-generated files by hand: `pkg/client/`, `deploy/examples/crds.yaml`, `deploy/examples/common.yaml`, `Documentation/Helm-Charts/*-chart.md`, `Documentation/CRDs/specification.md`. Regenerate with `make generate` / `make docs`.
