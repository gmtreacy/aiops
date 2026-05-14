+++
title = "About"
url = "/about/"
summary = "About this site"
hidemeta = true
ShowReadingTime = false
ShowBreadCrumbs = false
disableShare = true
+++

**AI Ops Field Notes** is a working notebook on running open-weights large language models on AWS. The recurring stack:

- **[Amazon EKS](https://aws.amazon.com/eks/)** for the cluster
- **[Karpenter](https://karpenter.sh/)** for GPU node provisioning
- **[vLLM](https://github.com/vllm-project/vllm)** for inference
- **[Terraform](https://www.terraform.io/)** for everything reproducible
- **[Prometheus](https://prometheus.io/)** + **[OpenTelemetry](https://opentelemetry.io/)** + **[CloudWatch](https://aws.amazon.com/cloudwatch/)** for the signals

## House rules

- Posts only ship after they've been run on real infrastructure.
- No marketing copy, no tier lists, no "10 prompts that will revolutionise your stand-up".
- Code blocks compile. Helm values render. Terraform plans.
- If something here is wrong, [open an issue or PR](https://github.com/gmtreacy/aiops). Corrections welcomed; opinions optional.

## Author

George Treacy — platform engineering on the AI/ML side. Reachable on [GitHub](https://github.com/gmtreacy).

## Colophon

Built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Cover art is hand-rolled SVG; no LLM was harmed in its production.
