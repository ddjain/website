---
title: PVC Scenario
description:
date: 2017-01-04
weight: 3
---
<krkn-hub-scenario id="pvc-scenarios">
Scenario to fill up a given PersistenVolumeClaim by creating a temp file on the PVC from a pod associated with it. The purpose of this scenario is to fill up a volume to understand faults caused by the application using this volume.
</krkn-hub-scenario>

## Purpose

The PVC scenario fills up a given PersistentVolumeClaim by creating a temporary file on the PVC from a pod associated with it. The purpose is to simulate disk pressure conditions and understand how applications behave when their underlying storage volume reaches capacity.

## Preconditions

Before running this scenario, ensure the following requirements are met:

- **Kubernetes 1.21+** cluster is running and accessible
- **kubeconfig** is configured and points to the target cluster
- **RBAC permissions** to list, get, and exec into pods in the target namespace
- **PVC in target namespace** must exist, be in `Bound` state, and be mounted to an active pod
- **Container runtime** (Podman or Docker) is available if using krkn-hub

## How to Run PVC Scenarios

Choose your preferred method to run PVC scenarios:

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

When the scenario runs successfully:

- The target PVC is identified and its current usage is calculated.
- A temporary file (`kraken.tmp`) filled with random data is created on the PVC mount path, bringing usage up to the configured `fill_percentage`.
- The disk fills up to the specified percentage, which may cause the application using the volume to exhibit degraded performance, throw errors, or enter a crash loop depending on how it handles storage pressure.
- After the configured `duration` elapses, the temporary file is automatically removed and the PVC returns to its original usage level.
- If Cerberus is enabled, the scenario reports a pass/fail status based on cluster health during and after the fault injection.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Scenario fails with "PVC not found" | The specified PVC name does not exist in the target namespace | Verify the PVC name and namespace are correct with `kubectl get pvc -n <namespace>` |
| Scenario fails with "PVC not bound" | The target PVC is in `Pending` or `Lost` state | Ensure the PVC is bound to a PersistentVolume and a pod is actively using it |
| Pod exec fails or times out | Insufficient RBAC permissions or the pod is in a non-running state | Check that your kubeconfig has exec permissions and that the pod mounting the PVC is in `Running` state |
| Fill percentage not reached | The PVC does not have enough free space to reach the target percentage, or the current usage already exceeds the target | Set `fill_percentage` to a value higher than the current usage; verify current usage with `kubectl exec` and `df` on the mount path |
| Temporary file not cleaned up | The scenario was interrupted before the duration expired (e.g., process killed) | Manually remove `/mount_path/kraken.tmp` from the pod using `kubectl exec` |
