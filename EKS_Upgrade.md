Yes. Since you want to manage the **entire EKS upgrade through Terraform**, the clean approach is to make the **EKS version, node-group version, add-on versions, and related configuration explicit in Terraform**.

Below is a production-style modular structure, but still small enough to understand and use for interview preparation.

I’ll use:

```text
EKS current version = 1.35
EKS target version  = 1.36
Region              = ap-south-1
Managed Node Groups
Terraform           = source of truth
```

> The exact EKS add-on version strings should be selected from AWS for your target Kubernetes version at upgrade time. Don't hardcode version numbers copied from an old tutorial.

---

# 1. What Terraform will manage

We want Terraform to control:

```text
                    Terraform
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
      EKS Cluster   EKS Add-ons   Node Groups
          |             |             |
          |             |             |
       1.36         CNI/CoreDNS    1.36
                     kube-proxy
                     EBS CSI
```

And preferably:

```text
Terraform
   |
   +-- VPC
   +-- EKS
   +-- IAM
   +-- Node Groups
   +-- EKS Add-ons
```

For an **upgrade-only** project, however, you may already have VPC/IAM modules. We can keep those separate.

---

# 2. Recommended repository structure

I'd use:

```text
eks-infrastructure/
│
├── environments/
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       ├── outputs.tf
│       └── providers.tf
│
└── modules/
    ├── eks/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── addons.tf
    │
    └── node-group/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

For a real platform repository, I would usually have:

```text
modules/
  vpc/
  eks/
  eks-node-group/
  eks-addons/
  iam/
```

---

# 3. First important concept

Don't create:

```text
cluster-version.tf
node-version.tf
addon-version.tf
```

with random versions everywhere.

Instead define the Kubernetes version centrally:

```hcl
kubernetes_version = "1.36"
```

Then use it consistently.

```text
                kubernetes_version
                       |
          +------------+------------+
          |            |            |
          v            v            v
       EKS CP       Node Group    Add-ons
        1.36           1.36       compatible
```

---

# 4. Provider configuration

Create:

```text
environments/prod/providers.tf
```

```hcl
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket = "my-company-terraform-state"
    key    = "eks/prod/terraform.tfstate"
    region = "ap-south-1"
  }
}

provider "aws" {
  region = var.aws_region
}
```

Your backend should ideally have:

```text
S3
 |
 +-- Versioning
 +-- Encryption
 +-- Restricted IAM
 +-- Optional DynamoDB/locking mechanism depending on your Terraform setup
```

Don't put production credentials in this file.

---

# 5. Environment variables

Create:

```text
environments/prod/variables.tf
```

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
  description = "Target Kubernetes version"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs for EKS"
  type        = list(string)
}
```

---

# 6. Production values

Create:

```text
environments/prod/terraform.tfvars
```

```hcl
aws_region = "ap-south-1"

cluster_name = "production-eks"

kubernetes_version = "1.36"

vpc_id = "vpc-xxxxxxxx"

private_subnet_ids = [
  "subnet-aaaa",
  "subnet-bbbb",
  "subnet-cccc"
]
```

In a real project, don't hardcode infrastructure IDs if they're already produced by your VPC module.

Instead:

```hcl
module.vpc.vpc_id
module.vpc.private_subnet_ids
```

---

# 7. EKS module variables

Create:

```text
modules/eks/variables.tf
```

```hcl
variable "cluster_name" {
  type = string
}

variable "kubernetes_version" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "node_groups" {
  type = map(object({
    instance_types = list(string)
    min_size       = number
    max_size       = number
    desired_size   = number
    capacity_type  = string
  }))
}
```

---

# 8. EKS cluster

Create:

```text
modules/eks/main.tf
```

```hcl
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids = var.private_subnet_ids

    endpoint_private_access = true
    endpoint_public_access  = true
  }

  access_config {
    authentication_mode = "API"
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}
```

The most important line for the upgrade is:

```hcl
version = var.kubernetes_version
```

When you change:

```hcl
kubernetes_version = "1.35"
```

to:

```hcl
kubernetes_version = "1.36"
```

Terraform detects the control-plane version change.

---

# 9. EKS cluster IAM role

Still in:

```text
modules/eks/main.tf
```

```hcl
resource "aws_iam_role" "eks_cluster" {
  name = "${var.cluster_name}-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "eks.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

Attach the EKS cluster policy:

```hcl
resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.eks_cluster.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
```

---

# 10. Managed node groups

Now create:

```text
modules/eks/node-groups.tf
```

```hcl
resource "aws_eks_node_group" "this" {
  for_each = var.node_groups

  cluster_name = aws_eks_cluster.this.name

  node_group_name = each.key

  node_role_arn = aws_iam_role.eks_nodes.arn

  subnet_ids = var.private_subnet_ids

  instance_types = each.value.instance_types

  capacity_type = each.value.capacity_type

  scaling_config {
    desired_size = each.value.desired_size
    min_size     = each.value.min_size
    max_size     = each.value.max_size
  }

  update_config {
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node,
    aws_iam_role_policy_attachment.eks_cni,
    aws_iam_role_policy_attachment.eks_container_registry
  ]
}
```

Notice something important:

There is **no separate Kubernetes version here**.

For managed node groups, you can specify the Kubernetes version in the node group resource when you want to control it explicitly. If omitted, the node group follows the cluster version when created, while upgrades are handled through the node group update. For an explicit upgrade workflow, I prefer making the desired node-group version visible in Terraform rather than relying on implicit behavior.

A more explicit configuration is:

```hcl
version = var.kubernetes_version
```

So:

```hcl
resource "aws_eks_node_group" "this" {
  for_each = var.node_groups

  cluster_name    = aws_eks_cluster.this.name
  node_group_name = each.key

  node_role_arn = aws_iam_role.eks_nodes.arn

  subnet_ids = var.private_subnet_ids

  version = var.kubernetes_version

  instance_types = each.value.instance_types
  capacity_type  = each.value.capacity_type

  scaling_config {
    desired_size = each.value.desired_size
    min_size     = each.value.min_size
    max_size     = each.value.max_size
  }

  update_config {
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node,
    aws_iam_role_policy_attachment.eks_cni,
    aws_iam_role_policy_attachment.eks_container_registry
  ]
}
```

That's the version I'd use for an explicit upgrade model.

---

# 11. Node IAM role

Add:

```hcl
resource "aws_iam_role" "eks_nodes" {
  name = "${var.cluster_name}-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "ec2.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })
}
```

Policies:

```hcl
resource "aws_iam_role_policy_attachment" "eks_worker_node" {
  role = aws_iam_role.eks_nodes.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}
```

```hcl
resource "aws_iam_role_policy_attachment" "eks_cni" {
  role = aws_iam_role.eks_nodes.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}
```

```hcl
resource "aws_iam_role_policy_attachment" "eks_container_registry" {
  role = aws_iam_role.eks_nodes.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}
```

For production, consider whether the VPC CNI permissions should instead be provided through its dedicated IAM role/service-account model rather than the node role.

---

# 12. Node group values

In `terraform.tfvars`:

```hcl
node_groups = {
  platform = {
    instance_types = ["m6i.large"]

    min_size = 2
    max_size = 5

    desired_size = 3

    capacity_type = "ON_DEMAND"
  }

  application = {
    instance_types = ["m6i.xlarge"]

    min_size = 3
    max_size = 10

    desired_size = 5

    capacity_type = "ON_DEMAND"
  }
}
```

Terraform creates:

```text
production-eks
      |
      +-- platform
      |     +-- node
      |     +-- node
      |     +-- node
      |
      +-- application
            +-- node
            +-- node
            +-- node
            +-- node
            +-- node
```

---

# 13. EKS add-ons

Create:

```text
modules/eks/addons.tf
```

I recommend making them configurable.

```hcl
variable "addons" {
  type = map(object({
    addon_version               = optional(string)
    resolve_conflicts_on_create = optional(string, "OVERWRITE")
    resolve_conflicts_on_update = optional(string, "PRESERVE")
  }))
}
```

Then:

```hcl
resource "aws_eks_addon" "this" {
  for_each = var.addons

  cluster_name = aws_eks_cluster.this.name

  addon_name = each.key

  addon_version = try(
    each.value.addon_version,
    null
  )

  resolve_conflicts_on_create = each.value.resolve_conflicts_on_create
  resolve_conflicts_on_update = each.value.resolve_conflicts_on_update

  depends_on = [
    aws_eks_cluster.this
  ]
}
```

---

# 14. Add-on values

In `terraform.tfvars`:

```hcl
addons = {
  vpc-cni = {
    addon_version = "USE_COMPATIBLE_VERSION"
  }

  coredns = {
    addon_version = "USE_COMPATIBLE_VERSION"
  }

  kube-proxy = {
    addon_version = "USE_COMPATIBLE_VERSION"
  }

  aws-ebs-csi-driver = {
    addon_version = "USE_COMPATIBLE_VERSION"
  }
}
```

Don't literally use:

```text
USE_COMPATIBLE_VERSION
```

That is just a placeholder.

Get the appropriate versions from AWS:

```bash
aws eks describe-addon-versions \
  --kubernetes-version 1.36 \
  --addon-name vpc-cni \
  --region ap-south-1
```

Then put the selected compatible version in Terraform.

---

# 15. Root module

Now:

```text
environments/prod/main.tf
```

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name = var.cluster_name

  kubernetes_version = var.kubernetes_version

  vpc_id = var.vpc_id

  private_subnet_ids = var.private_subnet_ids

  node_groups = var.node_groups

  addons = var.addons
}
```

---

# 16. Add variables to root

`environments/prod/variables.tf`:

```hcl
variable "node_groups" {
  type = map(object({
    instance_types = list(string)
    min_size       = number
    max_size       = number
    desired_size   = number
    capacity_type  = string
  }))
}

variable "addons" {
  type = map(object({
    addon_version               = optional(string)
    resolve_conflicts_on_create = optional(string, "OVERWRITE")
    resolve_conflicts_on_update = optional(string, "PRESERVE")
  }))
}
```

---

# 17. Outputs

Create:

```text
modules/eks/outputs.tf
```

```hcl
output "cluster_name" {
  value = aws_eks_cluster.this.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.this.endpoint
}

output "cluster_version" {
  value = aws_eks_cluster.this.version
}

output "node_groups" {
  value = {
    for name, ng in aws_eks_node_group.this :
    name => {
      status  = ng.status
      version = ng.version
    }
  }
}
```

Root:

```text
environments/prod/outputs.tf
```

```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_version" {
  value = module.eks.cluster_version
}

output "node_groups" {
  value = module.eks.node_groups
}
```

---

# 18. Now the important part — the upgrade

Suppose your current Terraform says:

```hcl
kubernetes_version = "1.35"
```

Your node groups:

```text
1.35
```

Your add-ons:

```text
1.35-compatible
```

Now AWS looks like:

```text
Control Plane → 1.35
Nodes         → 1.35
Add-ons       → 1.35
```

---

# 19. Step 1 — Upgrade Insights

Before modifying Terraform:

```text
AWS Console
 → EKS
 → production-eks
 → Upgrade Insights
```

Resolve important issues.

Also verify:

```bash
kubectl get pods -A
kubectl get nodes
kubectl get pdb -A
kubectl get crd
```

---

# 20. Step 2 — Change only the cluster version first

This is where I would **not** blindly change everything at once.

First:

```hcl
kubernetes_version = "1.36"
```

But temporarily keep your node group/add-on versions unchanged if your implementation allows you to stage them independently.

Then:

```bash
terraform plan
```

You want Terraform to show:

```text
aws_eks_cluster.this
~ version: "1.35" -> "1.36"
```

You do **not** want:

```text
50 resources destroyed
```

or unexpected:

```text
IAM replacement
VPC replacement
Node group replacement
```

---

# 21. Step 3 — Apply control-plane upgrade

```bash
terraform apply
```

Terraform asks:

```text
Do you want to perform these actions?
```

Review.

Then:

```text
aws_eks_cluster.this
Modifying...
```

Eventually:

```text
Apply complete!
```

---

# 22. Step 4 — Verify control plane

```bash
aws eks describe-cluster \
  --name production-eks \
  --region ap-south-1 \
  --query 'cluster.version'
```

Expected:

```text
1.36
```

Then:

```bash
kubectl get nodes
```

You may see:

```text
Control Plane = 1.36
Nodes         = 1.35
```

That's okay at this point.

---

# 23. Step 5 — Upgrade add-ons

Now change:

```hcl
addons = {
  vpc-cni = {
    addon_version = "compatible-1.36-version"
  }

  coredns = {
    addon_version = "compatible-1.36-version"
  }

  kube-proxy = {
    addon_version = "compatible-1.36-version"
  }

  aws-ebs-csi-driver = {
    addon_version = "compatible-1.36-version"
  }
}
```

Run:

```bash
terraform plan
```

You should see:

```text
aws_eks_addon.vpc_cni
~ addon_version

aws_eks_addon.coredns
~ addon_version

aws_eks_addon.kube_proxy
~ addon_version

aws_eks_addon.aws_ebs_csi_driver
~ addon_version
```

Then:

```bash
terraform apply
```

---

# 24. Step 6 — Verify add-ons

```bash
kubectl get pods -n kube-system
```

Then:

```bash
aws eks describe-addon \
  --cluster-name production-eks \
  --addon-name vpc-cni \
  --region ap-south-1
```

Repeat for:

```text
CoreDNS
kube-proxy
EBS CSI
```

---

# 25. Step 7 — Upgrade node groups

Now your Terraform variable is already:

```hcl
kubernetes_version = "1.36"
```

Because the node group uses:

```hcl
version = var.kubernetes_version
```

Terraform will see:

```text
Node Group
1.35 → 1.36
```

Run:

```bash
terraform plan
```

You should see something like:

```text
aws_eks_node_group.this["platform"]
~ version: "1.35" -> "1.36"

aws_eks_node_group.this["application"]
~ version: "1.35" -> "1.36"
```

---

# 26. Very important — don't blindly upgrade all node groups

For production, I'd consider doing one node group at a time.

For example:

```text
platform
   ↓
verify
   ↓
application-canary
   ↓
verify
   ↓
remaining application nodes
```

You can make node groups separately controllable in Terraform.

For example:

```hcl
node_groups = {
  platform = {
    ...
  }
}
```

Apply.

Then add/upgrade:

```hcl
application = {
   ...
}
```

depending on how your infrastructure is structured.

---

# 27. Node upgrade configuration

This is important:

```hcl
update_config {
  max_unavailable = 1
}
```

This tells EKS to limit how many nodes in that managed node group can be unavailable during an update.

For example:

```text
10 nodes

max_unavailable = 1

Node 1 → upgrade
Node 2 → upgrade
Node 3 → upgrade
...
```

instead of trying to take down a large number simultaneously.

The exact managed-node-group update behavior is controlled by EKS and the node group's update configuration.

---

# 28. Apply node upgrade

```bash
terraform apply
```

Terraform requests the node-group update.

Then EKS performs the rolling replacement/update.

Monitor:

```bash
kubectl get nodes -w
```

and:

```bash
kubectl get pods -A -w
```

---

# 29. Watch PDBs

During this phase:

```bash
kubectl get pdb -A
```

If a workload becomes blocked:

```text
Pod eviction blocked
       |
       v
PDB
       |
       v
minAvailable not satisfied
```

Investigate before forcing anything.

---

# 30. Verify all nodes

```bash
kubectl get nodes
```

You want:

```text
NAME       STATUS   VERSION
node-1     Ready    v1.36.x
node-2     Ready    v1.36.x
node-3     Ready    v1.36.x
```

---

# 31. Step 8 — Test DNS

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --rm -it \
  -- nslookup kubernetes.default
```

---

# 32. Step 9 — Test networking

Test:

```text
Pod → Pod
Pod → Service
Pod → External
Ingress → Service → Pod
```

Check:

```bash
kubectl get svc -A
kubectl get ingress -A
```

---

# 33. Step 10 — Test storage

```bash
kubectl get pvc -A
```

Everything important should remain:

```text
Bound
```

Test stateful applications.

---

# 34. Step 11 — Test Argo CD

```bash
kubectl get applications -n argocd
```

Expected:

```text
Synced
Healthy
```

---

# 35. Step 12 — Test monitoring

Check:

```text
Dynatrace
CloudWatch
Prometheus
Logs
Alerts
```

Make sure metrics weren't lost during node rotation.

---

# 36. Step 13 — Final Terraform plan

This is extremely important.

Run:

```bash
terraform plan
```

Expected:

```text
No changes.
```

You want:

```text
Terraform configuration
        =
Terraform state
        =
AWS infrastructure
```

If Terraform still wants to modify things, investigate.

---

# 37. Final architecture

After the upgrade:

```text
                         Terraform
                             |
                +------------+------------+
                |            |            |
                v            v            v
             EKS CP       Add-ons      Node Groups
              1.36       1.36-compatible     1.36
                |            |            |
                +------------+------------+
                             |
                             v
                         Kubernetes
                             |
              +--------------+--------------+
              |              |              |
             Apps           Argo           Monitoring
              |
              v
           Production
```

---

# 38. The Terraform Git workflow I recommend

Since this is production, don't directly edit production and immediately apply.

Use Git:

```text
main
 |
 +-- production
```

Create:

```text
feature/eks-1.36-upgrade
```

Change:

```diff
- kubernetes_version = "1.35"
+ kubernetes_version = "1.36"
```

Then:

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
```

Create PR.

Review plan.

Merge.

Then CI/CD:

```text
Git
 ↓
CI
 ↓
terraform plan
 ↓
Approval
 ↓
terraform apply
 ↓
EKS
```

---

# 39. Complete command sequence

Your actual operational sequence becomes:

```bash
# 1. Checkout upgrade branch
git checkout -b feature/eks-1.36-upgrade

# 2. Change Terraform version
# 1.35 → 1.36

# 3. Format
terraform fmt -recursive

# 4. Initialize
terraform init

# 5. Validate
terraform validate

# 6. Plan
terraform plan
```

Review.

Then:

```bash
# 7. Upgrade control plane
terraform apply
```

Verify:

```bash
aws eks describe-cluster \
  --name production-eks \
  --region ap-south-1 \
  --query 'cluster.version'

kubectl get nodes
```

Then:

```bash
# 8. Update addon versions in Terraform

terraform plan

terraform apply
```

Then:

```bash
# 9. Node groups

terraform plan

terraform apply
```

Then monitor:

```bash
kubectl get nodes -w
kubectl get pods -A
```

Finally:

```bash
terraform plan
```

Expected:

```text
No changes.
```

---

# 40. One improvement I'd make for production

I would **not** make the add-on version automatically equal to the Kubernetes version.

For example, don't do:

```hcl
addon_version = "v${var.kubernetes_version}"
```

because:

```text
Kubernetes 1.36
```

doesn't mean:

```text
VPC CNI = 1.36
CoreDNS = 1.36
kube-proxy = 1.36
```

Their versioning is different.

Instead:

```hcl
variable "addons" {
  ...
}
```

and explicitly specify:

```hcl
addons = {
  vpc-cni = {
    addon_version = "..."
  }

  coredns = {
    addon_version = "..."
  }

  kube-proxy = {
    addon_version = "..."
  }

  aws-ebs-csi-driver = {
    addon_version = "..."
  }
}
```

That makes your upgrade predictable and reviewable.

---

# 41. The most important Terraform concept

Your Terraform should have **one source of truth**:

```hcl
kubernetes_version = "1.36"
```

Then:

```text
                    Git
                     |
                     v
                Terraform
                     |
          +----------+----------+
          |          |          |
          v          v          v
      Control      Nodes      Add-ons
       Plane        1.36      explicit
       1.36
```

Not:

```text
Terraform → 1.35
AWS Console → 1.36
```

That creates drift.

---

# 42. What I would say in your Senior DevOps interview

If asked:

> **"How would you upgrade an EKS cluster using Terraform?"**

Say:

> "I keep Terraform as the source of truth and define the target Kubernetes version as an environment variable. Before changing it, I run EKS Upgrade Insights, check deprecated APIs, workload health, PDBs, CRDs, CSI drivers, admission webhooks and third-party controllers, and verify compatible versions of the EKS managed add-ons and node AMIs.
>
> I then update the control-plane version in Terraform and run `terraform plan` to make sure only the expected resources change. I apply the control-plane upgrade first and verify it. Then I update the EKS managed add-ons to versions compatible with the target Kubernetes version. After validating the add-ons, I update managed node groups and use controlled rolling updates with an appropriate `max_unavailable`, while monitoring PDBs and workload availability.
>
> Finally I validate nodes, DNS, networking, storage, ingress, autoscaling, Argo CD and monitoring, and run a final `terraform plan` to ensure there is no drift."

That answer demonstrates that you understand **Terraform + EKS + Kubernetes operations**, rather than just knowing the `aws eks update-cluster-version` command.

One final point: if your **existing EKS cluster is already created by Terraform**, I would **not recreate it using the sample resources above**. The correct approach is to map your existing Terraform modules/resources and make the version change in those modules. If you share your current `module "eks"` / EKS Terraform files, I can show you **exactly which lines to change for the upgrade and what the `terraform plan` should look like**, without risking replacement of your production cluster.
