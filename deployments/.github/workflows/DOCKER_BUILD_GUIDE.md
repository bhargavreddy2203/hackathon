# Docker Build and Push Pipeline

This document describes the Docker build and push workflow for all environments (dev, uat, prod).

## Workflow File

[`docker-build-push.yml`](docker-build-push.yml)

## Overview

Single unified pipeline that builds and pushes Docker images to Amazon ECR for all three microservices across all environments with environment-specific triggers and tagging strategies.

## Microservices

1. **patient-service** (Node.js)
2. **application-service** (Node.js)
3. **order-service** (Java/Spring Boot)

## Environment-Specific Behavior

### Development (dev)

**Trigger:** Automatic on PR merge to `dev` branch

**Tagging Strategy:**
- `latest` - Always points to the most recent dev build
- `<short-sha>` - Git commit SHA (first 7 characters)

**Example Tags:**
```
microservices-dev-patient-service:latest
microservices-dev-patient-service:a1b2c3d
```

**Workflow:**
1. Create PR to `dev` branch
2. Review and approve PR
3. Merge PR → **Automatic build and push**
4. Images tagged with `latest` and commit SHA

### UAT (uat)

**Trigger:** Manual with git release tag

**Tagging Strategy:**
- `<git-tag>` - Full git tag (e.g., v1.0.0)
- `<clean-tag>` - Git tag without 'v' prefix (e.g., 1.0.0)

**Example Tags:**
```
microservices-uat-patient-service:v1.0.0
microservices-uat-patient-service:1.0.0
```

**Workflow:**
1. Create and push git tag: `git tag v1.0.0 && git push origin v1.0.0`
2. Go to Actions → "Docker Build and Push to ECR"
3. Click "Run workflow"
4. Select: Environment=`uat`, Git Tag=`v1.0.0`
5. Click "Run workflow" → **Manual build and push**

### Production (prod)

**Trigger:** Manual with git release tag

**Tagging Strategy:**
- `<git-tag>` - Full git tag (e.g., v1.0.0)
- `<clean-tag>` - Git tag without 'v' prefix (e.g., 1.0.0)

**Example Tags:**
```
microservices-prod-patient-service:v1.0.0
microservices-prod-patient-service:1.0.0
```

**Workflow:**
1. Ensure tag exists: `git tag v1.0.0 && git push origin v1.0.0`
2. Go to Actions → "Docker Build and Push to ECR"
3. Click "Run workflow"
4. Select: Environment=`prod`, Git Tag=`v1.0.0`
5. Click "Run workflow" → **Manual build and push**

## Workflow Jobs

### 1. Setup Job
Determines environment and git tag based on trigger type:
- **PR merge to dev**: Sets environment=dev, git_tag=commit_sha
- **Manual dispatch**: Uses user-provided environment and git tag

### 2. Build and Push Job
Builds and pushes Docker images for all three services:
- Retrieves AWS credentials from Secrets Manager
- Logs into Amazon ECR
- Builds Docker image with BuildKit caching
- Pushes with environment-specific tags
- Initiates image vulnerability scan

### 3. Summary Job
Creates deployment summary with image details and pull commands

## Usage Examples

### Example 1: Deploy to Dev (Automatic)

```bash
# Create feature branch
git checkout -b feature/new-feature

# Make changes
vim patient-service/src/index.js

# Commit and push
git add .
git commit -m "Add new feature"
git push origin feature/new-feature

# Create PR to dev branch in GitHub UI
# Review and merge PR
# → Workflow automatically triggers
# → Images built and pushed with 'latest' and commit SHA tags
```

### Example 2: Deploy to UAT (Manual)

```bash
# Create and push release tag
git checkout dev
git pull origin dev
git tag v1.0.0
git push origin v1.0.0

# Go to GitHub Actions UI:
# 1. Actions → "Docker Build and Push to ECR"
# 2. Click "Run workflow"
# 3. Select:
#    - Environment: uat
#    - Git tag: v1.0.0
# 4. Click "Run workflow"
# → Images built from tag v1.0.0
# → Pushed with tags: v1.0.0 and 1.0.0
```

### Example 3: Deploy to Production (Manual)

```bash
# Verify tag exists
git tag -l v1.0.0

# If tag doesn't exist, create it
git checkout main
git pull origin main
git tag v1.0.0
git push origin v1.0.0

# Go to GitHub Actions UI:
# 1. Actions → "Docker Build and Push to ECR"
# 2. Click "Run workflow"
# 3. Select:
#    - Environment: prod
#    - Git tag: v1.0.0
# 4. Click "Run workflow"
# → Images built from tag v1.0.0
# → Pushed with tags: v1.0.0 and 1.0.0
```

## Tagging Strategy Summary

| Environment | Trigger | Tags | Example |
|-------------|---------|------|---------|
| **dev** | Auto (PR merge) | `latest`, `<sha>` | `latest`, `a1b2c3d` |
| **uat** | Manual | `<tag>`, `<clean-tag>` | `v1.0.0`, `1.0.0` |
| **prod** | Manual | `<tag>`, `<clean-tag>` | `v1.0.0`, `1.0.0` |

## Image Naming Convention

```
<project>-<environment>-<service>:<tag>
```

Examples:
- `microservices-dev-patient-service:latest`
- `microservices-uat-application-service:v1.0.0`
- `microservices-prod-order-service:1.0.0`

## Pull Images from ECR

### Login to ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com
```

### Pull Images

**Development:**
```bash
docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-patient-service:latest
docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-application-service:latest
docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-order-service:latest
```

**UAT/Production:**
```bash
docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/microservices-uat-patient-service:v1.0.0
docker pull <account-id>.dkr.ecr.us-east-1.amazonaws.com/microservices-prod-patient-service:v1.0.0
```

## Security Features

1. **AWS Secrets Manager Integration**
   - Credentials retrieved from Secrets Manager
   - No long-lived credentials in GitHub

2. **OIDC Authentication**
   - Bootstrap role uses OIDC
   - Temporary credentials only

3. **Image Scanning**
   - Automatic vulnerability scanning on push
   - Scan results available in ECR console

4. **Credential Masking**
   - All sensitive values masked in logs
   - No credential exposure

## Build Optimization

1. **Docker BuildKit**
   - Multi-stage builds
   - Layer caching

2. **GitHub Actions Cache**
   - Build cache stored in GitHub
   - Faster subsequent builds

3. **Parallel Builds**
   - All three services build in parallel
   - Matrix strategy for efficiency

## Monitoring and Artifacts

### Workflow Artifacts
Each run creates:
- Build logs for each service
- Image scan results
- Deployment summary

### Job Summary
After each run, view:
- Service names and tags
- ECR repository URLs
- Pull commands
- Build status

## Troubleshooting

### Issue: Tag already exists in ECR

**Solution**: ECR allows tag overwriting. The workflow will push successfully.

### Issue: Git tag not found

**Solution**: Ensure tag exists and is pushed:
```bash
git tag v1.0.0
git push origin v1.0.0
```

### Issue: ECR repository not found

**Solution**: Ensure Terraform infrastructure is deployed:
```bash
cd terraform
terraform apply
```

### Issue: Authentication failed

**Solution**: Verify AWS Secrets Manager secrets are configured correctly.

## Best Practices

1. **Semantic Versioning**
   - Use semantic versioning for tags: `v1.0.0`, `v1.1.0`, `v2.0.0`
   - Major.Minor.Patch format

2. **Tag Naming**
   - Always prefix with 'v': `v1.0.0` not `1.0.0`
   - Use consistent format across all releases

3. **Testing Before Production**
   - Always deploy to dev first
   - Test in UAT before production
   - Use same tag for UAT and prod

4. **Rollback Strategy**
   - Keep previous tags in ECR
   - Can redeploy any previous version
   - ECR lifecycle policies manage old images

5. **Change Management**
   - Document changes in git commit messages
   - Create GitHub releases for tags
   - Maintain CHANGELOG.md

## Integration with Kubernetes

After images are pushed, deploy to EKS:

```bash
# Update deployment with new image
kubectl set image deployment/patient-service \
  patient-service=<account-id>.dkr.ecr.us-east-1.amazonaws.com/microservices-prod-patient-service:v1.0.0

# Or apply updated manifests
kubectl apply -f k8s/patient-service-deployment.yaml
```

## Cost Considerations

- **ECR Storage**: $0.10 per GB per month
- **Data Transfer**: $0.09 per GB (out to internet)
- **Image Scanning**: First 100 scans per month free, then $0.09 per scan

## Additional Resources

- [Amazon ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Semantic Versioning](https://semver.org/)
