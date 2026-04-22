---
title: Service Disruption Scenarios
description:
date: 2017-01-04
weight: 3
---
## Purpose

Service disruption scenarios delete all objects within a specific namespace, or a namespace matching a certain regex string, to test whether services can recover gracefully after a full namespace-level disruption.

## Preconditions

| Requirement | Detail |
|---|---|
| Kubernetes version | 1.21+ |
| Kubeconfig | Valid kubeconfig with cluster-admin privileges |
| RBAC | Permissions to delete all resource types in the target namespace |
| Container runtime | CRI-O, containerd, or Docker |

## How to Run Service Disruption Scenarios

Choose your preferred method to run service disruption scenarios:

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

- All objects (pods, services, deployments, configmaps, secrets, etc.) in the targeted namespace are deleted.
- Controllers (Deployments, StatefulSets, DaemonSets, operators) should automatically recreate the deleted resources.
- Services backed by higher-level controllers should return to a healthy state without manual intervention.
- If the namespace itself is deleted, any namespace-scoped finalizers must complete before the namespace is fully removed.

## Failure Handling

| Symptom | Cause | Fix |
|---|---|---|
| Namespace stuck in `Terminating` state | Finalizers on one or more resources are not completing | Identify the resource with a pending finalizer (`kubectl get all -n <ns>`) and remove the finalizer manually, or resolve the underlying controller issue. |
| Services not recreated after deletion | No higher-level controller (Deployment, StatefulSet, Operator) managing the resource | Ensure workloads are managed by a controller that reconciles desired state; bare pods will not be recreated automatically. |
| RBAC permission denied errors during scenario run | Service account or kubeconfig lacks delete permissions on certain resource types | Grant cluster-admin or add explicit RBAC rules for all resource types in the target namespace. |
| Persistent volume data lost after namespace deletion | PersistentVolumeClaims with `Delete` reclaim policy were removed with the namespace | Use `Retain` reclaim policy on PersistentVolumes that must survive namespace-level disruptions. |

## Demo

You can find a link to a demo of the scenario [here](https://asciinema.org/a/kKPSI0D7upV9HQWH5jnroTaKb)
