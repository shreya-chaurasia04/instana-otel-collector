# Target Allocator Quick Start Guide

This guide will help you quickly set up the Target Allocator with consistent hashing strategy for the Instana Distribution of OpenTelemetry Collector.

In this chart, IDOT deploys both a DaemonSet and a StatefulSet collector. This guide uses the Target Allocator with the StatefulSet collector tier, while leaving the DaemonSet for node-local collection such as host and kubelet metrics.

## Prerequisites

- Kubernetes cluster (1.24+)
- Helm 3.9+
- kubectl configured to access your cluster
- Instana backend endpoint and agent key
- (Optional) Prometheus Operator CRDs installed for ServiceMonitor/PodMonitor support

## Quick Start (5 Minutes)

### Step 1: Install Prometheus Operator CRDs (if not already installed)

```bash
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_podmonitors.yaml
```

### Step 2: Create namespace

```bash
kubectl create namespace instana-collector
```

### Step 3: Install IDOT with Target Allocator

```bash
helm install instana-otel-collector \
  --repo https://instana.github.io/instana-otel-collector instana-otel-collector-chart \
  --namespace instana-collector \
  --set clusterName=my-cluster \
  --set instanaEndpoint=ingress-red-saas.instana.io:443 \
  --set instanaKey=YOUR_INSTANA_KEY \
  --set targetAllocator.enabled=true \
  --set targetAllocator.allocationStrategy=consistent-hashing \
  --set statefulset.replicaCount=3
```

### Step 4: Verify Installation

```bash
# Check all pods are running
kubectl get pods -n instana-collector

# Expected output:
# NAME                                    READY   STATUS    RESTARTS   AGE
# idot-daemonset-agent-xxxxx              1/1     Running   0          1m
# idot-statefulset-0                      1/1     Running   0          1m
# idot-statefulset-1                      1/1     Running   0          1m
# idot-statefulset-2                      1/1     Running   0          1m
# idot-targetallocator-xxxxxxxxxx-xxxxx   1/1     Running   0          1m

# Check target allocator logs
kubectl logs -l app.kubernetes.io/component=target-allocator -n instana-collector
```

### Step 5: Create a Test ServiceMonitor

Create a file `test-servicemonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubernetes-apiservers
  namespace: default
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    interval: 30s
    port: https
    scheme: https
    tlsConfig:
      caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      serverName: kubernetes
  jobLabel: component
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      component: apiserver
      provider: kubernetes
```

Apply it:

```bash
kubectl apply -f test-servicemonitor.yaml
```

### Step 6: Verify Target Allocation

```bash
# Port-forward to target allocator
kubectl port-forward svc/idot-targetallocator 8080:80 -n instana-collector

# In another terminal, check allocated targets
curl http://localhost:8080/jobs

# Check targets for a specific collector
curl http://localhost:8080/jobs/kubernetes-apiservers/targets
```

## Configuration Options

### Using a Custom Values File

Create `my-values.yaml`:

```yaml
clusterName: production-cluster
instanaEndpoint: ingress-red-saas.instana.io:443
instanaKey: YOUR_INSTANA_KEY

targetAllocator:
  enabled: true
  allocationStrategy: consistent-hashing
  replicas: 1
  prometheusCR:
    enabled: true
    serviceMonitorSelector:
      matchLabels:
        monitoring: enabled

statefulset:
  replicaCount: 3
  config:
    receivers:
      prometheus:
        config:
          global:
            scrape_interval: 30s
        target_allocator:
          endpoint: http://idot-targetallocator:80
          interval: 30s
          collector_id: ${POD_NAME}
```

Install with custom values:

```bash
helm install instana-otel-collector \
  --repo https://instana.github.io/instana-otel-collector instana-otel-collector-chart \
  --namespace instana-collector \
  -f my-values.yaml
```

## Scaling Collectors

To handle more scrape targets, scale the StatefulSet used for Prometheus scraping:

```bash
# Scale to 5 replicas
kubectl scale statefulset idot-statefulset --replicas=5 -n instana-collector

# Verify scaling
kubectl get pods -n instana-collector -l app.kubernetes.io/component=opentelemetry-collector

# Check target redistribution
kubectl port-forward svc/idot-targetallocator 8080:80 -n instana-collector
curl http://localhost:8080/jobs
```

The consistent hashing strategy ensures minimal target reassignment during scaling.

## Monitoring Target Allocator

### Check Target Allocator Health

```bash
# Health check endpoint
curl http://localhost:8080/health

# Metrics endpoint (if enabled)
curl http://localhost:8080/metrics
```

### View Target Allocator Logs

```bash
kubectl logs -f -l app.kubernetes.io/component=target-allocator -n instana-collector
```

### Common Log Messages

- `"Successfully loaded targets"` - Target discovery working
- `"Allocating targets to collectors"` - Distribution in progress
- `"Collector registered"` - New collector joined
- `"Collector unregistered"` - Collector removed

## Troubleshooting

### No Targets Discovered

**Check ServiceMonitors exist:**
```bash
kubectl get servicemonitors -A
```

**Check target allocator can access ServiceMonitors:**
```bash
kubectl auth can-i list servicemonitors \
  --as=system:serviceaccount:instana-collector:idot-targetallocator
```

**Check selector configuration:**
```bash
kubectl get servicemonitors -A --show-labels
# Ensure labels match your serviceMonitorSelector
```

### Targets Not Being Scraped

**Check collector logs:**
```bash
kubectl logs -l app.kubernetes.io/component=opentelemetry-collector -n instana-collector | grep prometheus
```

**Verify target allocator endpoint:**
```bash
kubectl get svc idot-targetallocator -n instana-collector
```

**Check collector can reach target allocator:**
```bash
kubectl exec -it idot-statefulset-0 -n instana-collector -- \
  wget -O- http://idot-targetallocator:80/jobs
```

### High Memory Usage

**Check number of targets:**
```bash
curl http://localhost:8080/jobs | jq '.[] | length'
```

**Increase resources:**
```yaml
targetAllocator:
  resources:
    limits:
      memory: 512Mi
    requests:
      memory: 256Mi
```

## Next Steps

1. **Add more ServiceMonitors** - Create ServiceMonitors for your applications
2. **Configure filtering** - Use label selectors to control which ServiceMonitors to use
3. **Scale collectors** - Add more replicas as your scrape targets grow
4. **Monitor in Instana** - View collected metrics in Instana dashboard
5. **Read full documentation** - See [docs/target-allocator.md](../docs/target-allocator.md)

## Example ServiceMonitors

### Scrape Application Metrics

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: default
  labels:
    monitoring: enabled
spec:
  selector:
    matchLabels:
      app: my-application
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Scrape Node Exporter

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    monitoring: enabled
spec:
  selector:
    matchLabels:
      app: node-exporter
  endpoints:
  - port: metrics
    interval: 30s
```

### Scrape Multiple Endpoints

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: multi-endpoint-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
  - port: admin
    path: /admin/metrics
    interval: 60s
```

## Cleanup

To remove the installation:

```bash
# Uninstall Helm release
helm uninstall instana-otel-collector -n instana-collector

# Delete namespace
kubectl delete namespace instana-collector

# Remove ServiceMonitors (if desired)
kubectl delete servicemonitors --all -A
```

## Support

- Full Documentation: [docs/target-allocator.md](../docs/target-allocator.md)

