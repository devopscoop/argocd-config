# ArgoCD Configuration

ArgoCD configuration for the gomplate-based config management plugin. This repository contains the Helm values and plugin configuration needed to enable dynamic, template-based application deployments.

## Overview

This configuration adds a custom plugin to ArgoCD that processes Helm charts with gomplate templating, enabling environment-specific configurations without duplicating directory structures.

**Part of the ArgoCD ApplicationSet Pattern:**
- Main pattern repository: [argocd-applicationset-pattern](https://github.com/arturo-builds-infra/argocd-applicationset-pattern)
- Container image repository: [argocd-gomplate](https://github.com/arturo-builds-infra/argocd-gomplate)
- **This repository**: ArgoCD configuration with plugin setup

## What's Included

- `values.yaml` - ArgoCD Helm values with plugin configuration
- Plugin sidecar container configuration
- Plugin script that processes templates
- Example cluster credentials with labels

## Prerequisites

- ArgoCD installed via Helm
- Access to pull from `ghcr.io/arturo-builds-infra/argocd-gomplate` (public image, no credentials needed)

## Installation

### Using Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f values.yaml
```

### Upgrading Existing ArgoCD

```bash
helm upgrade argocd argo/argo-cd \
  --namespace argocd \
  -f values.yaml
```

## Configuration

### Plugin Configuration

The plugin is configured as a sidecar container in the repo-server that:

1. Checks for `application.yaml.tpl` (ApplicationSet pattern)
2. Detects Helm chart type (OCI vs HTTPS)
3. Processes `.tpl` files with gomplate
4. Renders Helm charts with processed values
5. Concatenates pre, helm, and post manifests

### Cluster Labels

Cluster labels are exposed as environment variables to templates. Configure them when registering clusters:

```bash
kubectl label secret <cluster-secret> -n argocd \
  alias=banks-meowster \
  environment=prod \
  awsAccount=123456789012 \
  awsRegion=us-west-2
```

Add or modify labels in `values.yaml` under `configs.clusterCredentials` for the in-cluster configuration.

### Private Registry Access

**Note:** These credential configurations are only needed if you are deploying applications that use private Helm charts or pull from private Git repositories. The plugin container image itself is public and requires no credentials.

#### Helm Chart Registries

For applications using private Helm chart registries (OCI registries, private Helm repos), create a Docker config secret. The Docker config file can contain multiple registry configurations, allowing access to different private registries (Docker Hub, GHCR, ECR, etc.) with a single mounted file.

```bash
kubectl create secret generic ghcr-docker-config \
  --namespace argocd \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

Or create it manually with multiple registry auths:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-docker-config
  namespace: argocd
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

Example Docker config with multiple registries:

```json
{
  "auths": {
    "ghcr.io": {
      "auth": "<base64-encoded-username:token>"
    },
    "registry.example.com": {
      "auth": "<base64-encoded-username:password>"
    },
    "123456789012.dkr.ecr.us-west-2.amazonaws.com": {
      "auth": "<base64-encoded-aws-token>"
    }
  }
}
```

The secret is mounted to the plugin sidecar with proper read permissions and used for Helm registry authentication across all configured registries.

#### Git Repository Access

For private Git repositories containing your application code and templates, use ArgoCD's built-in repository credential management. These are configured separately from Helm registry credentials.

Create repository credentials via kubectl:

```bash
kubectl create secret generic repo-credentials \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/your-org/your-repo \
  --from-literal=password=<personal-access-token> \
  --from-literal=username=git

kubectl label secret repo-credentials -n argocd argocd.argoproj.io/secret-type=repository
```

Or add them through the ArgoCD UI under Settings > Repositories.

For SSH access:

```bash
kubectl create secret generic repo-credentials \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:your-org/your-repo.git \
  --from-file=sshPrivateKey=/path/to/private/key

kubectl label secret repo-credentials -n argocd argocd.argoproj.io/secret-type=repository
```

## Plugin Script

The plugin script processes applications in this order:

1. **ApplicationSet pattern**: If `application.yaml.tpl` exists, render it and exit
2. **Helm values**: Process `values.yaml.tpl` with gomplate
3. **Pre-deployment**: If `pre.yaml.tpl` exists, render and output it
4. **Helm chart**: Render the Helm chart with processed values
5. **Post-deployment**: If `post.yaml.tpl` exists, render and output it

All `.tpl` files have access to:
- Environment variables via `{{ .Env.VARIABLE_NAME }}`
- Override data via `{{ ds "env" }}` (if `overrides.yaml` exists)
- All gomplate functions and data sources

## Customization

### Adding Custom Environment Variables

Add environment variables to the plugin container in `values.yaml`:

```yaml
repoServer:
  extraContainers:
    - name: argocd-gomplate
      env:
        - name: CUSTOM_VAR
          value: "custom-value"
```

### Modifying the Plugin Script

The plugin logic is defined in `configs.cmp.plugins.argocd-gomplate.generate`. Modify the script to change how templates are processed.

### Additional ArgoCD Configuration

Add any additional ArgoCD configuration to `values.yaml` as needed:
- RBAC policies
- User accounts
- Repository credentials
- SSO configuration
- Notifications

## Usage

Once configured, use the plugin in your applications:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    plugin:
      name: argocd-gomplate
      env:
        - name: HELM_CHART_URL
          value: "https://charts.example.com"
        - name: HELM_CHART_NAME
          value: "my-chart"
        - name: HELM_CHART_VERSION
          value: "1.0.0"
```

For the complete application pattern, see [argocd-applicationset-pattern](https://github.com/arturo-builds-infra/argocd-applicationset-pattern).

## Troubleshooting

### Check Plugin Container Logs

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server -c argocd-gomplate
```

### Verify Plugin Registration

```bash
kubectl exec -n argocd -it <argocd-repo-server-pod> -- ls /home/argocd/cmp-server/config/
```

Should show `plugin.yaml`.

### Test Template Rendering

Create a test application and check the repo-server logs to see the rendered output.

## License

Apache 2.0
