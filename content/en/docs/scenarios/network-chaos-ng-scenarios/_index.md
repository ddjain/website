---
type: "docs/scenarios"
title: Network Chaos NG Scenarios
description: 
date: 2017-01-04
weight: 3
---

## Purpose

Network Chaos NG introduces a next-generation infrastructure for network fault injection, built on eBPF-based filtering to refactor and replace the legacy network chaos plugins. It provides more precise, lower-overhead network disruption capabilities at both the pod and node level.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with access to target namespaces and nodes
- RBAC: ability to `create`/`delete` privileged workloads and `get`/`list` pods and nodes in target namespaces
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods
- Kernel support for eBPF (Linux 4.15+ on cluster nodes)

### Available Scenarios

#### Network Chaos NG scenarios:
- [Pod Network Filter](/docs/scenarios/network-chaos-ng-scenarios/pod-network-filter/_index.md)
- [Node Network Filter](/docs/scenarios/network-chaos-ng-scenarios/node-network-filter/_index.md)
- [Node Interface Down](/docs/scenarios/network-chaos-ng-scenarios/node-interface-down/_index.md)

## Expected Behavior

When a Network Chaos NG scenario runs successfully:

- **Targeted traffic is filtered:** Ingress and/or egress traffic matching the configured rules is blocked on the target pods or nodes for the configured duration.
- **Applications experience connectivity issues:** Services relying on the disrupted network paths will encounter connection timeouts, refused connections, or packet loss.
- **Non-targeted traffic is unaffected:** Only the traffic matching the specified ports, interfaces, or selectors is disrupted; other network paths continue to operate normally.
- **Automatic recovery:** After the configured `test_duration` elapses, all iptables rules or interface changes are rolled back and normal connectivity is restored without manual intervention.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `no pods matching label selector` | The `label_selector` value does not match any pods or nodes in the target namespace | Verify labels with `kubectl get pods -n <namespace> --show-labels` or `kubectl get nodes --show-labels` and update the selector |
| `permission denied` or `privileged container required` | The service account lacks permissions to deploy privileged workloads needed for network manipulation | Ensure the kubeconfig user has cluster-admin or a ClusterRole that permits privileged pod creation |
| Network rules not restored after scenario | The scenario workload was interrupted or evicted before cleanup could run | Manually check and remove stale iptables rules on affected nodes/pods, or restart the affected pods to clear pod-level rules |
| `namespace not found` | The `namespace` parameter does not match any existing namespace | Check namespace exists with `kubectl get ns` and correct the configuration |
| `connection refused` or `cluster unreachable` | The kubeconfig points to an inaccessible cluster or the API server is down | Verify cluster access with `kubectl get nodes` and check the kubeconfig path |
