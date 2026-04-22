---
title: Container Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

<krkn-hub-scenario id="container-scenarios">
Kills specific containers within a pod using `kill` signals sent via `oc exec`. Use this to test how your application handles individual container crashes without full pod deletion — validating container restart policies and multi-container pod resilience.
</krkn-hub-scenario>

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with cluster-admin access
- RBAC: ability to `exec` into pods and `get`/`list` pods in target namespaces
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

## How to Run Container Scenarios

Choose your preferred method to run container scenarios:

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

- The targeted container is killed with the specified signal (e.g., SIGKILL, SIGTERM)
- Kubernetes restarts the container according to the pod's `restartPolicy`
- The pod itself may not be rescheduled — the container restarts in-place within the same pod
- Recovery metrics appear in telemetry output:

**Recovery Time Metrics:**

1. **pod_rescheduling_time** - Time for the cluster to reschedule the pod. Often 0.0 seconds for container kills since the pod stays on the same node.
2. **pod_readiness_time** - Time for the pod to become ready after the container restart.
3. **total_recovery_time** - Total time from container kill until the pod is fully ready.

```json
{
  "recovered": [
    {
      "pod_name": "backend-7d8f9c-xyz",
      "namespace": "production",
      "pod_rescheduling_time": 43.62235879898071,
      "pod_readiness_time": 0.0,
      "total_recovery_time": 43.62235879898071
    }
  ],
  "unrecovered": []
}
```

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `unable to exec into pod` | The pod or container does not exist, or RBAC lacks `exec` permission | Verify pod exists with `kubectl get pods -n <namespace>` and grant `pods/exec` permission |
| Container restarts but pod never becomes Ready | Readiness probe fails after container restart | Check probe configuration with `kubectl describe pod <pod>` and investigate application startup |
| `container not found` | The specified container name does not exist in the pod | List containers with `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].name}'` |

## Demo
See a demo of this scenario:
<script src="https://asciinema.org/a/452375.js" id="asciicast-452375" async="true" style="max-width:900px; max-height:400px; width:100%; aspect-ratio:20/9;"></script>
