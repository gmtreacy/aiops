+++
title = "Karpenter for LLM Autoscaling"
date = 2026-05-02T09:00:00+10:00
draft = false
summary = "A NodePool spec that brings GPU capacity online only when the queue demands it — and tears it down the moment the queue empties."
tags = ["karpenter", "eks", "kubernetes", "gpu", "autoscaling"]
categories = ["infrastructure"]
series = ["llm-platform"]
ShowToc = true
[cover]
  image = "/img/covers/karpenter-llm.svg"
  alt = "Karpenter autoscaling cover"
  caption = "Cover · Karpenter for LLMs"
+++

The cluster autoscaler is fine for CPU. For GPU it's a liability — you'll pay for an idle `p4d` for 10 minutes after the last request completes. [Karpenter](https://karpenter.sh/) does this better, and the LLM workload is exactly the shape it's good at: bursty, expensive, and indifferent to which instance type runs it as long as the GPU is there.

## NodePool: the elastic GPU pool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-spot
spec:
  template:
    metadata:
      labels:
        workload: gpu
        gpu.class: a10g
    spec:
      taints:
        - key: nvidia.com/gpu
          value: "true"
          effect: NoSchedule
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]      # spot first, on-demand fallback
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["g5.2xlarge", "g5.4xlarge", "g6.2xlarge", "g6.4xlarge"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-southeast-2a", "ap-southeast-2b", "ap-southeast-2c"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind:  EC2NodeClass
        name:  gpu
      expireAfter: 720h     # rotate weekly to pick up AMI updates
  limits:
    cpu: "1000"
    nvidia.com/gpu: "32"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
```

Two things to underline:

1. **`consolidationPolicy: WhenEmpty`**, not `WhenUnderutilized`. Underutilized consolidation will move your in-flight inference pod off its node mid-stream. WhenEmpty waits until the node has zero workloads.
2. **`consolidateAfter: 30s`**. With expensive GPUs you do not want a 5-minute idle tail.

## EC2NodeClass: the AMI choice

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: gpu
spec:
  amiFamily: AL2023
  amiSelectorTerms:
    - alias: al2023@v20260301              # pin; bump deliberately
  role: "KarpenterNodeRole-prod"
  subnetSelectorTerms:
    - tags: { "karpenter.sh/discovery": "prod" }
  securityGroupSelectorTerms:
    - tags: { "karpenter.sh/discovery": "prod" }
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 200Gi
        volumeType: gp3
        iops: 6000
        throughput: 250
        encrypted: true
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh prod \
      --container-runtime containerd \
      --kubelet-extra-args '--node-labels=nvidia.com/gpu=present'
```

The 200 GB disk is not optional — Llama-3-70B weights are ~140 GB on their own.

## Telling pods to use it

```yaml
spec:
  template:
    spec:
      nodeSelector:
        gpu.class: a10g
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: vllm
          resources:
            requests:  { nvidia.com/gpu: 1 }
            limits:    { nvidia.com/gpu: 1 }
```

The `nodeSelector` is what gives Karpenter permission to pick a GPU instance. Without it Karpenter sees a pod that "fits" on a CPU node and will not provision a GPU.

## What "good" looks like

Watch these in [Grafana](/posts/observability-llm-workloads/):

```promql
# how long pods wait for capacity
histogram_quantile(0.95,
  sum by (le) (rate(karpenter_pods_startup_time_seconds_bucket[5m]))
)

# how often nodes get consolidated
sum(rate(karpenter_nodes_terminated_total{reason="consolidation"}[1h]))

# spot interruptions
sum(rate(karpenter_interruption_actions_performed_total[1h]))
```

For a healthy LLM cluster: p95 startup ~60-90s (the GPU AMI takes a minute to come up), consolidations happen on every traffic trough, interruptions stay under 1/hr per 10 nodes.

## Spot, ODCR, and the floor

Run **two NodePools**:

- `gpu-spot` — the one above. `min = 0`, scales freely.
- `gpu-od-floor` — a small NodePool with `on-demand` only and a `topologySpreadConstraint` to keep at least one node alive across zones, for traffic you absolutely cannot drop.

Even better: combine `gpu-od-floor` with an [On-Demand Capacity Reservation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html) so you're paying for guaranteed capacity, not best-effort.

## Common failure modes

| Symptom | Cause |
| --- | --- |
| Pods stuck `Pending` despite NodePool with capacity | Missing toleration for `nvidia.com/gpu` taint |
| Karpenter provisions CPU node for a GPU pod | Pod missing `nvidia.com/gpu` resource request — it's the trigger |
| New nodes come up but pods still pending | NVIDIA device plugin DaemonSet doesn't tolerate the GPU taint |
| Spot churn breaks long requests | Use Spot for stateless inference; pin batch / fine-tune jobs to On-Demand |

## Reading list

- [Karpenter NodePool reference](https://karpenter.sh/docs/concepts/nodepools/)
- [Disruption budgets](https://karpenter.sh/docs/concepts/disruption/) — useful when you're ready
- [AWS blog: Karpenter for ML workloads](https://aws.amazon.com/blogs/containers/optimizing-deep-learning-workloads-on-amazon-eks-using-karpenter/)
