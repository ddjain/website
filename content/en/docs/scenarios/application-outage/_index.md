---
title: Application Outage Scenarios
description: 
date: 2017-01-04
weight: 3
---

## Purpose

<krkn-hub-scenario id="application-outages">
Blocks the traffic (Ingress/Egress) of an application matching the specified labels for a configured duration. Use this to understand how your service and its dependents behave during network isolation — helping you plan timeout improvements, alert tuning, and dependency resilience.
</krkn-hub-scenario>

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with cluster-admin access
- RBAC: ability to create and delete NetworkPolicies in target namespaces
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods
- Target namespace must support NetworkPolicy enforcement (CNI plugin must support it — e.g., Calico, Cilium, OVN-Kubernetes)

You can add your application's URL into the [health checks section](../../krkn/config.md#health-checks) of the config to track downtime during this scenario.

## How to Run Application Outage Scenarios

Choose your preferred method to run application outage scenarios:

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

- A NetworkPolicy named `kraken-deny` is created in the target namespace, blocking all Ingress and Egress traffic to matching pods
- During the outage window, the targeted application should be unreachable — dependent services should show timeouts or fallback behavior
- After the configured duration, Krkn automatically deletes the NetworkPolicy and traffic resumes
- Recovery indicators: application pods respond to health probes, dependent services reconnect, and monitoring alerts clear

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Traffic still flowing during outage | The CNI plugin does not support NetworkPolicy enforcement | Verify your CNI supports NetworkPolicy (Calico, Cilium, OVN-Kubernetes). Flannel does not support it by default. |
| NetworkPolicy not cleaned up after failure | Krkn crashed before the cleanup phase | Manually delete the policy: `kubectl delete networkpolicy kraken-deny -n <targeted-namespace>` |
| `forbidden: cannot create networkpolicies` | Insufficient RBAC permissions | Grant `create`/`delete` permissions on `networkpolicies` to the service account |
| Application does not recover after outage ends | The application lacks proper reconnection/retry logic | Check application health probes and restart pods if needed: `kubectl rollout restart deployment/<name> -n <namespace>` |
| Dependent services permanently degraded | Circuit breakers tripped and did not reset | Reset circuit breakers manually or wait for the configured reset timeout |

### Rollback Scenario Support

Krkn supports rollback for Application outages. For more details, please refer to the [Rollback Scenarios](../../rollback-scenarios/_index.md) documentation.

#### Demo
See a demo of this scenario:
<script src="https://asciinema.org/a/452403.js" id="asciicast-452403" async="true" style="max-width:900px; max-height:400px; width:100%; aspect-ratio:20/9;" ></script>
