# Separate Pipelines Guide - Environment-Specific CI/CD

This guide explains the separate pipeline structure for dev, uat, and prod environments.

## Overview

Your CI/CD setup now has **dedicated pipelines for each environment**:

```
.github/workflows/
├── terraform-dev.yml         # Terraform infrastructure for DEV
├── terraform-uat.yml         # Terraform infrastructure for UAT
├── terraform-prod.yml        # Terraform infrastructure for PROD
├── docker-build-dev.yml      # Docker build for DEV
├── docker-build-uat.yml      # Docker build for UAT
├── docker-build-prod.yml     # Docker build for PROD
├── k8s-deploy-dev.yml        # K8s deployment for DEV
├── k8s-deploy-uat.yml        # K8s deployment for UAT
└── k8s-deploy-prod.yml       # K8s deployment for PROD
```

**Total: 9 separate pipelines** (3 per environment)

## Pipeline Structure

### 🔵 Development Pipelines

#### Terraform Infrastructure - DEV ([`terraform-dev.yml`](terraform-dev.yml))

**Triggers:**
- ✅ **Manual only:** workflow_dispatch

**Actions:**
- `plan` - Preview infrastructure changes
- `deploy` - Apply infrastructure changes

**Usage:**
```bash
Actions → Terraform Infrastructure - DEV → Run workflow
- Action: plan (review changes first)
- Action: deploy (apply changes)
```

**Creates:**
- VPC: `microservices-dev-vpc`
- EKS Cluster: `microservices-dev-eks-cluster`
- ECR Repositories: `microservices-dev-*-service`

#### Docker Build - DEV ([`docker-build-dev.yml`](docker-build-dev.yml))

**Triggers:**
- ✅ **Automatic:** PR merged to `dev` branch
- ✅ **Manual:** workflow_dispatch

**Image Tags:**
- `latest`
- `<commit-sha>`
- `dev-<timestamp>`

**Usage:**
```bash
# Automatic: Merge PR to dev branch
# Manual: Actions → Docker Build - DEV → Run workflow
```

#### K8s Deploy - DEV ([`k8s-deploy-dev.yml`](k8s-deploy-dev.yml))

**Triggers:**
- ✅ **Manual only:** workflow_dispatch

**Inputs:**
- Service: all | patient-service | application-service | order-service
- Image Tag: (optional, defaults to latest)

**Usage:**
```bash
Actions → Deploy to Kubernetes - DEV → Run workflow
- Service: all
- Image Tag: (leave empty for latest)
```

---

---

### 🟡 UAT Pipelines

#### Terraform Infrastructure - UAT ([`terraform-uat.yml`](terraform-uat.yml))

**Triggers:**
- ✅ **Manual only:** workflow_dispatch

**Actions:**
- `plan` - Preview infrastructure changes
- `deploy` - Apply infrastructure changes

**Usage:**
```bash
Actions → Terraform Infrastructure - UAT → Run workflow
- Action: plan (review changes first)
- Action: deploy (apply changes)
```

**Creates:**
- VPC: `microservices-uat-vpc`
- EKS Cluster: `microservices-uat-eks-cluster`
- ECR Repositories: `microservices-uat-*-service`

#### Docker Build - UAT ([`docker-build-uat.yml`](docker-build-uat.yml))

**Triggers:**
- ✅ **Automatic:** Push to `uat` branch
- ✅ **Manual:** workflow_dispatch with optional git tag

**Image Tags:**
- `<git-tag>` (if provided)
- `<clean-tag>` (without 'v' prefix)
- `uat-latest`
- `uat-<timestamp>`

**Usage:**
```bash
# Automatic: Push to uat branch
git checkout uat
git merge dev
git push origin uat

# Manual with tag:
Actions → Docker Build - UAT → Run workflow
- Git tag: v1.0.0-rc1
```

#### K8s Deploy - UAT ([`k8s-deploy-uat.yml`](k8s-deploy-uat.yml))

**Triggers:**
- ✅ **Manual only:** workflow_dispatch

**Inputs:**
- Service: all | patient-service | application-service | order-service
- Image Tag: (optional, uses images.yaml if empty)

**Usage:**
```bash
Actions → Deploy to Kubernetes - UAT → Run workflow
- Service: all
- Image Tag: v1.0.0-rc1
```

---

---

### 🔴 Production Pipelines

#### Terraform Infrastructure - PROD ([`terraform-prod.yml`](terraform-prod.yml))

**Triggers:**
- ✅ **Manual only:** workflow_dispatch

**Actions:**
- `plan` - Preview infrastructure changes
- `deploy` - Apply infrastructure changes

**Usage:**
```bash
Actions → Terraform Infrastructure - PROD → Run workflow
- Action: plan (review changes carefully!)
- Action: deploy (apply changes)
```

**Creates:**
- VPC: `microservices-prod-vpc`
- EKS Cluster: `microservices-prod-eks-cluster`
- ECR Repositories: `microservices-prod-*-service`

**⚠️ Production Warnings:**
- Requires multiple approvals
- 90-day artifact retention
- Extra logging and warnings

#### Docker Build - PROD ([`docker-build-prod.yml`](docker-build-prod.yml))

**Triggers:**
- ✅ **Automatic:** Push to `main` branch (requires git tag)
- ✅ **Manual:** workflow_dispatch with **required** git tag

**Image Tags:**
- `<git-tag>` (required)
- `<clean-tag>` (without 'v' prefix)
- `prod-latest`

**Usage:**
```bash
# With tag:
git checkout main
git merge uat
git tag v1.0.0
git push origin main --tags

# Manual:
Actions → Docker Build - PROD → Run workflow
- Git tag: v1.0.0 (REQUIRED)
```

#### K8s Deploy - PROD ([`k8s-deploy-prod.yml`](k8s-deploy-prod.yml))

**Triggers:**
- ✅ **Manual only:** workflow_dispatch

**Inputs:**
- Service: all | patient-service | application-service | order-service
- Image Tag: **REQUIRED** for production

**Usage:**
```bash
Actions → Deploy to Kubernetes - PROD → Run workflow
- Service: all
- Image Tag: v1.0.0 (REQUIRED)
```

---

## Branch Mapping

| Branch | Environment | Docker Build | K8s Deploy |
|--------|-------------|--------------|------------|
| `dev` | Development | Automatic on PR merge | Manual |
| `uat` | UAT | Automatic on push | Manual |
| `main` | Production | Automatic on push (with tag) | Manual |

## Complete Workflows

### Development Workflow

```
1. Create feature branch
   git checkout -b feature/new-feature dev

2. Make changes and commit
   git add .
   git commit -m "Add new feature"
   git push origin feature/new-feature

3. Create PR to dev branch
   (GitHub UI)

4. Review and merge PR
   → Docker Build - DEV runs automatically
   → Builds all 3 services
   → Tags: latest, <sha>, dev-<timestamp>

5. Deploy to DEV
   Actions → Deploy to Kubernetes - DEV
   - Service: all
   - Image Tag: (empty for latest)
   → Deploys to dev EKS cluster
```

### UAT Workflow

```
1. Merge dev to uat
   git checkout uat
   git merge dev
   git push origin uat
   → Docker Build - UAT runs automatically

2. Or create release candidate tag
   git tag v1.0.0-rc1
   git push origin v1.0.0-rc1
   Actions → Docker Build - UAT
   - Git tag: v1.0.0-rc1

3. Deploy to UAT
   Actions → Deploy to Kubernetes - UAT
   - Service: all
   - Image Tag: v1.0.0-rc1
   → Deploys to uat EKS cluster

4. Test in UAT environment
```

### Production Workflow

```
1. Merge uat to main
   git checkout main
   git merge uat
   
2. Create production tag
   git tag v1.0.0
   git push origin main --tags
   → Docker Build - PROD runs automatically

3. Deploy to PRODUCTION
   Actions → Deploy to Kubernetes - PROD
   - Service: all
   - Image Tag: v1.0.0 (REQUIRED)
   → Deploys to prod EKS cluster

4. Verify production deployment
```

## Pipeline Features by Environment

### Development
- ✅ Automatic Docker builds on PR merge
- ✅ Latest tag always available
- ✅ Commit SHA tags for traceability
- ✅ Timestamp tags for history
- ✅ Manual K8s deployment
- ✅ Optional image tag override

### UAT
- ✅ Automatic Docker builds on push
- ✅ Git tag support for releases
- ✅ UAT-specific tags
- ✅ Manual K8s deployment
- ✅ Optional image tag override
- ⚠️ Requires approval (GitHub environment)

### Production
- ✅ Automatic Docker builds on push (with tag)
- ✅ **Required** git tag for builds
- ✅ **Required** image tag for deployments
- ✅ Manual K8s deployment only
- ⚠️ Requires multiple approvals (GitHub environment)
- ⚠️ Production warnings in logs

## GitHub Environment Configuration

### Setup Required

Configure these environments in GitHub repository settings:

#### Development Environment
```yaml
Name: dev
Protection rules:
  - Required reviewers: 0
  - Wait timer: 0 minutes
  - Deployment branches: dev
Secrets:
  - AWS_BOOTSTRAP_ROLE_ARN
```

#### UAT Environment
```yaml
Name: uat
Protection rules:
  - Required reviewers: 1
  - Wait timer: 0 minutes
  - Deployment branches: uat
Secrets:
  - AWS_BOOTSTRAP_ROLE_ARN
```

#### Production Environment
```yaml
Name: prod
Protection rules:
  - Required reviewers: 2+
  - Wait timer: 5 minutes
  - Deployment branches: main
Secrets:
  - AWS_BOOTSTRAP_ROLE_ARN
```

## Image Tagging Strategy

### Development
```
latest                    # Always points to latest dev build
abc1234                   # Commit SHA (short)
dev-20260331-120000      # Timestamp
```

### UAT
```
v1.0.0-rc1               # Git tag
1.0.0-rc1                # Clean tag (no 'v')
uat-latest               # Latest UAT build
uat-20260331-120000      # Timestamp
```

### Production
```
v1.0.0                   # Git tag (required)
1.0.0                    # Clean tag (no 'v')
prod-latest              # Latest prod build
```

## Service Selection

All K8s deployment pipelines support service selection:

### Deploy All Services
```
Service: all
→ Deploys patient-service, application-service, order-service
→ Updates ingress configuration
```

### Deploy Specific Service
```
Service: patient-service
→ Deploys only patient-service
→ Useful for hotfixes or individual updates
```

## Rollback Procedures

### Development Rollback
```bash
Actions → Deploy to Kubernetes - DEV
- Service: <affected-service>
- Image Tag: <previous-working-sha>
```

### UAT Rollback
```bash
Actions → Deploy to Kubernetes - UAT
- Service: <affected-service>
- Image Tag: <previous-working-tag>
```

### Production Rollback
```bash
Actions → Deploy to Kubernetes - PROD
- Service: <affected-service>
- Image Tag: <previous-production-tag>
```

## Monitoring Deployments

### View Pipeline Status
```
GitHub → Actions → Select workflow → View run
```

### Check Deployment Summary
Each pipeline provides:
- ✅ Image details and tags
- ✅ Deployment status
- ✅ Pod status
- ✅ Service URLs
- ✅ Ingress configuration

### Verify Deployment
```bash
# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name microservices-<env>-eks-cluster

# Check deployments
kubectl get deployments

# Check pods
kubectl get pods

# Check services
kubectl get svc

# Check ingress
kubectl get ingress
```

## Troubleshooting

### Issue: Docker Build Fails

**Dev:**
```bash
# Check if PR was actually merged
# Check service code for errors
# Review build logs in Actions
```

**UAT/Prod:**
```bash
# Verify git tag exists
git tag -l
# Check if tag was pushed
git ls-remote --tags origin
```

### Issue: K8s Deployment Fails

**All Environments:**
```bash
# Check if image exists in ECR
aws ecr describe-images --repository-name microservices-<env>-<service>

# Check pod logs
kubectl logs -f deployment/<service-name>

# Check pod events
kubectl describe pod <pod-name>
```

### Issue: Image Not Found

**Solution:**
```bash
# Verify image tag in images.yaml
cat deployments/images.yaml

# List images in ECR
aws ecr list-images --repository-name microservices-<env>-<service>

# Rebuild if necessary
Actions → Docker Build - <ENV> → Run workflow
```

## Best Practices

### Development
- ✅ Merge PRs frequently
- ✅ Test in dev before promoting to UAT
- ✅ Use commit SHAs for specific versions
- ✅ Deploy often to catch issues early

### UAT
- ✅ Use semantic versioning for tags (v1.0.0-rc1)
- ✅ Test thoroughly before promoting to prod
- ✅ Document test results
- ✅ Get stakeholder approval

### Production
- ✅ Always use git tags (v1.0.0)
- ✅ Never use 'latest' tag
- ✅ Require multiple approvals
- ✅ Deploy during maintenance windows
- ✅ Have rollback plan ready
- ✅ Monitor closely after deployment

## Security Considerations

### Secrets Management
- ✅ Use AWS Secrets Manager for credentials
- ✅ Use OIDC for temporary credentials
- ✅ Rotate secrets regularly
- ✅ Limit secret access by environment

### Access Control
- ✅ Restrict who can approve prod deployments
- ✅ Use branch protection rules
- ✅ Require code reviews
- ✅ Enable audit logging

### Image Security
- ✅ Scan images for vulnerabilities
- ✅ Use specific tags, not 'latest' in prod
- ✅ Sign images (optional)
- ✅ Use private ECR repositories

## Summary

✅ **6 separate pipelines** - 3 for Docker builds, 3 for K8s deployments  
✅ **Environment isolation** - Each environment has dedicated workflows  
✅ **Automatic triggers** - Dev and UAT can auto-build  
✅ **Manual deployments** - All K8s deployments require manual trigger  
✅ **Service selection** - Deploy all or individual services  
✅ **Tag management** - Environment-specific tagging strategies  
✅ **Safety controls** - Production requires tags and approvals  

This structure provides clear separation, safety, and flexibility for your CI/CD process!
