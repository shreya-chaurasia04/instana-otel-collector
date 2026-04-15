# Target Allocator with Consistent Hashing Strategy

## Overview

The Target Allocator is a component that enables horizontal scaling of Prometheus metric collection in OpenTelemetry Collector deployments. It distributes Prometheus scrape targets across multiple collector instances using a consistent hashing strategy, ensuring:

- **Even distribution** of scrape targets across collectors
- **Minimal reassignment** when collectors scale up or down
- **High availability** for metric collection
- **Efficient resource utilization**

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                              │
│  ┌──────────────────┐         ┌─────────────────────┐      │
│  │ ServiceMonitors  │────────▶│  Target Allocator   │      │
│  │  & PodMonitors   │         │ (Consistent Hashing)│      │
│  └──────────────────┘         └──────────┬──────────┘      │
│                                           │                  │
│                          ┌────────────────┼────────────┐    │
│                          │                │            │    │
│                          ▼                ▼            ▼    │
│                   ┌──────────┐    ┌──────────┐  ┌──────────┐
│                   │Collector │    │Collector │  │Collector │
│                   │ Pod 1    │    │ Pod 2    │  │ Pod 3    │
│                   └────┬─────┘    └────┬─────┘  └────┬─────┘
│                        │               │             │       │
│                        └───────────────┴─────────────┘       │
│                                    │                          │
│                                    ▼                          │
│                            ┌──────────────┐                  │
│                            │   Instana    │                  │
│                            │   Backend    │                  │
│                            └──────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## How Consistent Hashing Works

Consistent hashing ensures that:
1. Each scrape target is assigned to exactly one collector instance
2. When a collector is added or removed, only a minimal number of targets are reassigned
3. Targets are evenly distributed across all available collectors

### Benefits over other strategies:
- **least-weighted**: Distributes based on load, but can cause more reassignments
- **per-node**: Ties targets to specific nodes, less flexible for scaling
- **consistent-hashing**: Optimal for dynamic scaling with minimal disruption

## Configuration

In this IDOT chart, both a DaemonSet and a StatefulSet collector are deployed. For target allocation, the recommended pattern is:
- use the DaemonSet for node-local collection such as `hostmetrics` and `kubeletstats`
- use the StatefulSet as the centralized scalable Prometheus scraping tier with target allocator

### Step 1: Enable Target Allocator in values.yaml

```yaml
targetAllocator:
  enabled: true  # Enable target allocator
  image:
    repository: ghcr.io/open-telemetry/opentelemetry-operator/target-allocator
    tag: 0.95.0
  replicas: 1
  # Use consistent-hashing strategy for optimal distribution
  allocationStrategy: consistent-hashing
  service:
    type: ClusterIP
    port: 80
  # Enable Prometheus CR-based service discovery
  prometheusCR:
    enabled: true
    # Select which ServiceMonitors to use (empty = all)
    serviceMonitorSelector: {}
    # Select which PodMonitors to use (empty = all)
    podMonitorSelector: {}
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
```

### Step 2: Configure Prometheus Receiver in the StatefulSet

Enable the Prometheus receiver on the `statefulset` collector:

```yaml
statefulset:
  replicaCount: 3
  config:
    receivers:
      prometheus:
        config:
          global:
            scrape_interval: 30s
            scrape_timeout: 10s
        # Connect to target allocator
        target_allocator:
          endpoint: http://idot-targetallocator:80
          interval: 30s
          collector_id: ${POD_NAME}
```

### Step 3: Add Prometheus Receiver to the StatefulSet Metrics Pipeline

```yaml
statefulset:
  config:
    service:
      pipelines:
        metrics:
          receivers:
          - otlp
          - k8s_cluster
          - prometheus
          exporters:
          - otlp/instana
          processors:
          - resourcedetection/env
          - k8sattributes
          - memory_limiter
          - batch
```

### Step 4: Scale Collectors for Load Distribution

To benefit from target allocation, scale your collector deployment:

```bash
# For StatefulSet collectors
kubectl scale statefulset idot-statefulset --replicas=3 -n instana-collector

# For DaemonSet, collectors automatically scale with nodes
```

## RBAC Configuration

The target allocator requires permissions to discover ServiceMonitors and PodMonitors:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: idot-targetallocator
rules:
  # Required for ServiceMonitor discovery
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["servicemonitors", "podmonitors"]
    verbs: ["get", "list", "watch"]
  # Required for service discovery
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  # Required for node discovery
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

## Example: Scraping Application Metrics

### 1. Create a ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-metrics
  namespace: default
spec:
  selector:
    matchLabels:
      app: my-application
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### 2. Deploy Your Application with Metrics Endpoint

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-application
  labels:
    app: my-application
spec:
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
  selector:
    app: my-application
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-application
  template:
    metadata:
      labels:
        app: my-application
    spec:
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080
          name: metrics
```

### 3. Verify Target Allocation

Check the target allocator to see how targets are distributed:

```bash
# Port-forward to target allocator
kubectl port-forward svc/idot-targetallocator 8080:80 -n instana-collector

# View allocated targets
curl http://localhost:8080/jobs

# View targets for a specific collector
curl http://localhost:8080/jobs/<job-name>/targets?collector_id=<pod-name>
```

## Installation with Helm

### Basic Installation with Target Allocator

```bash
helm install instana-otel-collector \
  --repo https://instana.github.io/instana-otel-collector instana-otel-collector-chart \
  --namespace instana-collector \
  --create-namespace \
  --set clusterName=my-cluster \
  --set instanaEndpoint=ingress-red-saas.instana.io:443 \
  --set instanaKey=<YOUR_INSTANA_KEY> \
  --set targetAllocator.enabled=true \
  --set targetAllocator.allocationStrategy=consistent-hashing
```

### Installation with Custom Values File

Create a `custom-values.yaml`:

```yaml
clusterName: my-cluster
instanaEndpoint: ingress-red-saas.instana.io:443
instanaKey: <YOUR_INSTANA_KEY>

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
    service:
      pipelines:
        metrics:
          receivers:
          - otlp
          - k8s_cluster
          - prometheus
```

Install with custom values:

```bash
helm install instana-otel-collector \
  --repo https://instana.github.io/instana-otel-collector instana-otel-collector-chart \
  --namespace instana-collector \
  --create-namespace \
  -f custom-values.yaml
```

## Monitoring and Troubleshooting

### Check Target Allocator Status

```bash
# View target allocator logs
kubectl logs -l app.kubernetes.io/component=target-allocator -n instana-collector

# Check target allocator service
kubectl get svc idot-targetallocator -n instana-collector

# Describe target allocator pod
kubectl describe pod -l app.kubernetes.io/component=target-allocator -n instana-collector
```

### Verify Prometheus Receiver Configuration

```bash
# Check collector logs for Prometheus receiver
kubectl logs -l app.kubernetes.io/component=opentelemetry-collector -n instana-collector | grep prometheus

# Check if targets are being scraped
kubectl logs -l app.kubernetes.io/component=opentelemetry-collector -n instana-collector | grep "scrape"
```

### Common Issues

#### 1. No Targets Discovered

**Symptom**: Target allocator shows no targets

**Solution**:
- Verify ServiceMonitors/PodMonitors exist: `kubectl get servicemonitors -A`
- Check selector configuration in `prometheusCR.serviceMonitorSelector`
- Verify RBAC permissions for target allocator

#### 2. Targets Not Distributed

**Symptom**: All targets assigned to one collector

**Solution**:
- Ensure multiple collector replicas are running
- Verify `collector_id` is set correctly (should be `${POD_NAME}`)
- Check target allocator logs for allocation errors

#### 3. High Memory Usage

**Symptom**: Target allocator consuming excessive memory

**Solution**:
- Reduce number of ServiceMonitors/PodMonitors
- Increase resource limits
- Consider filtering targets with more specific selectors

## Performance Considerations

### Scaling Guidelines

| Scrape Targets | Recommended Collectors | Target Allocator Replicas |
|----------------|------------------------|---------------------------|
| < 100          | 1-2                    | 1                         |
| 100-500        | 2-5                    | 1                         |
| 500-1000       | 5-10                   | 1-2                       |
| > 1000         | 10+                    | 2-3                       |

### Resource Requirements

**Target Allocator**:
- CPU: ~50m per 1000 targets
- Memory: ~100Mi per 1000 targets

**Collector (with Prometheus receiver)**:
- CPU: ~100m per 100 targets
- Memory: ~200Mi per 100 targets

## Advanced Configuration

### Custom Allocation Strategy

While consistent-hashing is recommended, you can use other strategies:

```yaml
targetAllocator:
  allocationStrategy: least-weighted  # Alternative strategy
```

Available strategies:
- `consistent-hashing`: Best for dynamic scaling (recommended)
- `least-weighted`: Distributes based on current load
- `per-node`: Assigns targets based on node affinity

### Filtering Targets

Use label selectors to control which ServiceMonitors are used:

```yaml
targetAllocator:
  prometheusCR:
    serviceMonitorSelector:
      matchLabels:
        team: platform
        environment: production
    podMonitorSelector:
      matchExpressions:
      - key: monitoring
        operator: In
        values: [enabled, required]
```

### High Availability Setup

For production environments, run multiple target allocator replicas:

```yaml
targetAllocator:
  enabled: true
  replicas: 3  # Multiple replicas for HA
  allocationStrategy: consistent-hashing
```

## Migration Guide

### From Prometheus Operator

If migrating from Prometheus Operator:

1. Keep existing ServiceMonitors/PodMonitors
2. Enable target allocator in IDOT
3. Scale IDOT collectors to match Prometheus instances
4. Gradually reduce Prometheus Operator scrape load
5. Monitor metrics continuity in Instana

### From Static Prometheus Configuration

1. Convert static scrape configs to ServiceMonitors
2. Deploy ServiceMonitors to cluster
3. Enable target allocator
4. Verify metrics collection
5. Remove static configurations

## References

- [OpenTelemetry Collector Documentation](https://opentelemetry.io/docs/collector/)
- [Target Allocator Design](https://github.com/open-telemetry/opentelemetry-operator/blob/main/cmd/otel-allocator/README.md)
- [Prometheus Receiver Documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/prometheusreceiver)
- [Instana OpenTelemetry Documentation](https://www.ibm.com/docs/en/instana-observability/current?topic=apis-opentelemetry)

