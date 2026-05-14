+++
title = "A Terraform Module for EKS GPU Node Groups"
date = 2026-05-05T09:00:00+10:00
draft = false
summary = "A small, opinionated Terraform module for adding g5/p4d/p5 capacity to an existing EKS cluster — taints, labels, capacity reservations, and the IAM bits people forget."
tags = ["terraform", "eks", "aws", "gpu", "iac"]
categories = ["infrastructure"]
series = ["llm-platform"]
ShowToc = true
[cover]
  image = "/img/covers/terraform-eks-gpu.svg"
  alt = "Terraform GPU pools cover with HCL snippet"
  caption = "Cover · Terraform GPU pools"
+++

Most public EKS Terraform modules treat GPU node groups as an afterthought. This is the slim wrapper I keep reaching for, with the choices that have bitten me documented inline.

## What the module assumes

- You already have an EKS cluster managed by [terraform-aws-modules/eks/aws](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest).
- You will install the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) separately (Helm or via Karpenter's user-data).
- You want **Spot for elastic, On-Demand Capacity Reservation for the floor.**

## The module

```hcl
# modules/eks-gpu-nodegroup/variables.tf
variable "cluster_name"     { type = string }
variable "subnet_ids"       { type = list(string) }
variable "instance_types"   { type = list(string)  default = ["g5.2xlarge"] }
variable "capacity_type"    { type = string        default = "SPOT" }
variable "min_size"         { type = number        default = 0 }
variable "max_size"         { type = number        default = 8 }
variable "desired_size"     { type = number        default = 0 }
variable "capacity_reservation_id" { type = string default = null }
variable "labels"           { type = map(string)   default = {} }
variable "taints" {
  type = list(object({ key = string, value = string, effect = string }))
  default = [{
    key = "nvidia.com/gpu", value = "true", effect = "NO_SCHEDULE"
  }]
}
```

```hcl
# modules/eks-gpu-nodegroup/main.tf
locals {
  name = "${var.cluster_name}-gpu"
  base_labels = {
    "workload"          = "gpu"
    "nvidia.com/gpu"    = "present"
    "node.kubernetes.io/instance-type" = var.instance_types[0]
  }
}

resource "aws_eks_node_group" "this" {
  cluster_name    = var.cluster_name
  node_group_name = local.name
  node_role_arn   = aws_iam_role.this.arn
  subnet_ids      = var.subnet_ids

  ami_type       = "AL2023_x86_64_NVIDIA"
  instance_types = var.instance_types
  capacity_type  = var.capacity_type
  disk_size      = 200          # weights are big

  scaling_config {
    min_size     = var.min_size
    max_size     = var.max_size
    desired_size = var.desired_size
  }

  update_config { max_unavailable_percentage = 25 }

  labels = merge(local.base_labels, var.labels)

  dynamic "taint" {
    for_each = var.taints
    content {
      key    = taint.value.key
      value  = taint.value.value
      effect = taint.value.effect
    }
  }

  dynamic "capacity_reservation_specification" {
    for_each = var.capacity_reservation_id == null ? [] : [1]
    content {
      capacity_reservation_target {
        capacity_reservation_id = var.capacity_reservation_id
      }
    }
  }

  lifecycle { ignore_changes = [scaling_config[0].desired_size] }
  tags = { "k8s.io/cluster-autoscaler/${var.cluster_name}" = "owned" }
}
```

## The IAM piece people forget

The default node role docs forget that GPU nodes pulling weights from S3 also want `s3:GetObject` on the model bucket. Add it explicitly — never via `*`:

```hcl
resource "aws_iam_role_policy" "model_bucket_read" {
  role = aws_iam_role.this.name
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Resource = [
        "arn:aws:s3:::${var.model_bucket}",
        "arn:aws:s3:::${var.model_bucket}/*",
      ]
    }]
  })
}
```

Even better: do this at the **pod** level via [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) and skip it on the node role entirely. The node should know nothing about your weights.

## Using it

```hcl
module "gpu_g5" {
  source = "./modules/eks-gpu-nodegroup"

  cluster_name   = module.eks.cluster_name
  subnet_ids     = module.vpc.private_subnets
  instance_types = ["g5.2xlarge", "g5.4xlarge"]   # let ASG pick
  capacity_type  = "SPOT"
  min_size       = 0
  max_size       = 12
  labels         = { "gpu.class" = "a10g" }
}

module "gpu_p4d_floor" {
  source = "./modules/eks-gpu-nodegroup"

  cluster_name             = module.eks.cluster_name
  subnet_ids               = module.vpc.private_subnets
  instance_types           = ["p4d.24xlarge"]
  capacity_type            = "ON_DEMAND"
  min_size                 = 1
  max_size                 = 2
  desired_size             = 1
  capacity_reservation_id  = aws_ec2_capacity_reservation.p4d.id
  labels                   = { "gpu.class" = "a100" }
}
```

Two pools: one elastic on Spot, one fixed on a [Capacity Reservation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html). Pods that absolutely must run choose `gpu.class: a100` via `nodeSelector`; everything else schedules on the cheap pool first.

## Why managed node groups, not Karpenter, for the floor

Yes, I love [Karpenter](/posts/karpenter-llm-autoscaling/). But:

- Karpenter doesn't *own* a Capacity Reservation — it consumes capacity at request time.
- For your contractually-guaranteed baseline, you want a managed node group with `desired_size = N` and an attached ODCR. That capacity exists whether your software is healthy or not.
- Use Karpenter for the elastic shoulder above the baseline.

## Pitfalls

| Symptom | Cause |
| --- | --- |
| `0/3 nodes are available: 3 Insufficient nvidia.com/gpu` | Device plugin not installed, or AMI is wrong (`AL2_x86_64` instead of `AL2023_x86_64_NVIDIA`) |
| Spot interruptions during model load | Add the [Node Termination Handler](https://github.com/aws/aws-node-termination-handler) and a 120s `terminationGracePeriodSeconds` |
| Disk fills up after a week | Weight cache + container layers. Set `disk_size = 200` minimum, use [`gp3`](https://aws.amazon.com/ebs/general-purpose/) |

## Source

The full module lives at [github.com/gmtreacy/aiops](https://github.com/gmtreacy/aiops) under `examples/eks-gpu-nodegroup/`.
