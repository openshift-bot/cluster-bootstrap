# AI Agent Instructions — cluster-bootstrap

This document provides instructions for AI agents working in the `cluster-bootstrap` repository.

## What This Repository Does

The `cluster-bootstrap` binary orchestrates the transition from a temporary bootstrap control plane to a permanent self-hosted control plane during OpenShift cluster installation. It:

1. Starts static control plane pods on the bootstrap node
2. Waits for the permanent control plane to become available on master nodes
3. Tears down the temporary bootstrap control plane

**Critical context**: This code runs during cluster installation, not in a running cluster. The binary is invoked by the OpenShift installer on the bootstrap node.

## Key Constraints and Conventions

### Timing and Coordination

- **Bootstrap control plane lifetime**: The temporary control plane must stay alive until the self-hosted control plane is confirmed available *and* load balancers have time to discover the new API servers
- **Tear-down timing**: For HA clusters, a minimum delay (`minimumTeardownDelay = 30s`) is enforced before tear-down to give load balancers time to observe the self-hosted control plane — **do not reduce this delay**
- **Event-driven coordination**: The installer waits for the `bootstrap-success` event before removing the bootstrap node from the load balancer — **this event must be created before tear-down begins**

### Error Handling

- **User-facing messages**: Use `UserOutput()` for messages intended for humans watching the install — these go to stdout
- **Diagnostic messages**: Use `fmt.Fprintf(os.Stderr, ...)` for logs not intended for end users
- **Error wrapping**: Wrap errors with context using `fmt.Errorf("message: %w", err)` so the call stack is traceable

### Testing

- **No live cluster required**: Unit tests simulate the API with an `httptest` TLS server (see `pkg/start/bootstrap_test.go`) — the repo does **not** use `client-go`'s fake clientset, so follow the `httptest` pattern rather than introducing one
- **Table-driven tests**: When adding test cases, use table-driven tests for readability (see `TestWaitForAvailabilityBeforeTearDown` and `Test_parsePodPrefixes`)
- **End-to-end testing**: This repo has no e2e tests — e2e validation happens through full OpenShift installation

## Common Mistakes to Avoid

### 1. Removing or Reducing Tear-Down Delays

**Why it's wrong**: Load balancers need time to observe the self-hosted control plane before the bootstrap node is removed. Reducing or removing delays causes install failures when the load balancer still points to the now-dead bootstrap API server.

**What to do instead**: If tear-down seems slow, verify the delay is necessary for the deployment model (HA vs. SNO). Do not reduce `minimumTeardownDelay` for HA clusters.

### 2. Changing Event Names

**Why it's wrong**: The installer waits for specific event names (`bootstrap-success`, `bootstrap-finished`). Changing these names breaks the handshake between cluster-bootstrap and the installer.

**What to do instead**: If you need to add new events, coordinate with the installer team. Do not rename or remove existing events.

### 3. Ignoring HA vs. SNO Differences

**Why it's wrong**: Single-node OpenShift (SNO) and highly available (HA) clusters have different tear-down requirements. HA clusters need more time for load balancer discovery; SNO does not use load balancers.

**What to do instead**: Respect the `isHAControlPlane()` check and the conditional logic that applies delays only for HA clusters.

### 4. Disrupting Asset Creation Flow

**Why it's wrong**: Asset creation starts in the background immediately after the bootstrap control plane is started, running concurrently with the pod readiness check. Changing this order can cause the API server to be unavailable when assets are being created, or can delay the overall bootstrap process.

**What to do instead**: Preserve the existing flow: start the bootstrap control plane, kick off asset creation in the background, then wait for pods to be running. The background goroutine will retry on transient errors while the main thread validates pod readiness.

### 5. Using the Load Balancer Client Too Early

**Why it's wrong**: The bootstrap control plane runs on `localhost:6443`. Using the load balancer client (which points to the external API endpoint) before the self-hosted control plane is up causes timeouts and failures.

**What to do instead**: Use `localClientConfig` (pointing to `localhost:6443`) until the self-hosted control plane is confirmed available. Only switch to `restConfig` (load balancer) after the `bootstrap-success` event is created.

### 6. Assuming the API Server is Always Available

**Why it's wrong**: During bootstrap, the API server may restart or be temporarily unavailable. Code must handle transient errors gracefully.

**What to do instead**: Use `wait.PollImmediateUntil()` for operations that query the API server. Retry on transient errors; only fail on terminal errors.

## Architecture Overview

### Start Command Flow

```text
1. Parse config and flags
2. Build the primary Kubernetes client from the loopback admin kubeconfig
   (auth/kubeconfig-loopback), then clone that config and override Host to
   localhost:6443 for the operator client and background asset creation
3. Start bootstrap control plane (static pods)
4. Create assets in background (manifests → temporary API server)
5. Wait for bootstrap pods to be running
6. [HA only] Wait for self-hosted control plane availability
7. [HA only] Sleep for tear-down delay (load balancer discovery)
8. Create bootstrap-success event
9. [Optional] Wait for `--tear-down-event` from installer (can delay tear-down)
10. Tear down / finish, order depends on `--tear-down-early`:
    - **Early** (`--tear-down-early=true`): switch asset creation to the load
      balancer client → tear down the bootstrap control plane → create the
      bootstrap-finished event (the local control plane is already gone)
    - **Late** (`--tear-down-early=false`, what `bootkube.sh` sets today): finish
      remaining assets on the local client → create the bootstrap-finished event
      → tear down the bootstrap control plane
```

> Note: The primary `client` (used by `waitUntilPodsRunning`) is built from
> `restConfig` (the loopback admin kubeconfig). The `localhost:6443` clone is a
> separate config used only for the operator client and background asset creation
> — do not conflate the two.
>
> Note: The `bootstrap-finished` event's position relative to tear-down is
> branch-specific — it is created *before* tear-down in late mode and *after*
> the tear-down call in early mode. Both events are created through the local
> client, which is why late mode creates `bootstrap-finished` before the local
> control plane is torn down.

### Key Components

- **`cmd/cluster-bootstrap/start.go`**: CLI flag parsing and command setup
- **`pkg/start/start.go`**: Main start command orchestration — also holds `isHAControlPlane`, the tear-down delay logic, and the `create.EnsureManifestsCreated` call that applies manifests
- **`pkg/start/bootstrap.go`**: Bootstrap control plane lifecycle (start, tear down)
- **`pkg/start/asset.go`**: Asset path constants and `getInstallConfig()` only — it does *not* apply manifests
- **`pkg/start/ha_checker.go`**: Self-hosted control plane availability pollers (API/scheduler/KCM on ≥2 masters)
- **`pkg/start/status.go`**: `waitUntilPodsRunning` and the pod status controller

### Critical Files and Constants

- **`bootstrapPodsRunningTimeout = 20 * time.Minute`**: How long to wait for bootstrap pods to start
- **`controlPlaneAvailabaleWaitTimeout = 30 * time.Minute`**: How long to wait for the self-hosted control plane (HA only). Note: the identifier is misspelled ("Availabale") in `pkg/start/start.go` — grep for the exact spelling
- **`minimumTeardownDelay = 30 * time.Second`**: Minimum delay before tear-down for HA clusters
- **Default required pods**: `kube-system/kube-apiserver`, `kube-system/kube-scheduler`, `kube-system/kube-controller-manager`, `kube-system/pod-checkpointer`

## Making Changes

### Before You Change Timing Logic

- **Understand the deployment model**: Is this SNO, HA, or arbiter/two-node?
- **Coordinate with the installer team**: Changes to event names, timing, or tear-down logic affect the installer
- **Test with a real install**: Unit tests cannot catch timing-related bugs — verify with a full cluster install

### Before You Add New Flags

- **Check if the flag is installer-facing**: If the installer needs to pass a new value, coordinate with the installer team to ensure they pass it correctly
- **Document the flag**: Add a clear description in the `Flags()` setup and update `README.md`

### Before You Refactor the Start Flow

- **The order matters**: Assets must be created before tear-down, events must be created in the correct sequence, and delays must happen at the right time
- **Preserve event semantics**: The installer depends on `bootstrap-success` and `bootstrap-finished` events. `bootstrap-success` must **always** be created before tear-down begins — never move it after tear-down. `bootstrap-finished` is **mode-dependent**: it is created before tear-down in late mode and after the tear-down call in early mode, so leave it in the branch where it already is and do not move it relative to tear-down within a given mode. (In early mode it is intentionally created after tear-down, so the "before tear-down" rule applies to `bootstrap-success` only.)

## Useful References

- [OpenShift Installer](https://github.com/openshift/installer) — Invokes cluster-bootstrap
- [OpenShift Enhancements](https://github.com/openshift/enhancements) — Enhancement proposals for OpenShift, including bootstrap-related design docs
- [Kubernetes Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/) — Background on how static pods work

## Questions?

If you're unsure about a change:

1. Check the git history (`git log -p <file>`) to understand why the current behavior exists
2. Look for related issues or pull requests in this repo or the installer repo
3. Ask in the `#forum-installer` Slack channel or open a discussion issue

**When in doubt, preserve existing behavior** — this code runs during installation and errors are expensive to debug.
