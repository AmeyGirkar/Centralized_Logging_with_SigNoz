# Centralized Logging with SigNoz on Kubernetes

This repository contains the configuration and instructions to deploy SigNoz on a Kubernetes cluster for centralized logging, with support for log tagging (Namespace, Service name) and retention.

## Cluster Prerequisites

-   **Kubernetes Cluster**: Version 1.21 or higher.
-   **Helm**: Version 3.0 or higher.
-   **Resources**: SigNoz requires significant resources. Ensure your cluster has at least 4 vCPUs and 8GB RAM available for a basic deployment (ClickHouse, OTel Collector, Query Service, etc.).
-   **Storage**: A Default StorageClass is required for ClickHouse and Query Service persistence.

## 1. Deploy SigNoz

Follow these steps to deploy SigNoz using the provided `values.yaml`:

```bash
# Add SigNoz Helm repository
helm repo add signoz https://charts.signoz.io
helm repo update

# Create namespace
kubectl create namespace signoz

# Install SigNoz
helm install signoz signoz/signoz -n signoz -f k8s/signoz/values.yaml
```

## 2. Centralized Logging Configuration

The provided `k8s/signoz/values.yaml` configures the SigNoz OTel Agent to:
-   Automatically scrape logs from all pods.
-   Add Kubernetes metadata (`namespace`, `pod_name`, `container_name`).
-   Filter logs to ensure high-quality data.

### How it works:
-   **Namespace Tagging**: The `k8sattributes` processor extracts the namespace from the Kubernetes API or pod labels.
-   **Service Name Tagging**: SigNoz looks for the `service.name` attribute. If your pods have the label `service.name`, it will be automatically picked up.

## 3. How to Onboard a New Microservice

To ensure logs from your microservice are properly tagged and searchable in SigNoz:

1.  **Label your Pods**: Add the label `service.name` to your pod template.
2.  **Standard Labels**: SigNoz also attempts to derive service names from `app.kubernetes.io/name` or the Deployment name if `service.name` is missing.

**Example Pod Spec:**
```yaml
template:
  metadata:
    labels:
      service.name: "my-awesome-service"
```

## 4. Verification

Deploy the sample microservice to verify logging:

```bash
kubectl apply -f k8s/sample-app/service.yaml
```

1.  Port-forward the SigNoz Query Service:
    `kubectl port-forward -n signoz svc/signoz-query-service 3301:3301`
2.  Access the UI at `http://localhost:3301`.
3.  Go to the **Logs** tab.
4.  Filter by `k8s_namespace_name` = `sample-app` or `service_name` = `log-generator-service`.

## 5. Retention Policy

### Via UI (Recommended)
1.  Go to **Settings** > **General** in the SigNoz UI.
2.  Locate the **Log Retention** section.
3.  Set the desired number of days (N days).

### Via ClickHouse TTL (Advanced)
The retention is ultimately managed by ClickHouse TTL. You can set this in the Helm chart or by running a SQL command in ClickHouse:

```sql
ALTER TABLE signoz_logs.logs MODIFY COLUMN timestamp TTL timestamp + INTERVAL 7 DAY;
```
*(Default retention is often 30 days in SigNoz).*
