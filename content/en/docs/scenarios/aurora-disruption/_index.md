---
title: Aurora Disruption Scenario
description:
date: 2017-01-04
weight: 3
---

## Purpose

Blocks a pod's outgoing MySQL and PostgreSQL traffic, effectively preventing it from connecting to any AWS Aurora SQL engine. It works just as well for standard MySQL and PostgreSQL connections too. This uses the pod network filter scenario but set with specific parameters to disrupt Aurora.

## Preconditions

- Running Kubernetes (1.21+) or OpenShift cluster
- Valid kubeconfig with cluster-admin access
- RBAC: ability to `get`/`list` pods and manage network policies in target namespaces
- AWS Aurora endpoint accessible from the cluster (or standard MySQL/PostgreSQL endpoint)
- Container runtime: Docker (20.10+) or Podman (4.0+) — required for krkn-hub and krknctl methods

## How to Run Aurora Disruption Scenarios

Choose your preferred method to run aurora disruption scenarios:

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

- Outgoing traffic on MySQL (port 3306) and PostgreSQL (port 5432) ports is blocked from the targeted pod
- Database connections from the application fail with connection timeout or refused errors
- Applications relying on Aurora (or standard MySQL/PostgreSQL) see connection errors and may trigger retry logic or circuit breakers
- Once the disruption duration expires, network traffic is restored and database connections resume

## Failure Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Traffic is not blocked and database connections still succeed | The target pod label selector does not match any running pods | Verify pod labels with `kubectl get pods -n <namespace> --show-labels` and correct the selector |
| Disruption runs but application shows no errors | The application connects to the database through a sidecar or proxy not affected by the filter | Confirm the application pod directly initiates database connections and adjust the target accordingly |
| `iptables` or `tc` rule injection fails | The pod's network namespace is inaccessible or the node lacks required kernel modules | Check node kernel capabilities and ensure the container runtime grants the necessary privileges |
