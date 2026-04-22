---
title: ETCD Split Brain Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

This scenario isolates an etcd node by blocking its network traffic, forcing an etcd leader re-election and temporarily inducing a split-brain condition with two etcd leaders active simultaneously. It is particularly useful for testing the etcd cluster's resilience under such a challenging state.

{{< notice type="danger" >}} This scenario carries a significant risk: it **might break the cluster API**, making it impossible to automatically revert the applied network rules. The `iptables` rules will be printed to the console, allowing for manual reversal via a shell on the affected node. This scenario is **best suited for disposable clusters** and should be **used at your own risk**. {{< /notice >}}

## Preconditions

- Kubernetes 1.21+ cluster
- A valid `kubeconfig` with cluster-admin or equivalent RBAC permissions
- The etcd cluster must be accessible from the node where the scenario is executed
- Container runtime (Podman or Docker) installed on the host
- Network access to the target etcd node (iptables manipulation requires root-level privileges)

## How to Run ETCD Split Brain Scenarios

Choose your preferred method to run ETCD split brain scenarios:

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

- The targeted etcd node becomes isolated from the rest of the cluster due to blocked network traffic.
- The remaining etcd nodes detect the loss of the isolated member and trigger a **leader re-election**.
- The cluster temporarily exhibits a **split-brain condition**, with two etcd leaders active simultaneously.
- Once the network rules are reverted, the isolated node rejoins the cluster and **quorum is restored**.
- The Kubernetes API server recovers and resumes normal operation.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Kubernetes API becomes permanently unreachable after scenario | The iptables rules were not reverted and the cluster lost quorum | Manually remove the iptables rules printed to the console by connecting to the affected node via SSH |
| Etcd cluster does not recover quorum after scenario ends | The isolated node failed to rejoin the cluster or data diverged too far | Restart the etcd service on the isolated node; if the member is permanently out of sync, remove and re-add it to the cluster |
| Scenario fails to start with permission errors | Insufficient RBAC permissions or missing root-level access on the target node | Ensure the kubeconfig has cluster-admin privileges and the container runtime has access to manipulate iptables on the host |
| Network rules persist after the scenario timeout | The scenario process was interrupted before cleanup could run | Retrieve the printed iptables rules from the scenario logs and manually run the corresponding delete commands on the affected node |
