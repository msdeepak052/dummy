Yes. For the **exact structure you specified**, here is a complete working example.

This version keeps the VPC creation **inside the EKS module** so you only have one reusable module.

```text
eks-terraform/
├── providers.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    └── eks/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

## 1. `providers.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

---

## 2. `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR"
  type        = string
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "private_subnets" {
  description = "Private subnet CIDRs"
  type        = list(string)
}

variable "public_subnets" {
  description = "Public subnet CIDRs"
  type        = list(string)
}

variable "instance_types" {
  description = "EKS worker node instance types"
  type        = list(string)
}

variable "desired_size" {
  description = "Desired number of worker nodes"
  type        = number
}

variable "min_size" {
  description = "Minimum number of worker nodes"
  type        = number
}

variable "max_size" {
  description = "Maximum number of worker nodes"
  type        = number
}
```

---

# 3. `main.tf`

This is the root module.

```hcl
module "eks" {
  source = "./modules/eks"

  cluster_name       = var.cluster_name
  kubernetes_version = var.kubernetes_version

  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones

  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  instance_types = var.instance_types

  desired_size = var.desired_size
  min_size     = var.min_size
  max_size     = var.max_size
}
```

---

# 4. `outputs.tf`

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = module.eks.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.eks.private_subnet_ids
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = module.eks.public_subnet_ids
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "eks_cluster_endpoint" {
  description = "EKS API endpoint"
  value       = module.eks.cluster_endpoint
}

output "eks_cluster_arn" {
  description = "EKS cluster ARN"
  value       = module.eks.cluster_arn
}

output "node_group_arns" {
  description = "EKS managed node group ARNs"
  value       = module.eks.node_group_arns
}
```

---

# 5. `terraform.tfvars`

```hcl
aws_region = "ap-south-1"

cluster_name = "my-eks-cluster"

kubernetes_version = "1.33"

vpc_cidr = "10.0.0.0/16"

availability_zones = [
  "ap-south-1a",
  "ap-south-1b",
  "ap-south-1c"
]

private_subnets = [
  "10.0.1.0/24",
  "10.0.2.0/24",
  "10.0.3.0/24"
]

public_subnets = [
  "10.0.101.0/24",
  "10.0.102.0/24",
  "10.0.103.0/24"
]

instance_types = [
  "t3.medium"
]

desired_size = 2

min_size = 2

max_size = 3
```

---

# EKS Module

## 6. `modules/eks/variables.tf`

```hcl
variable "cluster_name" {
  type = string
}

variable "kubernetes_version" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "availability_zones" {
  type = list(string)
}

variable "private_subnets" {
  type = list(string)
}

variable "public_subnets" {
  type = list(string)
}

variable "instance_types" {
  type = list(string)
}

variable "desired_size" {
  type = number
}

variable "min_size" {
  type = number
}

variable "max_size" {
  type = number
}
```

---

# 7. `modules/eks/main.tf`

This creates:

* VPC
* Public subnets
* Private subnets
* Internet Gateway
* NAT Gateway
* EKS control plane
* EKS managed node group

```hcl
# ---------------------------------------------------------
# VPC
# ---------------------------------------------------------

module "vpc" {

  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "${var.cluster_name}-vpc"

  cidr = var.vpc_cidr

  azs = var.availability_zones

  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  enable_nat_gateway = true
  single_nat_gateway = true

  enable_dns_hostnames = true
  enable_dns_support   = true

  # Tags required by Kubernetes/AWS Load Balancer
  # for public load balancers.

  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }

  # Tags required for internal load balancers.

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }
}


# ---------------------------------------------------------
# EKS
# ---------------------------------------------------------

module "eks" {

  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0"

  name = var.cluster_name

  kubernetes_version = var.kubernetes_version

  # VPC created above

  vpc_id = module.vpc.vpc_id

  # EKS nodes will be placed
  # in private subnets.

  subnet_ids = module.vpc.private_subnets

  # Give the Terraform caller/admin
  # permissions on the cluster.

  enable_cluster_creator_admin_permissions = true

  # API endpoint

  endpoint_public_access = true

  # -------------------------------------------------------
  # Managed Node Group
  # -------------------------------------------------------

  eks_managed_node_groups = {

    default = {

      name = "${var.cluster_name}-nodes"

      instance_types = var.instance_types

      capacity_type = "ON_DEMAND"

      min_size     = var.min_size
      max_size     = var.max_size
      desired_size = var.desired_size

      disk_size = 30

      # Nodes are placed in private subnets.

      subnet_ids = module.vpc.private_subnets
    }
  }
}
```

---

# 8. `modules/eks/outputs.tf`

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnets
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = module.vpc.public_subnets
}

output "cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = module.eks.cluster_arn
}

output "node_group_arns" {
  description = "Managed node group ARNs"
  value       = module.eks.eks_managed_node_groups
}
```

## Complete flow

```text
                     Terraform
                         |
                         v
                 modules/eks
                         |
              +----------+----------+
              |                     |
              v                     v
         VPC Module            EKS Module
              |                     |
       +------+------+       +------+------+
       |             |       |             |
    Public        Private   Control      Managed
    Subnets       Subnets   Plane       Node Group
       |             |                     |
       |          NAT GW                   |
       |             |                     |
       +-------------+---------------------+
                     |
                     v
                  AWS EKS
```

### Run it

From `eks-terraform/`:

```bash
terraform init
```

Then:

```bash
terraform fmt -recursive
```

Validate:

```bash
terraform validate
```

Plan:

```bash
terraform plan
```

Apply:

```bash
terraform apply
```

After creation:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name my-eks-cluster
```

Then:

```bash
kubectl get nodes
```

Expected:

```text
NAME                          STATUS   ROLES
ip-10-0-1-xxx.ec2.internal    Ready    <none>
ip-10-0-2-xxx.ec2.internal    Ready    <none>
```

### One thing I'd change for a production design

For learning, the above structure is nice:

```text
root
  |
  +-- EKS module
        |
        +-- VPC
        +-- EKS
        +-- Node Groups
```

For a **real platform-engineering repository**, I'd split it further:

```text
modules/
├── vpc/
├── eks/
├── node-group/
├── security-group/
├── iam/
└── addons/
```

Then EKS becomes a composition of those modules. This makes things like **EKS upgrades, separate node groups, security groups, Pod Identity/IRSA, and EKS add-ons** much easier to manage independently.
