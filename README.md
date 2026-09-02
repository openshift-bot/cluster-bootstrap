# cluster-bootstrap

Cluster control plane bootstrapping logic for OpenShift.

## Overview

The `cluster-bootstrap` repository provides the orchestration logic that brings up a temporary bootstrap control plane during OpenShift cluster installation and manages the transition to a permanent, self-hosted control plane running on the cluster's master nodes.

## What This Does

During OpenShift cluster installation, this component:

1. **Starts a temporary bootstrap control plane** — launches static pods (kube-apiserver, kube-scheduler, kube-controller-manager) on the bootstrap node from manifests
2. **Creates initial cluster resources** — deploys manifests from the asset directory to the temporary control plane
3. **Waits for the self-hosted control plane to become available** — monitors for required control plane pods running on master nodes
4. **Tears down the bootstrap control plane** — removes the temporary components once the permanent control plane is ready and stable

The binary is packaged as a container image and invoked by the OpenShift installer during the bootstrap phase.

## Building Locally

### Prerequisites

- Go 1.25+
- Docker or Podman

### Build the Binary

```bash
make
```

This produces a `cluster-bootstrap` binary in the current directory.

### Build the Container Image

```bash
make images
```

## Running Locally

The `cluster-bootstrap` binary is designed to run during OpenShift installation, invoked by the installer. It is not typically run manually.

If you need to test or debug the start command:

```bash
./cluster-bootstrap start \
  --asset-dir=/path/to/cluster/assets \
  --pod-manifest-path=/etc/kubernetes/manifests
```

**Required flags:**
- `--asset-dir`: Path to the cluster asset directory containing manifests and kubeconfig
- `--pod-manifest-path`: Location where kubelet looks for static pod manifests (default: `/etc/kubernetes/manifests`)

**Common optional flags:**
- `--required-pods`: List of namespace/pod-prefix pairs that must be running before tear-down (default: `kube-system/kube-apiserver`, `kube-system/kube-scheduler`, `kube-system/kube-controller-manager`, `kube-system/pod-checkpointer`)
- `--tear-down-delay`: Duration to wait before tearing down the bootstrap control plane, giving load balancers time to observe the self-hosted control plane
- `--tear-down-early`: Tear down immediately after the self-hosted control plane is up (default: true)

## Repository Structure

```text
cluster-bootstrap/
├── cmd/cluster-bootstrap/    # Main entry point and CLI commands
│   ├── main.go               # Root command setup
│   ├── start.go              # Start command implementation
│   └── bootstrapinplace.go   # Bootstrap-in-place support
├── pkg/start/                # Core bootstrap orchestration logic
│   ├── start.go              # Main start command execution + manifest application
│   ├── bootstrap.go          # Bootstrap control plane lifecycle
│   ├── asset.go              # Asset path constants + install-config helper
│   ├── ha_checker.go         # Self-hosted control plane availability pollers
│   └── status.go             # Pod readiness (waitUntilPodsRunning)
├── manifests/                # Operator manifests for release payload
└── Dockerfile                # Container image definition
```

## Testing

```bash
# Run unit tests
make test-unit
```

**Note**: This repository does not currently have end-to-end tests. The `make test-e2e` target exists in the Makefile but is a placeholder. End-to-end testing of this component requires running a full OpenShift cluster installation.

## Additional Documentation

- [CONTRIBUTING.md](./CONTRIBUTING.md) — How to contribute to this repository
- [AGENTS.md](./AGENTS.md) — Instructions for AI agents working in this codebase
- [ARCHITECTURE.md](./ARCHITECTURE.md) — Design decisions and internal architecture

## Related Repositories

- [openshift/installer](https://github.com/openshift/installer) — Invokes cluster-bootstrap during installation
- [openshift/machine-config-operator](https://github.com/openshift/machine-config-operator) — Manages node configuration, including static pod manifests

## License

This repository is licensed under the Apache License 2.0. See [LICENSE](./LICENSE) for details.
