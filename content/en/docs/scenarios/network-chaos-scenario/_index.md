---
title: Network Chaos Scenario
description:
date: 2017-01-04
weight: 3
---

## Purpose

<krkn-hub-scenario id="network-chaos">
Introduces network latency, packet loss, and bandwidth restriction on the node's host network interface using `tc` and Netem. Use this to observe how your applications handle degraded network conditions.
</krkn-hub-scenario>

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with cluster-admin access
- `tc` (traffic control) and Netem kernel modules available on target nodes
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods
- SSH access to nodes (required for applying traffic control rules)

## How to Run Network Chaos Scenarios

Choose your preferred method to run network chaos scenarios:

{{< tabpane text=true >}}
  {{< tab header="**Krkn**" lang="krkn" >}}
{{< readfile file="_tab-krkn.md" >}}
  {{< /tab >}}
  {{< tab header="**Krkn-hub**" lang="krkn-hub" >}}
{{< readfile file="_tab-krkn-hub.md" >}}
  {{< /tab >}}
  {{< tab header="**Krknctl**" lang="krknctl" >}}
{{< readfile file="_tab-krknctl.md" >}}
  {{< /tab >}}
{{< /tabpane >}}

## Expected Behavior

- During the scenario, targeted nodes will experience the configured network degradation (latency, packet loss, or bandwidth restriction)
- Applications communicating through the affected nodes should show increased response times or errors
- After the scenario completes, Netem rules are removed and network conditions return to normal
- Monitor application metrics (latency percentiles, error rates) during the scenario to measure impact

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `tc command not found` | The `tc` utility is not installed on the target node | Install `iproute2` package on the node (`yum install iproute` or `apt install iproute2`) |
| `RTNETLINK answers: No such file or directory` | Netem kernel module is not loaded | Load the module with `modprobe sch_netem` on the target node |
| Network rules persist after scenario failure | The scenario crashed before cleanup | Manually remove rules with `tc qdisc del dev <interface> root` on the affected node |
| `connection refused` or SSH timeout | Cannot SSH into the target node to apply rules | Verify SSH key configuration and node security group allows SSH access |
| No observable network degradation | The wrong network interface was targeted | Check the node's primary interface name with `ip link show` and update the configuration |
