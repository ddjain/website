---
title: Power Outage Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

This scenario shuts down a Kubernetes/OpenShift cluster for a specified duration to simulate power outages, then brings it back online and verifies the cluster returns to a healthy state. It validates cluster resilience and recovery procedures when all nodes lose power simultaneously.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with cluster-admin access
- Cloud provider credentials (AWS, GCP, Azure, OpenStack, or Baremetal). See [cloud setup](/docs/scenarios/cloud_setup.md) for configuration details.
- Container runtime: Docker (20.10+) or Podman (4.0+) -- required for krkn-hub and krknctl methods

## How to Run Power Outage Scenarios

Choose your preferred method to run power outage scenarios:

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

When the scenario runs, all cluster nodes are stopped via the cloud provider API to simulate a complete power outage. After the configured `shut_down_duration` elapses, the nodes are restarted. The scenario then waits (up to the configured `timeout`) for every node to return to a `Ready` state and verifies overall cluster health. A successful run confirms the cluster can self-heal after a full power loss event.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cloud provider credentials not configured` | Missing or invalid cloud provider credentials for the target platform | Configure credentials for your cloud provider per the [cloud setup](/docs/scenarios/cloud_setup.md) guide (e.g., AWS `~/.aws/credentials`, Azure SP, GCP service account) |
| Nodes remain stopped after `shut_down_duration` | Cloud API call to restart nodes failed or timed out | Check cloud provider console for instance state; manually start the nodes and verify API credentials have start/stop permissions |
| `timeout waiting for node to be Ready` | One or more nodes did not recover within the configured timeout | Increase the `TIMEOUT` / `--timeout` value, or investigate node boot issues via the cloud provider console |
| Cluster components (API server, etcd) not healthy after restart | Control plane pods did not reschedule or start in time | Check `kubectl get pods -n openshift-kube-apiserver` (or equivalent) and node logs; consider increasing `shut_down_duration` to allow a cleaner shutdown |
| `unsupported cloud type` | The `cloud_type` value does not match a supported platform | Use one of the supported values: `aws`, `gcp`, `azure`, `openstack`, `bm` |

## Demo
See a demo of this scenario:
<script src="https://asciinema.org/a/r0zLbh70XK7gnc4s5v0ZzSXGo.js" id="asciicast-r0zLbh70XK7gnc4s5v0ZzSXGo" async="true"  style="max-width:900px; max-height:400px; width:100%; aspect-ratio:20/9;"></script>
