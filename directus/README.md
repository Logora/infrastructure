# Directus Kubernetes Deployment

This folder contains everything needed to deploy Directus on Kubernetes using Helm.

## Prerequisites
- Kubernetes cluster
- Helm installed
- A PostgreSQL database accessible from the cluster


## Configuration
1. Create a Kubernetes Secret from your env file:
   ```bash
   kubectl create secret generic directus-env --from-env-file=.env --namespace directus
   ```
   (Replace `.env` with your actual env file and `directus` with your namespace)
2. Edit `values.yaml` to match your environment (domain, resources, etc).

## Deployment Steps

1. Install the Helm chart:
   ```bash
   helm install directus ./ --namespace directus --values values.yaml
   ```
2. Upgrade the Helm chart with new values:
   ```bash
   helm upgrade directus ./ --namespace directus --values values.yaml
   ```
   (Use this command whenever you update `values.yaml` or want to apply new configuration changes.)

2. Check the deployment:
   ```bash
   kubectl get pods -n directus
   kubectl get svc -n directus
   kubectl get ingress -n directus
   ```

3. Access Directus at `https://directus.logora.fr` (or your configured domain).

## Environment Variables via Secret

+Your environment variables are stored in a Kubernetes Secret (`directus-env`). The deployment references this secret using `envFrom`:
```yaml
envFrom:
  - secretRef:
      name: directus-env
```

## Notes
- Make sure your database is reachable from the cluster.
- TLS/SSL is configured via cert-manager and Let's Encrypt in the provided ingress.
- For advanced configuration, see the [Directus documentation](https://docs.directus.io/self-hosted/kubernetes/).
