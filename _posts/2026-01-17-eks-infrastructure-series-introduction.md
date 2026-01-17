---
layout: post
title: "1.0 EKS Infrastructure Series: Building Production Kubernetes on AWS"
date: 2026-01-17 10:00:00 -0000
categories: kubernetes eks infrastructure aws
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes"]
series: "EKS Infrastructure Series"
series_part: "1.0"
---

Setting up a production-ready Kubernetes cluster on AWS isn't complicated, but it does require getting several pieces right. This series walks through building EKS infrastructure that you can actually deploy applications to - from container registry to ingress, storage, secrets management, and monitoring.

## The Architecture

Here's what we're building:

<div class="mermaid">
graph TB
    DEV[Developer] -->|Push images| ECR[ECR<br/>1.1]

    subgraph EKS Cluster - 1.2
        ALB[ALB<br/>1.3] --> SVC[Services]
        SVC --> PODS[Pods]
        PODS --> PI[Pod Identity<br/>1.5]
        PODS --> ES[External Secrets<br/>1.6]
        PODS --> EBS[EBS Volumes<br/>1.4]
        CW[CloudWatch<br/>1.7]
    end

    INT[Internet] --> ALB
    ECR -->|Pull images| PODS
    PI --> AWS[AWS APIs]
    ES --> SM[Secrets Manager]
    PODS -.->|Metrics & Logs| CW
</div>

Each component integrates with AWS services using IAM - no stored credentials, no external authentication systems.

## What This Series Covers

| Part | Component | Purpose |
|------|-----------|---------|
| 1.1 | [ECR Setup](/2026/01/17/eks-ecr-container-registry/) | Container registry for storing Docker images |
| 1.2 | [EKS Base Cluster](/2026/01/17/eks-base-cluster/) | Kubernetes control plane with auto-scaling nodes |
| 1.3 | [Ingress Controller](/2026/01/17/eks-ingress-controller/) | SSL termination and load balancing with ALB |
| 1.4 | [EBS Storage](/2026/01/17/eks-ebs-storage/) | Persistent volumes for stateful workloads |
| 1.5 | [Pod Identity](/2026/01/17/eks-pod-identity/) | Secure AWS API access without stored credentials |
| 1.6 | [External Secrets](/2026/01/17/eks-external-secrets/) | Automated secret sync from AWS Secrets Manager |
| 1.7 | [CloudWatch Monitoring](/2026/01/17/eks-cloudwatch-monitoring/) | Container Insights dashboards and log collection |
| 1.8 | [Production Architecture](/2026/01/17/eks-production-architecture/) | Operations guide, production checklist, architecture patterns |

Each article is self-contained with working examples. You can follow along sequentially or jump to specific topics.

## The Goal

By the end of this series, you'll have infrastructure that:

- Stores and scans container images in ECR
- Runs a cost-optimized EKS cluster with spot instances
- Exposes services via HTTPS with automatic certificate management
- Provides persistent storage for databases and stateful apps
- Accesses AWS services securely without stored credentials
- Syncs secrets automatically from AWS Secrets Manager
- Monitors cluster health with dashboards and logs
- Follows proven architecture patterns for different workload types

## How Components Connect

Understanding how these pieces fit together helps when debugging issues:

**Image Flow**: You build images locally → push to ECR → EKS pulls from ECR when deploying pods.

**Traffic Flow**: Internet → Route 53 → ALB (created by Ingress Controller) → Kubernetes Service → Pods.

**AWS Access Flow**: Pod → Pod Identity Agent → STS AssumeRole → AWS API (S3, DynamoDB, etc.).

**Secrets Flow**: AWS Secrets Manager → External Secrets Operator → Kubernetes Secret → Pod (mounted as file or env var).

**Storage Flow**: Pod requests PVC → EBS CSI Driver creates EBS volume → Volume attached to node → Mounted in pod.

**Monitoring Flow**: CloudWatch Agent (DaemonSet) collects metrics → CloudWatch. Fluent Bit collects logs → CloudWatch Logs.

## Prerequisites

Before starting, you'll need:

```bash
# Required tools
aws --version        # AWS CLI v2
eksctl version       # eksctl for EKS management
kubectl version      # Kubernetes CLI
helm version         # Helm v3 for package management
docker --version     # Docker for building images
```

You'll also need:
- An AWS account with appropriate permissions
- A domain name (for SSL certificates)
- Basic familiarity with Kubernetes concepts

## Understanding EKS Add-ons

EKS has an add-on system that simplifies installing and managing common components. Some add-ons are AWS-managed (installed via `aws eks create-addon` or `eksctl`), while others are community tools installed via Helm.

**What we use in this series:**

| Component | Type | Install Method |
|-----------|------|----------------|
| VPC CNI | EKS Add-on | Pre-installed with cluster |
| CoreDNS | EKS Add-on | Pre-installed with cluster |
| kube-proxy | EKS Add-on | Pre-installed with cluster |
| EBS CSI Driver | EKS Add-on | `eksctl create addon` |
| Pod Identity Agent | EKS Add-on | `eksctl create addon` |
| CloudWatch Observability | EKS Add-on | `aws eks create-addon` |
| AWS Load Balancer Controller | Helm | `helm install` |
| External Secrets Operator | Helm | `helm install` |

**Available EKS add-ons not covered** (for future exploration):

| Add-on | Purpose | Install |
|--------|---------|---------|
| **Karpenter** | Intelligent node provisioning and scaling | Helm chart |
| **AWS Distro for OpenTelemetry** | Metrics and traces collection | EKS Add-on |
| **Mountpoint for S3 CSI** | Mount S3 buckets as file systems | EKS Add-on |
| **EFS CSI Driver** | Shared persistent storage (NFS-like) | EKS Add-on |
| **Secrets Store CSI Driver** | Mount secrets as volumes | EKS Add-on |
| **CoreDNS (advanced)** | Custom DNS configuration | EKS Add-on |
| **Calico** | Network policies | Helm chart |
| **Metrics Server** | Pod resource metrics for HPA | Helm chart |

List all available EKS add-ons:
```bash
aws eks describe-addon-versions --query 'addons[*].addonName' --output table
```

## Why This Order?

We start with ECR before the cluster because:

1. **You need somewhere to store images** - Before deploying anything to Kubernetes, you need container images. ECR gives you a private registry integrated with AWS IAM.

2. **Security scanning from the start** - ECR's built-in vulnerability scanning means you're checking images before they ever reach your cluster.

3. **IAM integration** - ECR authentication uses the same IAM patterns we'll use throughout the series, so you learn the approach early.

Then we build outward:

- **Cluster (1.2)** - The foundation
- **Ingress (1.3)** - How traffic gets in
- **Storage (1.4)** - Where data persists
- **Pod Identity (1.5)** - How pods access AWS
- **External Secrets (1.6)** - How secrets get into pods
- **Monitoring (1.7)** - How you see what's happening
- **Architecture (1.8)** - How to use it all effectively

## Design Principles

A few guiding principles for this infrastructure:

**Keep it simple** - Every component should solve a specific problem. No over-engineering for hypothetical future needs.

**Security by default** - Private networking, no stored credentials, least-privilege IAM policies. These aren't add-ons; they're the baseline.

**Cost awareness** - Spot instances, right-sized resources, and clear cost breakdowns. Production-ready doesn't mean expensive.

**Maintainable** - Scripts and configurations you can understand, modify, and debug. No magic.

## Estimated Costs

Running this infrastructure costs approximately:

| Component | Monthly Cost (estimate) |
|-----------|------------------------|
| EKS Control Plane | $73 |
| EC2 Nodes (2x t3.medium spot) | ~$15-25 |
| ALB | ~$20-30 |
| EBS Storage (50GB gp3) | ~$4 |
| CloudWatch (logs + metrics) | ~$15-30 |
| ECR | ~$1-5 |
| **Total** | **~$130-170/month** |

Costs vary based on traffic, log volume, and storage usage. The series includes tips for cost optimization.

## Let's Get Started

The first article covers setting up Amazon ECR - your private container registry. We'll configure image scanning, lifecycle policies, and the IAM permissions needed to push and pull images.

**Next**: [1.1 ECR: Container Registry Setup](/2026/01/17/eks-ecr-container-registry/)

---

*Questions about this series? Reach out on socials.*
