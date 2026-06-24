# Architecture — cluster-bootstrap

This document describes the internal architecture, key design decisions, and tradeoffs in the `cluster-bootstrap` repository.

## Purpose and Scope

The `cluster-bootstrap` binary orchestrates the transition from a **temporary bootstrap control plane** (running on the bootstrap node during installation) to a **permanent self-hosted control plane** (running on the cluster's master nodes).

**What this is**:
- A one-time orchestrator that runs during OpenShift cluster installation
- Responsible for starting the temporary control plane, waiting for the permanent control plane to be ready, and tearing down the temporary control plane

**What this is not**:
- An operator that runs in a live cluster
- Responsible for installing master nodes or the permanent control plane — that's handled by the installer and cluster operators

## High-Level Architecture

### Bootstrap Process Flow

```text
┌─────────────────────────────────────────────────────────────────┐
│ OpenShift Installer                                             │
│  - Provisions bootstrap node                                    │
│  - Invokes cluster-bootstrap binary on bootstrap node           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ cluster-bootstrap (on bootstrap node)                           │
│  1. Start temporary control plane (static pods)                 │
│  2. Create initial cluster resources (manifests)                │
│  3. Wait for self-hosted control plane to be available          │
│  4. Tear down temporary control plane                           │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Self-Hosted Control Plane (on master nodes)                     │
│  - kube-apiserver, kube-scheduler, kube-controller-manager      │
│  - Running as regular pods managed by cluster operators         │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Start Command (`pkg/start/start.go`)

The `start` command is the main entry point and the orchestrator. It:

- **Builds the Kubernetes clients**: The primary client is built from the loopback admin kubeconfig (`auth/kubeconfig-loopback`); a cloned config with `Host` overridden to `localhost:6443` is used for the operator client and background asset creation
- **Drives the bootstrap control plane lifecycle**: Calls `bcp.Start()` to launch static pods, then `bcp.Teardown()` when ready
- **Applies manifests**: Calls `create.EnsureManifestsCreated()` (from `library-go/pkg/assets/create`) in a background goroutine to apply manifests from `<asset-dir>/manifests`
- **Waits for readiness**: Waits for the required bootstrap pods (`waitUntilPodsRunning`) and, on HA clusters, for the self-hosted control plane
- **Detects HA topology**: `isHAControlPlane()` reads the install config and returns true when `ControlPlane.Replicas >= 3`
- **Enforces the tear-down delay**: Applies `minimumTeardownDelay` before tear-down on HA clusters
- **Coordinates via events**: Creates events (`bootstrap-success`, `bootstrap-finished`) that the installer waits for

#### 2. Bootstrap Control Plane (`pkg/start/bootstrap.go`)

Manages the lifecycle of the temporary control plane running on the bootstrap node:

- **Start**: Copies TLS secrets (from `<asset-dir>/tls`) and the admin kubeconfig into the bootstrap secrets directory, copies the static pod manifests from `<asset-dir>/bootstrap-manifests` into the kubelet's pod manifest path (default `/etc/kubernetes/manifests`), then waits for the kube-apiserver `/readyz` endpoint to respond
- **Teardown**: Removes the manifests it copied and the secrets directory, then optionally waits for the API to terminate (`/version` connection refused)

The set of static pods launched is whatever the installer places in `bootstrap-manifests` — cluster-bootstrap copies them as-is rather than hardcoding a list.

#### 3. Asset Path Constants and Install Config (`pkg/start/asset.go`)

This file does **not** apply manifests. It holds the asset path constants (`assetPathSecrets`, `assetPathAdminKubeConfig`, `assetPathClusterConfig`, `assetPathManifests`, `assetPathBootstrapManifests`) and the `getInstallConfig()` helper, which reads the cluster-config ConfigMap (`manifests/cluster-config.yaml`) and unmarshals its embedded `install-config` data. The actual manifest application lives in `start.go` (`create.EnsureManifestsCreated`).

#### 4. Self-Hosted Availability Checker (`pkg/start/ha_checker.go`)

Waits for the self-hosted control plane to become available before tear-down. `waitForSelfHostedControlPlaneAvailabilityBeforeTearDown()` runs three concurrent pollers against the operator API, requiring each to be available on **at least two** master nodes (`CurrentRevision >= 1`):

- kube-apiserver (`kubeapiservers/cluster`)
- kube-scheduler (`kubeschedulers/cluster`)
- kube-controller-manager (`kubecontrollermanagers/cluster`)

Note: HA *detection* (`isHAControlPlane`) and the tear-down delay both live in `start.go`, not here.

#### 5. Pod Status Controller (`pkg/start/status.go`)

`waitUntilPodsRunning()` and the `statusController` watch the required pod prefixes (default: `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `pod-checkpointer` in `kube-system`) and block until all required pods are running and ready.

## Design Decisions

### 1. Localhost API Server for Initial Operations

**Decision**: Use `localhost:6443` to talk to the bootstrap API server during initial asset creation, not the load balancer endpoint.

**Rationale**:
- The bootstrap control plane runs on the local node — no network hop required
- The load balancer may not be configured yet or may not point to a working API server
- Faster and more reliable than going through the load balancer

**Tradeoff**:
- Only **early** tear-down switches asset creation to the load balancer client (`restConfig`) after the `bootstrap-success` event; **late** tear-down keeps asset creation on the local client (`localClientConfig`) the whole time, since the local control plane is still running
- Requires managing two client configurations (`localClientConfig` and `restConfig`)

**Where this is implemented**:
- `pkg/start/start.go` — creates `localClientConfig` pointing to `localhost:6443`
- The first asset round always uses `localClientConfig`. After the `bootstrap-success` event, asset creation switches to `restConfig` **only if `--tear-down-early` is enabled**; otherwise it stays on `localClientConfig`

### 2. Event-Driven Coordination with the Installer

**Decision**: Use Kubernetes Events to signal readiness to the installer instead of RPC or file-based coordination.

**Rationale**:
- Events are a native Kubernetes primitive — no additional infrastructure required
- The installer already watches Kubernetes resources — no need for a separate communication channel
- Events are timestamped and can be inspected after installation for debugging

**Events created**:
- `bootstrap-success`: Created after the self-hosted control plane is confirmed available (HA paths only) and the tear-down delay has elapsed; signals the installer that tear-down can begin
- `bootstrap-finished`: Signals that asset creation is complete. Its position relative to tear-down is **mode-dependent** — in late tear-down mode it is created *before* the bootstrap control plane is torn down; in early tear-down mode the bootstrap control plane has already been torn down by the time this event is created

**Tradeoff**:
- Event names are hardcoded and must match between cluster-bootstrap and the installer — changes require coordination
- Both events are created through the local (`localhost:6443`) client, so any event that must reach the API has to be created before that local control plane is torn down. This is why, in late tear-down mode, `bootstrap-finished` is created before tear-down (see the `// client actually refers to localhost` note in `pkg/start/start.go`)

**Where this is implemented**:
- `pkg/start/start.go` — creates events using `client.CoreV1().Events("kube-system").Create()`

### 3. Minimum Tear-Down Delay for HA Clusters

**Decision**: Enforce a 30-second minimum delay before tearing down the bootstrap control plane on HA clusters, even if the self-hosted control plane is ready earlier.

**Rationale**:
- Load balancers (AWS ELB, GCP LB, etc.) perform periodic health checks to discover backend API servers
- If the bootstrap control plane is torn down before the load balancer discovers the self-hosted API servers, the installer loses API access and the install fails
- 30 seconds is a conservative estimate based on typical load balancer health check intervals

**Tradeoff**:
- Adds 30 seconds to every HA cluster install, even if the load balancer is fast
- Does not apply to SNO (single-node OpenShift) clusters, which don't use load balancers

**Where this is implemented**:
- `pkg/start/start.go` — checks `isHAControlPlane()` and enforces `minimumTeardownDelay = 30 * time.Second`

### 4. Early vs. Late Tear-Down

**Decision**: Provide a `--tear-down-early` flag to control when the bootstrap control plane is torn down.

**Flag default vs. installer invocation**: The CLI flag defaults to `true` (see `cmd/cluster-bootstrap/start.go`), but the OpenShift installer's `bootkube.sh` currently invokes the binary with `--tear-down-early=false`, i.e. **late tear-down is what runs in a real install today** (see the `// currently it is set to false by bootkube.sh` note in `pkg/start/start.go`). Do not assume early tear-down is the effective behavior just because it is the CLI default.

**Early tear-down** (`--tear-down-early=true`):
- Switches asset creation over to the load balancer client, then tears down the bootstrap control plane **before** the `bootstrap-finished` event is created
- The optional `--tear-down-event` (if set) is waited for *before* this tear-down, so the installer can further delay it (e.g. to remove the bootstrap node from the load balancer first)

**Late tear-down** (`--tear-down-early=false`, the installer's current setting):
- Keeps the bootstrap control plane (and its local client) running while the remaining assets are created
- Creates the `bootstrap-finished` event and only **then** tears the bootstrap control plane down

**Rationale**:
- Early tear-down reduces the time the bootstrap node is running, saving cost on cloud providers
- Late tear-down is safer if asset creation might fail — the temporary control plane is still available, and the `bootstrap-finished` event can still be created through the local client

**Tradeoff**:
- Early tear-down requires switching clients mid-process, adding complexity, and means `bootstrap-finished` is created after the local control plane is gone
- Late tear-down keeps the bootstrap node alive longer, increasing cost

**Where this is implemented**:
- `pkg/start/start.go` — checks `b.earlyTearDown` and calls `bcp.Teardown()` either before or after the `bootstrap-finished` event; the `--tear-down-event` wait sits between the `bootstrap-success` event and tear-down

### 5. Static Pod Manifests for the Bootstrap Control Plane

**Decision**: Use static pod manifests (written to `/etc/kubernetes/manifests`) instead of running control plane components as systemd units or direct processes.

**Rationale**:
- The kubelet is already running on the bootstrap node — no need for additional process management
- Static pods are automatically restarted by the kubelet if they crash
- The `pod-checkpointer` ensures static pods survive kubelet restarts

**Tradeoff**:
- Static pods cannot be managed via the Kubernetes API — must manipulate files on disk
- Tear-down requires removing files and waiting for the kubelet to notice (no immediate shutdown)

**Where this is implemented**:
- `pkg/start/bootstrap.go` — copies manifests to the pod manifest path, removes them on tear-down

## Dependencies

### External Dependencies

- **Kubernetes client-go**: Interacts with the Kubernetes API server
- **openshift/client-go**: OpenShift-specific client for operator resources (e.g., checking kube-apiserver operator status)
- **openshift/library-go**: Shared OpenShift libraries for asset creation and common utilities
- **spf13/cobra**: CLI framework for command-line argument parsing

### Internal Dependencies

- **`pkg/version`**: Provides version information for the binary
- **`pkg/start`**: Core start command logic
- **`pkg/bootstrapinplace`**: Support for bootstrap-in-place (SNO) installations
- **`pkg/dependencymagent`**: A build-only dependency magnet (`//go:build tools`, package `dependencymagnet`). It blank-imports `github.com/openshift/build-machinery-go` so `go mod` keeps it vendored for the Makefile. **Do not delete** — removing it breaks the `make` targets that rely on build-machinery-go.

### Runtime Dependencies

At runtime, `cluster-bootstrap` depends on:

- **kubelet**: Must be running on the bootstrap node to start static pods
- **Kubernetes API server**: Temporary bootstrap API server must be available for asset creation
- **Asset directory**: Must contain manifests, kubeconfig, and install config

## Data Flow

### Assets and Configuration

The asset directory layout is defined by the path constants in `pkg/start/asset.go`:

```text
Asset Directory Structure:
/path/to/assets/
├── auth/
│   └── kubeconfig-loopback       # Admin kubeconfig (assetPathAdminKubeConfig)
├── tls/                          # TLS secrets copied into the bootstrap
│   └── ...                       #   secrets dir (assetPathSecrets)
├── manifests/                    # Manifests applied to the API server
│   ├── cluster-config.yaml       #   ConfigMap wrapping install-config
│   │                             #   (assetPathClusterConfig)
│   └── ...                       #   (assetPathManifests)
└── bootstrap-manifests/          # Static pod manifests for the kubelet
    └── ...                       #   (assetPathBootstrapManifests)
```

**Flow**:
1. Installer generates assets during the `create ignition-configs` phase
2. Assets are written to the bootstrap node's filesystem
3. `cluster-bootstrap start --asset-dir=/path/to/assets`, in implementation order:
   - first reads `manifests/cluster-config.yaml` via `isHAControlPlane()` to determine HA topology (this happens *before* the control plane is started)
   - copies `bootstrap-manifests/` to the kubelet's pod manifest path to launch the temporary control plane (`bcp.Start()`)
   - applies everything under `manifests/` to the API server via the background asset-creation goroutine

### Event Flow

The two tear-down modes differ in where tear-down sits relative to the
`bootstrap-finished` event.

```text
Shared prefix (both modes):
T0:  cluster-bootstrap starts
T1:  cluster-config read (isHAControlPlane) — before the control plane starts
T2:  Bootstrap control plane static pods launched
T3:  Bootstrap control plane API server is ready (localhost:6443)
T4:  Manifests applied to bootstrap API server (background goroutine)
T5:  Required bootstrap pods running
T6:  [HA only] Self-hosted control plane API servers become available
T7:  [HA only] Sleep for tear-down delay (load balancer discovery)
T8:  bootstrap-success event created
T9:  [Optional] Wait for --tear-down-event from installer

Late tear-down (--tear-down-early=false, installer's current setting):
T10: Remaining assets finish (kept on the local client)
T11: bootstrap-finished event created
T12: Bootstrap control plane torn down

Early tear-down (--tear-down-early=true):
T10: Asset creation switched to the load balancer client
T11: Bootstrap control plane torn down
T12: Remaining assets finish
T13: bootstrap-finished event created (local control plane already gone)
```

## Known Limitations and Tradeoffs

### 1. No Validation of Asset Directory Contents

**Limitation**: `cluster-bootstrap` assumes the asset directory is well-formed and contains all required files. If files are missing or malformed, the error may be cryptic.

**Workaround**: The installer validates assets before invoking `cluster-bootstrap`.

**Future work**: Add validation of asset directory structure and required files.

### 2. Tear-Down Delay is Not Dynamic

**Limitation**: The 30-second tear-down delay for HA clusters is hardcoded and does not adapt to actual load balancer health check intervals.

**Workaround**: 30 seconds is a conservative estimate that works for most load balancers.

**Future work**: Query the load balancer's health check interval (if available) and use it instead of a fixed delay.

### 3. No Support for Control Plane on Different Nodes

**Limitation**: `cluster-bootstrap` assumes the bootstrap control plane runs on the same node as the `cluster-bootstrap` binary. It cannot manage a remote bootstrap control plane.

**Workaround**: This is by design — the bootstrap control plane is always local to the bootstrap node.

## Testing Strategy

- **`cmd/cluster-bootstrap/start_test.go`**: `Test_parsePodPrefixes` covers the required-pods clause parsing
- **`pkg/start/bootstrap_test.go`**: `TestBootstrapControlPlane` and `TestBootstrapControlPlaneNoOverwrite` exercise the copy/start/teardown lifecycle against an `httptest.NewTLSServer` stand-in for the API
- **`pkg/start/ha_checker_test.go`**: `TestWaitForAvailabilityBeforeTearDown` covers the poller aggregation logic with synthetic conditions
- **No fake clientset**: The tests do not use `client-go`'s fake clientset — API interactions are simulated with an `httptest` server. Follow that pattern rather than introducing a fake client

### Integration Tests

- **None currently**: Integration tests would require a running Kubernetes cluster, which is not practical for this repository

### End-to-End Tests

- **Handled by the installer**: Full OpenShift installation tests validate `cluster-bootstrap` behavior
- **CI**: Every pull request to `openshift/installer` runs installation tests that exercise `cluster-bootstrap`

## Future Enhancements

### 1. Dynamic Tear-Down Delay Based on Load Balancer Health Checks

Instead of a fixed 30-second delay, query the load balancer's health check interval and wait for at least one health check cycle.

### 2. More Granular Control Plane Readiness Checks

Currently, we check if control plane pods are running. We could also check:
- API server `/healthz` endpoint returns 200
- Scheduler and controller manager have successfully elected a leader
- API server can serve requests (e.g., list nodes)

### 3. Structured Logging

Replace `UserOutput()` and `fmt.Fprintf(os.Stderr, ...)` with structured logging (e.g., `logr` or `klog`) for better observability.

### 4. Metrics and Instrumentation

Expose metrics (e.g., time to bootstrap control plane ready, time to self-hosted control plane ready) for monitoring and performance analysis.

## Questions and Decisions to Revisit

### Should `--tear-down-early` be the default?

**Current CLI default**: `true`

**Effective behavior in a real install**: `false` — the installer's `bootkube.sh` passes `--tear-down-early=false`, so late tear-down is what actually runs. The CLI default and the installer's setting disagree today.

**Pros of early**: Reduces bootstrap node lifetime, saving cloud costs

**Cons of early**: Adds complexity (client switching), and `bootstrap-finished` is created after the local control plane is gone, making that event best-effort

**Decision**: The in-code TODO questions whether `--tear-down-early` is meaningful at all and suggests removing the flag (or scoping it to SNO). Until that is resolved, keep the CLI default at `true` but be aware the installer overrides it to `false`.

### Should we support non-HA deployments differently?

**Current behavior**: HA and SNO have different tear-down logic (delay for HA, no delay for SNO)

**Future**: Two-node and arbiter topologies may need different behavior — revisit when those topologies are fully supported.

## Related Documentation

- [README.md](./README.md) — User-facing documentation
- [CONTRIBUTING.md](./CONTRIBUTING.md) — Contribution guidelines
- [AGENTS.md](./AGENTS.md) — AI agent instructions
- [OpenShift Installer Documentation](https://github.com/openshift/installer/tree/master/docs)

## Contact

For questions about architecture or design decisions, reach out in:
- **Slack**: `#forum-installer` or `#forum-control-plane`
- **GitHub**: Open an issue or discussion in this repository
