# Branch Mapping Strategy in CI/CD Workflows

This document explains how branches map to environments in your CI/CD pipelines.

## Overview

Your CI/CD setup uses a **hybrid approach**:
- **Automatic triggers** for development
- **Manual triggers** for UAT and production

## Branch to Environment Mapping

```
┌─────────────┬──────────────────┬─────────────────┬──────────────┐
│   Branch    │   Environment    │   Trigger Type  │   Workflow   │
├─────────────┼──────────────────┼─────────────────┼──────────────┤
│   dev       │   Development    │   Automatic     │   Docker     │
│   uat       │   UAT            │   Manual        │   Docker     │
│   main      │   Production     │   Manual        │   Docker     │
│   Any       │   dev/uat/prod   │   Manual        │   Terraform  │
│   Any       │   dev/uat/prod   │   Manual        │   K8s Deploy │
└─────────────┴──────────────────┴─────────────────┴──────────────┘
```

## Workflow 1: Docker Build & Push

**File:** [`.github/workflows/docker-build-push.yml`](.github/workflows/docker-build-push.yml)

### Automatic Trigger (Development)

**Lines 3-12:**
```yaml
on:
  pull_request:
    types: [closed]
    branches:
      - dev
    paths:
      - 'patient-service/**'
      - 'application-service/**'
      - 'order-service/**'
      - '.github/workflows/docker-build-push.yml'
```

**Mapping:**
- **Branch:** `dev`
- **Environment:** `dev`
- **Trigger:** When PR is merged to `dev` branch
- **Image Tags:** `latest` and `<commit-sha>`

**Logic (Lines 49-52):**
```yaml
elif [ "${{ github.event_name }}" == "pull_request" ] && [ "${{ github.event.pull_request.merged }}" == "true" ]; then
  ENVIRONMENT="dev"
  GIT_TAG="${{ github.sha }}"
  SHOULD_RUN="true"
```

### Manual Trigger (UAT/Production)

**Lines 14-26:**
```yaml
workflow_dispatch:
  inputs:
    environment:
      description: 'Environment to deploy'
      required: true
      type: choice
      options:
        - uat
        - prod
    git_tag:
      description: 'Git release tag (e.g., v1.0.0)'
      required: true
      type: string
```

**Mapping:**
- **Branch:** Any (you specify git tag)
- **Environment:** `uat` or `prod` (you choose)
- **Trigger:** Manual workflow dispatch
- **Image Tags:** `<git-tag>` and `<clean-tag>` (without 'v' prefix)

**Logic (Lines 44-48):**
```yaml
if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
  ENVIRONMENT="${{ github.event.inputs.environment }}"
  GIT_TAG="${{ github.event.inputs.git_tag }}"
  SHOULD_RUN="true"
```

### Image Tagging Strategy

**Development (Lines 164-176):**
```yaml
Build and Push Docker Image (Dev)
tags: |
  ${{ steps.ecr-uri.outputs.repository_uri }}:latest
  ${{ steps.ecr-uri.outputs.repository_uri }}:${{ steps.meta.outputs.short_sha }}
```

**UAT/Production (Lines 178-190):**
```yaml
Build and Push Docker Image (UAT/PROD)
tags: |
  ${{ steps.ecr-uri.outputs.repository_uri }}:${{ env.GIT_TAG }}
  ${{ steps.ecr-uri.outputs.repository_uri }}:${{ steps.meta.outputs.clean_tag }}
```

## Workflow 2: Terraform Infrastructure

**File:** [`.github/workflows/terraform-infrastructure.yml`](.github/workflows/terraform-infrastructure.yml)

### Manual Trigger Only

**Lines 3-21:**
```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - uat
          - prod
      action:
        description: 'Action to perform'
        required: true
        type: choice
        options:
          - plan
          - deploy
```

**Mapping:**
- **Branch:** Any (runs from current branch)
- **Environment:** `dev`, `uat`, or `prod` (you choose)
- **Trigger:** Manual workflow dispatch only
- **Actions:** `plan` or `deploy`

**No automatic branch mapping** - you explicitly select the environment.

## Workflow 3: Kubernetes Deployment

**File:** [`.github/workflows/k8s-deploy.yml`](.github/workflows/k8s-deploy.yml)

### Manual Trigger Only

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - dev
          - uat
          - prod
      service:
        description: 'Service to deploy'
        required: true
        type: choice
        options:
          - all
          - patient-service
          - application-service
          - order-service
```

**Mapping:**
- **Branch:** Any (runs from current branch)
- **Environment:** `dev`, `uat`, or `prod` (you choose)
- **Service:** All or specific service (you choose)
- **Trigger:** Manual workflow dispatch only

## Complete CI/CD Flow by Environment

### Development Environment

```
1. Developer creates PR to 'dev' branch
   ↓
2. Code review and approval
   ↓
3. PR merged to 'dev' branch
   ↓
4. Docker Build workflow AUTOMATICALLY triggers
   - Builds all 3 services
   - Tags: latest, <commit-sha>
   - Pushes to ECR dev repositories
   ↓
5. MANUALLY trigger K8s Deploy workflow
   - Environment: dev
   - Service: all (or specific)
   - Deploys to dev EKS cluster
```

### UAT Environment

```
1. Create git tag (e.g., v1.0.0-rc1)
   git tag v1.0.0-rc1
   git push origin v1.0.0-rc1
   ↓
2. MANUALLY trigger Docker Build workflow
   - Environment: uat
   - Git tag: v1.0.0-rc1
   - Builds from tagged commit
   - Pushes to ECR uat repositories
   ↓
3. MANUALLY trigger K8s Deploy workflow
   - Environment: uat
   - Service: all
   - Deploys to uat EKS cluster
```

### Production Environment

```
1. Create production git tag (e.g., v1.0.0)
   git tag v1.0.0
   git push origin v1.0.0
   ↓
2. MANUALLY trigger Docker Build workflow
   - Environment: prod
   - Git tag: v1.0.0
   - Builds from tagged commit
   - Pushes to ECR prod repositories
   ↓
3. MANUALLY trigger K8s Deploy workflow
   - Environment: prod
   - Service: all
   - Deploys to prod EKS cluster
```

## Branch Strategy Recommendations

### Recommended Git Branch Structure

```
main (production)
  ↓
uat (UAT/staging)
  ↓
dev (development)
  ↓
feature/* (feature branches)
```

### Workflow

1. **Feature Development:**
   ```bash
   git checkout -b feature/new-feature dev
   # Make changes
   git commit -m "Add new feature"
   git push origin feature/new-feature
   # Create PR to dev
   ```

2. **Development Deployment:**
   ```bash
   # PR merged to dev → Automatic Docker build
   # Manual K8s deployment to dev
   ```

3. **UAT Deployment:**
   ```bash
   # Merge dev to uat
   git checkout uat
   git merge dev
   git push origin uat
   
   # Create release candidate tag
   git tag v1.0.0-rc1
   git push origin v1.0.0-rc1
   
   # Manual Docker build with tag
   # Manual K8s deployment to uat
   ```

4. **Production Deployment:**
   ```bash
   # Merge uat to main
   git checkout main
   git merge uat
   git push origin main
   
   # Create production tag
   git tag v1.0.0
   git push origin v1.0.0
   
   # Manual Docker build with tag
   # Manual K8s deployment to prod
   ```

## Environment Variables by Workflow

### Docker Build Workflow

```yaml
ENVIRONMENT: dev | uat | prod
GIT_TAG: <commit-sha> | <git-tag>
AWS_REGION: us-east-1
PROJECT_NAME: microservices
```

### Terraform Workflow

```yaml
ENVIRONMENT: dev | uat | prod
ACTION: plan | deploy
AWS_REGION: us-east-1
PROJECT_NAME: microservices
ECR_REPOS: patient-service,application-service,order-service
```

### K8s Deploy Workflow

```yaml
ENVIRONMENT: dev | uat | prod
SERVICE: all | patient-service | application-service | order-service
IMAGE_TAG: (optional override)
AWS_REGION: us-east-1
```

## GitHub Environments

Each environment should be configured in GitHub with protection rules:

### Development
- **Required reviewers:** 0
- **Wait timer:** 0 minutes
- **Deployment branches:** `dev` only
- **Secrets:** AWS credentials for dev

### UAT
- **Required reviewers:** 1
- **Wait timer:** 0 minutes
- **Deployment branches:** `uat` only
- **Secrets:** AWS credentials for uat

### Production
- **Required reviewers:** 2+
- **Wait timer:** 5 minutes
- **Deployment branches:** `main` only
- **Secrets:** AWS credentials for prod

## Summary

### Automatic Triggers
✅ **Dev environment only** - PR merge to `dev` branch triggers Docker build

### Manual Triggers
✅ **UAT/Prod Docker builds** - Specify environment and git tag  
✅ **All Terraform operations** - Specify environment and action  
✅ **All K8s deployments** - Specify environment and service  

### Key Points

1. **No branch-based automatic deployments for UAT/Prod** - Prevents accidental production deployments
2. **Git tags for UAT/Prod** - Ensures traceability and version control
3. **Manual approval gates** - GitHub environment protection rules
4. **Explicit environment selection** - No ambiguity about target environment
5. **Service-level deployment control** - Deploy all or specific services

This approach provides:
- 🔒 **Safety** - Manual controls for production
- 🚀 **Speed** - Automatic dev deployments
- 📋 **Traceability** - Git tags for releases
- 🎯 **Flexibility** - Deploy any service to any environment
