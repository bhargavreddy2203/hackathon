# GitHub Actions CI/CD Workflows

This directory contains GitHub Actions workflows for infrastructure deployment and application deployment.

## Workflows Overview

### 1. Terraform Infrastructure (`terraform-infrastructure.yml`)
Manual Terraform deployment pipeline for AWS infrastructure (VPC, EKS, ECR).

### 2. Docker Build & Push (`docker-build-push.yml`)
Automated Docker image builds and pushes to Amazon ECR.

### 3. Kubernetes Deployment (`k8s-deploy.yml`)
Manual deployment of microservices to Kubernetes (EKS) with service selection.

---

## 1. Terraform Infrastructure Workflow

Manual Terraform deployment pipeline - **All deployments require manual trigger**.

## Workflow Trigger

**Manual Only** - No automatic deployments on push or PR.

All workflows must be triggered manually from GitHub Actions UI.

## Workflow Behavior

### Manual Dispatch (Only Trigger Method)
- **Trigger**: Manual workflow run from GitHub UI
- **Required Inputs**:
  - **Environment**: Choose dev/uat/prod
  - **Action**: Choose plan or deploy
- **Use Cases**:
  - Run plan to preview changes
  - Deploy infrastructure after reviewing plan
  - All infrastructure changes

## Workflow Job

### Terraform Job
Single job that runs the CI/CD pipeline script.

**Steps:**
1. Checkout code
2. Configure AWS credentials
3. Setup Terraform and kubectl
4. Make scripts executable
5. **Run `cicd-pipeline.sh`** with selected action and environment
6. Upload artifacts (plan, outputs, deployment info)
7. Create job summary

## Required GitHub Secrets

Configure these secrets in your GitHub repository:

### AWS Credentials (Option 1: Access Keys)
```
AWS_ACCESS_KEY_ID       # AWS Access Key ID
AWS_SECRET_ACCESS_KEY   # AWS Secret Access Key
```

### AWS Credentials (Option 2: OIDC - Recommended)
```
AWS_ROLE_ARN           # ARN of IAM role to assume
```

## Environment Protection Rules

Configure environment protection rules in GitHub:

### Development (dev)
- **Required reviewers**: 0
- **Wait timer**: 0 minutes
- **Deployment branches**: dev only

### UAT (uat)
- **Required reviewers**: 1
- **Wait timer**: 0 minutes
- **Deployment branches**: uat only

### Production (prod)
- **Required reviewers**: 2
- **Wait timer**: 5 minutes
- **Deployment branches**: main only

## Usage Examples

### Example 1: Plan Infrastructure Changes
1. Go to **Actions** tab in GitHub
2. Select **"Terraform Infrastructure"** workflow
3. Click **"Run workflow"** button
4. Select inputs:
   - **Environment**: `dev` (or `uat`/`prod`)
   - **Action**: `plan`
5. Click **"Run workflow"**
6. Wait for workflow to complete
7. Review plan output in:
   - Job logs
   - Downloaded artifacts

### Example 2: Deploy Infrastructure
1. Go to **Actions** tab in GitHub
2. Select **"Terraform Infrastructure"** workflow
3. Click **"Run workflow"** button
4. Select inputs:
   - **Environment**: `dev` (or `uat`/`prod`)
   - **Action**: `deploy`
5. Click **"Run workflow"**
6. Wait for deployment (15-20 minutes for EKS)
7. Review deployment summary in:
   - Job summary
   - Downloaded artifacts

### Example 3: Typical Workflow
```
1. Plan Dev:
   Actions → Run workflow → env=dev, action=plan
   → Review plan output

2. Deploy Dev:
   Actions → Run workflow → env=dev, action=deploy
   → Infrastructure deployed to dev

3. Plan UAT:
   Actions → Run workflow → env=uat, action=plan
   → Review plan output

4. Deploy UAT:
   Actions → Run workflow → env=uat, action=deploy
   → Infrastructure deployed to uat

5. Plan Prod:
   Actions → Run workflow → env=prod, action=plan
   → Review plan output (requires approval)

6. Deploy Prod:
   Actions → Run workflow → env=prod, action=deploy
   → Infrastructure deployed to prod (requires approval)
```

## Workflow Outputs

### Artifacts
Each workflow run creates artifacts:

**Plan Artifacts** (retained for 5 days):
- `terraform-plan-{env}`: Contains tfplan and plan output

**Deployment Artifacts** (retained for 30 days):
- `deployment-info-{env}`: Contains ECR URLs and cluster info

### Job Summary
After deployment, a summary is added to the workflow run:
- Environment details
- EKS cluster name and endpoint
- Next steps for kubectl configuration

## Customization

### Modify Terraform Variables
Edit the "Generate Terraform Variables" step in the workflow:

```yaml
- name: Generate Terraform Variables
  working-directory: terraform
  run: |
    cat > terraform.tfvars << EOF
    # Customize these values
    node_instance_types = ["t3.large"]  # Change instance type
    node_desired_size = 5               # Change node count
    single_nat_gateway = true           # Use single NAT for cost savings
    EOF
```

### Change AWS Region
Update the workflow environment variable:

```yaml
env:
  AWS_REGION: us-west-2  # Change region
```

### Add More Environments
1. Create new branch (e.g., `staging`)
2. Update workflow triggers:
```yaml
on:
  push:
    branches:
      - dev
      - uat
      - staging  # Add new branch
      - main
```
3. Update environment mapping in setup job

## Monitoring and Debugging

### View Workflow Runs
1. Go to Actions tab
2. Select workflow run
3. View job logs

### Debug Failed Runs
1. Check job logs for errors
2. Review Terraform plan output
3. Verify AWS credentials
4. Check backend state

### Re-run Failed Jobs
1. Go to failed workflow run
2. Click "Re-run failed jobs"
3. Or "Re-run all jobs"

## Cost Optimization

### Reduce Costs in Non-Prod
Modify terraform.tfvars generation for dev/uat:

```yaml
- name: Generate Terraform Variables
  run: |
    if [ "${{ env.ENVIRONMENT }}" != "prod" ]; then
      # Use smaller instances for non-prod
      NODE_TYPE="t3.small"
      NODE_COUNT=2
      SINGLE_NAT=true
    else
      NODE_TYPE="t3.medium"
      NODE_COUNT=3
      SINGLE_NAT=false
    fi
```

## Security Best Practices

1. **Use OIDC instead of access keys**
   - More secure
   - Temporary credentials
   - No long-lived secrets

2. **Enable environment protection**
   - Require approvals for prod
   - Restrict deployment branches

3. **Use least privilege IAM**
   - Grant only required permissions
   - Separate roles per environment

4. **Enable audit logging**
   - CloudTrail for AWS actions
   - GitHub audit log for workflow runs

5. **Scan Terraform code**
   - Add tfsec or checkov steps
   - Fail on critical issues

## Troubleshooting

### Issue: Backend not found
**Solution**: Run backend setup manually first:
```bash
cd terraform/backend-setup
terraform init
terraform apply
```

### Issue: AWS credentials invalid
**Solution**: Verify secrets in GitHub repository settings

### Issue: Plan shows unexpected changes
**Solution**: Check if state file is in sync with actual infrastructure

### Issue: Apply fails with timeout
**Solution**: EKS cluster creation takes 15-20 minutes, increase timeout if needed

---

## 2. Docker Build & Push Workflow

See [`docker-build-push.yml`](docker-build-push.yml) for automated Docker image builds.

**Key Features:**
- Builds Docker images for all microservices
- Pushes to Amazon ECR
- Supports dev/uat/prod environments
- Automatic tagging with commit SHA and environment tags

**Triggers:**
- **Dev**: Automatic on PR merge to `dev` branch
- **UAT/Prod**: Manual trigger with git tag

**Documentation:** See workflow file for detailed configuration.

---

## 3. Kubernetes Deployment Workflow

**File:** [`k8s-deploy.yml`](k8s-deploy.yml)

Deploy microservices to Kubernetes with flexible service selection.

### Quick Start

1. **Navigate to Actions Tab**
   - Go to GitHub repository → Actions
   - Select "Deploy to Kubernetes"
   - Click "Run workflow"

2. **Configure Parameters**
   - **Environment**: dev, uat, or prod
   - **Service**: all, patient-service, application-service, or order-service
   - **Image Tag**: (optional) Override default tag from images.yaml

3. **Run Deployment**
   - Click "Run workflow" button
   - Monitor deployment progress
   - View deployment summary and service URLs

### Deployment Options

#### Deploy All Services to Dev
```
Environment: dev
Service: all
Image Tag: (leave empty)
```

#### Deploy Specific Service to UAT
```
Environment: uat
Service: patient-service
Image Tag: (leave empty)
```

#### Deploy with Custom Tag to Prod
```
Environment: prod
Service: order-service
Image Tag: v1.5.0
```

### Workflow Features

✅ **Service Selection** - Deploy all services or individual services
✅ **Environment Support** - dev, uat, prod with separate configurations
✅ **Image Tag Override** - Use custom tags or defaults from images.yaml
✅ **Deployment Status** - Real-time rollout monitoring
✅ **Service URLs** - Automatic display of load balancer endpoints
✅ **Rollback Support** - Easy redeployment of previous versions

### Deployment Process

1. **Authentication** - Uses AWS OIDC and Secrets Manager
2. **EKS Configuration** - Connects to target EKS cluster
3. **Image Resolution** - Reads from images.yaml or uses override
4. **Deployment** - Applies Kubernetes manifests
5. **Monitoring** - Waits for rollout completion
6. **Reporting** - Displays deployment status and URLs

### Output Example

After deployment, you'll see:

```
Deployment Status
Environment: prod
Service: patient-service
Image Tag: v1.5.0

Deployments:
NAME              READY   UP-TO-DATE   AVAILABLE
patient-service   3/3     3            3

Service URLs:
Load Balancer: abc123-xyz.us-east-1.elb.amazonaws.com
- Patient Service: http://abc123-xyz.us-east-1.elb.amazonaws.com/patients
- Application Service: http://abc123-xyz.us-east-1.elb.amazonaws.com/appointments
- Order Service: http://abc123-xyz.us-east-1.elb.amazonaws.com/orders
```

### Complete Documentation

For detailed usage, troubleshooting, and best practices, see:
📖 **[K8S_DEPLOYMENT_GUIDE.md](K8S_DEPLOYMENT_GUIDE.md)**

---

## Complete CI/CD Pipeline Flow

```
1. Code Push
   ↓
2. Docker Build & Push (docker-build-push.yml)
   - Builds images
   - Pushes to ECR
   - Tags with commit SHA
   ↓
3. Kubernetes Deployment (k8s-deploy.yml)
   - Manual trigger
   - Select environment & service
   - Deploy to EKS
   ↓
4. Verification
   - Check deployment status
   - Test service endpoints
   - Monitor logs
```

## Prerequisites

Before using these workflows:

1. ✅ **AWS Account** - With appropriate permissions
2. ✅ **GitHub Secrets** - AWS credentials configured
3. ✅ **Terraform Backend** - S3 and DynamoDB created
4. ✅ **Infrastructure** - VPC, EKS, ECR deployed
5. ✅ **Docker Images** - Built and pushed to ECR

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Terraform GitHub Actions](https://github.com/hashicorp/setup-terraform)
- [AWS GitHub Actions](https://github.com/aws-actions/configure-aws-credentials)
- [Terraform Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html)
- [Kubernetes Deployment Guide](K8S_DEPLOYMENT_GUIDE.md)
- [Docker Build Documentation](docker-build-push.yml)
