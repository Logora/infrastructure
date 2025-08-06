# Budibase on Kubernetes

This guide explains how to deploy Budibase on Kubernetes, following the official documentation: [Budibase Kubernetes Guide](https://docs.budibase.com/docs/kubernetes-k8s).

## Prerequisites

- A running Kubernetes cluster (local or cloud)
- `kubectl` installed and configured
- (Optional) Helm installed for easier management

## Steps


### 1. Add Budibase Helm Repository

Add the official Budibase Helm chart repository:

```bash
helm repo add budibase https://budibase.github.io/budibase/
helm repo update
```

### 2. Configure Values

Use the `values.yaml` file to set your configuration and environment variables. Refer to the [Budibase Helm chart documentation](https://github.com/Budibase/budibase/tree/master/charts/budibase) for available options.

### 3. Install Budibase with Helm

Install Budibase using the Helm chart:

```bash
helm install --create-namespace --namespace budibase budibase budibase/budibase -f values.yaml
```

This will deploy all required Budibase services (server, worker, minio, couchdb, redis, proxy) automatically.

### 4. Verify Deployment

Check that all pods are running:

```bash
kubectl get pods -n budibase
```

Access Budibase via the exposed service or ingress URL.

## Update the Budibase Helm Chart

To update Budibase to the latest chart version, first update the Helm repository:

```bash
helm repo update
```

Then upgrade your deployment:

```bash
helm upgrade -n budibase budibase budibase/budibase -f values.yaml
```

This will apply any changes in your `values.yaml` and update Budibase to the latest version available in the chart repository.

## Uninstall

To remove the OpenTelemetry Collector:
```bash
helm uninstall -n budibase budibase
```

## Useful Links

- [Budibase Kubernetes Docs](https://docs.budibase.com/docs/kubernetes-k8s)
- [Budibase GitHub](https://github.com/Budibase/budibase)
