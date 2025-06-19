# ArgoCD Templates Usage Guide

This guide explains how to use the ArgoCD templates for application and project management instead of static YAML files.

## Overview

The template-based approach provides:
- **Reusability**: Generic templates for multiple applications and environments
- **Consistency**: Standardized configurations across deployments
- **Maintainability**: Single source of truth for configuration patterns
- **Flexibility**: Easy customization through values files

## Template Structure

```
Kubernetes with ArgoCD/
├── templates/
│   ├── appproject-template.yaml      # Generic AppProject template
│   └── application-template.yaml     # Generic Application template
├── values/
│   ├── appproject-values.yaml        # AppProject configuration
│   ├── applicationset-values.yaml    # ApplicationSet configuration
│   └── django-app-values.yaml        # Django app configuration
├── applicationset-with-templates.yaml # ApplicationSet using templates
└── [static YAML files]               # Legacy individual manifests
```

## Template Components

### 1. AppProject Template

**File**: `templates/appproject-template.yaml`

**Purpose**: Defines project-level policies, RBAC, and resource permissions.

**Key Features**:
- Source repository restrictions
- Destination cluster/namespace permissions
- RBAC roles and policies
- Sync windows for controlled deployments
- Resource whitelisting

### 2. Application Template

**File**: `templates/application-template.yaml`

**Purpose**: Generic template for ArgoCD Applications supporting both Helm charts and Git repositories.

**Key Features**:
- Helm chart support with values, parameters, and value files
- Git repository support with Kustomize
- Flexible sync policies
- Ignore differences configuration
- Notification settings

### 3. ApplicationSet Values File

**File**: `values/applicationset-values.yaml`

**Purpose**: Centralized configuration for ApplicationSet template generation.

**Key Features**:
- Project-level settings
- Repository configuration
- Global domain and image registry settings
- Environment-specific overrides (dev, staging, prod)
- Database configuration
- Image tags per environment
- Sync wave orchestration
- Automated sync policies

**Structure**:
```yaml
project:
  name: "multi-env-project"
  
repository:
  url: "https://github.com/your-username/your-repo.git"
  
global:
  domain: "example.com"
  imageRegistry: "varunsimha"
  
environments:
  dev:
    domain: "dev.example.com"
    replicas:
      django: 1
      postgres: 1
    resources:
      django:
        cpu: "200m"
        memory: "256Mi"
```

### 4. Other Values Files

**Purpose**: Component-specific configurations.

**Types**:
- `appproject-values.yaml`: Project-level settings
- `django-app-values.yaml`: Django application configuration

## Usage Methods

### Method 1: Using ApplicationSet with Values (Recommended)

**Files**: 
- `applicationset-with-templates.yaml`
- `values/applicationset-values.yaml`

This method uses the ApplicationSet with centralized values configuration:

```bash
# 1. Update repository URL in values file
sed -i 's|https://github.com/your-username/your-repo.git|https://github.com/YOUR_ORG/YOUR_REPO.git|g' values/applicationset-values.yaml

# 2. Apply the ApplicationSet
kubectl apply -f applicationset-with-templates.yaml

# 3. Verify deployment
kubectl get applicationset -n argocd
kubectl get applications -n argocd
```

**Benefits**:
- Centralized configuration management via values file
- Environment-specific resource allocation
- Automated multi-environment deployment
- Sync wave orchestration (database first, then application)
- Consistent deployment patterns across environments

### Method 2: Using Templates with Helm

```bash
# Create AppProject using template
helm template appproject ./templates/appproject-template.yaml \
  --values ./values/appproject-values.yaml | kubectl apply -f -

# Create Django application using template
helm template django-app ./templates/application-template.yaml \
  --values ./values/django-app-values.yaml | kubectl apply -f -

# Note: PostgreSQL configuration is now handled via ApplicationSet values
```

### Method 3: Direct Template Substitution

```bash
# Using envsubst or similar tools
export PROJECT_NAME="my-project"
export ENVIRONMENT="dev"
envsubst < templates/application-template.yaml | kubectl apply -f -
```

## Quick Start Configuration

### 1. Update Repository Settings
```bash
# Edit values/applicationset-values.yaml
vim values/applicationset-values.yaml

# Update these key values:
# - repository.url: Your Git repository URL
# - global.domain: Your base domain
# - global.imageRegistry: Your container registry
# - image.django.repository: Your Django app image
```

### 2. Environment Customization
```yaml
# Add new environments in applicationset-values.yaml
environments:
  staging:
    domain: "staging.example.com"
    replicas:
      django: 2
      postgres: 1
    resources:
      django:
        cpu: "500m"
        memory: "512Mi"
```

### 3. Deploy
```bash
kubectl apply -f applicationset-with-templates.yaml
```

## Configuration Examples

### Creating a New Environment

1. **Add to ApplicationSet**:
```yaml
# In applicationset-with-templates.yaml
- name: django-app-test
  chart: django-app
  environment: test
  appType: web
  syncWave: "2"
  replicaCount: "1"
  imageTag: test
  host: test.example.com
  # ... other parameters
```

2. **Create Values File**:
```yaml
# values/django-app-test-values.yaml
application:
  name: "django-app-test"
  environment: "test"
  # ... configuration
```

### Adding a New Application Type

1. **Create Helm Chart**:
```bash
mkdir -p charts/redis
# Create Chart.yaml, values.yaml, templates/
```

2. **Add to ApplicationSet**:
```yaml
- name: redis-dev
  chart: redis
  environment: dev
  appType: cache
  syncWave: "1"
  # ... parameters
```

3. **Update Template Values**:
```yaml
{{- else if eq .chart "redis" }}
# Redis-specific configuration
replicaCount: {{replicaCount}}
image:
  repository: redis
  tag: "6.2"
# ... redis config
{{- end }}
```

## Migration from Static Files

### Step 1: Backup Current Configuration
```bash
cp -r "Kubernetes with ArgoCD" "Kubernetes with ArgoCD.backup"
```

### Step 2: Update Repository URL
```bash
# Replace <your-repo-url> in templates and ApplicationSet
sed -i 's|<your-repo-url>|https://github.com/your-org/your-repo.git|g' \
  applicationset-with-templates.yaml values/*.yaml
```

### Step 3: Apply Templates
```bash
# Remove old ApplicationSet (optional)
kubectl delete applicationset django-app-set -n argocd

# Apply new template-based ApplicationSet
kubectl apply -f applicationset-with-templates.yaml
```

### Step 4: Verify Deployment
```bash
# Check ApplicationSet
kubectl get applicationset -n argocd

# Check generated applications
kubectl get applications -n argocd

# Check application status
kubectl describe application django-app-dev -n argocd
```

## Best Practices

### 1. Template Organization
- Keep templates generic and reusable
- Use values files for environment-specific configurations
- Maintain clear naming conventions

### 2. Values Management
- Separate values by environment and application type
- Use hierarchical values (global → environment → application)
- Version control all values files

### 3. Security
- Use AppProject to restrict permissions
- Implement RBAC policies
- Use sync windows for production deployments
- Store secrets separately (not in values files)

### 4. Monitoring
- Set up notifications for sync failures
- Monitor application health
- Use ArgoCD UI for visualization

## Troubleshooting

### Common Issues

1. **Template Rendering Errors**:
```bash
# Test template rendering
helm template test ./templates/application-template.yaml \
  --values ./values/django-app-values.yaml --debug
```

2. **ApplicationSet Not Generating Apps**:
```bash
# Check ApplicationSet status
kubectl describe applicationset multi-env-project -n argocd

# Check ApplicationSet controller logs
kubectl logs -n argocd deployment/argocd-applicationset-controller
```

3. **Values File Issues**:
```bash
# Validate values file syntax
yamllint values/applicationset-values.yaml

# Check ApplicationSet template rendering
kubectl get applicationset multi-env-project -n argocd -o yaml
```

4. **Sync Failures**:
```bash
# Check application events
kubectl describe application django-app-dev -n argocd

# Force refresh
kubectl patch application django-app-dev -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

### Validation Commands

```bash
# Validate YAML syntax
yamllint templates/*.yaml values/*.yaml

# Validate Kubernetes resources
kubectl apply --dry-run=client -f applicationset-with-templates.yaml

# Test Helm template rendering
helm template test templates/application-template.yaml \
  --values values/django-app-values.yaml
```

## Advanced Features

### 1. Multi-Source Applications
```yaml
sources:
  - repoURL: https://github.com/your-org/helm-charts
    chart: django-app
    targetRevision: 1.0.0
  - repoURL: https://github.com/your-org/app-config
    path: environments/dev
    targetRevision: HEAD
```

### 2. Progressive Sync
```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true
  managedNamespaceMetadata:
    labels:
      environment: dev
      managed-by: argocd
```

### 3. Resource Hooks
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

This template-based approach provides a robust, scalable foundation for managing ArgoCD applications across multiple environments while maintaining consistency and reducing configuration drift.