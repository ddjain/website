---
type: "docs/scenarios"
title: Hog Scenarios
description: 
date: 2017-01-04
weight: 3
---

## Purpose

Hog scenarios push the limits of memory, CPU, or I/O on one or more nodes in your cluster. They deploy stress workloads that consume a predetermined amount of resources for a specified duration. Use this to evaluate whether your cluster can withstand resource-intensive pods and to test eviction, throttling, and OOM-killer behavior.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with cluster-admin access
- RBAC: ability to create pods and deployments in the target namespace
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods
- The stress workload image (`quay.io/krkn-chaos/krkn-hog`) must be pullable from the cluster

## Config Options

#### Common options

| Option  | Type     | Description    |
|---------|-------------------------------------|---------------------------------------------------------------------------------------------------------------|
| `duration` | number  | the duration of the stress test in seconds       |
| `workers` | number (Optional)  | the number of threads instantiated by stress-ng, if left empty the number of workers will match the number of available cores in the node.   |
| `hog-type` | string (Enum)  | can be cpu, memory or io.    |
| `image` | string  | the container image of the stress workload  (quay.io/krkn-chaos/krkn-hog)  |
| `namespace` | string   | the namespace where the stress workload will be deployed   |
| `node-selector` | string (Optional) | defines the node selector for choosing target nodes. If not specified, one schedulable node in the cluster will be chosen at random. If multiple nodes match the selector, all of them will be subjected to stress. If number-of-nodes is specified, that many nodes will be randomly selected from those identified by the selector. |
|`taints`| list (Optional) default [] | list of taints for which tolerations need to be created. Example: ["node-role.kubernetes.io/master:NoSchedule"]|
| `number-of-nodes` | number (Optional) | restricts the number of selected nodes by the selector|

### Available Scenarios

- [CPU Hog](/docs/scenarios/hog-scenarios/cpu-hog-scenario/_index.md)
- [Memory Hog](/docs/scenarios/hog-scenarios/memory-hog-scenario/_index.md)
- [I/O Hog](/docs/scenarios/hog-scenarios/io-hog-scenario/_index.md)

## Expected Behavior

- During the scenario, the stress workload pod deploys on the target node(s) and consumes the configured amount of CPU, memory, or I/O
- **CPU Hog**: Node CPU utilization spikes; pods with CPU limits may get throttled
- **Memory Hog**: Node memory pressure triggers pod evictions if limits are not set; OOM-killer may terminate the stress pod or other pods
- **I/O Hog**: Disk I/O saturation causes increased latency for volume-dependent workloads
- After the configured `duration` expires, the stress workload completes and is cleaned up automatically
- Monitor node metrics (`kubectl top nodes`) and pod status during the scenario to observe resource pressure effects

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Stress pod stuck in `Pending` state | No node matches the `node-selector` or insufficient resources to schedule | Verify node labels with `kubectl get nodes --show-labels` and check available resources with `kubectl describe node` |
| `ImagePullBackOff` on stress pod | Cannot pull `quay.io/krkn-chaos/krkn-hog` image | Verify cluster has internet access or mirror the image to a private registry |
| Node becomes unresponsive during memory hog | OOM-killer or kernel panic due to excessive memory consumption | Reduce the memory target or add `number-of-nodes` to limit scope; reboot the node if unresponsive |
| Stress pod not cleaned up after scenario | Scenario crashed before cleanup phase | Manually delete the stress pods with `kubectl delete pod -l <hog-label> -n <namespace>` or use the [rollback command](../../rollback-scenarios/) |
| `forbidden: pod security policy` error | Pod security policies or admission controllers block privileged pods | Grant the service account appropriate security context or create a policy exception |

### Rollback Scenario Support

Krkn supports rollback for all available Hog scenarios. For more details, please refer to the [Rollback Scenarios](../../rollback-scenarios/_index.md) documentation.
