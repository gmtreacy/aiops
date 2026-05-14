+++
title = "Welcome to AI Ops Field Notes"
date = 2026-05-14T08:00:00+10:00
draft = false
summary = "What this site is, who it's for, and the rough territory it covers — running LLM workloads on AWS without losing the plot."
tags = ["meta", "introduction"]
categories = ["meta"]
weight = 1
[cover]
  image = "/img/covers/welcome.svg"
  alt = "Terminal-styled cover image with the title 'AI Ops Field Notes' and a fake kubectl get gpu output"
  caption = "Cover · welcome"
  relative = false
+++

This is a working notebook for one specific intersection: **large language models** and **AWS-native Kubernetes**. The pieces that show up over and over here are:

- [Amazon EKS](https://aws.amazon.com/eks/) for the cluster
- [Karpenter](https://karpenter.sh/) for GPU node provisioning
- [vLLM](https://github.com/vllm-project/vllm) for inference
- [Terraform](https://www.terraform.io/) / [Terragrunt](https://terragrunt.gruntwork.io/) for everything reproducible
- [Prometheus](https://prometheus.io/), [OpenTelemetry](https://opentelemetry.io/), and [CloudWatch](https://aws.amazon.com/cloudwatch/) for the signals

## Who this is for

Engineers who already know how to read a `kubectl describe pod` and now have a model-serving SLO on their plate. We will not stop to explain what a Pod is.

## What it isn't

A vendor blog, a benchmark leaderboard, or anything written by an LLM with the temperature cranked up. Posts here only ship after they've been run on real infrastructure.

## Getting around

| Section | What's there |
| --- | --- |
| [Posts](/posts/) | Every article in reverse-chronological order |
| [Categories](/categories/) | Larger themes — `infrastructure`, `inference`, `observability`, `finops` |
| [Tags](/tags/) | Specific tools — `eks`, `vllm`, `karpenter`, `terraform`, etc. |
| [Archive](/archives/) | Year-grouped index |

If something here is wrong, [open an issue or PR](https://github.com/gmtreacy/aiops). Corrections welcomed; opinions optional.
