# Building-an-End-to-End-DevOps-Project-on-AWS

In this project, I build a full end-to-end DevOps project on AWS with GitOps workflow. The entire infrastructure was provisioned using Terraform, with state and lock management in AWS S3. The application code is managed on GitHub, and a webhook triggers Jenkins to clone the repo, build a Docker image, push it to Amazon ECR, and update a GitOps-managed repo. ArgoCD watches this repo and automatically deploys to Amazon EKS, using Kubernetes Deployments for app pods and StatefulSets for database pods, backed by Amazon EFS for persistent storage. External access is routed via an Ingress Controller using AWS ALB, secured by AWS Certificate Manager (ACM) and Route 53 for DNS. The entire stack is monitored by Prometheus and visualized through Grafana, with RBAC controlling access and alerts sent via email for any failures in Jenkins pipelines or unhealthy services.

The tech stack includes Terraform, GitHub, Jenkins, Docker, ArgoCD, Helm, Kubernetes, AWS EKS, ECR, EFS, ALB, ACM, Route 53, Prometheus, Grafana, ConfigMap, Secrets, RBAC, and more — delivering a robust, automated, scalable, and secure DevOps pipeline.

![DevOps](https://github.com/user-attachments/assets/9cdbbd38-930b-4b7e-87f7-dd3e98c25f4a)

## ArgoCD ApplicationSet for Auto-Provisioning

This repository has been enhanced with ArgoCD ApplicationSet for automated multi-environment application provisioning using Helm charts.

### Architecture Overview

The project implements a GitOps workflow with:
- **Django Application**: Web application deployed using Helm charts
- **PostgreSQL Database**: Persistent database deployed using Helm charts
- **ArgoCD ApplicationSet**: Automated multi-environment deployment
- **Multi-Environment Support**: Dev, Staging, and Production environments

### Repository Structure

```
.
├── charts/
│   ├── django-app/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── ingress.yaml
│   │       └── _helpers.tpl
│   └── postgres/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── statefulset.yaml
│           ├── service.yaml
│           └── _helpers.tpl
├── Kubernetes with ArgoCD/
│   ├── apps/
│   │   └── applicationset.yaml
│   └── templates/
│       └── applicationset-template.yaml
└── appdeployment.yml (legacy)
```

### Helm Charts

#### Django Application Chart
- **Location**: `charts/django-app/`
- **Features**: Deployment, Service, Ingress, ConfigMaps
- **Configurable**: Replicas, image tags, resources, environment variables

#### PostgreSQL Database Chart
- **Location**: `charts/postgres/`
- **Features**: StatefulSet, Persistent Volume Claims, Service
- **Configurable**: Storage size, resources, database credentials

### ArgoCD ApplicationSet Configuration

The ApplicationSet (`Kubernetes with ArgoCD/apps/applicationset.yaml`) manages:

#### Multi-Environment Deployment
- **Development**: 1 Django replica, 5Gi PostgreSQL storage
- **Staging**: 2 Django replicas, 10Gi PostgreSQL storage
- **Production**: 3 Django replicas, 20Gi PostgreSQL storage

#### Sync Wave Orchestration
- **Wave 1**: PostgreSQL databases (dependencies first)
- **Wave 2**: Django applications (after databases are ready)

#### Auto-Provisioning Features
- Automatic namespace creation
- Self-healing and pruning
- Environment-specific configurations
- Resource scaling per environment

### Setup Instructions

#### Prerequisites
1. Kubernetes cluster
2. ArgoCD installed and configured
3. kubectl access to the cluster

#### Installation Steps

1. **Update repository URL**:
   Edit `Kubernetes with ArgoCD/apps/applicationset.yaml` and replace `<your-repo-url>` with your actual repository URL.

2. **Apply the ApplicationSet**:
   ```bash
   kubectl apply -f "Kubernetes with ArgoCD/apps/applicationset.yaml"
   ```

3. **Verify deployment**:
   ```bash
   # Check ApplicationSet
   kubectl get applicationset -n argocd
   
   # Check generated Applications
   kubectl get applications -n argocd
   
   # Check deployed resources
   kubectl get pods -n dev
   kubectl get pods -n staging
   kubectl get pods -n prod
   ```

### Environment Configuration

#### Development Environment
- **Namespace**: `dev`
- **Django**: 1 replica, 200m CPU, 256Mi memory
- **PostgreSQL**: 5Gi storage, 300m CPU, 512Mi memory
- **Host**: `dev.example.com`

#### Staging Environment
- **Namespace**: `staging`
- **Django**: 2 replicas, 300m CPU, 512Mi memory
- **PostgreSQL**: 10Gi storage, 500m CPU, 1Gi memory
- **Host**: `staging.example.com`

#### Production Environment
- **Namespace**: `prod`
- **Django**: 3 replicas, 500m CPU, 1Gi memory
- **PostgreSQL**: 20Gi storage, 1000m CPU, 2Gi memory
- **Host**: `prod.example.com`

### Monitoring and Troubleshooting

#### Common Commands
```bash
# Check ApplicationSet status
kubectl describe applicationset multi-app-set -n argocd

# Check application sync status
kubectl get applications -n argocd

# Check pod logs
kubectl logs -f deployment/django-app-dev -n dev
kubectl logs -f statefulset/postgres-dev -n dev
```

### Benefits of ApplicationSet Auto-Provisioning

1. **Consistency**: Identical deployment patterns across environments
2. **Scalability**: Easy addition of new environments or applications
3. **Automation**: Zero-touch deployment and updates
4. **GitOps**: Declarative configuration management
5. **Multi-tenancy**: Environment isolation with shared templates
6. **Dependency Management**: Proper orchestration with sync waves

## ArgoCD Templates for Generic Configuration

The repository now includes generic ArgoCD templates that replace static YAML configurations, providing better reusability and maintainability.

### Template Structure

```
Kubernetes with ArgoCD/
├── templates/
│   ├── appproject-template.yaml      # Generic AppProject template
│   └── application-template.yaml     # Generic Application template
├── values/
│   ├── appproject-values.yaml        # Project configuration
│   ├── django-app-values.yaml        # Django app configuration
│   └── postgres-values.yaml          # PostgreSQL configuration
├── applicationset-with-templates.yaml # Template-based ApplicationSet
└── TEMPLATE_USAGE.md                 # Comprehensive usage guide
```

### Key Template Features

#### AppProject Template
- **RBAC Policies**: Role-based access control for teams
- **Resource Whitelisting**: Controlled resource permissions
- **Sync Windows**: Deployment time restrictions
- **Multi-Environment Support**: Dev, staging, production configurations

#### Application Template
- **Helm Chart Support**: Dynamic values and parameters
- **Git Repository Support**: Kustomize integration
- **Flexible Sync Policies**: Automated and manual sync options
- **Notification Integration**: Slack/email alerts
- **Ignore Differences**: Handle dynamic fields

### Usage Options

#### Option 1: Template-Based ApplicationSet (Recommended)
```bash
# Apply the template-based ApplicationSet
kubectl apply -f "Kubernetes with ArgoCD/applicationset-with-templates.yaml"
```

#### Option 2: Individual Templates with Helm
```bash
# Create AppProject
helm template appproject ./templates/appproject-template.yaml \
  --values ./values/appproject-values.yaml | kubectl apply -f -

# Create Applications
helm template django-app ./templates/application-template.yaml \
  --values ./values/django-app-values.yaml | kubectl apply -f -
```

### Template Benefits

1. **Reusability**: Single template for multiple applications and environments
2. **Maintainability**: Centralized configuration patterns
3. **Consistency**: Standardized deployments across teams
4. **Flexibility**: Easy customization through values files
5. **Security**: Built-in RBAC and resource restrictions
6. **Scalability**: Simple addition of new environments or applications

### Migration from Static Files

1. **Backup existing configuration**:
   ```bash
   cp -r "Kubernetes with ArgoCD" "Kubernetes with ArgoCD.backup"
   ```

2. **Update repository URLs** in template files

3. **Apply template-based configuration**:
   ```bash
   kubectl apply -f "Kubernetes with ArgoCD/applicationset-with-templates.yaml"
   ```

4. **Verify deployment**:
   ```bash
   kubectl get applicationset -n argocd
   kubectl get applications -n argocd
   ```

For detailed usage instructions, see <mcfile name="TEMPLATE_USAGE.md" path="/Users/sajid/Documents/Tech-LnD/DevOps/Building-an-End-to-End-DevOps-Project-on-AWS/Kubernetes with ArgoCD/TEMPLATE_USAGE.md"></mcfile>
