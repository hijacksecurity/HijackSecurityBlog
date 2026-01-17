---
layout: post
title: "1.0 EKS Infrastructure Series: Building Production Kubernetes on AWS"
date: 2025-08-29 10:00:00 -0000
categories: kubernetes eks infrastructure aws
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes"]
series: "EKS Infrastructure Series"
series_part: "1.0"
---

Setting up a production-ready Kubernetes cluster on AWS isn't complicated, but it does require getting several pieces right. This series walks through building EKS infrastructure that you can actually deploy applications to - from container registry to ingress, storage, and secrets management.

## What This Series Covers

We're building a complete EKS environment with these components:

**Phase 1: Infrastructure Foundation**

| Part | Component | Purpose |
|------|-----------|---------|
| 1.1 | [ECR Setup](/2025/08/29/eks-ecr-container-registry/) | Container registry for storing Docker images |
| 1.2 | EKS Base Cluster | Kubernetes control plane with auto-scaling nodes |
| 1.3 | Ingress Controller | SSL termination and load balancing with ALB |
| 1.4 | EBS Storage | Persistent volumes for stateful workloads |
| 1.5 | Pod Identity | Secure AWS API access without stored credentials |
| 1.6 | External Secrets | Automated secret sync from AWS Secrets Manager |

Each article is self-contained with working examples. You can follow along sequentially or jump to specific topics.

## The Goal

By the end of this series, you'll have infrastructure that:

- Stores and scans container images in ECR
- Runs a cost-optimized EKS cluster with spot instances
- Exposes services via HTTPS with automatic certificate management
- Provides persistent storage for databases and stateful apps
- Accesses AWS services securely without storing credentials
- Syncs secrets automatically from AWS Secrets Manager

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

## Why This Order?

We start with ECR before the cluster because:

1. **You need somewhere to store images** - Before deploying anything to Kubernetes, you need container images. ECR gives you a private registry integrated with AWS IAM.

2. **Security scanning from the start** - ECR's built-in vulnerability scanning means you're checking images before they ever reach your cluster.

3. **IAM integration** - ECR authentication uses the same IAM patterns we'll use throughout the series, so you learn the approach early.

## Design Principles

A few guiding principles for this infrastructure:

**Keep it simple** - Every component should solve a specific problem. No over-engineering for hypothetical future needs.

**Security by default** - Private networking, no stored credentials, least-privilege IAM policies. These aren't add-ons; they're the baseline.

**Cost awareness** - Spot instances, right-sized resources, and clear cost breakdowns. Production-ready doesn't mean expensive.

**Maintainable** - Scripts and configurations you can understand, modify, and debug. No magic.

## Let's Get Started

The first article covers setting up Amazon ECR - your private container registry. We'll configure image scanning, lifecycle policies, and the IAM permissions needed to push and pull images.

**Next**: [1.1 ECR: Container Registry Setup](/2025/08/29/eks-ecr-container-registry/)

---

*This series is a work in progress. Articles will be published as they're completed.*
