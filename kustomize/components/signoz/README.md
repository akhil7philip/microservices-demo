# Integrate Online Boutique with SigNoz

This component enables observability for Online Boutique using [SigNoz](https://signoz.io/), an open-source observability platform that provides logs, metrics, and traces visualization.

## Overview

SigNoz is an open-source alternative to DataDog, New Relic, etc. It uses OpenTelemetry for instrumentation and ClickHouse for storage. This integration enables:

- **Traces**: Distributed tracing across all microservices
- **Metrics**: Application and infrastructure metrics
- **Logs**: Kubernetes pod logs and application logs

## Prerequisites

- A GKE (Google Kubernetes Engine) cluster
  - **Standard GKE**: Full SigNoz functionality (logs, metrics, traces)
  - **GKE Autopilot**: Traces work perfectly; logs/metrics need alternatives (see [AUTOPILOT-OPTIONS.md](AUTOPILOT-OPTIONS.md))
- `kubectl` configured to access your cluster
- `helm` CLI installed (v3.0+)
- Sufficient cluster resources (SigNoz requires ~2-4GB RAM and 2-3 CPUs)

## Installation Steps

### Step 1: Install SigNoz on GKE

SigNoz requires **RWO (ReadWriteOnce)** block storage for ClickHouse. On GKE, you can use:
- **`standard-rwo`** (default, cheaper) - backed by PD CSI driver
- **`premium-rwo`** (faster, more expensive) - backed by PD CSI driver

#### Create SigNoz values file

Create the values file in your current directory (or any convenient location):

```bash
# From project root or any directory
cat > signoz-values.yaml <<'YAML'
global:
  storageClass: standard-rwo

clickhouse:
  installCustomStorageClass: true
YAML
```

**Note**: You can create this file anywhere - it's just a Helm values file. The path will be referenced in the `helm install` command below. Common locations:
- Project root directory
- `kustomize/components/signoz/` directory (for keeping SigNoz configs together)
- Any temporary directory

#### Add SigNoz Helm repository and install

```bash
helm repo add signoz https://charts.signoz.io
helm repo update

helm install signoz signoz/signoz \
  -n signoz --create-namespace \
  -f signoz-values.yaml \
  --wait --timeout 1h
```

**Note**: The installation may take 5-10 minutes. Wait for all pods to be ready:

```bash
kubectl get pods -n signoz
```

#### Access SigNoz UI

Port-forward to access the SigNoz UI:

```bash
kubectl port-forward -n signoz svc/signoz 8080:8080
```

Then open `http://localhost:8080` in your browser.

### Step 2: Install k8s-infra Agent for Logs and Metrics

The `k8s-infra` Helm chart collects Kubernetes cluster telemetry (logs and metrics) and sends it to SigNoz's OpenTelemetry collector.

#### ⚠️ GKE Autopilot Limitation

**If you're using GKE Autopilot**, the `k8s-infra` chart **will not work** because:
- Autopilot doesn't allow hostPath volumes (except `/var/log/`)
- Autopilot doesn't allow host ports
- k8s-infra requires these for collecting node-level metrics and logs

**For GKE Autopilot users**, see [Alternative: Collecting Logs and Metrics on Autopilot](#alternative-collecting-logs-and-metrics-on-gke-autopilot) below.

#### Find the SigNoz OTel collector service

```bash
kubectl get -n signoz svc | grep -E 'otel|collector'
```

The service is typically named `signoz-otel-collector`.

#### Install k8s-infra (Standard GKE only)

```bash
helm install -n signoz k8s-infra signoz/k8s-infra \
  --set otelCollectorEndpoint="signoz-otel-collector.signoz.svc.cluster.local:4317" \
  --set otelInsecure=true
```

#### Verify k8s-infra is running

```bash
kubectl get pods -n signoz | grep k8s-infra
```

### Step 3: Enable SigNoz Tracing for Online Boutique

From the `kustomize/` folder at the root level of this repository, execute:

```bash
kustomize edit add component components/signoz
```

This will update the `kustomize/kustomization.yaml` file to include:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- base
components:
- components/signoz
```

#### Deploy the updated configuration

```bash
kubectl apply -k .
```

This will patch the microservices to send traces to SigNoz's OpenTelemetry collector.

## What Gets Configured

When enabling this kustomize component, services will be patched with environment variables similar to:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      containers:
        - name: server
          env:
          - name: COLLECTOR_SERVICE_ADDR
            value: "signoz-otel-collector.signoz.svc.cluster.local:4317"
          - name: ENABLE_TRACING
            value: "1"
          - name: OTEL_SERVICE_NAME
            value: "frontend"
```

This configuration:
- Sets `COLLECTOR_SERVICE_ADDR` to point to SigNoz's OTel collector
- Enables tracing via `ENABLE_TRACING=1`
- Sets the service name for proper identification in SigNoz

### Alternative: Collecting Logs and Metrics on GKE Autopilot

Since `k8s-infra` doesn't work on GKE Autopilot, you have several options. **See [AUTOPILOT-OPTIONS.md](AUTOPILOT-OPTIONS.md) for detailed comparison and implementation guides.**

#### Quick Options Summary

1. **Use Standard GKE** (Recommended for full SigNoz experience)
   - ✅ k8s-infra works perfectly
   - ✅ Full logs, metrics, and traces in SigNoz
   - ⚠️ Requires migrating from Autopilot

2. **Export Cloud Logging to SigNoz** (Stay on Autopilot)
   - ✅ Keep Autopilot cluster
   - ✅ Get logs in SigNoz via Pub/Sub + Cloud Function
   - ⚠️ Metrics still need separate solution
   - 📖 See [AUTOPILOT-OPTIONS.md](AUTOPILOT-OPTIONS.md#option-2-export-cloud-logging-to-signoz-stay-on-autopilot)

3. **Use Grafana Cloud** (Best Autopilot compatibility)
   - ✅ Works perfectly on Autopilot
   - ✅ Free tier available
   - ✅ Full logs, metrics, and traces
   - 📖 See [AUTOPILOT-OPTIONS.md](AUTOPILOT-OPTIONS.md#option-3-use-grafana-cloud-best-for-autopilot)

4. **Hybrid: SigNoz Traces + Cloud Operations**
   - ✅ Traces in SigNoz
   - ✅ Logs/Metrics in Google Cloud Console
   - ✅ No additional setup needed

## Verifying the Integration

### Check Traces in SigNoz

1. Generate some traffic to your application:
   ```bash
   # If loadgenerator is running, it will generate traffic automatically
   # Or access the frontend in your browser
   ```

2. In SigNoz UI (`http://localhost:8080`):
   - Navigate to **Traces** section
   - Filter by service names: `frontend`, `checkoutservice`, `paymentservice`, etc.
   - You should see distributed traces showing request flow across services

### Check Metrics in SigNoz

**Note**: If you're on GKE Autopilot, metrics collection via k8s-infra won't work. Use Google Cloud Monitoring instead.

1. Navigate to **Dashboards** in SigNoz UI
2. Look for Kubernetes dashboards showing:
   - Node metrics (CPU, memory, disk)
   - Pod metrics (CPU, memory usage per pod)
   - Container metrics

**For GKE Autopilot**: Check metrics in Google Cloud Monitoring:
```bash
# Open Cloud Monitoring in your browser
echo "https://console.cloud.google.com/monitoring"
```

### Check Logs in SigNoz

**Note**: If you're on GKE Autopilot, logs collection via k8s-infra won't work. Use Google Cloud Logging instead.

1. Navigate to **Logs** section in SigNoz UI
2. Filter logs by:
   - **Namespace**: `default` (or your Online Boutique namespace)
   - **Workload**: `frontend`, `loadgenerator`, `checkoutservice`, etc.
3. You should see logs from all Online Boutique microservices

**For GKE Autopilot**: Check logs in Google Cloud Logging:
```bash
# View logs for a specific pod
kubectl logs -n default deployment/frontend

# Or open Cloud Logging in browser
echo "https://console.cloud.google.com/logs/query"
```

## Storage Considerations

- **Storage Class**: Use `standard-rwo` for cost-effective storage or `premium-rwo` for better performance
- **Storage Size**: SigNoz typically requires 20-50GB for ClickHouse data, depending on retention period
- **Retention**: Configure retention policies in SigNoz UI to manage storage costs

## Troubleshooting

### SigNoz pods not starting

```bash
# Check pod status
kubectl get pods -n signoz

# Check pod logs
kubectl logs -n signoz <pod-name>

# Verify storage class exists
kubectl get storageclass standard-rwo
```

### No traces appearing in SigNoz

1. Verify services are patched correctly:
   ```bash
   kubectl get deployment frontend -o yaml | grep COLLECTOR_SERVICE_ADDR
   ```

2. Check if traces are being sent:
   ```bash
   kubectl logs -n signoz -l app=signoz-otel-collector | grep -i trace
   ```

3. Verify network connectivity:
   ```bash
   kubectl run -it --rm debug --image=busybox --restart=Never -- \
     nc -zv signoz-otel-collector.signoz.svc.cluster.local 4317
   ```

### k8s-infra not collecting logs/metrics

**If you're on GKE Autopilot**: This is expected - k8s-infra cannot run on Autopilot. Use Google Cloud Operations instead (see [Alternative: Collecting Logs and Metrics on GKE Autopilot](#alternative-collecting-logs-and-metrics-on-gke-autopilot)).

For standard GKE clusters:

1. Check k8s-infra pod logs:
   ```bash
   kubectl logs -n signoz -l app=k8s-infra
   ```

2. Verify the collector endpoint is correct:
   ```bash
   kubectl get configmap -n signoz k8s-infra -o yaml
   ```

3. Check if pods are running:
   ```bash
   kubectl get pods -n signoz | grep k8s-infra
   ```

## Uninstalling SigNoz

To remove SigNoz and k8s-infra:

```bash
# Remove k8s-infra (if installed)
helm uninstall -n signoz k8s-infra 2>/dev/null || echo "k8s-infra not installed or already removed"

# Clean up any remaining k8s-infra resources (if installation partially failed)
kubectl delete daemonset -n signoz k8s-infra-otel-agent 2>/dev/null || true
kubectl delete deployment -n signoz k8s-infra-otel-deployment 2>/dev/null || true
kubectl delete svc -n signoz k8s-infra-otel-agent k8s-infra-otel-deployment 2>/dev/null || true

# Remove SigNoz
helm uninstall -n signoz signoz

# Remove namespace (optional)
kubectl delete namespace signoz
```

**Note**: Removing the namespace will delete all SigNoz data. Back up important data before uninstalling.

## Additional Resources

- [SigNoz Documentation](https://signoz.io/docs/)
- [SigNoz Helm Chart](https://github.com/SigNoz/charts)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [k8s-infra Chart](https://github.com/SigNoz/charts/tree/main/charts/k8s-infra)

