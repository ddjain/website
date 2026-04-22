---
title: Pod Network Scenarios
description:
date: 2017-01-04
weight: 3
---

## Purpose

Pod network scenarios introduce network-level chaos at the pod layer, including traffic blocking (ingress/egress outage), latency injection, packet loss, and bandwidth restriction. These scenarios help validate how services and their dependencies behave under degraded network conditions, enabling teams to improve timeouts, alerting, and overall resilience.

## Preconditions

| Requirement | Details |
|---|---|
| Kubernetes version | 1.21+ |
| kubeconfig | Valid kubeconfig with access to the target cluster |
| RBAC permissions | Permissions to list, get, and exec into pods in the target namespace |
| Container runtime | Supported container runtime (CRI-O, containerd, Docker) |
| Network utilities | `tc` and `netem` kernel modules available on cluster nodes |
| Network plugin | OpenShiftSDN or OVNKubernetes (for pod outage OVS flow rules) |

<krkn-hub-scenario id="pod-network-chaos">

### Pod outage
Scenario to block the traffic (Ingress/Egress) of a pod matching the labels for the specified duration of time to understand the behavior of the service/other services which depend on it during downtime. This helps with planning the requirements accordingly, be it improving the timeouts or tweaking the alerts etc.
With the current network policies, it is not possible to explicitly block ports which are enabled by allowed network policy rule. This chaos scenario addresses this issue by using OVS flow rules to block ports related to the pod. It supports OpenShiftSDN and OVNKubernetes based networks.

#### Excluding Pods from Network Outage

The pod outage scenario now supports excluding specific pods from chaos testing using the `exclude_label` parameter. This allows you to target a namespace or group of pods with your chaos testing while deliberately preserving certain critical workloads.

##### Why Use Pod Exclusion?

This feature addresses several common use cases:

- Testing resiliency of an application while keeping critical monitoring pods operational
- Preserving designated "control plane" pods within a microservice architecture
- Allowing targeted chaos without affecting auxiliary services in the same namespace
- Enabling more precise pod selection when network policies require all related services to be in the same namespace

##### How to Use the `exclude_label` Parameter

The `exclude_label` parameter works alongside existing pod selection parameters (`label_selector` and `pod_name`). The system will:
1. Identify all pods in the target namespace
2. Exclude pods matching the `exclude_label` criteria (in format "key=value")
3. Apply the existing filters (`label_selector` or `pod_name`)
4. Apply the chaos scenario to the resulting pod list



##### Example Configurations

**Basic exclude configuration:**
```yaml
- id: pod_network_outage
  config:
    namespace: my-application
    label_selector: "app=my-service"
    exclude_label: "critical=true"
    direction:
      - egress
    test_duration: 600
```

In this example, network disruption is applied to all pods with the label `app=my-service` in the `my-application` namespace, except for those that also have the label `critical=true`.

**Complete scenario example:**
```yaml
- id: pod_network_outage
  config:
    namespace: openshift-console
    direction:
      - ingress
    ingress_ports:
      - 8443
    label_selector: 'component=ui'
    exclude_label: 'excluded=true'
    test_duration: 600
```

This scenario blocks ingress traffic on port 8443 for pods matching `component=ui` label in the `openshift-console` namespace, but will skip any pods labeled with `excluded=true`.

The `exclude_label` parameter is also supported in the pod network shaping scenarios (`pod_egress_shaping` and `pod_ingress_shaping`), allowing for the same selective application of network latency, packet loss, and bandwidth restriction.

</krkn-hub-scenario>

## How to Run Pod Network Scenarios

Choose your preferred method to run pod network scenarios:

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

When a pod network scenario runs successfully:

- **Targeted pods** experience network degradation matching the configured chaos type (outage, latency, packet loss, or bandwidth restriction) for the specified `test_duration`.
- **Ingress/Egress traffic** is selectively disrupted based on the `direction` parameter; only the configured directions and ports are affected.
- **Monitoring metrics** show increased latency, elevated packet loss rates, or reduced throughput on the affected pods during the chaos window.
- **Dependent services** exhibit timeout errors, retry storms, or degraded responses when communicating with the disrupted pods.
- **Recovery** occurs automatically after the test duration expires; network conditions return to normal without manual intervention.
- **Excluded pods** (those matching `exclude_label`) continue operating normally throughout the test.

## Failure Handling

| Symptom | Cause | Fix |
|---|---|---|
| Scenario fails to start with permission error | Insufficient RBAC permissions to exec into pods or manage network policies | Ensure the service account has `pods/exec` and `networkpolicies` permissions in the target namespace |
| Network disruption not applied to target pods | Label selector does not match any running pods in the namespace | Verify `label_selector` and `namespace` values; run `kubectl get pods -n <namespace> -l <label>` to confirm matches |
| Traffic still flows during outage scenario | Network plugin is not OpenShiftSDN or OVNKubernetes, so OVS flow rules cannot be applied | Confirm the cluster uses a supported network plugin; for unsupported plugins, use shaping scenarios instead |
| Pods remain disrupted after test duration | The cleanup process was interrupted or the krkn pod was terminated mid-test | Manually remove OVS flow rules or network policies; restart affected pods to restore normal traffic |
| `tc`/`netem` errors in scenario logs | The `tc` utility or `netem` kernel module is not available on the node | Ensure the node image includes `iproute2` and the `sch_netem` kernel module is loaded |
