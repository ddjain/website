---
title: Zone Outage Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

Zone outage scenarios simulate a full availability-zone failure in a public cloud to measure the impact on both the Kubernetes/OpenShift control plane and the applications running on worker nodes in that zone. Use this to validate that your cluster and workloads can tolerate the loss of an entire zone and continue operating.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster deployed across multiple availability zones
- Valid kubeconfig with cluster-admin access
- Cloud provider credentials: AWS (with permissions to modify VPC network ACLs) or GCP (with permissions to stop/start compute instances)
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

## How It Works

<krkn-hub-scenario id="zone-outages">
Scenario to create outage in a targeted zone in the public cloud to understand the impact on both Kubernetes/OpenShift control plane as well as applications running on the worker nodes in that zone.

There are 2 ways these scenarios run:
For AWS, it tweaks the network acl of the zone to simulate the failure and that in turn will stop both ingress and egress traffic from all the nodes in a particular zone for the specified duration and reverts it back to the previous state.

For GCP, it in a specific zone you want to target and finds the nodes (master, worker, and infra) and stops the nodes for the set duration and then starts them back up. The reason we do it this way is because any edits to the nodes require you to first stop the node before performing any updates. So, editing the network as the AWS way would still require you to stop the nodes first.

</krkn-hub-scenario>

## How to Run Zone Outage Scenarios

Choose your preferred method to run zone outage scenarios:

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

- **Subnet disruption (AWS):** All ingress and egress traffic to the targeted zone's subnets is blocked for the configured duration, causing nodes in that zone to become `NotReady`. After the duration, network ACLs are reverted to their original state and nodes recover.
- **Node stop (GCP):** All nodes (master, worker, and infra) in the targeted zone are stopped for the configured duration, then started back up. Pods on those nodes are evicted and rescheduled to nodes in surviving zones.
- **Pod rescheduling across zones:** Workloads with multiple replicas and topology spread constraints should automatically reschedule onto nodes in the remaining healthy zones, maintaining service availability.
- **Control plane resilience:** If the targeted zone hosts a control-plane node, the cluster API should remain available through control-plane nodes in other zones (requires an HA control plane with 3+ nodes spread across zones).

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cloud provider credentials not configured` | Missing or invalid AWS/GCP credentials | For AWS, configure `~/.aws/credentials` with permissions for VPC and network ACL operations. For GCP, set `GOOGLE_APPLICATION_CREDENTIALS` to a service account JSON key with Compute Engine admin access. |
| Nodes remain `NotReady` after scenario duration ends | Network ACL revert failed (AWS) or node restart failed (GCP) | Manually restore the original network ACL rules via the AWS console, or start the stopped instances via the GCP console. Verify with `kubectl get nodes`. |
| Pods not rescheduling to healthy zones | Missing topology spread constraints or insufficient resources in surviving zones | Ensure workloads define `topologySpreadConstraints` and that nodes in remaining zones have enough CPU/memory. Check with `kubectl describe node`. |
| `timeout waiting for zone recovery` | Zone recovery takes longer than the configured timeout | Increase the scenario timeout or investigate cloud provider status for the targeted zone and region. |
| `permission denied` modifying network ACLs (AWS) | IAM policy lacks `ec2:ReplaceNetworkAclEntry` or related permissions | Update the IAM role/policy attached to the credentials to include VPC network ACL modification permissions. |

## Demo
You can find a link to a demo of the scenario [here](https://asciinema.org/a/452672?speed=3&theme=solarized-dark)
