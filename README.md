# Logora Infrastructure

This repository contains Kubernetes deployment configurations for Logora's application infrastructure. Each application is deployed using either official Helm charts or custom Helm charts, following consistent patterns for ingress, TLS certificates, and resource management.

## Applications

### [Budibase](./budibase/)
Low-code platform for building internal tools and applications.
- **Deployment**: Official Helm chart
- **Domain**: budibase.logora.fr
- **Database**: CouchDB (included)
- **Storage**: MinIO (included)

### [Directus](./directus/)
Headless CMS for content management and API generation.
- **Deployment**: Custom Helm chart
- **Domain**: directus.logora.fr / directus-staging.logora.fr
- **Database**: PostgreSQL (external)
- **Features**: Multi-environment support (staging/prod)

### [Metabase](./metabase/)
Business intelligence and analytics platform.
- **Deployment**: Direct Kubernetes YAML
- **Domain**: data.logora.com
- **Database**: PostgreSQL (external)

### [OpenTelemetry](./opentelemetry/)
Observability and telemetry collection for monitoring and tracing.
- **Deployment**: Official Helm chart
- **Purpose**: Centralized telemetry collection
- **Integration**: Sends data to Uptrace platform

### [Tolgee](./tolgee/)
Developer-friendly localization platform for managing translations.
- **Deployment**: Custom Helm chart with included PostgreSQL database
- **Domain**: tolgee.logora.fr / tolgee-staging.logora.fr
- **Database**: PostgreSQL (included as Helm dependency)
- **Features**: Multi-environment support (staging/prod), persistent database storage

## Common Infrastructure Patterns

### Ingress and TLS
All applications use:
- **Ingress Controller**: nginx
- **TLS Certificates**: Automatic provisioning via cert-manager and Let's Encrypt
- **SSL Redirect**: Enforced HTTPS for all applications

### Resource Management
Consistent resource allocation patterns:
- **Memory Requests**: 256Mi (typical)
- **Memory Limits**: 512Mi (typical)
- **Storage Class**: `csi-cinder-high-speed` for persistent volumes

### Environment Management
- **Secrets**: Environment variables stored in Kubernetes secrets
- **Configuration**: `.env.example` files provide templates
- **Multi-environment**: Staging and production configurations where applicable

## Prerequisites

Before deploying any application, ensure you have:

1. **Kubernetes Cluster**: Running cluster with appropriate permissions
2. **Helm**: Version 3.x installed and configured
3. **kubectl**: Configured to connect to your cluster
4. **nginx-ingress-controller**: Installed in your cluster
5. **cert-manager**: Installed for automatic TLS certificate management
6. **Database Management**: PostgreSQL instances (external for Directus/Metabase, included for Tolgee)

## General Deployment Process

1. **Prepare Environment**:
   ```bash
   # Create namespace
   kubectl create namespace <app-name>
   
   # Create secrets from .env file
   kubectl create secret generic <app-name>-env --from-env-file=.env --namespace <app-name>
   ```

2. **Deploy Application**:
   ```bash
   # For Helm charts
   helm install <app-name> ./<app-name> --namespace <app-name> --values <app-name>/values.yaml
   
   # For direct YAML
   kubectl apply -f <app-name>/
   ```

3. **Verify Deployment**:
   ```bash
   kubectl get pods -n <app-name>
   kubectl get svc -n <app-name>
   kubectl get ingress -n <app-name>
   ```

## Domain Configuration

All applications follow the domain pattern:
- **Production**: `<app-name>.logora.fr`
- **Staging**: `<app-name>-staging.logora.fr`
- **Exception**: Metabase uses `data.logora.com`

## Monitoring and Observability

The OpenTelemetry collector is configured to:
- Collect metrics from all applications and infrastructure
- Aggregate logs from container workloads
- Forward telemetry data to Uptrace for analysis
- Provide comprehensive observability across the entire platform

## Security Considerations

- All sensitive configuration is stored in Kubernetes secrets
- TLS encryption is enforced for all external traffic
- Database connections use secure authentication
- Environment files (`.env`) are excluded from version control

## Support and Documentation

Each application folder contains:
- `README.md`: Detailed deployment instructions
- `.env.example`: Configuration template
- Helm charts or Kubernetes manifests
- Environment-specific configuration files

For application-specific issues, refer to the individual README files in each application directory.
