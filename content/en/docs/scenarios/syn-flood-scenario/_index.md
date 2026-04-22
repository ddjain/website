---
title: Syn Flood Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

Syn flood scenarios generate a substantial amount of TCP SYN flood traffic directed at one or more Kubernetes services to test the server's resiliency under extreme traffic conditions and validate DDoS resilience. This helps teams understand how their infrastructure responds to connection exhaustion attacks and whether rate-limiting, firewall rules, and auto-scaling policies are properly configured.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with access to target namespaces
- RBAC: ability to `create`/`delete` pods and `get`/`list` services in target namespaces
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

## Use Cases

<krkn-hub-scenario id="syn-flood">

This scenario generates a substantial amount of TCP traffic directed at one or more Kubernetes services within
the cluster to test the server's resiliency under extreme traffic conditions.
It can also target hosts outside the cluster by specifying a reachable IP address or hostname.
This scenario leverages the distributed nature of Kubernetes clusters to instantiate multiple instances
of the same pod against a single host, significantly increasing the effectiveness of the attack.
The configuration also allows for the specification of multiple node selectors, enabling Kubernetes to schedule
the attacker pods on a user-defined subset of nodes to make the test more realistic.



The attacker container source code is available [here](https://github.com/krkn-chaos/krkn-syn-flood).

</krkn-hub-scenario>

## How to Run Syn Flood Scenarios

Choose your preferred method to run syn flood scenarios:

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

- **TCP Connection Exhaustion:** The target service experiences a surge in half-open TCP connections, consuming available connection slots and kernel resources.
- **Increased Latency:** Response times for the target service increase significantly as the connection backlog fills and SYN queue overflows.
- **Service Degradation Under Load:** Legitimate client connections may be delayed or dropped while the flood is active, revealing the service's DDoS tolerance threshold.
- **Recovery After Scenario Ends:** Once attacker pods are terminated, the target service should recover to normal operation without manual intervention, confirming that connection state cleanup and resource reclamation work correctly.
- **Infrastructure Countermeasures Validated:** SYN cookies, rate-limiting rules, network policies, and cloud-provider DDoS protections should mitigate the impact — if they are properly configured.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Attacker pods stuck in `Pending` state | Node selectors do not match any available nodes, or the cluster lacks sufficient resources to schedule the attacker pods | Verify node labels with `kubectl get nodes --show-labels` and adjust the node selectors in the scenario configuration |
| Target service unreachable before scenario starts | The target service IP or hostname is incorrect, or the service is not running | Confirm the target is reachable with `kubectl get svc -n <namespace>` and test connectivity with `curl` or `nc` before running the scenario |
| No measurable impact on target service | The flood intensity is too low (too few attacker pods or insufficient traffic rate) or cloud-provider DDoS protection is absorbing the traffic | Increase the number of attacker pod replicas, adjust node selectors to spread across more nodes, or verify that cloud DDoS shields are not silently dropping traffic |
| `connection refused` or `cluster unreachable` | The kubeconfig points to an inaccessible cluster | Verify cluster access with `kubectl get nodes` and check the kubeconfig path |
| Attacker pods consume excessive node resources | The attacker pods are not resource-limited and are starving other workloads on the same node | Set CPU and memory limits on attacker pods, or use dedicated node selectors to isolate attacker workloads from production services |
