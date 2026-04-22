---
title: Time Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

Time scenarios change the system time and/or date for targeted pods or nodes on a Kubernetes/OpenShift cluster. Use this to validate that your applications, scheduled jobs, and time-dependent services handle clock skew or time jumps without data corruption or service disruption.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with access to target namespaces and nodes
- RBAC: ability to `exec` into pods, `get`/`list` pods and nodes in target namespaces
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

<krkn-hub-scenario id="time-scenarios">

Using this type of scenario configuration, one is able to change the time and/or date of the system for pods or nodes.

</krkn-hub-scenario>

## How to Run Time Scenarios

Choose your preferred method to run time scenarios:

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

- **Time Shift Applied:** The targeted pods or nodes reflect the configured time/date offset during the chaos window.
- **Application Resilience:** Time-dependent services (cron jobs, certificate validation, token expiry, log timestamps) continue to function correctly despite the clock skew.
- **Automatic Recovery:** After the scenario completes, the system time on affected pods or nodes returns to the correct value without manual intervention.
- **No Data Corruption:** Databases, caches, and event logs remain consistent and do not produce duplicate or out-of-order entries due to the time shift.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `no pods matching label selector` | The label selector does not match any pods in the target namespace | Verify labels with `kubectl get pods -n <namespace> --show-labels` and update the selector |
| `permission denied` when setting system time | The pod or node lacks the required capabilities (e.g., `SYS_TIME`) | Ensure the target container has `SYS_TIME` capability or run with sufficient privileges |
| Time does not revert after scenario completes | The recovery mechanism failed or the scenario was interrupted | Manually sync the clock with `ntpdate` or `chronyc makestep`, or restart the affected pod |
| `connection refused` or `cluster unreachable` | The kubeconfig points to an inaccessible cluster | Verify cluster access with `kubectl get nodes` and check the kubeconfig path |

## Demo
See a demo of this scenario:
<script src="https://asciinema.org/a/zMBAdzHE40oPXkFVIz0exEoOX.js" id="asciicast-zMBAdzHE40oPXkFVIz0exEoOX" async="true"  style="max-width:900px; max-height:400px; width:100%; aspect-ratio:20/9;"></script>
