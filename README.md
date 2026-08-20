# DevOps Microservices -- AWS EKS CI/CD Deployment

A practical DevOps project demonstrating how to build, containerize,
push, and deploy Python microservices to **Amazon EKS** using **GitHub
Actions**, **Amazon ECR**, and **AWS IAM OIDC**.

------------------------------------------------------------------------

### 🔐 Security Note

This README intentionally uses placeholders for AWS account IDs, GitHub owner/repository names, and other account-specific identifiers.

Before publishing the repository, replace placeholders such as:

```text
<AWS_ACCOUNT_ID>
<GITHUB_OWNER>
<REPOSITORY>
```

with values only in your private deployment configuration or local environment. Do **not** commit AWS access keys, secret keys, session tokens, passwords, private keys, or other credentials.

# 🏗️ Architecture

``` text
Developer
   │
   │ git push
   ▼
GitHub Repository
   │
   ▼
GitHub Actions (CI/CD)
   ├── Docker Build ──► Amazon ECR
   └── AWS OIDC ──────► IAM Role
                              │
                              ▼
                       Amazon EKS Cluster
                              │
                              ▼
┌──────────────┐ → ┌───────────────┐ → ┌──────────────────┐
│ user-service │   │ order-service │   │ PostgreSQL       │
│ Deployment   │   │ Deployment    │   │ Deployment / Pod │
└──────┬───────┘   └───────┬───────┘   └────────┬─────────┘
       │                   │                     │
       ▼                   ▼                     ▼
   load balancer IP     ClusterIP                PVC
    Service             Service                  │
                                                 ▼
                                                PV
                                                 │
                                                 ▼
                                            Amazon EBS

------------------------------------------------------------------------

## 📁 Project Structure

``` text
devops-microservice-/
│
├── .github/
│   └── workflows/
│       └── main.yml
│
├── k8s/
│   ├── postgres-pvc.yaml
│   ├── postgres-service.yaml
│   ├── postgres.yaml
│   ├── user-service.yaml
│   ├── order-service.yaml
│   └── ...
│
├── user-service/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
├── order-service/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
├── docker-compose.yml
├── README.md
└── .gitignore
```

> The exact Kubernetes filenames may differ depending on the current
> version of the repository. The deployment workflow must reference
> files that actually exist under `k8s/`.

------------------------------------------------------------------------

# 1. Microservices

The project contains two Python services:

### User Service

Location:

``` text
user-service/
```

Files:

``` text
Dockerfile
app.py
requirements.txt
```

### Order Service

Location:

``` text
order-service/
```

Files:

``` text
Dockerfile
app.py
requirements.txt
```

Both services are containerized using Docker.

------------------------------------------------------------------------

# 2. Docker

Each service has a Dockerfile similar to:

``` dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

> Important: Docker instructions are case-insensitive, but the standard
> form is `FROM` and `RUN`.

Build locally:

``` bash
docker build -t user-service ./user-service
docker build -t order-service ./order-service
```

------------------------------------------------------------------------

# 3. Amazon ECR

Two ECR repositories are used:

``` text
user-service
order-service
```

Check repositories:

``` bash
aws ecr describe-repositories \
  --region ap-south-1 \
  --query 'repositories[].repositoryName' \
  --output table
```

Expected:

``` text
order-service
user-service
```

List images:

``` bash
aws ecr list-images \
  --repository-name user-service \
  --region ap-south-1
```

``` bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

------------------------------------------------------------------------

# 4. AWS IAM OIDC Authentication

GitHub Actions does not need long-lived AWS access keys.

Instead, GitHub Actions uses **OIDC** to assume the IAM role:

``` text
GitHubActions-EKS-Deploy
```

Role ARN:

``` text
arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActions-EKS-Deploy
```

OIDC provider:

``` text
arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

The GitHub workflow requires:

``` yaml
permissions:
  id-token: write
  contents: read
```

AWS authentication:

``` yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActions-EKS-Deploy
    aws-region: ap-south-1
```

------------------------------------------------------------------------

# 5. IAM Trust Policy

The IAM role trusts GitHub's OIDC provider.

The important claims are:

``` text
aud = sts.amazonaws.com
sub = repo:<GITHUB_OWNER>/<REPOSITORY>:ref:refs/heads/main
```

Example:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_OWNER>/<REPOSITORY>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

This restricts the role to the repository's `main` branch.

------------------------------------------------------------------------

# 6. IAM Permissions

The GitHub Actions role needs permissions for the AWS resources used by
the pipeline.

For EKS cluster discovery:

``` json
{
  "Effect": "Allow",
  "Action": "eks:DescribeCluster",
  "Resource": "arn:aws:eks:ap-south-1:<AWS_ACCOUNT_ID>:cluster/devops-microservice-cluster"
}
```

The role also requires appropriate ECR permissions for pushing images.

For a production setup, follow the principle of least privilege and
grant only the actions actually required by the workflow.

------------------------------------------------------------------------

# 7. EKS Cluster

Cluster:

``` text
devops-microservice-cluster
```

Region:

``` text
ap-south-1
```

Update kubeconfig manually:

``` bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name devops-microservice-cluster
```

Verify:

``` bash
kubectl get nodes
```

------------------------------------------------------------------------

# 8. EKS Access Entry

The IAM role also needs Kubernetes/EKS access.

The role used by GitHub Actions:

``` text
arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActions-EKS-Deploy
```

An EKS access entry was created for this role:

``` bash
aws eks update-access-entry \
  --cluster-name devops-microservice-cluster \
  --principal-arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActions-EKS-Deploy \
  --username github-actions \
  --region ap-south-1
```

Check the access entry:

``` bash
aws eks describe-access-entry \
  --cluster-name devops-microservice-cluster \
  --principal-arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActions-EKS-Deploy \
  --region ap-south-1
```

Associated EKS access policies can be checked with:

``` bash
aws eks list-associated-access-policies \
  --cluster-name devops-microservice-cluster \
  --principal-arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActions-EKS-Deploy \
  --region ap-south-1
```

------------------------------------------------------------------------

# 9. GitHub Actions CI/CD

The workflow is triggered when code is pushed to `main`.

Basic configuration:

``` yaml
name: Build and Push to ECR

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read
```

Environment:

``` yaml
env:
  AWS_REGION: ap-south-1
  AWS_ACCOUNT_ID: "<AWS_ACCOUNT_ID>"
```

------------------------------------------------------------------------

# 10. Build and Push Images

GitHub Actions logs in to ECR:

``` yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
```

User service:

``` yaml
- name: Build and push user-service
  env:
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    IMAGE_TAG: ${{ github.sha }}
  run: |
    docker build \
      -t $REGISTRY/user-service:$IMAGE_TAG \
      ./user-service

    docker push \
      $REGISTRY/user-service:$IMAGE_TAG
```

Order service:

``` yaml
- name: Build and push order-service
  env:
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    IMAGE_TAG: ${{ github.sha }}
  run: |
    docker build \
      -t $REGISTRY/order-service:$IMAGE_TAG \
      ./order-service

    docker push \
      $REGISTRY/order-service:$IMAGE_TAG
```

Using:

``` text
${{ github.sha }}
```

means every build gets a unique image tag.

Example:

``` text
<AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/user-service:<commit-sha>
```

------------------------------------------------------------------------

# 11. Update kubeconfig in GitHub Actions

The workflow connects to EKS:

``` yaml
- name: Update kubeconfig
  run: |
    aws eks update-kubeconfig \
      --region $AWS_REGION \
      --name devops-microservice-cluster
```

This step previously failed with:

``` text
AccessDeniedException
eks:DescribeCluster
```

The issue was fixed by adding the required IAM permission.

------------------------------------------------------------------------

# 12. Test EKS Access

The workflow verifies Kubernetes access:

``` yaml
- name: Test EKS access
  run: |
    kubectl get nodes
    kubectl get pods -A
```

The important distinction is:

``` text
AWS IAM permission
        ↓
eks:DescribeCluster
        ↓
kubectl configuration
        ↓
EKS access entry / policy
        ↓
Kubernetes API permissions
```

Both AWS-level and EKS/Kubernetes-level permissions must be configured.

------------------------------------------------------------------------

# 13. Kubernetes Deployment

The deployment stage should apply the Kubernetes manifests that actually
exist in:

``` text
k8s/
```

Example:

``` bash
kubectl apply -f k8s/
```

Or apply individual manifests:

``` bash
kubectl apply -f k8s/postgres-pvc.yaml
kubectl apply -f k8s/postgres-service.yaml
kubectl apply -f k8s/postgres.yaml
```

Then deploy the application services.

------------------------------------------------------------------------

# 14. Update Application Images

After the new images are pushed to ECR, Kubernetes deployments can be
updated with the current Git SHA.

Example:

``` bash
kubectl set image deployment/user-service \
  user-service=$REGISTRY/user-service:$IMAGE_TAG
```

``` bash
kubectl set image deployment/order-service \
  order-service=$REGISTRY/order-service:$IMAGE_TAG
```

This allows every GitHub commit to deploy a specific image version.

------------------------------------------------------------------------

# 15. Verify Rollout

Wait for the user-service deployment:

``` bash
kubectl rollout status deployment/user-service
```

Wait for the order-service deployment:

``` bash
kubectl rollout status deployment/order-service
```

Check deployments:

``` bash
kubectl get deployments
```

Check pods:

``` bash
kubectl get pods
```

Check services:

``` bash
kubectl get services
```

Check everything together:

``` bash
kubectl get pods,deployments,services
```

------------------------------------------------------------------------

# 16. Complete Deployment Flow

A successful deployment should look like:

``` text
Developer
   |
   | git push origin main
   v
GitHub Repository
   |
   v
GitHub Actions
   |
   +--> Checkout
   |
   +--> AWS OIDC authentication
   |
   +--> Login to ECR
   |
   +--> Build user-service
   |
   +--> Push user-service to ECR
   |
   +--> Build order-service
   |
   +--> Push order-service to ECR
   |
   +--> Update EKS kubeconfig
   |
   +--> Test EKS access
   |
   +--> Deploy Kubernetes manifests
   |
   +--> Update user-service image
   |
   +--> Update order-service image
   |
   +--> Wait for rollout
   |
   +--> Verify deployment
   |
   v
Amazon EKS
   |
   +--> user-service
   |
   +--> order-service
   |
   +--> PostgreSQL
          |
          v
   PersistentVolumeClaim
          |
          v
    Persistent Volume
          |
          v
    Persistent Storage
```

------------------------------------------------------------------------

## 🚀 Result

The project demonstrates a complete AWS-based CI/CD workflow:

**GitHub → GitHub Actions → OIDC → IAM → ECR → EKS → Kubernetes →
Microservices**

The pipeline uses short-lived AWS credentials through GitHub OIDC
instead of storing long-lived AWS access keys in GitHub.
