# OpenTelemetry Collector Deployment

This guide explains how to deploy the OpenTelemetry Collector to collect Kubernetes metrics using Helm.

## Prerequisites

- Kubernetes cluster
- Helm 3.x installed
- `kubectl` configured to connect to your cluster

## Installation Steps

1. Add the OpenTelemetry Helm repository:
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
```

2. Use the provided `values.yaml` file in this directory which contains the OpenTelemetry Collector configuration with:
   - Kubelet stats receiver setup
   - Debug and OTLP exporters configuration
   - Resource limits and requests
   - RBAC settings
   - Required environment variables

3. Create the namespace:
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

5. Install the OpenTelemetry Collector:
```bash
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  -f values.yaml
```

## Verification

Check if the collector is running:
```bash
kubectl -n monitoring get pods -l app.kubernetes.io/name=opentelemetry-collector
```

View the collector logs:
```bash
kubectl -n monitoring logs -l app.kubernetes.io/name=opentelemetry-collector
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

## Configuration

The `values.yaml` file contains:
- Kubelet stats receiver configuration
- Debug and OTLP exporters setup
- Resource limits and requests
- RBAC configuration
- Environment variables for node access
- Pipeline configuration for metrics collection

Modify the `values.yaml` file to adjust any of these settings according to your needs.
