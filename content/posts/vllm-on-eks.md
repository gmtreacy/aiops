+++
title = "vLLM on EKS, Properly Tuned"
date = 2026-05-08T09:00:00+10:00
draft = false
summary = "Helm values, GPU memory tuning, and the half-dozen knobs that decide whether your tokens-per-second number is real or aspirational."
tags = ["vllm", "eks", "kubernetes", "gpu", "inference"]
categories = ["inference"]
series = ["llm-platform"]
ShowToc = true
[cover]
  image = "/img/covers/vllm-on-eks.svg"
  alt = "vLLM on EKS cover with a Helm install command"
  caption = "Cover · vLLM on EKS"
+++

[vLLM](https://github.com/vllm-project/vllm) is the right default for self-hosted LLM serving in 2026. The first time you deploy it you will get something that "works" and uses 30% of the GPU. This post is how to get the other 70%.

## Prerequisites

- An EKS cluster with a GPU node group (see [Terraform GPU node group](/posts/terraform-eks-gpu-nodegroup/))
- The [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) installed
- [`nvidia.com/gpu`](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/) showing up under `kubectl describe node`

If `kubectl get pods -n kube-system | grep nvidia` is empty, stop and fix that first. Nothing else will work.

## The minimum viable Helm values

```yaml
# values.yaml
image:
  repository: vllm/vllm-openai
  tag: v0.6.3

model: meta-llama/Llama-3.1-8B-Instruct

resources:
  limits:
    nvidia.com/gpu: 1
    memory: 32Gi
  requests:
    nvidia.com/gpu: 1
    memory: 24Gi

extraArgs:
  - --gpu-memory-utilization=0.92
  - --max-model-len=8192
  - --max-num-seqs=128
  - --enable-prefix-caching
  - --kv-cache-dtype=fp8
```

Apply with:

```bash
helm repo add vllm https://vllm-project.github.io/vllm
helm upgrade --install vllm vllm/vllm -n inference -f values.yaml
```

## The knobs that actually matter

### `--gpu-memory-utilization`

The default is `0.9`. On a clean A10G with no other tenants you can push to **0.92**. Don't go higher: vLLM needs scratch space for activations and the OOM killer is a poor diagnostic tool.

### `--max-model-len`

Set this to the **longest prompt + response you actually serve**, not the model's max. An 8B model nominally supports 128K but the KV cache for that is *enormous* — it costs you batch slots. If 95% of traffic is < 8K, set 8192 and refuse the rest.

### `--max-num-seqs`

The continuous-batching ceiling. Higher = more concurrency, lower per-request latency variance, more memory pressure. Start at 128, watch `vllm:num_requests_running` in [Prometheus](/posts/observability-llm-workloads/), nudge until p95 latency starts to climb.

### `--enable-prefix-caching`

Free 2-5× speedup if your traffic has shared prefixes (system prompts, RAG context). Off by default. Turn it on.

### `--kv-cache-dtype=fp8`

Halves KV cache memory at the cost of about 0.5% on quality benchmarks. On Hopper / Ada this is essentially free. On Ampere, benchmark first.

## Tensor parallelism: when, not whether

For 70B-class models on a single `p4d.24xlarge`:

```yaml
extraArgs:
  - --tensor-parallel-size=8
```

Rules:

1. `tensor-parallel-size` must divide the model's attention head count.
2. All ranks must be on the **same node** unless you have NVLink fabric. Cross-node tensor parallel over the EKS pod network is a recipe for sadness.
3. Use `topology.kubernetes.io/zone` and a `podAntiAffinity` to keep one TP group per node.

## The pod spec, with the bits people forget

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama3-8b
spec:
  replicas: 2
  template:
    spec:
      runtimeClassName: nvidia
      terminationGracePeriodSeconds: 120  # let in-flight requests drain
      containers:
        - name: vllm
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 90        # weights take a minute to load
            periodSeconds: 5
          startupProbe:
            httpGet: { path: /health, port: 8000 }
            failureThreshold: 60           # don't kill it during model load
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "30"]   # give the LB time to deregister
```

The startup probe is the one people skip. Without it Kubernetes will kill a 70B pod halfway through loading because the readiness probe failed for the 4th time.

## Verifying it's actually fast

```bash
kubectl port-forward -n inference svc/vllm 8000:8000

curl -s http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.1-8B-Instruct",
    "prompt": "Write a haiku about Karpenter.",
    "max_tokens": 64,
    "stream": false
  }' | jq '.usage'
```

Then look at the `/metrics` endpoint and confirm `vllm:gpu_cache_usage_perc` sits around 0.7-0.85 under load. If it's 0.2 you're not actually batching.

## Further reading

- [vLLM production deployment guide](https://docs.vllm.ai/en/latest/serving/deploying_with_k8s.html)
- [PagedAttention paper](https://arxiv.org/abs/2309.06180)
- [Continuous batching explained](https://www.anyscale.com/blog/continuous-batching-llm-inference)
