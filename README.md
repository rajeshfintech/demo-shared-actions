# Shared Actions: Build Once, Promote by Digest, Deploy

This repository provides reusable GitHub Actions workflows that implement a secure, efficient CI/CD pattern: build once, promote by digest, and deploy. This approach ensures consistency across environments while avoiding unnecessary rebuilds.

## Prerequisites

- Set up appropriate secrets for Kubernetes deployment (kubeconfig or AWS OIDC)

## Workflows

### 1. `reusable-build-and-publish.yml`

Builds, tests, and publishes Docker images to GitHub Container Registry (GHCR).

**Key Features:**
- Automatic Python environment setup and testing (if `requirements*.txt` files exist)
- Multi-architecture build support via QEMU/Buildx
- Automatic tagging with commit SHA (`sha-<commit>`)
- Optional latest tag promotion on main branch
- Security scanning with Trivy (optional)
- Build caching for faster subsequent builds
- Automatic build metadata injection (commit hash, build time, branch)

**Inputs:**
- `app_name` (required): Application name for image naming
- `context` (optional): Docker build context, defaults to "."
- `dockerfile` (optional): Dockerfile path, defaults to "Dockerfile"
- `image_name` (optional): Custom image name, defaults to `ghcr.io/{owner}/{app_name}`
- `test_command` (optional): Test command to run, defaults to "pytest -q"
- `python_version` (optional): Python version for testing, defaults to "3.11"
- `push_latest` (optional): Push latest tag on main branch, defaults to false
- `run_trivy` (optional): Run Trivy security scan, defaults to false
- `build_args` (optional): Additional Docker build arguments

**Outputs:**
- `image`: Canonical image reference with digest (`image@sha256:...`)
- `image_tag`: The SHA-based tag that was pushed (`sha-<commit>`)

**Example Usage:**
```yaml
jobs:
  build:
    uses: your-org/shared-actions/.github/workflows/reusable-build-and-publish.yml@main
    with:
      app_name: my-app
      push_latest: true
      run_trivy: true
    secrets: inherit
```

### 2. `reusable-promote-image.yml`

Promotes existing images by retagging them without rebuilding, ensuring the exact same image is deployed across environments.

**Key Features:**
- Uses `skopeo` for efficient manifest copying
- Supports multiple target tags in a single promotion
- No rebuilding - promotes by digest reference
- Maintains image integrity across environments

**Inputs:**
- `image_digest_ref` (required): Full image reference with digest from build workflow
- `promote_to` (required): Comma-separated list of target tags (e.g., "dev,staging,prod")

**Secrets:**
- `GHCR_TOKEN` (optional): Custom GHCR token, falls back to `GITHUB_TOKEN`

**Example Usage:**
```yaml
jobs:
  promote-to-staging:
    uses: your-org/shared-actions/.github/workflows/reusable-promote-image.yml@main
    with:
      image_digest_ref: ${{ needs.build.outputs.image }}
      promote_to: "staging,v1.2.3"
    secrets: inherit
```

### 3. `reusable-deploy-k8s.yml`

Deploys applications to Kubernetes clusters with support for both AWS EKS (via OIDC) and traditional kubeconfig methods.

**Key Features:**
- AWS OIDC integration for EKS clusters (recommended)
- Fallback to kubeconfig secrets
- Support for both standard kubectl and Kustomize deployments
- Environment-specific manifest support
- Automatic namespace creation
- Binary caching for faster runs
- Rollout status monitoring

**Inputs:**
- `namespace` (required): Kubernetes namespace for deployment
- `manifest_path` (required): Path to Kubernetes manifests
- `deployment` (required): Deployment name to update
- `container` (required): Container name within the deployment
- `image_ref` (required): Full image reference to deploy
- `aws_region` (optional): AWS region for EKS
- `aws_role_to_assume` (optional): AWS IAM role for OIDC authentication
- `cluster_name` (optional): EKS cluster name
- `generate_kubeconfig` (optional): Generate kubeconfig via AWS CLI, defaults to true
- `use_kustomize` (optional): Use Kustomize for deployment, defaults to false

**Secrets:**
- `KUBE_CONFIG` (optional): Base64-encoded kubeconfig (fallback method)

**Manifest Structure:**
The workflow supports environment-specific manifests:
- `namespace-{environment}.yaml` or `namespace.yaml`
- `deployment-{environment}.yaml` or `deployment.yaml`
- `service.yaml`

**Example Usage (AWS EKS with OIDC):**
```yaml
jobs:
  deploy:
    uses: your-org/shared-actions/.github/workflows/reusable-deploy-k8s.yml@main
    with:
      aws_region: us-west-2
      aws_role_to_assume: arn:aws:iam::123456789012:role/GitHubActionsRole
      cluster_name: my-eks-cluster
      namespace: production
      manifest_path: k8s/
      deployment: my-app
      container: app
      image_ref: ${{ needs.build.outputs.image }}
```

**Example Usage (Traditional kubeconfig):**
```yaml
jobs:
  deploy:
    uses: your-org/shared-actions/.github/workflows/reusable-deploy-k8s.yml@main
    with:
      generate_kubeconfig: false
      namespace: production
      manifest_path: k8s/
      deployment: my-app
      container: app
      image_ref: ${{ needs.build.outputs.image }}
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
```

## Complete Workflow Example

Here's how to use all three workflows together:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: your-org/shared-actions/.github/workflows/reusable-build-and-publish.yml@main
    with:
      app_name: my-application
      push_latest: ${{ github.ref == 'refs/heads/main' }}
      run_trivy: true
    secrets: inherit

  promote-dev:
    if: github.ref == 'refs/heads/develop'
    needs: build
    uses: your-org/shared-actions/.github/workflows/reusable-promote-image.yml@main
    with:
      image_digest_ref: ${{ needs.build.outputs.image }}
      promote_to: "dev"
    secrets: inherit

  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    needs: [build, promote-dev]
    uses: your-org/shared-actions/.github/workflows/reusable-deploy-k8s.yml@main
    with:
      aws_region: us-west-2
      aws_role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
      cluster_name: dev-cluster
      namespace: development
      manifest_path: k8s/
      deployment: my-application
      container: app
      image_ref: ${{ needs.build.outputs.image }}

  promote-prod:
    if: github.ref == 'refs/heads/main'
    needs: build
    uses: your-org/shared-actions/.github/workflows/reusable-promote-image.yml@main
    with:
      image_digest_ref: ${{ needs.build.outputs.image }}
      promote_to: "prod,latest"
    secrets: inherit

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: [build, promote-prod]
    uses: your-org/shared-actions/.github/workflows/reusable-deploy-k8s.yml@main
    with:
      aws_region: us-west-2
      aws_role_to_assume: ${{ secrets.AWS_ROLE_ARN }}
      cluster_name: prod-cluster
      namespace: production
      manifest_path: k8s/
      deployment: my-application
      container: app
      image_ref: ${{ needs.build.outputs.image }}
```

## Security Best Practices

- Use AWS OIDC for EKS authentication instead of long-lived credentials
- Enable Trivy scanning in the build workflow for security vulnerabilities
- Use digest-based image references to ensure immutable deployments
- Limit `GITHUB_TOKEN` permissions to only what's needed
- Store sensitive data in GitHub Secrets, not in workflow files
