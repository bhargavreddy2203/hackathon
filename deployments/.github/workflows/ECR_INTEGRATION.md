# ECR Integration for EKS Deployments

This document explains how your EKS pods pull container images from Amazon ECR (Elastic Container Registry).

## Overview

Your Kubernetes deployments are fully configured to pull images from ECR. Here's how it works:

## Architecture

```
Docker Build → Push to ECR → EKS Pulls from ECR → Pods Running
```

## Configuration Files

### 1. Images Configuration ([`deployments/images.yaml`](../deployments/images.yaml))

Contains ECR repository URLs for each environment:

```yaml
dev:
  patient-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-patient-service
    tag: latest
  application-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-application-service
    tag: latest
  order-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-order-service
    tag: latest
```

**Key Points:**
- `<ACCOUNT_ID>` is automatically replaced with your AWS Account ID during deployment
- Each environment (dev/uat/prod) has separate ECR repositories
- Tags can be customized per environment (latest, v1.0.0, etc.)

### 2. Kubernetes Manifests ([`deployments/base/`](../deployments/base/))

Base manifests use placeholder images that get replaced during deployment:

```yaml
spec:
  containers:
  - name: patient-service
    image: patient-service:latest  # ← Replaced during deployment
```

## How ECR Pull Works

### Step 1: Image Build & Push
The [`docker-build-push.yml`](docker-build-push.yml) workflow:
1. Builds Docker images
2. Authenticates to ECR
3. Pushes images to ECR repositories

Example ECR image URL:
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-patient-service:latest
```

### Step 2: Deployment
The [`k8s-deploy.yml`](k8s-deploy.yml) workflow:
1. Reads image URLs from [`images.yaml`](../deployments/images.yaml)
2. Replaces placeholder images in manifests
3. Applies manifests to EKS

**Before replacement:**
```yaml
image: patient-service:latest
```

**After replacement:**
```yaml
image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-patient-service:latest
```

### Step 3: EKS Pulls from ECR
When Kubernetes creates pods:
1. EKS node receives pod specification with ECR image URL
2. Node uses IAM role to authenticate to ECR
3. Node pulls image from ECR
4. Container starts with the pulled image

## IAM Permissions

### EKS Node IAM Role
Your EKS nodes need the following IAM policy to pull from ECR:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

**Note:** This is automatically configured when you create EKS cluster with Terraform.

## Image Pull Process

### Automatic Authentication
EKS nodes automatically authenticate to ECR using their IAM role. **No image pull secrets needed!**

### Image Pull Policy
Kubernetes uses these policies:
- `IfNotPresent` - Pull only if image not cached (default)
- `Always` - Always pull latest version
- `Never` - Never pull, use cached only

Current configuration uses default (`IfNotPresent`).

## Verification

### Check if Pods are Pulling from ECR

```bash
# Get pod details
kubectl describe pod <pod-name>

# Look for:
# - Image: Should show full ECR URL
# - Events: Should show "Pulled" or "Already present"
```

Example output:
```
Containers:
  patient-service:
    Image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-patient-service:latest
    
Events:
  Normal  Pulling    10s   kubelet  Pulling image "123456789012.dkr.ecr.us-east-1.amazonaws.com/microservices-dev-patient-service:latest"
  Normal  Pulled     8s    kubelet  Successfully pulled image
  Normal  Created    8s    kubelet  Created container patient-service
  Normal  Started    8s    kubelet  Started container patient-service
```

### Check ECR Repositories

```bash
# List ECR repositories
aws ecr describe-repositories --region us-east-1

# List images in a repository
aws ecr list-images \
  --repository-name microservices-dev-patient-service \
  --region us-east-1
```

## Troubleshooting

### Issue: ImagePullBackOff

**Symptom:**
```bash
kubectl get pods
# Shows: ImagePullBackOff or ErrImagePull
```

**Possible Causes & Solutions:**

#### 1. Image doesn't exist in ECR
```bash
# Check if image exists
aws ecr describe-images \
  --repository-name microservices-dev-patient-service \
  --image-ids imageTag=latest \
  --region us-east-1

# Solution: Build and push image first
```

#### 2. Wrong image tag
```bash
# Check images.yaml has correct tag
cat deployments/images.yaml

# Solution: Update tag in images.yaml or override in workflow
```

#### 3. IAM permissions issue
```bash
# Check node IAM role has ECR permissions
aws iam get-role --role-name <eks-node-role-name>

# Solution: Add ECR permissions to node role
```

#### 4. Wrong AWS account ID
```bash
# Verify account ID in image URL
kubectl describe pod <pod-name> | grep Image:

# Should match your AWS account ID
aws sts get-caller-identity --query Account --output text
```

### Issue: Old Image Version Running

**Symptom:** Deployed new version but pods still running old code

**Solution:**
```bash
# Force pull new image
kubectl rollout restart deployment/patient-service

# Or delete pods to force recreation
kubectl delete pod -l app=patient-service
```

## Best Practices

### 1. Use Specific Tags
❌ **Don't use in production:**
```yaml
tag: latest
```

✅ **Do use in production:**
```yaml
tag: v1.2.3
```

### 2. Tag Strategy by Environment

**Development:**
```yaml
tag: latest  # or commit SHA
```

**UAT:**
```yaml
tag: uat-v1.2.3  # or release candidate
```

**Production:**
```yaml
tag: v1.2.3  # semantic versioning
```

### 3. Image Lifecycle Policy

Set ECR lifecycle policies to clean up old images:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

### 4. Image Scanning

Enable ECR image scanning for security:

```bash
aws ecr put-image-scanning-configuration \
  --repository-name microservices-dev-patient-service \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

## Environment-Specific ECR Repositories

Your setup uses separate ECR repositories per environment:

```
Development:
- microservices-dev-patient-service
- microservices-dev-application-service
- microservices-dev-order-service

UAT:
- microservices-uat-patient-service
- microservices-uat-application-service
- microservices-uat-order-service

Production:
- microservices-prod-patient-service
- microservices-prod-application-service
- microservices-prod-order-service
```

**Benefits:**
- ✅ Environment isolation
- ✅ Different retention policies per environment
- ✅ Separate access controls
- ✅ Clear image promotion path

## Image Promotion Workflow

### Dev → UAT → Prod

```bash
# 1. Build and push to dev
# (Automated via docker-build-push.yml)

# 2. Test in dev environment
# (Manual testing)

# 3. Promote to UAT
aws ecr batch-get-image \
  --repository-name microservices-dev-patient-service \
  --image-ids imageTag=sha-abc123 \
  --query 'images[].imageManifest' \
  --output text | \
aws ecr put-image \
  --repository-name microservices-uat-patient-service \
  --image-tag v1.2.3 \
  --image-manifest -

# 4. Deploy to UAT
# (Via k8s-deploy.yml workflow)

# 5. Test in UAT
# (Manual testing)

# 6. Promote to prod (same process)
```

## Summary

✅ **Your setup is already configured correctly!**

- Images are stored in ECR
- EKS nodes authenticate automatically via IAM
- Deployment workflow replaces placeholder images with ECR URLs
- Pods pull images from ECR when created
- No manual configuration needed

**The workflow handles everything automatically:**
1. Reads ECR URLs from [`images.yaml`](../deployments/images.yaml)
2. Replaces placeholders in manifests
3. Applies to EKS
4. EKS pulls from ECR using node IAM role
