---
title: DNS Outage Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

This scenario blocks all outgoing DNS traffic from a specific pod, effectively preventing it from resolving any hostnames or service names. It validates that applications handle DNS resolution failures gracefully and that monitoring systems detect connectivity degradation caused by DNS unavailability.

## Preconditions

- Kubernetes cluster version 1.21 or later
- A valid `kubeconfig` with permissions to create, list, and delete pods and network policies in the target namespace
- RBAC configured to allow Krkn to manage network policy resources
- Container runtime (CRI-O or containerd) accessible from the cluster nodes
- Target pod must be running and reachable before the scenario begins

## How to Run DNS Outage Scenarios

Choose your preferred method to run DNS outage scenarios:

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

When the DNS outage scenario is active:

- Targeted pods cannot resolve internal or external hostnames via DNS queries (port 53 UDP/TCP traffic is blocked)
- Services that depend on hostname resolution fail to establish new connections to external endpoints, APIs, or other cluster services referenced by DNS name
- Existing connections that were established before the outage may continue to function until they are recycled or time out
- Applications without proper retry logic or DNS caching may experience cascading failures
- Once the scenario completes, DNS traffic is restored and pods should resume normal name resolution

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| DNS queries time out but are not fully blocked | Network policy not applied to the correct pod or namespace | Verify the target pod label selectors and namespace in the scenario configuration match the running pod |
| Scenario fails to start with RBAC errors | Service account lacks permissions to create `NetworkPolicy` resources | Grant the Krkn service account `create`, `get`, `list`, and `delete` permissions on `networking.k8s.io/networkpolicies` in the target namespace |
| DNS resolution still works during the scenario | A node-level DNS cache (e.g., NodeLocal DNSCache) is serving cached responses | Disable or bypass the node-local DNS cache, or extend the scenario duration beyond the cache TTL to observe failures after cached entries expire |
| Pod does not recover DNS resolution after scenario ends | Network policy was not cleaned up properly | Manually delete the `NetworkPolicy` created by Krkn in the target namespace and restart the affected pod |
