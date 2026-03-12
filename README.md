# OpenShift GitOps Configuration Repository

This repository contains GitOps configurations for OpenShift clusters using ArgoCD ApplicationSet and Kustomize.

## Repository Structure

```
ocp-ztp-gitops/
├── applicationsets/
│   └── cluster-config.yaml       # ApplicationSet that generates apps per cluster
├── cluster-config/
│   ├── base/                     # Base Kustomize configurations
│   │   ├── kustomization.yaml
│   │   ├── namespaces/           # Namespace configurations
│   │   │   ├── kustomization.yaml
│   │   │   └── example-namespace.yaml
│   │   └── project-templates/    # OpenShift project templates
│   │       ├── kustomization.yaml
│   │       └── project-template.yaml
│   └── overlays/                 # Cluster-specific overlays
│       ├── cluster-dev/          # Development cluster overlay
│       │   └── kustomization.yaml
│       └── cluster-prod/         # Production cluster overlay
│           └── kustomization.yaml
└── README.md
```

## Prerequisites

- OpenShift cluster(s) with OpenShift GitOps (ArgoCD) installed
- Access to create ApplicationSets in the `openshift-gitops` namespace
- Git repository accessible from the cluster

## Setup Instructions

### 1. Install OpenShift GitOps Operator

If not already installed, deploy the OpenShift GitOps operator:

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### 2. Configure Repository Access

Update the `repoURL` in `applicationsets/cluster-config.yaml` to point to your Git repository:

```yaml
source:
  repoURL: https://github.com/your-org/ocp-ztp-gitops.git
```

### 3. Update Cluster Information

Edit the ApplicationSet generator in `applicationsets/cluster-config.yaml` to match your clusters:

```yaml
generators:
  - list:
      elements:
        - cluster: cluster-dev
          url: https://api.dev.example.com:6443
          environment: development
        - cluster: cluster-prod
          url: https://api.prod.example.com:6443
          environment: production
```

### 4. Deploy the ApplicationSet

```bash
oc apply -f applicationsets/cluster-config.yaml
```

## Adding New Clusters

1. Create a new overlay directory:

```bash
mkdir -p cluster-config/overlays/cluster-new
```

2. Create a `kustomization.yaml` in the new overlay:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

commonLabels:
  cluster: cluster-new
  environment: staging
```

3. Add the cluster to the ApplicationSet generator in `applicationsets/cluster-config.yaml`

4. Commit and push the changes

## Adding New Configurations

### Adding a New Namespace

1. Create the namespace YAML in `cluster-config/base/namespaces/`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-new-namespace
```

2. Add it to `cluster-config/base/namespaces/kustomization.yaml`:

```yaml
resources:
  - example-namespace.yaml
  - my-new-namespace.yaml
```

### Customizing Per Cluster

Use Kustomize patches in the overlay `kustomization.yaml` to customize configurations per cluster.

## Included Examples

### Namespace with Resource Quotas

The `example-namespace.yaml` creates:
- A namespace called `example-app`
- ResourceQuota limiting CPU, memory, and pod counts
- LimitRange setting default container resource limits

### Project Template

The `project-template.yaml` creates a default project request template that includes:
- Project creation with metadata
- Admin RoleBinding
- Default ResourceQuota
- Default LimitRange
- Network policies for isolation

To activate the project template, configure the cluster:

```bash
oc patch project.config.openshift.io/cluster --type=merge \
  -p '{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'
```

## Validation

Test Kustomize builds locally before committing:

```bash
# Validate base
kustomize build cluster-config/base

# Validate dev overlay
kustomize build cluster-config/overlays/cluster-dev

# Validate prod overlay
kustomize build cluster-config/overlays/cluster-prod
```

## Best Practices

1. **Always test locally** - Use `kustomize build` to validate configurations
2. **Use overlays for cluster-specific settings** - Keep base configurations generic
3. **Follow GitOps principles** - All changes should go through Git
4. **Use meaningful labels** - Helps with resource discovery and management
5. **Set appropriate sync policies** - Use `selfHeal` and `prune` carefully in production
