# Field Content Workload

This workload enables self-service deployment of field-developed content using GitOps patterns. It allows developers to create their own automation in a GitOps repository and deploy it through the Red Hat Demo Platform without needing deep knowledge of AgnosticD.

## Overview

The Field Content workload creates an ArgoCD Application pointing to a developer's GitOps repository. The developer controls all aspects of their deployment through their repository, while this role provides minimal orchestration and integration with RHDP infrastructure.

## Hybrid Deployment Model

All deployments use Helm charts as the source type. This enables a **hybrid approach** where developers can combine:

- **Helm templates**: Standard Kubernetes manifests for operators, applications, configurations
- **Ansible Jobs**: Kubernetes Jobs running ansible-runner for complex orchestration (waiting for operators, generating secrets, calling APIs, conditional logic)

Use ArgoCD **sync waves** to control the order of deployment. For example:
- Wave 0: Namespace, RBAC setup
- Wave 1: Operator installation
- Wave 2: Ansible Job (waits for operator, performs complex setup)
- Wave 3: Application deployment

## Required Variables

```yaml
ocp4_workload_field_content_gitops_repo_url: "https://github.com/developer/my-field-content"
```

## Optional Variables

```yaml
ocp4_workload_field_content_gitops_repo_revision: "main"         # Git branch/tag
ocp4_workload_field_content_gitops_repo_path: ""                 # Path within repo
ocp4_workload_field_content_helm_values: {}                      # Additional Helm values
```

## Developer Repository Structure

Your GitOps repository must be a Helm chart:

```
my-field-content/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── namespace.yaml           # Wave 0: Create namespace
    ├── operator-subscription.yaml # Wave 1: Install operator
    ├── ansible-job.yaml         # Wave 2: Run Ansible for complex setup (optional)
    ├── application.yaml         # Wave 3: Deploy application
    └── userinfo-configmap.yaml  # RHDP integration
```

For Ansible automation, include the ansible-runner Job in your templates and store playbooks in your repository or a separate Git repo.

## Integration with RHDP

### Data Flow Back to AgnosticD

Create ConfigMaps with the `demo.redhat.com/userinfo` label to pass data back to AgnosticD:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-field-content-info
  labels:
    demo.redhat.com/userinfo: ""
data:
  demo_url: "https://my-demo.apps.cluster.example.com"
  users_json: '{"users": {"user1": {"password": "generated-password"}}}'
```

### Health Checking

Applications with the `demo.redhat.com/application` label will be monitored for health and sync status:

```yaml
metadata:
  labels:
    demo.redhat.com/application: "my-field-content"
```

## Available Deployer Values

The following values are automatically injected into your Helm chart and can be used in templates:

```yaml
deployer:
  domain: "apps.cluster-guid.guid.sandbox.opentlc.com"
  apiUrl: "https://api.cluster-guid.guid.sandbox.opentlc.com:6443"
```

Use these in your templates like:
```yaml
# In templates/route.yaml
spec:
  host: my-app.{{ .Values.deployer.domain }}
```
