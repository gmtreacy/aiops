+++
title = "Observability for LLM Workloads"
date = 2026-04-28T09:00:00+10:00
draft = false
summary = "The minimum viable signals — and dashboards — for self-hosted inference. Tokens per second, p95 latency, and dollars per million tokens."
tags = ["observability", "prometheus", "grafana", "vllm", "opentelemetry"]
categories = ["observability"]
series = ["llm-platform"]
ShowToc = true
[cover]
  image = "/img/covers/observability.svg"
  alt = "Observability cover with three time-series lines"
  caption = "Cover · Tokens, latency, cost"
+++

Most monitoring stacks for LLM platforms are inherited from web services: requests, errors, durations. That's necessary but not sufficient. Tokens are not requests, and a p95 latency number that doesn't condition on prompt length is a lie.

## The three signals

| Signal | Where it comes from | What it tells you |
| --- | --- | --- |
| **Tokens/sec per pod** | vLLM `/metrics` | Are we using the GPU? |
| **TTFT and per-token latency** | vLLM `/metrics` + ingress traces | Are users waiting? |
| **$/Million tokens** | Cost Explorer + token counter | Is this affordable? |

## Scraping vLLM

vLLM exposes Prometheus metrics out of the box on `:8000/metrics`. Add a `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm
  namespace: inference
spec:
  selector:
    matchLabels: { app: vllm }
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

The metrics that matter:

- `vllm:num_requests_running` — current batch size
- `vllm:num_requests_waiting` — queue depth (if this is non-zero you need more pods)
- `vllm:gpu_cache_usage_perc` — KV cache pressure; sustained > 0.95 means OOM is one big request away
- `vllm:time_to_first_token_seconds` — TTFT histogram
- `vllm:time_per_output_token_seconds` — generation speed
- `vllm:prompt_tokens_total` / `vllm:generation_tokens_total` — counters for cost math

## A useful PromQL kit

```promql
# tokens per second, per pod, smoothed
sum by (pod) (rate(vllm:generation_tokens_total[1m]))

# p95 TTFT, per model
histogram_quantile(0.95,
  sum by (le, model) (rate(vllm:time_to_first_token_seconds_bucket[5m]))
)

# queue pressure: pods where wait > running for more than 2m
(
  avg_over_time(vllm:num_requests_waiting[2m])
  > avg_over_time(vllm:num_requests_running[2m])
)

# KV cache headroom
1 - max by (pod) (vllm:gpu_cache_usage_perc)
```

That last one is the alert that prevents the most pages.

## Tracing: just enough OpenTelemetry

vLLM doesn't emit traces natively (yet). Put the OTel collector in front of it as a sidecar or run it as an [OTel-aware proxy](https://opentelemetry.io/docs/specs/otel/protocol/) at the Envoy / Ingress layer. You want one trace per **request** with these attributes:

- `gen_ai.system = "vllm"`
- `gen_ai.request.model`
- `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens`
- `gen_ai.response.finish_reason`

Ship to [AWS Distro for OpenTelemetry](https://aws-otel.github.io/) → Tempo / X-Ray. Skip Jaeger; it cannot keep up with serious token volume.

## Dollars per million tokens

This is the metric your CFO wants and the one you should put on the wall. The math is unglamorous:

```text
$/1M tokens = (instance $/hr × pod_count × hours)
              ───────────────────────────────────
              (Σ generation_tokens_total / 1e6)
```

A small Lambda that runs every hour:

```python
# lambda/cost_per_mtok.py
import boto3
import requests

PROM = "http://prometheus.observability:9090"

def query(q):
    r = requests.get(f"{PROM}/api/v1/query", params={"query": q}, timeout=10)
    return float(r.json()["data"]["result"][0]["value"][1])

def handler(_, __):
    tokens = query("sum(increase(vllm:generation_tokens_total[1h]))")
    pods   = query("sum(kube_pod_info{namespace='inference'})")
    rate   = 1.21  # g5.2xlarge $/hr; pull from Pricing API in real life

    cost_per_mtok = (rate * pods) / (tokens / 1e6) if tokens else 0

    boto3.client("cloudwatch").put_metric_data(
        Namespace="aiops",
        MetricData=[{
            "MetricName": "DollarsPerMillionTokens",
            "Value": cost_per_mtok,
            "Unit": "None",
        }],
    )
```

Now you can alert on it: if `$/1M tokens` jumps 30% week-over-week, **something** changed — usually a model upgrade or a regression in batching efficiency.

There's a longer post on the cost side at [$/Million Tokens](/posts/finops-dollars-per-million-tokens/).

## A starter Grafana dashboard

Three rows, six panels:

1. **Capacity** — tokens/sec per pod (timeseries), GPU utilization (gauge)
2. **Quality** — TTFT p50/p95 (timeseries), per-token latency p95 (timeseries)
3. **Cost** — $/1M tokens (stat), spot interruption rate (timeseries)

That's the whole dashboard. Anything beyond is for the deep-dive panel.

## Alerts that earn their keep

| Alert | Threshold |
| --- | --- |
| `vllm:num_requests_waiting > 50` for 5m | Capacity is short — scale or rate-limit |
| `vllm:gpu_cache_usage_perc > 0.95` for 2m | KV pressure — restart imminent |
| `karpenter_pods_startup_time_seconds:p95 > 180s` for 15m | Scale-up is slow — capacity issues |
| `aiops_dollars_per_million_tokens > $1.50` | Anomaly — investigate before next bill |

Skip the rest until they bite you.

## References

- [vLLM metrics docs](https://docs.vllm.ai/en/latest/serving/metrics.html)
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [AWS Managed Prometheus / Grafana](https://aws.amazon.com/prometheus/)
