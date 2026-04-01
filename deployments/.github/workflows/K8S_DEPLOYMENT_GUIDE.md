# Kubernetes Deployment Pipeline Guide

This guide explains how to use the GitHub Actions workflow to deploy microservices to Kubernetes (EKS).

## Overview

The **Deploy to Kubernetes** workflow allows you to:
- Deploy to **dev**, **uat**, or **prod** environments
- Deploy **all services** or a **specific service** (patient-service, application-service, order-service)
- Override image tags or use defaults from `images.yaml`
- Get deployment status and service URLs automatically

## Workflow File

**Location:** `.github/workflows/k8s-deploy.yml`

## Prerequisites

Before using this workflow, ensure:

1. ✅ **Infrastructure is deployed** (VPC, EKS cluster, ECR repositories)
2. ✅ **Docker images are built and pushed** to ECR
3. ✅ **AWS Secrets Manager is configured** with credentials
4. ✅ **GitHub environments are set up** (dev, uat, prod)
5. ✅ **AWS Bootstrap Role is configured** for OIDC authentication

## How to Use

### Step 1: Navigate to Actions Tab

1. Go to your GitHub repository
2. Click on the **Actions** tab
3. Select **Deploy to Kubernetes** from the workflows list
4. Click **Run workflow** button

### Step 2: Configure Deployment Parameters

You'll see three input fields:

#### 1. Environment (Required)
Choose the target environment:
- **dev** - Development environment
- **uat** - User Acceptance Testing environment
- **prod** - Production environment

#### 2. Service (Required)
Choose which service to deploy:
- **all** - Deploy all three services
- **patient-service** - Deploy only patient service
- **application-service** - Deploy only application service
- **order-service** - Deploy only order service

#### 3. Image Tag (Optional)
- **Leave empty** - Use image tags from `deployments/images.yaml`
- **Specify tag** - Override with custom tag (e.g., `v1.2.3`, `sha-abc123`, `latest`)

### Step 3: Run the Workflow

Click **Run workflow** to start the deployment.

## Usage Examples

### Example 1: Deploy All Services to Dev
```
Environment: dev
Service: all
Image Tag: (leave empty)
```
This deploys all services using the image tags defined in `images.yaml` for dev environment.

### Example 2: Deploy Specific Service to UAT
```
Environment: uat
Service: patient-service
Image Tag: (leave empty)
```
This deploys only the patient-service to UAT using the image tag from `images.yaml`.

### Example 3: Deploy with Custom Image Tag
```
Environment: prod
Service: order-service
Image Tag: v1.5.0
```
This deploys order-service to production using the specific image tag `v1.5.0`.

### Example 4: Hotfix Deployment
```
Environment: prod
Service: application-service
Image Tag: sha-a1b2c3d
```
This deploys a specific commit SHA to production for a hotfix.

## Workflow Steps

The workflow performs the following steps:

1. **Checkout Code** - Gets the latest code from repository
2. **Configure AWS Credentials (Bootstrap)** - Uses OIDC role for initial authentication
3. **Retrieve Secrets** - Gets AWS credentials from Secrets Manager
4. **Configure AWS Credentials (Main)** - Authenticates with retrieved credentials
5. **Setup kubectl** - Installs Kubernetes CLI tool
6. **Configure kubectl for EKS** - Connects to the EKS cluster
7. **Install yq** - YAML processor for reading `images.yaml`
8. **Update images.yaml** - Replaces `<ACCOUNT_ID>` with actual AWS account ID
9. **Override Image Tag** - (If provided) Updates image tags in `images.yaml`
10. **Deploy Services** - Applies Kubernetes manifests
11. **Wait for Deployment** - Monitors rollout status
12. **Get Deployment Status** - Shows deployment details in summary
13. **Get Service URL** - Displays load balancer URL and endpoints

## Deployment Output

After deployment, the workflow provides:

### Deployment Summary
- Environment deployed to
- Service(s) deployed
- Image tag used
- Deployment status
- Pod status
- Ingress configuration

### Service URLs
The workflow displays the load balancer URL and service endpoints:
```
Load Balancer: abc123-xyz.us-east-1.elb.amazonaws.com

Endpoints:
- Patient Service: http://abc123-xyz.us-east-1.elb.amazonaws.com/patients
- Application Service: http://abc123-xyz.us-east-1.elb.amazonaws.com/appointments
- Order Service: http://abc123-xyz.us-east-1.elb.amazonaws.com/orders
```

## Image Configuration

The workflow uses `deployments/images.yaml` for image configuration:

```yaml
dev:
  patient-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/patient-service
    tag: latest
  application-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/application-service
    tag: latest
  order-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/order-service
    tag: latest

uat:
  patient-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/patient-service
    tag: uat-latest
  # ... similar for other services

prod:
  patient-service:
    repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/patient-service
    tag: v1.0.0
  # ... similar for other services
```

## Deployment Strategies

### Development Environment
- **Trigger:** Manual or automated after Docker build
- **Image Tag:** `latest` or commit SHA
- **Frequency:** Multiple times per day
- **Service Selection:** Often individual services for testing

### UAT Environment
- **Trigger:** Manual after dev testing
- **Image Tag:** `uat-latest` or release candidate tag
- **Frequency:** Daily or per sprint
- **Service Selection:** Usually all services together

### Production Environment
- **Trigger:** Manual only
- **Image Tag:** Semantic version (e.g., `v1.2.3`)
- **Frequency:** Scheduled releases
- **Service Selection:** All services or specific for hotfixes

## Rollback Procedure

If a deployment fails or causes issues:

### Option 1: Redeploy Previous Version
1. Run the workflow again
2. Select the same environment and service
3. Specify the previous working image tag
4. Click **Run workflow**

### Option 2: Use kubectl Directly
```bash
# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name microservices-prod-eks-cluster

# Rollback deployment
kubectl rollout undo deployment/patient-service

# Check rollback status
kubectl rollout status deployment/patient-service
```

## Monitoring Deployment

### View Logs in GitHub Actions
1. Go to the workflow run
2. Click on the job name
3. Expand each step to view logs
4. Check the deployment summary at the bottom

### View Kubernetes Status
```bash
# Get deployment status
kubectl get deployments

# Get pod status
kubectl get pods

# View pod logs
kubectl logs -f deployment/patient-service

# Describe deployment
kubectl describe deployment patient-service
```

## Troubleshooting

### Issue: Deployment Timeout
**Symptom:** Workflow fails with "rollout status timeout"

**Solutions:**
- Check pod logs: `kubectl logs -f deployment/<service-name>`
- Check pod events: `kubectl describe pod <pod-name>`
- Verify image exists in ECR
- Check resource limits in deployment manifests

### Issue: Image Pull Error
**Symptom:** Pods show "ImagePullBackOff" or "ErrImagePull"

**Solutions:**
- Verify image tag exists in ECR
- Check ECR repository permissions
- Ensure node IAM role has ECR pull permissions
- Verify image URL format in `images.yaml`

### Issue: Service Not Accessible
**Symptom:** Load balancer URL returns 503 or timeout

**Solutions:**
- Check pod status: `kubectl get pods`
- Verify service configuration: `kubectl get svc`
- Check ingress: `kubectl describe ingress microservices-ingress`
- Verify security group rules allow traffic
- Check application logs for startup errors

### Issue: AWS Authentication Failed
**Symptom:** "Unable to locate credentials" or "Access Denied"

**Solutions:**
- Verify AWS Secrets Manager contains correct credentials
- Check GitHub environment secrets are configured
- Ensure Bootstrap Role ARN is correct
- Verify IAM permissions for the role

## Security Best Practices

1. **Use Specific Image Tags** - Avoid `latest` in production
2. **Environment Protection** - Enable required reviewers for prod
3. **Secrets Management** - Store credentials in AWS Secrets Manager
4. **OIDC Authentication** - Use temporary credentials via OIDC
5. **Least Privilege** - Grant minimal required IAM permissions
6. **Audit Logs** - Review deployment history regularly

## Integration with Other Workflows

### Complete CI/CD Flow

1. **Code Push** → Triggers Docker build workflow
2. **Docker Build** → Builds and pushes images to ECR
3. **Manual Trigger** → Run K8s deployment workflow
4. **Deployment** → Updates services in EKS
5. **Verification** → Test endpoints and monitor logs

### Automated Deployment (Optional)

To automate deployment after Docker build, add to `docker-build-push.yml`:

```yaml
jobs:
  trigger-deployment:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Trigger K8s Deployment
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'k8s-deploy.yml',
              ref: 'main',
              inputs: {
                environment: 'dev',
                service: 'all',
                image_tag: '${{ github.sha }}'
              }
            });
```

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

## Support

For issues or questions:
1. Check workflow logs in GitHub Actions
2. Review Kubernetes pod logs
3. Consult the troubleshooting section above
4. Contact the DevOps team
