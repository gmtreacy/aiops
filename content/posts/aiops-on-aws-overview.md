+++
title = "An LLM Platform on AWS, End to End"
date = 2026-05-10T09:00:00+10:00
draft = false
summary = "A reference architecture for hosting open-weights models on AWS — from edge to GPU — and the boring decisions that decide whether it actually works."
tags = ["aws", "eks", "architecture", "vllm", "karpenter"]
categories = ["infrastructure"]
series = ["llm-platform"]
ShowToc = true
TocOpen = true
[cover]
  image = "/img/covers/aiops-overview.svg"
  alt = "Reference architecture diagram cover"
  caption = "Cover · AI Ops on AWS"
+++

Most "LLM platform" diagrams hand-wave the parts that matter. This one tries not to.

## The shape of it

```text
   ┌─────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
   │  Client │ ─▶ │ ALB / API GW │ ─▶ │ EKS Ingress  │ ─▶ │  vLLM Pods  │
   └─────────┘    └──────────────┘    └──────────────┘    └─────────────┘
                       │                     │                   │
                       ▼                     ▼                   ▼
                  WAF + Cognito        Karpenter scales       g5/p4d/p5
                  rate limit + JWT      GPU NodePools        SPOT + ODCR
                       │                     │                   │
                       └────────── OpenTelemetry / Prometheus ───┘
                                            │
                                            ▼
                                  Managed Grafana + S3 logs
```

Everything that follows is a justification or a regret about one of those boxes.

## Edge: ALB vs API Gateway

For LLM traffic you almost always want **ALB + WAF**, not API Gateway. Three reasons:

1. **Streaming**: vLLM speaks SSE / chunked transfer. ALB streams it; API Gateway buffers up to its hard timeout.
2. **Long tails**: a single 70B-class request can run 60+ seconds. API Gateway's 29s integration timeout will quietly murder it.
3. **Cost**: at 1k req/s the per-request charge on API Gateway dwarfs the LB hour.

If you need [WebSockets](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html), API Gateway is back on the table — but most chat clients still get more done with SSE.

## Cluster: one EKS, multiple node groups

```hcl
node_groups = {
  system  = { instance_types = ["m6i.large"],   min = 2, max = 4  }
  cpu     = { instance_types = ["m6i.2xlarge"], min = 2, max = 20 }
  gpu_g5  = { instance_types = ["g5.2xlarge"],  min = 0, max = 12, capacity_type = "SPOT" }
  gpu_p4d = { instance_types = ["p4d.24xlarge"],min = 0, max = 2,  capacity_type = "ON_DEMAND" }
}
```

The system pool runs the cluster's nervous system: CoreDNS, Karpenter itself, the AWS Load Balancer Controller, [`cert-manager`](https://cert-manager.io/), [`external-secrets`](https://external-secrets.io/). Don't let Karpenter manage these — circular dependency.

## Inference: vLLM, not "roll-your-own Transformers"

I have seen at least four teams write their own batching server because it "looked simple." None of them shipped. vLLM gets you:

- [PagedAttention](https://blog.vllm.ai/2023/06/20/vllm.html) — KV cache that doesn't fragment
- Continuous batching — fills the GPU instead of starving it between requests
- An OpenAI-compatible API — your client SDKs Just Work

There's a deeper post on this at [vLLM on EKS](/posts/vllm-on-eks/).

## GPUs: the only line item that matters

Pricing is from [AWS On-Demand Pricing](https://aws.amazon.com/ec2/pricing/on-demand/), as of writing — verify before you quote anyone.

| Instance | GPU | VRAM | $/hr (on-demand) | Use it for |
| --- | --- | --- | --- | --- |
| `g5.2xlarge` | 1× A10G | 24 GB | ~$1.21 | 7-13B serving |
| `g6.12xlarge` | 4× L4 | 96 GB | ~$4.92 | mid-size + multi-tenant |
| `p4d.24xlarge` | 8× A100 | 320 GB | ~$32.77 | 70B serving / fine-tune |
| `p5.48xlarge` | 8× H100 | 640 GB | ~$98 | frontier serving |

Mix **Spot** for elastic serving and an **[On-Demand Capacity Reservation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)** for the floor you cannot lose. [Karpenter](/posts/karpenter-llm-autoscaling/) handles the rest.

## Identity: IRSA or pod identity, never node IAM

Bedrock, S3, KMS — every one of those calls should be made from the pod's own role, never the node's. Use [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) on new clusters; [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) on older ones. Pod Identity is faster and avoids OIDC trust-policy gymnastics.

## Observability: three signals

You only need three to start:

- **Tokens/sec per pod** (from vLLM's `/metrics`) — capacity
- **End-to-end p95** (from your Envoy / ingress) — quality
- **$/1M tokens** (CloudWatch + Cost Explorer + a small lambda) — sanity

More on that in [Observability for LLM workloads](/posts/observability-llm-workloads/).

## What this series will cover

- [vLLM on EKS, properly tuned](/posts/vllm-on-eks/)
- [Terraform module for GPU node groups](/posts/terraform-eks-gpu-nodegroup/)
- [Karpenter for LLM autoscaling](/posts/karpenter-llm-autoscaling/)
- [Observability for LLM workloads](/posts/observability-llm-workloads/)
- [The $/Million-Tokens metric](/posts/finops-dollars-per-million-tokens/)

If a box in the diagram is missing a post, that's the backlog.
