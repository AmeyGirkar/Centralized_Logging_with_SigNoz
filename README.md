# Centralized Logging with SigNoz on Kubernetes

This repository contains the configuration and instructions to deploy SigNoz on a Kubernetes cluster for centralized logging, with support for multi-node clusters via infrastructure agents.

## Cluster Prerequisites

-   **Kubernetes Cluster**: Version 1.21 or higher.
-   **Helm**: Version 3.0 or higher.
-   **Resources**: SigNoz requires significant resources (at least 4 vCPUs and 8GB RAM).
-   **Storage**: A Default StorageClass is required for persistence.

## 1. Deploy SigNoz (Platform)

The main chart installs the SigNoz backend, UI, and Clickhouse.

```bash
# Add SigNoz Helm repository
helm repo add signoz https://charts.signoz.io
helm repo update

# Create namespace
kubectl create namespace signoz

# Install SigNoz Platform
helm upgrade --install signoz signoz/signoz -n signoz -f k8s/signoz/values.yaml
```

## 2. Deploy SigNoz k8s-infra (Agents for Logs & Metrics)

The `k8s-infra` chart installs the agents as a **DaemonSet** on every node. This is required to collect logs from your application pods.

```bash
# Install Infrastructure Agents
helm upgrade --install signoz-infra signoz/k8s-infra -n signoz -f k8s/signoz/k8s-infra-values.yaml
```

## 3. Centralized Logging Configuration

The `k8s/signoz/k8s-infra-values.yaml` configures the agents to:
-   **Application Logs**: Scrape logs from all pods (except `kube-system`).
-   **Metadata**: Automatically tag logs with Namespace, Pod Name, and Service Name.
-   **Multi-node Support**: Works reliably on remote clusters (not just single-node).

## 4. Verification

1.  **Check Pods**: Verify both the platform and the agents are running.
    ```bash
    kubectl get pods -n signoz
    ```
    *You should see `signoz-otel-collector-*` AND multiple `signoz-infra-otel-agent-*` pods.*

2.  **Access UI**: The SigNoz UI is on NodePort `30080`.
    Access at `http://<NODE_IP>:30080`.

3.  **Check Logs**: Go to the **Logs** tab and filter by `k8s_namespace_name`.

## 5. Retention Policy

Retention is configured for 7 days in `k8s/signoz/values.yaml`:
```yaml
clickhouse:
  configData:
    ttl_for_logs: 7
```

## 6. Troubleshooting

If logs are missing:
1.  Check if agents are running: `kubectl get ds -n signoz`.
2.  Check agent logs for errors: `kubectl logs -n signoz -l app.kubernetes.io/component=otel-agent`.
3.  Ensure the `otelCollector` endpoint is reachable from the agents.
