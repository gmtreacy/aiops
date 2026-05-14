+++
title = "FinOps for LLMs: $/Million Tokens or Bust"
date = 2026-04-22T09:00:00+10:00
draft = false
summary = "How to actually compare self-hosting vs Bedrock vs OpenAI without lying to yourself. The unit metric is dollars per million tokens — everything else is vanity."
tags = ["finops", "aws", "bedrock", "vllm", "cost"]
categories = ["finops"]
series = ["llm-platform"]
ShowToc = true
[cover]
  image = "/img/covers/finops.svg"
  alt = "FinOps cover with a price comparison table"
  caption = "Cover · $/Million Tokens"
+++

A team will spend two weeks debating model choice and forty seconds on cost. This post tries to invert that.

## The only number that matters

```text
$/1M tokens = total infrastructure cost (in window)
              ──────────────────────────────────────
              total tokens generated (in window)
```

Not requests. Not pods. **Tokens.** Anything else makes self-hosted look cheap when it isn't, or expensive when it isn't.

## Where the numbers come from

| Component | Source |
| --- | --- |
| Instance $/hr | [AWS On-Demand Pricing](https://aws.amazon.com/ec2/pricing/on-demand/) or [Pricing API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-pelong.html) |
| Spot discount | [Spot Pricing History](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances-history.html) |
| Pod-hours | Prometheus `kube_pod_info` integrated over time |
| Tokens generated | vLLM `vllm:generation_tokens_total` counter |
| Bedrock cost | `bedrock-runtime` invocation metrics + [Bedrock Pricing](https://aws.amazon.com/bedrock/pricing/) |

Use the Pricing API in production. Hard-coded `$/hr` constants will bite you within a quarter.

## A worked comparison

Same workload — 5 RPS of 2K-prompt / 500-token responses (~12.5M tokens/hr generated). All numbers below are illustrative; verify with current pricing.

| Approach | Hardware / model | $/hr | tok/s | $/1M tokens |
| --- | --- | --- | --- | --- |
| Self-host Llama-3-8B | 1× `g5.2xlarge` Spot | ~$0.60 | 3,200 | **~$0.05** |
| Self-host Llama-3-8B | 1× `g5.2xlarge` On-Demand | ~$1.21 | 3,200 | ~$0.10 |
| Self-host Llama-3-70B | 1× `p4d.24xlarge` ODCR | ~$32.77 | 9,800 | ~$0.93 |
| Bedrock — Llama-3-8B | n/a | per-token | n/a | ~$0.30 |
| Bedrock — Claude Haiku | n/a | per-token | n/a | ~$0.25 |
| Bedrock — Claude Sonnet | n/a | per-token | n/a | ~$3.00 |
| OpenAI — GPT-4o-mini | n/a | per-token | n/a | ~$0.60 |

Two reads from this table:

1. Self-hosting an 8B on a saturated `g5` is **5-6× cheaper** than the equivalent managed model. **If** it's saturated.
2. Self-hosting a 70B on a `p4d` is **only** ~3× cheaper than Sonnet, and Sonnet is a measurably better model. The math swings hard at this size.

## Where self-hosting goes wrong

It's almost always one of three things:

### 1. The GPU isn't saturated

A `g5.2xlarge` running at 20% utilisation has the same $/hr as one at 95%, and 4× the $/1M tokens. Continuous batching ([covered here](/posts/vllm-on-eks/)) is non-negotiable, and you must either:

- run multiple tenants on one pod (`--max-num-seqs` cranked up), or
- scale pods down aggressively when traffic dips ([Karpenter](/posts/karpenter-llm-autoscaling/) helps).

### 2. Idle capacity for "just in case"

Three replicas "for HA" on a model getting 0.5 RPS at 03:00 will demolish your unit cost. Use a Spot pool with `min = 0` and a small On-Demand floor of one or two pods.

### 3. Engineer-hours

A team of two SREs costs ~$400k/year fully loaded. That's $46/hr — i.e., the cost of running ~38 idle `g5.2xlarge` instances around the clock. If the platform takes 25% of two engineers to keep upright, you owe that to the cost model.

A useful rule of thumb: **self-hosting is cheaper than Bedrock above roughly 50M tokens/day of sustained traffic on a given model.** Below that, the engineering tax dominates.

## Tagging that survives a finance audit

Tag every GPU resource so Cost Explorer can split it cleanly:

```hcl
tags = {
  "cost-center"  = "ai-platform"
  "workload"     = "inference"
  "model"        = "llama-3-8b"
  "tenant"       = "internal-search"
  "environment"  = "prod"
}
```

For Bedrock, use the [Application Inference Profile](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html) feature to tag invocations the same way. Then the cost-per-tenant and cost-per-model dashboards are one Cost Explorer query away.

## Putting `$/1M tokens` on the wall

The implementation lives in [Observability for LLM workloads](/posts/observability-llm-workloads/). The short version:

1. Lambda runs hourly.
2. Pulls Prometheus tokens, Pricing API rates, pod-hours.
3. Writes a CloudWatch metric `aiops/DollarsPerMillionTokens`.
4. Grafana panel with WoW delta, alert at +30%.

Once it's on a screen, decisions get faster: a PR that drops tok/s by 8% is no longer "weird perf regression" — it's a $7,000/month bill.

## Bedrock isn't the enemy

A short defense of the managed path: for low-volume, spiky, or compliance-sensitive workloads, **the cheaper $/1M tokens of self-hosting is wiped out by the cost of one outage you should not have had**. Use Bedrock as the floor and self-host where the volume actually justifies it. Most platforms end up with a mix.

## Reading list

- [AWS Bedrock pricing](https://aws.amazon.com/bedrock/pricing/)
- [AWS Pricing API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-pelong.html)
- [Cost Explorer with Resource Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/aws-cost-explorer-what-is.html)
- [FinOps Foundation — AI cost management](https://www.finops.org/)
