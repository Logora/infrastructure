# Tolgee Kubernetes Deployment

This folder contains everything needed to deploy Tolgee on Kubernetes using Helm.

## Prerequisites
- Kubernetes cluster
- Helm installed
- Storage class `csi-cinder-high-speed` available in your cluster (for PostgreSQL persistent volumes)

## About Tolgee

Tolgee is a developer-friendly localization platform that helps manage translations for web applications. Since no official Helm chart is available, this custom chart provides a production-ready deployment following Kubernetes best practices.

## Configuration
1. Create a Kubernetes Secret from your env file:
   ```bash
   kubectl create secret generic tolgee-env --from-env-file=.env --namespace tolgee
   ```
   (Replace `.env` with your actual env file and `tolgee` with your namespace)
2. Edit `values.yaml` to match your environment (domain, resources, etc).

## Deployment Steps

1. Install the Helm chart:
   ```bash
   helm install tolgee ./ --namespace tolgee --values values.yaml
   ```
2. Upgrade the Helm chart with new values:
   ```bash
   helm upgrade tolgee ./ --namespace tolgee --values values.yaml
   ```
   (Use this command whenever you update `values.yaml` or want to apply new configuration changes.)

3. Check the deployment:
   ```bash
   kubectl get pods -n tolgee
   kubectl get svc -n tolgee
   kubectl get ingress -n tolgee
   ```

4. Access Tolgee at `https://tolgee.logora.fr` (or your configured domain).

## Environment Variables via Secret

Your environment variables are stored in a Kubernetes Secret (`tolgee-env`). The deployment references this secret using `envFrom`:
```yaml
envFrom:
  - secretRef:
      name: tolgee-env
```

## Required Environment Variables

The following environment variables must be configured in your `.env` file:

### Database Configuration (Included)
This deployment includes a PostgreSQL database as a dependency. The database configuration is automatically handled:
- `TOLGEE_POSTGRES_HOST`: Set to `tolgee-postgresql` (internal service)
- `TOLGEE_POSTGRES_PORT`: Set to `5432`
- `TOLGEE_POSTGRES_USER`: Set to `tolgee`
- `TOLGEE_POSTGRES_PASSWORD`: Environment-specific password
- `TOLGEE_POSTGRES_DATABASE`: Set to `tolgee`

To use an external database instead, set `postgresql.enabled: false` in your values file.

### Authentication (Required)
- `TOLGEE_JWT_SECRET`: JWT secret key for authentication (generate a random string)
- `TOLGEE_INITIAL_USERNAME`: Initial admin username
- `TOLGEE_INITIAL_PASSWORD`: Initial admin password

### SMTP Configuration (Optional)
- `TOLGEE_SMTP_HOST`: SMTP server host
- `TOLGEE_SMTP_PORT`: SMTP server port
- `TOLGEE_SMTP_USERNAME`: SMTP username
- `TOLGEE_SMTP_PASSWORD`: SMTP password
- `TOLGEE_SMTP_FROM`: From email address

### Machine Translation APIs (Optional)
- `TOLGEE_GOOGLE_API_KEY`: Google Translate API key
- `TOLGEE_AWS_ACCESS_KEY`: AWS Translate access key
- `TOLGEE_AWS_SECRET_KEY`: AWS Translate secret key
- `TOLGEE_AZURE_API_KEY`: Azure Translator API key

## Environment-Specific Deployments

### Staging
```bash
helm install tolgee ./ --namespace tolgee-staging --values values.staging.yaml
```
Access at: `https://tolgee-staging.logora.fr`

### Production
```bash
helm install tolgee ./ --namespace tolgee-prod --values values.prod.yaml
```
Access at: `https://tolgee.logora.fr`

## Notes
- PostgreSQL database is included as a dependency with persistent storage using `csi-cinder-high-speed` storage class.
- Database data is persisted with an 8Gi volume that survives pod restarts.
- TLS/SSL is configured via cert-manager and Let's Encrypt in the provided ingress.
- The deployment includes health checks for both liveness and readiness probes.
- Resource limits are configured to use 256Mi request / 512Mi limit for memory for both Tolgee and PostgreSQL.
- To disable the included PostgreSQL and use an external database, set `postgresql.enabled: false` in your values file.
- For advanced configuration, see the [Tolgee documentation](https://tolgee.io/platform/self_hosting/configuration).

## Troubleshooting

If you experience issues:

1. Check pod status:
```bash
kubectl -n tolgee describe pod -l app.kubernetes.io/name=tolgee
```

2. Check ingress status:
```bash
kubectl -n tolgee describe ingress tolgee
```

3. Check the logs:
```bash
kubectl -n tolgee logs -l app.kubernetes.io/name=tolgee --tail=100
```

## Uninstall

To completely remove Tolgee:
```bash
# Delete the Helm release
helm uninstall tolgee -n tolgee

# Delete the secret
kubectl delete secret tolgee-env -n tolgee

# Delete the namespace
kubectl delete namespace tolgee
```
