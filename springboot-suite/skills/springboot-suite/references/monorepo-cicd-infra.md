# Infrastructure Reference Templates

Use these templates for Terraform, Argo CD, and shared infra files.

Variables used:
- `{{PROJECT_NAME}}` — e.g. `my-app`
- `{{AWS_REGION}}` — e.g. `us-east-1`
- `{{AWS_ACCOUNT_ID}}` — e.g. `123456789012`
- `{{GITHUB_ORG}}` — e.g. `acme-corp`
- `{{APPS}}` — which apps exist: `frontend`, `backend`, or both

---

## infra/terraform/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "{{PROJECT_NAME}}-terraform-state"
    key            = "infra/terraform.tfstate"
    region         = "{{AWS_REGION}}"
    encrypt        = true
    dynamodb_table = "{{PROJECT_NAME}}-terraform-locks"
  }
}
```

Note: The S3 bucket and DynamoDB table must be created manually ONCE before running
`terraform init`. Tell the user this in the README and next steps.

Commands to create them (include in README):
```bash
aws s3api create-bucket --bucket {{PROJECT_NAME}}-terraform-state --region {{AWS_REGION}} \
  --create-bucket-configuration LocationConstraint={{AWS_REGION}}
aws s3api put-bucket-versioning --bucket {{PROJECT_NAME}}-terraform-state \
  --versioning-configuration Status=Enabled
aws dynamodb create-table \
  --table-name {{PROJECT_NAME}}-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region {{AWS_REGION}}
```

---

## infra/terraform/main.tf

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}
```

---

## infra/terraform/variables.tf

```hcl
variable "project_name" {
  description = "Project name used to prefix all resources"
  type        = string
  default     = "{{PROJECT_NAME}}"
}

variable "aws_region" {
  description = "AWS region to deploy to"
  type        = string
  default     = "{{AWS_REGION}}"
}

variable "cluster_version" {
  description = "Kubernetes version for EKS"
  type        = string
  default     = "1.29"
}

variable "node_instance_type" {
  description = "EC2 instance type for EKS nodes"
  type        = string
  default     = "t3.medium"
}

variable "node_min_size" {
  type    = number
  default = 1
}

variable "node_max_size" {
  type    = number
  default = 4
}

variable "node_desired_size" {
  type    = number
  default = 2
}
```

---

## infra/terraform/vpc.tf

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

---

## infra/terraform/eks.tf

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${var.project_name}-cluster"
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    main = {
      instance_types = [var.node_instance_type]
      min_size       = var.node_min_size
      max_size       = var.node_max_size
      desired_size   = var.node_desired_size
    }
  }

  tags = {
    Project = var.project_name
  }
}

resource "kubernetes_namespace" "app" {
  metadata {
    name = var.project_name
  }
}
```

---

## infra/terraform/ecr.tf

Generate one `aws_ecr_repository` block per app in `{{APPS}}`.

```hcl
# Frontend ECR repo (include if frontend is in scope)
resource "aws_ecr_repository" "frontend" {
  name                 = "${var.project_name}-frontend"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  tags = {
    Project = var.project_name
  }
}

# Backend ECR repo (include if backend is in scope)
resource "aws_ecr_repository" "backend" {
  name                 = "${var.project_name}-backend"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  tags = {
    Project = var.project_name
  }
}
```

---

## infra/terraform/iam.tf

```hcl
# IAM user for GitHub Actions to push to ECR
resource "aws_iam_user" "github_actions" {
  name = "${var.project_name}-github-actions"
}

resource "aws_iam_access_key" "github_actions" {
  user = aws_iam_user.github_actions.name
}

resource "aws_iam_user_policy" "ecr_push" {
  name = "${var.project_name}-ecr-push"
  user = aws_iam_user.github_actions.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:PutImage"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## infra/terraform/outputs.tf

```hcl
output "cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
}

output "configure_kubectl" {
  description = "Command to configure kubectl"
  value       = "aws eks update-kubeconfig --region ${var.aws_region} --name ${module.eks.cluster_name}"
}

# Include only repos in scope:
output "ecr_frontend_url" {
  description = "Frontend ECR repository URL"
  value       = aws_ecr_repository.frontend.repository_url
}

output "ecr_backend_url" {
  description = "Backend ECR repository URL"
  value       = aws_ecr_repository.backend.repository_url
}

output "github_actions_access_key_id" {
  description = "AWS access key ID for GitHub Actions — add as GitHub Secret"
  value       = aws_iam_access_key.github_actions.id
  sensitive   = true
}

output "github_actions_secret_access_key" {
  description = "AWS secret access key for GitHub Actions — add as GitHub Secret"
  value       = aws_iam_access_key.github_actions.secret
  sensitive   = true
}
```

---

## argocd/frontend-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{PROJECT_NAME}}-frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/{{GITHUB_ORG}}/{{PROJECT_NAME}}.git
    targetRevision: HEAD
    path: infra/k8s/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: {{PROJECT_NAME}}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## argocd/backend-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{PROJECT_NAME}}-backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/{{GITHUB_ORG}}/{{PROJECT_NAME}}.git
    targetRevision: HEAD
    path: infra/k8s/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: {{PROJECT_NAME}}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## .gitignore

```
# Terraform
**/.terraform/
*.tfstate
*.tfstate.*
*.tfvars
.terraform.lock.hcl

# Environment files
.env
.env.*
!.env.example

# Node
node_modules/
dist/
.next/
build/

# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/

# Go
vendor/

# Docker
*.log

# OS
.DS_Store
Thumbs.db
```
