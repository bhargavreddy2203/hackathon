# Complete CI/CD Architecture Diagram

## 🏗️ Overall Architecture Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEVELOPER WORKFLOW                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GITHUB REPOSITORY                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                            │
│  │  patient-  │  │application-│  │   order-   │                            │
│  │  service   │  │  service   │  │  service   │                            │
│  │  (Node.js) │  │  (Node.js) │  │   (Java)   │                            │
│  └────────────┘  └────────────┘  └────────────┘                            │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    Terraform Infrastructure                         │    │
│  │  • VPC Configuration    • EKS Cluster    • ECR Repositories        │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GITHUB ACTIONS CI/CD                                │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  1. TERRAFORM PLAN ON PR (Automatic)                                 │  │
│  │     terraform-plan-pr.yml                                            │  │
│  │     ├─ Triggers: PR to dev/uat/main                                  │  │
│  │     ├─ Format Check                                                  │  │
│  │     ├─ Validation                                                    │  │
│  │     ├─ Plan Generation                                               │  │
│  │     └─ PR Comment with Plan Output                                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  2. TERRAFORM DEPLOY (Manual)                                        │  │
│  │     terraform-deploy.yml                                             │  │
│  │     ├─ Input: Environment (dev/uat/prod)                             │  │
│  │     ├─ AWS Authentication via Secrets Manager                        │  │
│  │     ├─ Terraform Init                                                │  │
│  │     ├─ Terraform Apply                                               │  │
│  │     └─ Creates: VPC, EKS Cluster, ECR Repositories                  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  3. DOCKER BUILD & PUSH (Manual)                                     │  │
│  │     docker-build.yml                                                 │  │
│  │     ├─ Input: Environment (dev/uat/prod), Git Tag                    │  │
│  │     ├─ AWS Authentication via Secrets Manager                        │  │
│  │     ├─ ECR Login                                                     │  │
│  │     ├─ Build Docker Images (3 services)                              │  │
│  │     ├─ Tag Images (git-tag, sha, timestamp)                          │  │
│  │     └─ Push to ECR                                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  4. KUBERNETES DEPLOY/ROLLBACK (Manual)                              │  │
│  │     k8s-deploy.yml                                                   │  │
│  │     ├─ Input: Environment, Action (deploy/rollback), Image Tag       │  │
│  │     ├─ AWS Authentication via Secrets Manager                        │  │
│  │     ├─ Configure kubectl for EKS                                     │  │
│  │     ├─ Read images.yaml                                              │  │
│  │     ├─ Deploy: Process Manifests & Deploy ALL Services               │  │
│  │     ├─ Rollback: Undo to previous or specific version                │  │
│  │     ├─ Deploy Ingress (deploy only)                                  │  │
│  │     └─ Wait for Rollout                                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS INFRASTRUCTURE                              │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        AWS SECRETS MANAGER                            │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │  │
│  │  │ github-actions/  │  │ github-actions/  │  │ github-actions/  │  │  │
│  │  │ dev/aws-creds    │  │ uat/aws-creds    │  │ prod/aws-creds   │  │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    AMAZON ECR (Container Registry)                    │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  DEV Repositories                                             │   │  │
│  │  │  • microservices-dev-patient-service                          │   │  │
│  │  │  • microservices-dev-application-service                      │   │  │
│  │  │  • microservices-dev-order-service                            │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  UAT Repositories                                             │   │  │
│  │  │  • microservices-uat-patient-service                          │   │  │
│  │  │  • microservices-uat-application-service                      │   │  │
│  │  │  • microservices-uat-order-service                            │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  PROD Repositories                                            │   │  │
│  │  │  • microservices-prod-patient-service                         │   │  │
│  │  │  • microservices-prod-application-service                     │   │  │
│  │  │  • microservices-prod-order-service                           │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    VPC (Per Environment)                              │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │  Public Subnets (2 AZs)                                         │ │  │
│  │  │  ├─ NAT Gateway 1                                               │ │  │
│  │  │  └─ NAT Gateway 2                                               │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │  Private Subnets (2 AZs)                                        │ │  │
│  │  │  ├─ EKS Worker Nodes                                            │ │  │
│  │  │  └─ Application Pods                                            │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    EKS CLUSTER (Per Environment)                      │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │  Control Plane (Managed by AWS)                                 │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │  Worker Nodes (Managed Node Group)                              │ │  │
│  │  │  ├─ Instance Type: t3.medium                                    │ │  │
│  │  │  ├─ Min: 3, Max: 5, Desired: 3                                  │ │  │
│  │  │  └─ IAM Role with ECR Pull Permissions                          │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │  Kubernetes Resources                                           │ │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │  │
│  │  │  │  Deployments                                              │ │ │  │
│  │  │  │  ├─ patient-service (2 replicas)                          │ │ │  │
│  │  │  │  ├─ application-service (2 replicas)                      │ │ │  │
│  │  │  │  └─ order-service (2 replicas)                            │ │ │  │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │  │
│  │  │  │  Services (ClusterIP)                                     │ │ │  │
│  │  │  │  ├─ patient-service:80 → 3000                             │ │ │  │
│  │  │  │  ├─ application-service:80 → 3000                         │ │ │  │
│  │  │  │  └─ order-service:80 → 8080                               │ │ │  │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐ │ │  │
│  │  │  │  Ingress (ALB)                                            │ │ │  │
│  │  │  │  ├─ /patients → patient-service                           │ │ │  │
│  │  │  │  ├─ /appointments → application-service                   │ │ │  │
│  │  │  │  └─ /orders → order-service                               │ │ │  │
│  │  │  └──────────────────────────────────────────────────────────┘ │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              END USERS                                       │
│                                                                               │
│  Internet → ALB → Ingress → Services → Pods (Containers from ECR)           │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 🔄 Complete Deployment Flow

### Step 1: Infrastructure Setup (One-time per environment)

```
Developer → GitHub Actions → Terraform Deploy
                                    │
                                    ▼
                            AWS Secrets Manager
                                    │
                                    ▼
                            Terraform Apply
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
                   VPC             EKS             ECR
```

### Step 2: Application Build & Push

```
Developer → GitHub Actions → Docker Build
                                    │
                                    ▼
                            AWS Secrets Manager
                                    │
                                    ▼
                            ECR Login
                                    │
                                    ▼
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            Build Patient    Build Application  Build Order
              Service           Service          Service
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                            Push to ECR
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            patient-service  application-service  order-service
            :latest          :latest              :latest
            :abc1234         :abc1234             :abc1234
            :dev-timestamp   :dev-timestamp       :dev-timestamp
```

### Step 3: Kubernetes Deployment

```
Developer → GitHub Actions → K8s Deploy
                                    │
                                    ▼
                            AWS Secrets Manager
                                    │
                                    ▼
                            Configure kubectl
                                    │
                                    ▼
                            Read images.yaml
                                    │
                                    ▼
                    Process K8s Manifests
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            Deploy Patient    Deploy Application  Deploy Order
              Service           Service          Service
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                            Deploy Ingress
                                    │
                                    ▼
                            EKS Cluster
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            Pods Pull from ECR (using IAM role)
                    │               │               │
                    ▼               ▼               ▼
            patient-service  application-service  order-service
            containers       containers           containers
            running          running              running
```

## 🔐 Security Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions                            │
│                          │                                   │
│                          ▼                                   │
│              AWS Bootstrap Role (OIDC)                       │
│                          │                                   │
│                          ▼                                   │
│              AWS Secrets Manager                             │
│                          │                                   │
│                          ▼                                   │
│              Environment-Specific Credentials                │
│                          │                                   │
│          ┌───────────────┼───────────────┐                  │
│          ▼               ▼               ▼                  │
│         Dev            UAT             Prod                 │
│      Credentials    Credentials     Credentials             │
└─────────────────────────────────────────────────────────────┘
```

## 📊 Data Flow

```
User Request
     │
     ▼
Internet
     │
     ▼
AWS ALB (Application Load Balancer)
     │
     ▼
Kubernetes Ingress
     │
     ├─ /patients → patient-service:80 → Pod:3000
     │
     ├─ /appointments → application-service:80 → Pod:3000
     │
     └─ /orders → order-service:80 → Pod:8080
```

## 🎯 Environment Isolation

```
┌──────────────┬──────────────┬──────────────┐
│     DEV      │     UAT      │     PROD     │
├──────────────┼──────────────┼──────────────┤
│ VPC (dev)    │ VPC (uat)    │ VPC (prod)   │
│ EKS (dev)    │ EKS (uat)    │ EKS (prod)   │
│ ECR (dev)    │ ECR (uat)    │ ECR (prod)   │
│ Secrets(dev) │ Secrets(uat) │ Secrets(prod)│
└──────────────┴──────────────┴──────────────┘
```

This architecture provides complete isolation between environments with manual control over all deployments!
