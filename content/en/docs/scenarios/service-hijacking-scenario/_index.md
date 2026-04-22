---
title: Service Hijacking Scenario
description:
date: 2017-01-04
weight: 3
---

## Purpose

Service Hijacking Scenarios hijack HTTP traffic to Kubernetes Services by replacing the target Service's pod selector with a mock web service that returns custom responses. Use this to validate that your application handles unexpected or degraded HTTP responses gracefully and recovers once the original service is restored.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with access to target namespaces
- RBAC: ability to `get`/`update` Services and `create`/`delete` Deployments in target namespaces
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

## How It Works

<krkn-hub-scenario id="service-hijacking">
Service Hijacking Scenarios aim to simulate fake HTTP responses from a workload targeted by a Service already deployed in the cluster. This scenario is executed by deploying a custom-made web service and modifying the target Service selector to direct traffic to this web service for a specified duration. 

The web service's source code is available [here](https://github.com/krkn-chaos/krkn-service-hijacking). 
</krkn-hub-scenario>


It employs a time-based test plan from the scenario configuration file, which specifies the behavior of resources during the chaos scenario as follows:

The scenario will focus on the `service_name` within the `service_namespace`, 
substituting the selector with a randomly generated one, which is added as a label in the mock service manifest.
This allows multiple scenarios to be executed in the same namespace, each targeting different services without causing conflicts.

The newly deployed mock web service will expose a `service_target_port`, 
which can be either a named or numeric port based on the service configuration. 
This ensures that the Service correctly routes HTTP traffic to the mock web service during the chaos run.

Each step will last for `duration` seconds from the deployment of the mock web service in the cluster. 
For each HTTP resource, defined as a top-level YAML property of the plan 
(it could be a specific resource, e.g., /list/index.php, or a path-based resource typical in MVC frameworks), 
one or more HTTP request methods can be specified. Both standard and custom request methods are supported.

During this time frame, the web service will respond with:

- `status`: The [HTTP status code](https://datatracker.ietf.org/doc/html/rfc7231#section-6) (can be standard or custom).
- `mime_type`: The [MIME type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types) (can be standard or custom).
- `payload`: The response body to be returned to the client.

At the end of the step `duration`, the web service will proceed to the next step (if available) until 
the global `chaos_duration` concludes. At this point, the original service will be restored, 
and the custom web service and its resources will be undeployed.

__NOTE__: Some clients (e.g., cURL, jQuery) may optimize queries using lightweight methods (like HEAD or OPTIONS) 
to probe API behavior. If these methods are not defined in the test plan, the web service may respond with 
a `405` or `404` status code. If you encounter unexpected behavior, consider this use case.


## How to Run Service Hijacking Scenarios

Choose your preferred method to run service hijacking scenarios:

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

- **Targeted Service Returns Custom Responses:** During the chaos window, the hijacked Service routes all traffic to the mock web service, which responds with the configured `status`, `mime_type`, and `payload` for each HTTP resource and method defined in the test plan.
- **Clients See Hijacked Responses:** Any client connecting to the Service (other pods, ingress controllers, external consumers) receives the mock responses instead of the real backend, allowing you to observe how upstream and downstream components handle error codes, unexpected payloads, or degraded responses.
- **Automatic Restoration:** Once the `chaos_duration` expires, the original Service selector is restored, the mock web service Deployment is deleted, and traffic resumes flowing to the real backend pods without manual intervention.

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `service not found` | The `service_name` or `service_namespace` does not match any Service in the cluster | Verify the Service exists with `kubectl get svc -n <namespace>` and correct the name or namespace in the scenario config |
| Mock web service pod stays in `Pending` or `CrashLoopBackOff` | Insufficient cluster resources or the `service_target_port` is invalid | Check pod events with `kubectl describe pod -n <namespace>` and ensure the target port matches the Service spec |
| Clients still receive real backend responses during chaos | The Service selector was not updated, or the mock pod is not ready before traffic is routed | Confirm the Service selector was modified with `kubectl get svc <name> -n <namespace> -o yaml` and check mock pod readiness |
| `405 Method Not Allowed` or `404 Not Found` from mock service | The HTTP method used by the client (e.g., HEAD, OPTIONS) is not defined in the test plan | Add the missing HTTP methods to the test plan configuration for the targeted resource |
| Original Service not restored after chaos ends | The scenario process was interrupted or killed before cleanup | Manually restore the original selector on the Service with `kubectl edit svc <name> -n <namespace>` and delete any leftover mock Deployments |
