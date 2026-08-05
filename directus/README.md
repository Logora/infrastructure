# Directus Kubernetes Deployment

This folder contains everything needed to deploy Directus on Kubernetes using Helm.

## Prerequisites
- Kubernetes cluster
- Helm installed
- A PostgreSQL database accessible from the cluster

## Configuration

Directus configuration is provided entirely through environment variables mounted from a Kubernetes Secret.

1. Create the Secret from an env file:
   ```bash
   kubectl create secret generic directus-env --from-env-file=.env --namespace directus
   ```
   (Replace `.env` with your actual env file and `directus` with your namespace.)
2. Edit `values.yaml` (and/or an environment file such as `values.staging.yaml`) to match your domain, resources, etc.

### Database configuration (important)

Use **individual `DB_*` variables**, not `DB_CONNECTION_STRING`:

- Directus builds the connection as `connectionString || connectionConfig`.
- So when `DB_CONNECTION_STRING` is set, **`DB_SSL` is ignored** — TLS certificate verification cannot be disabled.
- This breaks connections to OVH-managed PostgreSQL (self-signed CA): you get
  `SELF_SIGNED_CERT_IN_CHAIN`. Use `DB_HOST`/`DB_PORT`/`DB_USER`/`DB_PASSWORD`/`DB_DATABASE` + `DB_SSL={"rejectUnauthorized":false}` instead.

#### Schema (`DB_SEARCH_PATH`)

- `DB_SEARCH_PATH` selects the PostgreSQL schema Directus operates in (Knex top-level `searchPath`, independent of the connection).
- Point it at the schema that contains the collections you want to manage (e.g. `public`, or your application's schema).
- Directus only sees tables in that schema, and existing tables become collections in **Settings → Data Model**.

See [`.env.example`](./.env.example) for the complete list of variables.

## Deployment Steps

Base chart values are in `values.yaml` so the chart is usable on its own. Environment-specific
overrides live in `values.staging.yaml` and `values.prod.yaml`.

1. Install the Helm chart:
   ```bash
   helm install directus ./ --namespace directus --values values.yaml
   ```
2. Upgrade with new values:
   ```bash
   helm upgrade directus ./ --namespace directus --values values.yaml
   ```
3. Check the deployment:
   ```bash
   kubectl get pods -n directus
   kubectl get svc -n directus
   kubectl get ingress -n directus
   ```
4. Access Directus at `https://directus.example.com` (or your configured domain).

## Environment-Specific Deployments

### Staging (OpenModeration)
```bash
kubectl create namespace open-moderation-staging
kubectl create secret generic directus-env --from-env-file=directus/.env.staging -n open-moderation-staging
helm install directus ./directus -n open-moderation-staging -f values.staging.yaml
```
Access at: `https://directus-staging.openmoderation.com`

### Production (OpenModeration)
```bash
kubectl create namespace open-moderation-prod
kubectl create secret generic directus-env --from-env-file=directus/.env.prod -n open-moderation-prod
helm install directus ./directus -n open-moderation-prod -f values.prod.yaml
```
Access at: `https://directus.openmoderation.com`

> One Directus instance per namespace, so the release and resource names (`directus`,
> `directus-env`) stay identical across environments. If you ever run two instances in the
> *same* namespace, override `envFromSecret` and the resource names.

## Environment Variables via Secret

Environment variables are stored in a Kubernetes Secret (`directus-env` by default, configurable
via the `envFromSecret` value). The deployment references it using `envFrom`:
```yaml
envFrom:
  - secretRef:
      name: directus-env
```

## Notes
- TLS/SSL is configured via cert-manager and Let's Encrypt in the provided ingress.
- For advanced configuration, see the [Directus documentation](https://docs.directus.io/self-hosted/kubernetes/).
