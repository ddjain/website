---
title: Troubleshooting
description: Common errors and how to fix them when running Krkn chaos scenarios
weight: 13
---

This page covers cross-cutting errors you may encounter when running chaos scenarios with Krkn, krkn-hub, or krknctl. For scenario-specific errors, see the **Failure Handling** section on each scenario page.

## Common Errors

| # | Symptom | Cause | Fix |
|---|---------|-------|-----|
| 1 | `invalid kubeconfig` or `unable to load kubeconfig` | Kubeconfig file is missing, corrupted, or points to a non-existent cluster | Verify path: `ls ~/.kube/config`. Regenerate if needed: `kubectl config view --flatten > ~/.kube/config`. For krknctl, pass explicitly: `--kubeconfig /path/to/config`. |
| 2 | `Unauthorized` or `forbidden: User cannot...` | The kubeconfig user or service account lacks required RBAC permissions | Create a ClusterRole with the needed verbs (`get`, `list`, `delete`, `create`) and bind it: `kubectl create clusterrolebinding krkn-admin --clusterrole=cluster-admin --serviceaccount=<ns>:<sa>` |
| 3 | `Cannot connect to the Docker daemon` or `dial unix /run/podman/podman.sock: connect: no such file or directory` | Container runtime (Docker/Podman) is not running or the socket is not accessible | **Docker:** `sudo systemctl start docker`. **Podman:** `systemctl enable --user --now podman.socket`. Verify: `docker ps` or `podman ps`. |
| 4 | `ImagePullBackOff` or `ErrImagePull` on krkn-hub containers | Cannot pull container images from `quay.io/krkn-chaos/krkn-hub` | Check internet connectivity from the cluster. For air-gapped environments, mirror images to a private registry and configure krknctl with `--private-registry` flags. |
| 5 | `dial tcp: lookup <cluster-api>: no such host` or `connection refused` | Cluster API server is unreachable from where you're running krkn | Verify cluster access: `kubectl get nodes`. Check VPN/firewall rules. Ensure the API server URL in kubeconfig is correct. |
| 6 | `cerberus is not running` or health check failures | Cerberus is configured but not started, or the URL is incorrect | Start Cerberus before running scenarios. Verify the URL in your config matches where Cerberus is listening. If you don't need Cerberus, set `CERBERUS_ENABLED=false`. |
| 7 | Scenario times out with no visible action | The timeout is too short for the operation, or the target selector matches nothing | Increase timeout values. Verify selectors match actual resources: `kubectl get pods -l <label> -n <namespace>`. |
| 8 | `namespace not found` or no pods matching selector | The namespace regex or label selector does not match any resources | Check with `kubectl get ns` and `kubectl get pods --show-labels -n <namespace>`. Fix the regex pattern or label value. |
| 9 | Node stuck in `NotReady` after node scenario | The node failed to recover after crash or stop scenario | Manually reboot the node from your cloud console. For `node_crash_scenario`, this is expected behavior — see the [Node Scenarios](../scenarios/node-scenarios/) page. |
| 10 | `resource limits exceeded` or pod evicted during hog scenario | The hog workload exceeded node capacity and triggered eviction | Reduce the stress parameters (memory target, CPU workers) or target a node with more available resources. |

## Reading Krkn Logs

Krkn outputs structured logs that help diagnose issues. Key patterns to look for:

**Successful run:**
```
[INFO] Starting chaos scenario: pod_disruption_scenarios
[INFO] Pod killed: backend-7d8f9c-xyz in namespace production
[INFO] Waiting for pod recovery...
[INFO] Pod recovered in 8.0 seconds
[INFO] Scenario completed successfully
```

**Failed run:**
```
[ERROR] Pod did not recover within 120 seconds
[ERROR] Unrecovered pods: [backend-7d8f9c-xyz]
[WARNING] Scenario marked as failed
```

**Common log patterns:**

| Log Pattern | Meaning |
|-------------|---------|
| `[INFO] Starting chaos scenario` | Scenario execution begins |
| `[INFO] Scenario completed successfully` | Scenario passed — all targets recovered |
| `[ERROR] ... did not recover within` | Recovery timeout exceeded — check the target workload |
| `[WARNING] Cerberus check failed` | Cluster health check detected an issue during the scenario |
| `[ERROR] Failed to connect to` | Network or authentication issue — check kubeconfig and connectivity |

### Viewing logs by method

**krknctl:**
```bash
# Attached mode — logs stream to terminal automatically
krknctl run pod-scenarios --namespace default --pod-label "app=test"

# Detached mode — reattach to see logs
krknctl run pod-scenarios --detached --namespace default --pod-label "app=test"
krknctl attach <scenario-id>
```

**krkn-hub:**
```bash
# Stream container logs
podman logs -f <container-name>
# or
docker logs -f <container-name>

# Check exit code (0 = pass, non-zero = fail)
podman inspect <container-name> --format "{{.State.ExitCode}}"
```

**krkn (standalone):**
```bash
# Logs output to terminal during execution
python run_kraken.py --config config/config.yaml

# Check the log file if configured
cat /tmp/kraken.log
```

## Cerberus Health Checks

[Cerberus](../cerberus/) monitors cluster health during chaos scenarios. If configured, it runs continuous checks and fails the scenario if the cluster becomes unhealthy.

**Verifying Cerberus is running:**
```bash
curl http://<cerberus-url>:8080/api/v1/status
```

Expected response: `{"status": "healthy"}`

**Common Cerberus issues:**

| Issue | Fix |
|-------|-----|
| Cerberus not reachable | Check the URL and port. Ensure Cerberus pod is running: `kubectl get pods -n <cerberus-ns>`. |
| False-positive failures | Review Cerberus config — it may be checking components not relevant to your scenario. Adjust watch namespaces. |
| Cerberus crashes during scenario | Increase Cerberus resource limits. It may run out of memory when monitoring many namespaces. |
