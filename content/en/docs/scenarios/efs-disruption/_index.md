---
title: EFS Disruption Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

This scenario creates an outgoing firewall rule on specific nodes in your cluster, chosen by node name or a selector. This rule blocks connections to AWS EFS, leading to a temporary failure of any EFS volumes mounted on those affected nodes.

## Preconditions

- Kubernetes cluster version 1.21 or later
- A valid `kubeconfig` with permissions to list nodes and create/delete network policies or iptables rules (RBAC)
- One or more AWS EFS volumes mounted in the cluster
- Container runtime (CRI-O, containerd, or Docker) running on the target nodes
- Network access from the nodes to the AWS EFS endpoint (so the firewall rule has traffic to block)

## How to Run EFS Disruption Scenarios

Choose your preferred method to run EFS disruption scenarios:

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

When this scenario is active:

- EFS-backed volumes on affected nodes become unavailable, and any read/write operations against those mounts will stall or fail.
- Pods relying on EFS volumes experience I/O errors, which may cause application-level failures, container restarts, or liveness/readiness probe failures.
- Once the firewall rule is removed (after the configured duration), EFS connectivity is restored and pods recover automatically or through their restart policy.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Scenario fails to start; firewall rule is not created | Insufficient RBAC permissions or inability to run `iptables` on the target node | Ensure the service account has privileged access and that the node allows iptables modifications |
| Pods do not show I/O errors even though the scenario is running | EFS mount target uses an IP address or DNS endpoint not covered by the firewall rule | Verify that the scenario configuration targets the correct EFS endpoint IP range or DNS name |
| EFS volumes remain unavailable after the scenario ends | The iptables cleanup step failed or was interrupted | Manually remove the blocking iptables rule on the affected nodes using `iptables -D` or restart the node's network stack |
