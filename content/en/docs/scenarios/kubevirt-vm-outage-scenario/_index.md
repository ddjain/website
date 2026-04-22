---
title: KubeVirt VM Outage Scenario
description: Simulating VM-level disruptions in KubeVirt/OpenShift CNV environments
date: 2017-01-04
weight: 3
---

## Purpose

The `kubevirt_vm_outage` scenario deletes KubeVirt Virtual Machine Instances (VMIs) to test VM-level resilience and recovery in clusters where KubeVirt or OpenShift Containerized Network Virtualization (CNV) is installed. This helps users validate that VM monitoring, automatic restart policies, and high availability configurations work as expected when a VM crashes or is lost.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with access to target namespaces
- RBAC: ability to `delete`, `get`, `list`, and `create` VMIs in target namespaces
- KubeVirt or OpenShift CNV installed in the cluster
- Target VMI(s) exist and are in Running state in the specified namespace
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

## Parameters

<krkn-hub-scenario id="kubevirt-outage">

The scenario supports the following parameters:

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| vm_name | The name of the VMI to delete | Yes | N/A |
| namespace | The namespace where the VMI is located | No | "default" |
| timeout | How long to wait (in seconds) before attempting recovery for VMI to start running again | No | 60 |
| kill_count | How many VMI's to kill serially | No | 1 |

</krkn-hub-scenario>

## Use Cases

### Testing High Availability VM Configurations

This scenario is particularly useful for testing high availability configurations, such as:

- Clustered applications running across multiple VMs
- VMs with automatic restart policies
- Applications with cross-VM resilience mechanisms

### Validating VMI SSH Connection

While the KubeVirt outage is running you can enable KubeVirt checks to verify SSH connectivity to a list of VMIs, testing whether an outage of one VMI affects others becoming unready or unconnectable.
See more details on how to enable these checks in [KubeVirt checks](../../krkn/virt-checks.md).

## How to Run KubeVirt VM Outage Scenarios

Choose your preferred method to run KubeVirt VM outage scenarios:

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

When executed, the scenario will:

1. Validate that KubeVirt is installed and the target VMI exists
2. Save the initial state of the VMI
3. Delete the VMI
4. Wait for the VMI to become running or hit the timeout
5. Attempt to recover the VMI:
   - If the VMI is managed by a VirtualMachine resource with `runStrategy: Always`, it will automatically recover
   - If automatic recovery doesn't occur, the plugin will manually recreate the VMI using the saved state
6. Validate that the VMI is running again

{{% alert title="Note" %}}If the VM is managed by a VirtualMachine resource with `runStrategy: Always`, KubeVirt will automatically try to recreate the VMI after deletion. In this case, the scenario will wait for this automatic recovery to complete.{{% /alert %}}

### Recovery Strategies

The plugin implements two recovery strategies:

1. **Automated Recovery**: If the VM is managed by a VirtualMachine resource with `runStrategy: Always`, the plugin will wait for KubeVirt's controller to automatically recreate the VMI.

2. **Manual Recovery**: If automatic recovery doesn't occur within the timeout period, the plugin will attempt to manually recreate the VMI using the saved state from before the deletion.

### Recovery Time Metrics in Krkn Telemetry

Krkn tracks three key recovery time metrics for each affected VMI:

1. **pod_rescheduling_time** - The time (in seconds) that the Kubernetes cluster took to reschedule the VMI after it was deleted. This measures the cluster's scheduling efficiency and includes the time from VMI deletion until the replacement VMI is scheduled on a node.

2. **pod_readiness_time** - The time (in seconds) the VMI took to become ready after being scheduled. This measures VMI startup time, including container image pulls, VM boot process, and readiness probe success.

3. **total_recovery_time** - The total amount of time (in seconds) from VMI deletion until the replacement VMI became fully ready and available. This is the sum of rescheduling time and readiness time.

These metrics appear in the telemetry output under `PodsStatus.recovered` for successfully recovered VMIs. VMIs that fail to recover within the timeout period appear under `PodsStatus.unrecovered` without timing data.

**Example telemetry output:**
```json
{
  "recovered": [
    {
      "pod_name": "virt-launcher-fedora-vm-xyz",
      "namespace": "default",
      "pod_rescheduling_time": 3.2,
      "pod_readiness_time": 12.5,
      "total_recovery_time": 15.7
    }
  ],
  "unrecovered": []
}
```

### Rollback Scenario Support

Krkn supports rollback for KubeVirt VM Outage Scenario. For more details, please refer to the [Rollback Scenarios](../../rollback-scenarios/_index.md) documentation.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `VMI not found` or target VMI does not exist | The `vm_name` does not match any VMI in the specified namespace | Verify VMI name with `kubectl get vmi -n <namespace>` and correct the parameter |
| `namespace not found` | The `namespace` value does not match any namespace in the cluster | Check namespace exists with `kubectl get ns` and fix the namespace parameter |
| `insufficient RBAC for VMI deletion` | The kubeconfig user/service account lacks permissions to delete or create VMIs | Create a ClusterRole with `virtualmachineinstances: [get, list, delete, create]` and bind it to the service account |
| VMI not recovered within timeout | The VirtualMachine resource does not have `runStrategy: Always`, or the manual recreation failed | Increase the `timeout` parameter, verify `runStrategy` with `kubectl get vm <name> -o yaml`, or check KubeVirt controller logs |
| `KubeVirt not installed` or CRDs missing | KubeVirt or OpenShift CNV is not properly installed in the cluster | Install KubeVirt following the [official docs](https://kubevirt.io/user-guide/) and verify CRDs with `kubectl get crd \| grep kubevirt` |
| VMI spec changed during outage | The VM spec was modified between deletion and manual recovery, causing a mismatch | Re-run the scenario after the spec stabilizes; the saved state reflects the pre-deletion spec |

### Limitations

- The scenario currently supports deleting a single VMI at a time
- If VM spec changes during the outage window, the manual recovery may not reflect those changes
- The scenario doesn't simulate partial VM failures (e.g., VM freezing) — only complete VM outage
