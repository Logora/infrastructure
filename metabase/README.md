# Metabase Deployment

This guide explains how to deploy Metabase on Kubernetes.

## Prerequisites

- Kubernetes cluster
- `kubectl` configured to connect to your cluster
- nginx-ingress-controller installed in your cluster
- cert-manager installed in your cluster (for SSL/TLS)
- A PostgreSQL database
- `.env` file with database connection URL (see `.env.example` for format)

## Files

- `.env`: Contains environment variables (gitignored)
- `.env.example`: Template for environment variables

- `deployment.yaml`: Defines the Metabase deployment with container settings, resources, and health checks
- `service.yaml`: Creates a ClusterIP service to expose Metabase internally
- `ingress.yaml`: Configures the Nginx ingress for accessing Metabase at data.logora.com with TLS

## Installation Steps

1. Create the namespace:
```bash
kubectl create namespace metabase
```

2. Copy the .env.example file to create your .env file

3. Create a secret from the .env file:
```bash
# Create the secret using the .env file
kubectl create secret generic metabase-db-secret \
  --namespace metabase \
  --from-env-file=.env
```

3. Deploy Metabase:

```bash
# Apply the deployment
kubectl apply -f deployment.yaml

# Apply the service
kubectl apply -f service.yaml

# Apply the ingress
kubectl apply -f ingress.yaml
```

## Configuration Details

### Database Configuration

The deployment uses a Kubernetes secret `metabase-db-secret` to store the database connection URL. Make sure to replace the placeholder in the secret creation command with your actual database URL.

### Resource Limits

The deployment is configured with the following resource limits:
- Memory: 2Gi (request) / 4Gi (limit)
- CPU: 1 (request) / 2 (limit)

### Ingress Configuration

The ingress is configured to:
- Use nginx-ingress-controller
- Automatically provision TLS certificates using cert-manager
- Access Metabase at https://data.logora.com

## Verification

Check if all resources are created:
```bash
# Check the namespace
kubectl get namespace metabase

# Check the deployment status
kubectl -n metabase get deployments

# Check if pods are running
kubectl -n metabase get pods

# Check the service
kubectl -n metabase get services

# Check the ingress
kubectl -n metabase get ingress

# View the Metabase logs
kubectl -n metabase logs -l app=metabase
```

## Troubleshooting

If you experience issues:

1. Check pod status:
```bash
kubectl -n metabase describe pod -l app=metabase
```

2. Check ingress status:
```bash
kubectl -n metabase describe ingress metabase
```

3. Check the logs:
```bash
kubectl -n metabase logs -l app=metabase --tail=100
```

## Uninstall

To completely remove Metabase:
```bash
# Delete all resources
kubectl delete -f ingress.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml

# Delete the secret
kubectl delete secret metabase-db-secret -n metabase

# Delete the namespace
kubectl delete namespace metabase
```
