---
title: ManagedCluster Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

ManagedCluster scenarios provide a way to integrate Krkn with [Open Cluster Management (OCM)](https://open-cluster-management.io/) and [Red Hat Advanced Cluster Management for Kubernetes (ACM)](https://www.redhat.com/en/technologies/management/advanced-cluster-management) by injecting faults into [ManagedClusters](https://open-cluster-management.io/concepts/managedcluster/) via [ManifestWorks](https://open-cluster-management.io/concepts/manifestwork/).

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster configured as an OCM/ACM hub
- Valid kubeconfig with access to the hub cluster
- RBAC: ability to `get`/`list`/`create`/`delete` ManagedCluster and ManifestWork resources
- Cloud provider credentials for the underlying infrastructure of managed clusters
- Container runtime: Docker (20.10+) or Podman (4.0+)

## Supported Scenarios

The following ManagedCluster chaos scenarios are supported:

1. **managedcluster_start_scenario**: Scenario to start the ManagedCluster instance.
2. **managedcluster_stop_scenario**: Scenario to stop the ManagedCluster instance.
3. **managedcluster_stop_start_scenario**: Scenario to stop and then start the ManagedCluster instance.
4. **start_klusterlet_scenario**: Scenario to start the klusterlet of the ManagedCluster instance.
5. **stop_klusterlet_scenario**: Scenario to stop the klusterlet of the ManagedCluster instance.
6. **stop_start_klusterlet_scenario**: Scenario to stop and start the klusterlet of the ManagedCluster instance.

## How to Run ManagedCluster Scenarios

ManagedCluster scenarios can be injected by placing the ManagedCluster scenarios config files under `managedcluster_scenarios` option in the Kraken config. Refer to [managedcluster_scenarios_example](https://github.com/redhat-chaos/krkn/blob/main/scenarios/kube/managedcluster_scenarios_example.yml) config file.

```yaml
managedcluster_scenarios:
  - actions:                                                        # ManagedCluster chaos scenarios to be injected
    - managedcluster_stop_start_scenario
    managedcluster_name: cluster1                                   # ManagedCluster on which scenario has to be injected; can set multiple names separated by comma
    # label_selector:                                               # When managedcluster_name is not specified, a ManagedCluster with matching label_selector is selected for ManagedCluster chaos scenario injection
    instance_count: 1                                               # Number of managedcluster to perform action/select that match the label selector
    runs: 1                                                         # Number of times to inject each scenario under actions (will perform on same ManagedCluster each time)
    timeout: 420                                                    # Duration to wait for completion of ManagedCluster scenario injection
                                                                    # For OCM to detect a ManagedCluster as unavailable, have to wait 5*leaseDurationSeconds
                                                                    # (default leaseDurationSeconds = 60 sec)
  - actions:
    - stop_start_klusterlet_scenario
    managedcluster_name: cluster1
    # label_selector:
    instance_count: 1
    runs: 1
    timeout: 60
```

## Expected Behavior

- **Stop scenario:** The ManagedCluster transitions to an unavailable state. OCM/ACM hub detects the cluster as unreachable after `5 * leaseDurationSeconds` (default 300 seconds). Workloads on the managed cluster continue running locally but are no longer managed from the hub.
- **Start scenario:** The ManagedCluster comes back online, re-registers with the hub, and the klusterlet re-establishes communication. ManifestWorks are reconciled and workloads return to a managed state.
- **Stop-start scenario:** The ManagedCluster is stopped, confirmed as unavailable, then restarted. After restart, full connectivity and management are restored within the configured timeout.
- **Klusterlet scenarios:** The klusterlet agent on the managed cluster is stopped or restarted. When stopped, the hub loses visibility into the managed cluster's status. When restarted, the agent reconnects and resumes reporting.
- **Recovery is automatic:** After a start or stop-start scenario, no manual intervention should be needed for the managed cluster to rejoin the hub.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ManagedCluster not found` | The `managedcluster_name` does not match any registered ManagedCluster on the hub | Verify managed clusters with `kubectl get managedclusters` and correct the name in the config |
| Timeout waiting for ManagedCluster to become unavailable | The `timeout` value is too low relative to `leaseDurationSeconds` | Increase `timeout` to at least `5 * leaseDurationSeconds` (default minimum 300s) |
| Klusterlet fails to restart | The klusterlet pod is stuck in CrashLoopBackOff or the managed cluster node is unhealthy | Check klusterlet pod logs with `kubectl logs -n open-cluster-management-agent` on the managed cluster and verify node health |
| `no ManagedCluster matching label_selector` | The `label_selector` does not match any ManagedCluster resources | List available labels with `kubectl get managedclusters --show-labels` and update the selector |
| `RBAC: forbidden` on ManifestWork creation | The kubeconfig user lacks permissions to create ManifestWork resources | Create a ClusterRole with ManifestWork and ManagedCluster permissions and bind it to the service account |
