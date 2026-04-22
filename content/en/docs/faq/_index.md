---
title: FAQ
description: Frequently asked questions about running Krkn chaos scenarios
weight: 14
---

### Why is my scenario failing silently?

Check the scenario exit code. For krkn-hub: `podman inspect <container> --format "{{.State.ExitCode}}"`. A non-zero exit code means the scenario failed. Enable verbose logging or check `CERBERUS_ENABLED` — Cerberus may be detecting cluster issues silently. For krknctl, run in attached mode (without `--detached`) to see real-time logs.

### Can I run multiple scenarios in parallel?

**krknctl**: Yes — use the `graph` command to define parallel execution plans. See [graph orchestration](../krknctl/usage.md) for details.

**krkn (standalone)**: Yes — you can list multiple scenario types in a single `config.yaml` file and they execute sequentially within one run. For true parallelism, run multiple krkn instances.

**krkn-hub**: Each container runs one scenario. Run multiple containers simultaneously for parallel execution.

### What are the risks of running chaos scenarios in production?

Chaos scenarios intentionally disrupt your systems. Key risks:
- **Data loss** if targeting stateful workloads without backups
- **Extended outages** if recovery mechanisms are misconfigured
- **Cascading failures** across dependent services

Mitigate by: starting in staging, using `exclude_label` to protect critical pods, setting conservative timeouts, and monitoring with Cerberus.

### How do I roll back a failed scenario?

Krkn supports automatic rollback via versioned rollback files. If a scenario fails mid-execution:

```bash
# List pending rollbacks
python run_kraken.py list-rollback --config config/config.yaml

# Execute rollback
python run_kraken.py execute-rollback --config config/config.yaml
```

For manual rollback of specific scenarios:
- **Application outage**: `kubectl delete networkpolicy kraken-deny -n <namespace>`
- **Hog scenarios**: `kubectl delete pod -l <hog-label> -n <namespace>`

See [Rollback Scenarios](../rollback-scenarios/) for full details.

### Which scenario should I start with?

Start with **Pod Scenarios** using the `pod-scenarios` image. It's the simplest and safest — it deletes pods and verifies Kubernetes recreates them. Use the [Getting Started](../getting-started/) guide for a copy-paste walkthrough. Once comfortable, try `container-scenarios` or `time-scenarios` before moving to more disruptive scenarios.

### Do I need Cerberus running?

No. Cerberus is optional but recommended. Without it, scenarios still run but you won't get automated cluster health verification during the chaos injection. To disable it, set `CERBERUS_ENABLED=false` in your environment or config.

### How do I run scenarios in CI/CD?

Use **krkn-hub** containers or **krknctl** in detached mode:

```bash
# krkn-hub in CI
podman run --net=host \
  -v $KUBECONFIG:/home/krkn/.kube/config:Z \
  -e NAMESPACE=default \
  -e POD_LABEL="app=my-app" \
  quay.io/krkn-chaos/krkn-hub:pod-scenarios

# Check pass/fail
EXIT_CODE=$(podman inspect <container> --format "{{.State.ExitCode}}")
if [ "$EXIT_CODE" -ne 0 ]; then echo "Chaos test failed"; exit 1; fi
```

```bash
# krknctl in CI
krknctl run pod-scenarios --namespace default --pod-label "app=my-app"
# Exit code is automatically returned
```

Use the `query-status` command for graph-based executions to check overall status.

### What Kubernetes versions are supported?

Krkn supports Kubernetes 1.21+ and OpenShift 4.x. Specific cloud provider scenarios (node stop/start, zone outage) require the corresponding cloud CLI and credentials. Check each scenario's **Preconditions** section for provider-specific requirements.

### How do I target specific pods or nodes?

**Targeting pods:**
- `--pod-label` / `label_selector`: Match pods by label (e.g., `app=backend`)
- `--name-pattern` / `name_pattern`: Match pods by regex pattern
- `--namespace` / `namespace_pattern`: Scope to specific namespaces (supports regex)
- `--exclude-label` / `exclude_label`: Exclude specific pods from chaos

**Targeting nodes:**
- `node_label_selector`: Target nodes by label (e.g., `node-role.kubernetes.io/worker=`)
- `node_names`: Target specific nodes by name

See [Pod Scenarios](../scenarios/pod-scenario/) for detailed examples.

### What's the difference between krknctl, krkn-hub, and krkn?

| Tool | What It Is | Best For |
|------|-----------|----------|
| **krknctl** | CLI tool with auto-completion and validation | Interactive use, quick experiments, orchestrated runs |
| **krkn-hub** | Pre-built container images for each scenario | CI/CD pipelines, reproducible tests, air-gapped environments |
| **krkn** | Core Python chaos engine | Running multiple scenario types in a single execution, maximum control |

All three use the same underlying chaos logic. krknctl and krkn-hub use container images; krkn runs directly as a Python program.
