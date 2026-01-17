---
layout: post
title: "1.8 EKS: Production Architecture and Operations Guide"
date: 2026-01-17 18:00:00 -0000
categories: kubernetes eks infrastructure aws architecture
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS", "Architecture"]
series: "EKS Infrastructure Series"
series_part: "1.8"
---

You've built the infrastructure. Now let's talk about using it properly - the operational commands you'll run daily, getting production-ready, and most importantly, how to architect applications that take full advantage of what you've built.

This concluding article ties everything together. We'll cover the commands you need, the checklist before going live, and architectural patterns for different application types.

## Quick Reference: Daily Operations

These are the commands you'll use regularly. Bookmark this section.

### Cluster Health

```bash
# Node status and resource usage
kubectl get nodes
kubectl top nodes

# Pod status across all namespaces
kubectl get pods -A

# Events (recent cluster activity)
kubectl get events --sort-by='.lastTimestamp' | tail -20

# Resource quotas and limits
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Deployments

```bash
# Rollout status
kubectl rollout status deployment/my-app

# Rollback a deployment
kubectl rollout undo deployment/my-app

# Scale a deployment
kubectl scale deployment/my-app --replicas=5

# Restart pods (rolling)
kubectl rollout restart deployment/my-app
```

### Debugging

```bash
# Pod logs
kubectl logs -f deployment/my-app

# Previous container logs (after crash)
kubectl logs pod/my-app-xxx --previous

# Exec into a pod
kubectl exec -it pod/my-app-xxx -- /bin/sh

# Describe pod (events, status, conditions)
kubectl describe pod/my-app-xxx
```

### Secrets and Config

```bash
# List external secrets and sync status
kubectl get externalsecrets -A

# Force secret refresh
kubectl annotate externalsecret my-secret force-sync=$(date +%s) --overwrite

# View secret (base64 decoded)
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

### Images and Registry

```bash
# List ECR images
aws ecr describe-images --repository-name my-app --query 'imageDetails[*].[imageTags,imagePushedAt]'

# Get ECR login token
aws ecr get-login-password | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

# Check image vulnerabilities
aws ecr describe-image-scan-findings --repository-name my-app --image-id imageTag=latest
```

### Monitoring

```bash
# Pod resource usage
kubectl top pods

# CloudWatch log groups for the cluster
aws logs describe-log-groups --log-group-name-prefix /aws/containerinsights/my-cluster

# Recent logs from CloudWatch
aws logs filter-log-events \
    --log-group-name /aws/containerinsights/my-cluster/application \
    --start-time $(($(date +%s) - 3600))000 \
    --filter-pattern "ERROR"
```

### Storage

```bash
# List persistent volumes and claims
kubectl get pv,pvc -A

# Check EBS CSI driver status
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

## Production Readiness Checklist

Before serving traffic, verify each item. This assumes you're cloning your test infrastructure scripts for production - same patterns, different cluster name and values.

### Infrastructure

| Item | How to Verify | Why It Matters |
|------|---------------|----------------|
| **Multi-AZ nodes** | `kubectl get nodes -o wide` shows nodes in different AZs | Survives AZ outages |
| **Node auto-scaling** | Check managed node group min/max in AWS Console | Handles traffic spikes |
| **Private subnets** | Nodes have private IPs only | Reduces attack surface |
| **ECR scanning enabled** | `aws ecr describe-repositories` shows `scanOnPush: true` | Catches vulnerabilities before deployment |

### Security

| Item | How to Verify | Why It Matters |
|------|---------------|----------------|
| **Pod Identity working** | Pods can call AWS APIs without stored credentials | No leaked keys |
| **Secrets in Secrets Manager** | No secrets in ConfigMaps, env vars, or git | Centralized, auditable secrets |
| **HTTPS only** | Ingress redirects HTTP to HTTPS | Encrypted traffic |
| **Network policies** | `kubectl get networkpolicies -A` | Limits pod-to-pod traffic |
| **IAM least privilege** | Review IAM policies for overly broad access | Blast radius reduction |

### Reliability

| Item | How to Verify | Why It Matters |
|------|---------------|----------------|
| **Resource requests/limits set** | `kubectl describe pod` shows requests and limits | Prevents resource starvation |
| **Liveness/readiness probes** | Deployment spec includes probes | Automatic recovery |
| **PodDisruptionBudgets** | `kubectl get pdb -A` | Survives node drains |
| **Multiple replicas** | `kubectl get deployment` shows replicas > 1 | No single point of failure |

### Observability

| Item | How to Verify | Why It Matters |
|------|---------------|----------------|
| **CloudWatch Observability** | Addon is ACTIVE, dashboards show data | See what's happening |
| **Log retention set** | Check CloudWatch log group retention | Control costs |
| **Alarms configured** | CloudWatch alarms for CPU, memory, errors | Get notified of issues |
| **Application metrics** | App exposes /metrics or sends custom metrics | Business visibility |

### Operations

| Item | How to Verify | Why It Matters |
|------|---------------|----------------|
| **Backup strategy** | EBS snapshots scheduled, PV backup tested | Disaster recovery |
| **Upgrade plan** | Know current EKS version, target version | Security patches |
| **Runbooks documented** | Common operations and troubleshooting steps | Team can respond |
| **Access controls** | RBAC configured, audit logging enabled | Know who did what |

## Test vs Production: What Changes

Your test cluster scripts are the foundation. For production, these values typically change:

| Setting | Test | Production |
|---------|------|------------|
| **Cluster name** | `my-cluster-test` | `my-cluster-prod` |
| **Node count** | 2 nodes minimum | 3+ nodes across AZs |
| **Instance types** | Smaller, more spot | Mix of on-demand and spot |
| **Secrets path** | `EKS/test/*` | `EKS/prod/*` |
| **Log retention** | 7 days | 30-90 days |
| **Alarms** | Optional | Required |
| **Backup frequency** | Manual | Automated daily |

The scripts themselves stay identical - just update the configuration variables at the top.

## Architecture Patterns

Here's where everything comes together. Different application types need different combinations of what we've built.

### Pattern 1: Stateless Web Application

**Example**: REST API, web frontend, microservice

**What You Need**:
- ECR for container images
- EKS with Deployment (multiple replicas)
- Ingress Controller for HTTPS/load balancing
- External Secrets for database credentials, API keys
- CloudWatch for logs and metrics
- Pod Identity if calling AWS services (S3, SQS, etc.)

**What You Don't Need**:
- EBS Storage (stateless - no persistent volumes)

**Architecture**:

<div class="mermaid">
graph LR
    subgraph EKS Cluster
        ALB[ALB] --> SVC[Service]
        SVC --> DEP[Deployment<br/>3+ replicas]
        DEP --> ES[External Secrets]
        DEP --> PI[Pod Identity]
    end
    INT[Internet] --> ALB
    ES --> SM[Secrets Manager]
    PI --> AWS[AWS Services]
</div>

**Key Decisions**:
- Set CPU/memory requests based on load testing
- Use HorizontalPodAutoscaler for traffic-based scaling
- Configure readiness probes to prevent traffic to unhealthy pods
- Use rolling deployments with `maxUnavailable: 0` for zero-downtime deploys

**Deployment Spec Essentials**:
```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

### Pattern 2: Stateful Application (Database, Cache)

**Example**: PostgreSQL, Redis, Elasticsearch

**What You Need**:
- Everything from Pattern 1, plus:
- EBS Storage for persistent volumes
- StatefulSet instead of Deployment
- Headless Service for stable network identity

**Architecture**:

<div class="mermaid">
graph TB
    subgraph EKS Cluster
        APP[App Pods] --> HS[Headless Service]
        HS --> SS[StatefulSet]
        SS --> PVC[PersistentVolumeClaim]
    end
    PVC --> EBS[EBS Volume]
</div>

**Key Decisions**:
- Use `volumeBindingMode: WaitForFirstConsumer` to ensure volume and pod are in same AZ
- Set appropriate storage class (gp3 for general use, io2 for high IOPS)
- Configure backup strategy (EBS snapshots or application-level backups)
- Consider managed services (RDS, ElastiCache) for less operational burden

**When to Use EKS vs Managed Service**:

| Use EKS StatefulSet When | Use Managed Service When |
|--------------------------|--------------------------|
| Need specific version/config not in managed service | Want zero operational overhead |
| Cost-sensitive at scale | Need multi-AZ automatic failover |
| Already have K8s expertise | Team is small |
| Custom extensions required | Standard configuration is fine |

**StatefulSet Essentials**:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-sc
      resources:
        requests:
          storage: 100Gi
```

### Pattern 3: Background Worker / Queue Processor

**Example**: Job processor, message consumer, scheduled tasks

**What You Need**:
- ECR for container images
- EKS with Deployment (scales based on queue depth)
- Pod Identity for SQS/SNS access
- External Secrets for credentials
- CloudWatch for logs
- No Ingress (not internet-facing)

**Architecture**:

<div class="mermaid">
graph TB
    subgraph EKS Cluster
        WD[Worker Deployment<br/>autoscaled] --> ES[External Secrets]
        WD --> PI[Pod Identity]
    end
    ES --> SM[Secrets Manager]
    PI --> SQS[SQS Queue]
</div>

**Key Decisions**:
- Scale based on queue depth, not CPU (use KEDA or custom metrics)
- Set appropriate visibility timeout in SQS
- Implement graceful shutdown (handle SIGTERM)
- Use spot instances (workers are interruptible)

**Pod Identity for SQS**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes"
            ],
            "Resource": "arn:aws:sqs:us-east-1:ACCOUNT:my-queue"
        }
    ]
}
```

### Pattern 4: Scheduled Jobs (Cron)

**Example**: Daily reports, cleanup tasks, data exports

**What You Need**:
- ECR for container images
- EKS with CronJob
- Pod Identity for AWS access
- External Secrets for credentials
- CloudWatch for logs

**Architecture**:

<div class="mermaid">
graph TB
    subgraph EKS Cluster
        CJ[CronJob] --> JOB[Job]
        JOB --> POD[Pod]
        POD --> PI[Pod Identity]
    end
    PI --> AWS[AWS Services]
</div>

**Key Decisions**:
- Set `concurrencyPolicy: Forbid` to prevent overlapping runs
- Configure `startingDeadlineSeconds` for missed schedule handling
- Set `activeDeadlineSeconds` to kill hung jobs
- Use `ttlSecondsAfterFinished` to clean up completed jobs

**CronJob Essentials**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      activeDeadlineSeconds: 3600
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: report
            image: my-report:latest
```

### Pattern 5: Multi-Tier Application

**Example**: Frontend + API + Database + Cache + Workers

**What You Need**:
- All components from previous patterns
- Network policies for isolation
- Multiple namespaces for organization

**Architecture**:

<div class="mermaid">
graph TB
    subgraph EKS Cluster
        ALB[ALB] --> FE[Frontend]
        FE --> SVC[Internal Service]
        SVC --> API[Backend API]
        API --> PG[PostgreSQL<br/>StatefulSet]
        API --> RD[Redis Cache<br/>StatefulSet]
        PG --> WK[Worker]
    end
    INT[Internet] --> ALB
    SQS[SQS] --> WK
</div>

**Namespace Organization**:
```
my-app-prod/
├── frontend/       # Web UI
├── backend/        # API services
├── data/           # Databases, caches
└── workers/        # Background processors
```

**Key Decisions**:
- Use NetworkPolicies to restrict traffic between tiers
- Separate External Secrets per namespace (different IAM permissions)
- Consider service mesh for mTLS between services
- Use resource quotas per namespace to prevent noisy neighbors

## Component Decision Matrix

When do you need each component we've built?

| Component | Stateless API | Stateful DB | Worker | CronJob | Multi-Tier |
|-----------|:-------------:|:-----------:|:------:|:-------:|:----------:|
| **ECR** | Yes | Yes | Yes | Yes | Yes |
| **EKS Base** | Yes | Yes | Yes | Yes | Yes |
| **Ingress** | Yes | No | No | No | Frontend only |
| **EBS Storage** | No | Yes | No | No | Data tier |
| **Pod Identity** | If AWS calls | If AWS calls | Usually | Usually | Per service |
| **External Secrets** | Yes | Yes | Yes | Yes | Yes |
| **CloudWatch** | Yes | Yes | Yes | Yes | Yes |

## Cost Optimization

EKS costs come from several sources. Here's how to optimize each:

### Compute (Biggest Cost)

**Use Spot Instances for**:
- Stateless workloads (web servers, APIs)
- Workers and background jobs
- Development/test environments

**Use On-Demand for**:
- Stateful workloads (databases)
- Critical single-replica services
- Production control plane adjacent services

**Right-size Resources**:
```bash
# Check actual vs requested resources
kubectl top pods
kubectl describe pod my-pod | grep -A 2 "Requests:"
```

If actual usage is consistently 20% of requests, you're overprovisioned.

### Storage

- Use gp3 instead of gp2 (20% cheaper, better performance)
- Set appropriate volume sizes (can expand later, can't shrink)
- Delete unused PVCs: `kubectl get pvc -A | grep -v Bound`

### Networking

- ALB is charged per hour + LCU. Consolidate ingresses where possible
- Use internal load balancers for service-to-service communication
- Enable VPC endpoints for ECR, S3, CloudWatch to avoid NAT charges

### Monitoring

- Set log retention policies (don't keep logs forever)
- Use metric filters instead of storing all logs
- Disable Container Insights if using alternative monitoring

## Upgrade Strategy

EKS releases new Kubernetes versions regularly. Plan for upgrades:

### Before Upgrading

1. **Check version compatibility**:
   ```bash
   eksctl utils describe-addon-versions --cluster my-cluster
   ```

2. **Review deprecation notices**:
   ```bash
   kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
   ```

3. **Test in non-production first** - Clone your scripts, create a test cluster with new version

### Upgrade Order

1. **Control plane first**: `eksctl upgrade cluster --name my-cluster --version 1.XX`
2. **Update addons**: EBS CSI, Pod Identity Agent, CloudWatch
3. **Update node groups**: Rolling update with new AMI
4. **Update kubectl**: Match cluster version

### Addon Compatibility

Check addon versions before cluster upgrade:
```bash
aws eks describe-addon-versions --addon-name vpc-cni --kubernetes-version 1.XX
```

## Disaster Recovery

### What to Back Up

| Component | Backup Method | Frequency |
|-----------|---------------|-----------|
| **Application state** | EBS snapshots | Daily |
| **Cluster config** | GitOps (manifests in git) | Every change |
| **Secrets** | Secrets Manager (built-in) | Automatic |
| **ECR images** | Cross-region replication | Automatic |

### Recovery Scenarios

**Pod failure**: Kubernetes handles automatically (restarts, reschedules)

**Node failure**: Auto-scaling group replaces node, pods reschedule

**AZ failure**: Pods reschedule to nodes in other AZs (if multi-AZ)

**Cluster failure**:
1. Create new cluster from scripts
2. Restore EBS volumes from snapshots
3. Deploy applications from git
4. Secrets sync automatically from Secrets Manager

**Region failure**:
1. Requires pre-planned multi-region setup
2. ECR cross-region replication
3. Secrets Manager cross-region replication
4. DNS failover (Route 53)

## Common Pitfalls

### 1. No Resource Limits

**Problem**: One pod consumes all node resources, starving others.

**Solution**: Always set requests and limits. Start conservative, adjust based on metrics.

### 2. Single Replica in Production

**Problem**: Any pod restart causes downtime.

**Solution**: Minimum 2 replicas for anything user-facing. Use PodDisruptionBudgets.

### 3. Secrets in Environment Variables

**Problem**: Secrets visible in `kubectl describe pod`, logged accidentally.

**Solution**: Use External Secrets → Kubernetes Secrets → Volume mounts.

### 4. No Readiness Probes

**Problem**: Traffic sent to pods that aren't ready, causing errors.

**Solution**: Always configure readiness probes. Test they actually verify readiness.

### 5. Ignoring Pod Identity

**Problem**: AWS credentials in environment variables or mounted files.

**Solution**: Use Pod Identity for all AWS access. Zero stored credentials.

### 6. Oversized Persistent Volumes

**Problem**: Provisioned 500GB, using 10GB, paying for 500GB.

**Solution**: Start small, monitor usage, expand as needed (can't shrink).

### 7. No Log Retention

**Problem**: CloudWatch costs grow unbounded.

**Solution**: Set retention policies on all log groups. 7-30 days for most use cases.

## What's Next

This series covered the foundational EKS infrastructure. Here's what to explore next, with how to install each:

### Deployment Automation

| Tool | Purpose | Install Method |
|------|---------|----------------|
| **ArgoCD** | GitOps continuous delivery | Helm: `helm install argocd argo/argo-cd` |
| **Flux** | GitOps toolkit | CLI: `flux bootstrap github` |
| **GitHub Actions** | CI/CD pipelines | No cluster install (runs in GitHub) |

### Advanced Scaling

| Tool | Purpose | Install Method |
|------|---------|----------------|
| **Karpenter** | Intelligent node provisioning | Helm: `helm install karpenter oci://public.ecr.aws/karpenter/karpenter` |
| **KEDA** | Event-driven pod autoscaling | Helm: `helm install keda kedacore/keda` |
| **Metrics Server** | Pod metrics for HPA | Helm: `helm install metrics-server metrics-server/metrics-server` |

### Networking & Service Mesh

| Tool | Purpose | Install Method |
|------|---------|----------------|
| **Istio** | Service mesh, mTLS | CLI: `istioctl install` |
| **Linkerd** | Lightweight service mesh | CLI: `linkerd install \| kubectl apply -f -` |
| **Calico** | Network policies | Helm: `helm install calico projectcalico/tigera-operator` |

### Security

| Tool | Purpose | Install Method |
|------|---------|----------------|
| **OPA Gatekeeper** | Policy enforcement | Helm: `helm install gatekeeper gatekeeper/gatekeeper` |
| **Falco** | Runtime threat detection | Helm: `helm install falco falcosecurity/falco` |
| **GuardDuty EKS** | AWS threat detection | AWS Console: Enable in GuardDuty settings |
| **Trivy Operator** | Vulnerability scanning | Helm: `helm install trivy-operator aqua/trivy-operator` |

### Observability (Beyond CloudWatch)

| Tool | Purpose | Install Method |
|------|---------|----------------|
| **Prometheus** | Metrics collection | Helm: `helm install prometheus prometheus-community/prometheus` |
| **Grafana** | Dashboards | Helm: `helm install grafana grafana/grafana` |
| **Loki** | Log aggregation | Helm: `helm install loki grafana/loki-stack` |

### Cost Management

| Tool | Purpose | Install Method |
|------|---------|----------------|
| **Kubecost** | Cost allocation | Helm: `helm install kubecost kubecost/cost-analyzer` |
| **AWS Cost Explorer** | AWS-level costs | AWS Console (no cluster install) |

Most tools use Helm charts. Add the repos first:
```bash
# Common Helm repos
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo add aqua https://aquasecurity.github.io/helm-charts
helm repo add kubecost https://kubecost.github.io/cost-analyzer
helm repo update
```

## Series Summary

You've built production-ready EKS infrastructure:

| Part | Component | What It Provides |
|------|-----------|------------------|
| 1.1 | ECR | Secure container registry with vulnerability scanning |
| 1.2 | EKS Base | Kubernetes control plane with auto-scaling nodes |
| 1.3 | Ingress | HTTPS termination and load balancing |
| 1.4 | EBS Storage | Persistent volumes for stateful workloads |
| 1.5 | Pod Identity | Secure AWS access without credentials |
| 1.6 | External Secrets | Automated secrets sync from Secrets Manager |
| 1.7 | CloudWatch | Dashboards, metrics, and log collection |
| 1.8 | This Guide | Operations, production readiness, architecture |

The infrastructure is ready. Build something great.

---

*Questions about EKS architecture? Reach out on socials.*
