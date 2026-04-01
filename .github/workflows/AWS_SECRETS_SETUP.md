# AWS Secrets Manager Setup for GitHub Actions

This guide explains how to configure AWS Secrets Manager to store credentials for GitHub Actions workflows.

## Architecture

The workflow uses a two-step authentication process:

1. **Bootstrap Authentication**: GitHub Actions authenticates to AWS using OIDC or minimal bootstrap credentials
2. **Retrieve Main Credentials**: Fetches environment-specific credentials from AWS Secrets Manager
3. **Use Main Credentials**: Uses retrieved credentials for Terraform operations

## Prerequisites

- AWS CLI installed and configured
- Appropriate IAM permissions to create secrets and IAM roles
- GitHub repository configured

## Setup Steps

### Step 1: Create IAM OIDC Provider for GitHub Actions

```bash
# Create OIDC provider for GitHub
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### Step 2: Create Bootstrap IAM Role

Create a role that GitHub Actions will assume to read secrets.

**trust-policy.json:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

**Create the role:**
```bash
# Replace ACCOUNT_ID with your AWS account ID
sed -i 's/ACCOUNT_ID/123456789012/g' trust-policy.json

# Create IAM role
aws iam create-role \
  --role-name GitHubActionsBootstrapRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Bootstrap role for GitHub Actions to access Secrets Manager"
```

### Step 3: Create IAM Policy for Secrets Access

**secrets-policy.json:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:*:*:secret:github-actions/*"
      ]
    }
  ]
}
```

**Attach policy to role:**
```bash
# Create policy
aws iam create-policy \
  --policy-name GitHubActionsSecretsManagerAccess \
  --policy-document file://secrets-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name GitHubActionsBootstrapRole \
  --policy-arn arn:aws:iam::ACCOUNT_ID:policy/GitHubActionsSecretsManagerAccess
```

### Step 4: Create Secrets in AWS Secrets Manager

Create secrets for each environment (dev, uat, prod).

**For Development Environment:**
```bash
aws secretsmanager create-secret \
  --name github-actions/dev/aws-credentials \
  --description "AWS credentials for GitHub Actions - Dev Environment" \
  --secret-string '{
    "aws_access_key_id": "AKIA...",
    "aws_secret_access_key": "..."
  }' \
  --region us-east-1
```

**For UAT Environment:**
```bash
aws secretsmanager create-secret \
  --name github-actions/uat/aws-credentials \
  --description "AWS credentials for GitHub Actions - UAT Environment" \
  --secret-string '{
    "aws_access_key_id": "AKIA...",
    "aws_secret_access_key": "..."
  }' \
  --region us-east-1
```

**For Production Environment:**
```bash
aws secretsmanager create-secret \
  --name github-actions/prod/aws-credentials \
  --description "AWS credentials for GitHub Actions - Prod Environment" \
  --secret-string '{
    "aws_access_key_id": "AKIA...",
    "aws_secret_access_key": "..."
  }' \
  --region us-east-1
```

### Step 5: Configure GitHub Repository Secrets

Add the bootstrap role ARN to GitHub repository secrets:

1. Go to GitHub repository → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Add secret:
   - **Name**: `AWS_BOOTSTRAP_ROLE_ARN`
   - **Value**: `arn:aws:iam::ACCOUNT_ID:role/GitHubActionsBootstrapRole`

## Secret Structure

Each secret in AWS Secrets Manager should have this JSON structure:

```json
{
  "aws_access_key_id": "AKIA...",
  "aws_secret_access_key": "..."
}
```

## Secret Naming Convention

```
github-actions/{environment}/aws-credentials
```

Examples:
- `github-actions/dev/aws-credentials`
- `github-actions/uat/aws-credentials`
- `github-actions/prod/aws-credentials`

## Updating Secrets

### Update a secret value:
```bash
aws secretsmanager update-secret \
  --secret-id github-actions/dev/aws-credentials \
  --secret-string '{
    "aws_access_key_id": "NEW_KEY",
    "aws_secret_access_key": "NEW_SECRET"
  }'
```

### Rotate secrets:
```bash
# Enable automatic rotation (optional)
aws secretsmanager rotate-secret \
  --secret-id github-actions/dev/aws-credentials \
  --rotation-lambda-arn arn:aws:lambda:REGION:ACCOUNT:function:SecretsManagerRotation
```

## Verification

### Test secret retrieval:
```bash
# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id github-actions/dev/aws-credentials \
  --region us-east-1 \
  --query SecretString \
  --output text | jq .
```

### Test OIDC authentication:
```bash
# This will be done automatically by GitHub Actions
# You can test locally using aws-vault or similar tools
```

## Security Best Practices

1. **Use OIDC Instead of Long-Lived Credentials**
   - OIDC provides temporary credentials
   - No long-lived secrets in GitHub

2. **Principle of Least Privilege**
   - Bootstrap role only has Secrets Manager read access
   - Main credentials have only required permissions

3. **Separate Credentials per Environment**
   - Different credentials for dev/uat/prod
   - Limits blast radius of compromised credentials

4. **Enable Secret Rotation**
   - Regularly rotate credentials
   - Use AWS Secrets Manager rotation features

5. **Audit Access**
   - Enable CloudTrail logging
   - Monitor secret access patterns

6. **Encrypt Secrets**
   - Use AWS KMS for encryption
   - Separate KMS keys per environment

## Troubleshooting

### Issue: "Access Denied" when retrieving secrets

**Solution**: Verify IAM role has correct permissions:
```bash
aws iam get-role-policy \
  --role-name GitHubActionsBootstrapRole \
  --policy-name SecretsManagerAccess
```

### Issue: "Secret not found"

**Solution**: Verify secret exists and name is correct:
```bash
aws secretsmanager list-secrets \
  --filters Key=name,Values=github-actions
```

### Issue: OIDC authentication fails

**Solution**: Verify OIDC provider and trust policy:
```bash
aws iam list-open-id-connect-providers
aws iam get-role --role-name GitHubActionsBootstrapRole
```

## Alternative: Using Bootstrap Access Keys

If OIDC is not available, you can use minimal bootstrap access keys:

1. Create IAM user with only Secrets Manager read access
2. Generate access keys
3. Store in GitHub Secrets:
   - `AWS_BOOTSTRAP_ACCESS_KEY_ID`
   - `AWS_BOOTSTRAP_SECRET_ACCESS_KEY`

4. Update workflow to use access keys instead of OIDC

## Cost Considerations

- **Secrets Manager**: $0.40 per secret per month
- **API Calls**: $0.05 per 10,000 API calls
- **Estimated Cost**: ~$1.20/month for 3 secrets (dev/uat/prod)

## Additional Resources

- [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/)
- [GitHub OIDC Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
