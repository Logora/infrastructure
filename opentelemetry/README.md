# OpenTelemetry Collector Deployment

This guide explains how to deploy the OpenTelemetry Collector to collect Kubernetes metrics using Helm.

## Overview

This deployment consists of two separate OpenTelemetry Collector instances:

1. **Cluster Collector** (`values.yaml`) - Deployment mode
   - Runs as a single instance
   - Collects cluster-level metrics via k8s_cluster receiver
   - Provides inventory and state information (pod phases, deployments, node conditions, etc.)
   - Emits Kubernetes entity events as logs

2. **Node Collector** (`values-daemonset.yaml`) - DaemonSet mode
   - Runs one instance per node
   - Collects runtime metrics via kubeletstats receiver
   - Provides actual resource usage (CPU, memory, network, filesystem)
   - Per-pod, per-container, and per-node utilization data

## Prerequisites

- Kubernetes cluster
- Helm 3.x installed
- `kubectl` configured to connect to your cluster
- Uptrace account with DSN

## Installation Steps

1. Add the OpenTelemetry Helm repository:
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

2. Create the namespace:
```bash
kubectl create namespace monitoring
```

4. Create a secret from the .env file:
```bash
# Create the secret using the .env file
kubectl create secret generic otel-collector-secrets \
  --namespace monitoring \
  --from-env-file=.env
```

5. Apply the ClusterRole for RBAC permissions:
```bash
kubectl apply -f templates/otel-collector-clusterrole.yaml
```

6. Install the Cluster Collector (k8s_cluster receiver):
```bash
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  -f values.yaml
```

7. Install the Node Collector (kubeletstats receiver):
```bash
helm install otel-collector-daemonset open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  -f values-daemonset.yaml
```

## Verification

Check if both collectors are running:
```bash
# Check cluster collector (deployment)
kubectl -n monitoring get pods -l app.kubernetes.io/instance=otel-collector

# Check node collectors (daemonset)
kubectl -n monitoring get pods -l app.kubernetes.io/instance=otel-collector-daemonset
```

View the collector logs:
```bash
# Cluster collector logs
kubectl -n monitoring logs -l app.kubernetes.io/instance=otel-collector

# Node collector logs (pick any pod)
kubectl -n monitoring logs -l app.kubernetes.io/instance=otel-collector-daemonset --tail=100
```

## Uninstall

To remove the OpenTelemetry Collector:
```bash
helm uninstall otel-collector -n monitoring
```

## Upgrading

To upgrade the collector with new configuration:
```bash
helm upgrade otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  -f values.yaml
```

## Troubleshooting

If you see any issues with metrics collection, check:
1. The collector logs for any errors
2. That the serviceAccount has the correct permissions
3. That the NODE_IP environment variable is properly set
4. That the Uptrace DSN is correctly configured in the secret
