---
layout: post
title: "1.2 EKS: Creating the Base Cluster with Auto-Scaling"
date: 2026-01-17 12:00:00 -0000
categories: kubernetes eks infrastructure aws
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS"]
series: "EKS Infrastructure Series"
series_part: "1.2"
---

With ECR ready to store our container images, it's time to create the Kubernetes cluster itself. We'll build a cost-optimized EKS cluster with spot instances, auto-scaling, and private networking - production patterns at development prices.

This is part 1.2 of the EKS Infrastructure Series. We're building on the container registry from [1.1 ECR Setup](/2026/01/17/eks-ecr-container-registry/).

## What We're Building

By the end of this article, you'll have:

- **EKS cluster** with Kubernetes control plane managed by AWS
- **Auto-scaling node group** that scales from 1-5 nodes based on demand
- **Spot instances** for significant cost savings (up to 90% vs on-demand)
- **Private networking** with nodes in private subnets only
- **Cluster Autoscaler** for automatic node scaling
- **IAM OIDC provider** for secure service account authentication

## What EKS Actually Provides

Before diving in, it's worth understanding what you're paying for with EKS versus running Kubernetes yourself.

### The Shared Responsibility Model

<div class="mermaid">
graph TB
    subgraph AWS Manages
        CP[Control Plane]
        ETCD[etcd]
        API[API Server]
        SCHED[Scheduler]
        CM[Controller Manager]
        UP[Upgrades & Patches]
        HA[High Availability]
    end
    subgraph You Manage
        NODES[Worker Nodes]
        APPS[Applications]
        ADDONS[Add-ons]
        NET[Networking Config]
        SEC[Security Policies]
        MON[Monitoring Setup]
    end
</div>

| Component | Self-Managed K8s | EKS |
|-----------|------------------|-----|
| **Control plane servers** | You provision, patch, scale | AWS manages completely |
| **etcd cluster** | You back up, maintain, monitor | AWS handles (3 AZs, encrypted) |
| **API server availability** | You configure HA, load balancing | 99.95% SLA, auto-scaled |
| **Kubernetes upgrades** | You plan, test, execute | One command, managed rollout |
| **Security patches** | You monitor CVEs, apply patches | AWS patches automatically |
| **Worker nodes** | You manage | You manage |
| **Applications** | You deploy | You deploy |

### What You Don't Have to Do with EKS

Running Kubernetes yourself means:

- **Provisioning control plane VMs** - At least 3 for HA, sized appropriately
- **Installing and configuring etcd** - Distributed consensus is hard to get right
- **Setting up TLS certificates** - For API server, kubelet, etcd communication
- **Configuring the API server** - Dozens of flags for auth, admission controllers, feature gates
- **Managing etcd backups** - Regular snapshots, tested restore procedures
- **Handling control plane upgrades** - Careful sequencing, rollback plans
- **Monitoring control plane health** - API server latency, etcd disk I/O, scheduler queue depth

With EKS, all of this is handled. You get a Kubernetes API endpoint that just works.

### What You Still Manage

EKS doesn't eliminate all operational work. You're responsible for:

- **Worker nodes** - EC2 instances that run your pods (though managed node groups help)
- **Cluster add-ons** - CNI, CSI drivers, ingress controllers, monitoring
- **Application deployments** - Your actual workloads
- **Security configuration** - RBAC, network policies, pod security
- **Cost optimization** - Right-sizing, spot instances, scaling policies

### The Real Value

The $73/month EKS control plane cost buys you:

- **Time savings** - No control plane maintenance, focus on applications
- **Reliability** - AWS's operational expertise, multi-AZ by default
- **Security** - Automatic patches, AWS security team monitoring
- **Integration** - Native IAM, CloudWatch, VPC integration
- **Support** - AWS support covers EKS issues

For comparison, running your own HA control plane on EC2:
- 3x t3.medium instances: ~$90/month
- Plus your time managing them
- Plus the risk of misconfiguration

EKS is genuinely cheaper when you factor in operational overhead.

## Why eksctl?

We're using `eksctl` instead of Terraform, CloudFormation, or the AWS Console. Here's why:

**One command creates everything.** A single `eksctl create cluster` command provisions:

| Component | What eksctl creates automatically |
|-----------|----------------------------------|
| VPC | New VPC with proper CIDR blocks |
| Subnets | Public and private subnets across 2-3 AZs |
| NAT Gateways | For outbound internet from private subnets |
| Internet Gateway | For public subnet routing |
| Route Tables | Properly configured for public/private traffic |
| Security Groups | EKS-specific rules for cluster communication |
| IAM Roles | Node role, cluster role with correct policies |
| EKS Control Plane | Managed Kubernetes API server |
| Node Group | EC2 instances with Auto Scaling Group |

**Compare this to Terraform**, where you'd write 300+ lines of HCL across multiple files to achieve the same result. Or the AWS Console, where you'd click through dozens of screens.

**eksctl is opinionated in the right ways.** It follows AWS best practices by default - private subnets for nodes, proper IAM boundaries, security group rules that actually make sense.

**It's still Infrastructure as Code.** You can export the config to YAML, version it, and reproduce clusters exactly.

## Why This Configuration?

**Spot instances** - AWS sells unused EC2 capacity at steep discounts. For development and testing workloads that can handle occasional interruptions, this cuts costs dramatically.

**Auto-scaling** - Pay for what you use. The cluster scales down to one node when idle and up to five when busy.

**Private networking** - Nodes have no public IP addresses. All traffic goes through the VPC, reducing attack surface.

**OIDC provider** - Enables pods to assume IAM roles without storing credentials. This is the foundation for secure AWS API access in later articles.

## Prerequisites

Ensure you have:

```bash
# AWS CLI configured
aws --version
aws sts get-caller-identity

# eksctl for cluster management
eksctl version

# kubectl for Kubernetes operations
kubectl version --client

# Completed 1.1 ECR Setup (optional but recommended)
```

Your AWS user needs permissions to create EKS clusters, EC2 instances, IAM roles, and VPC resources.

## Project Structure

```
eks-base/
├── create-cluster.sh           # Cluster creation script
├── delete-cluster.sh           # Cleanup script
└── cluster-autoscaler-patch.yaml  # Autoscaler configuration
```

## Step 1: Create the Cluster Configuration Script

This script creates the EKS cluster with all our cost and security optimizations:

**create-cluster.sh**
```bash
#!/bin/bash

# Configuration - modify these for your environment
CLUSTER_NAME="my-cluster"
REGION="us-east-1"
NODE_TYPE="t3.medium"
MIN_NODES=1
MAX_NODES=5
K8S_VERSION="latest"
USE_SPOT=true

echo "EKS Cluster Setup"
echo "=================="
echo "Cluster Name:    $CLUSTER_NAME"
echo "Region:          $REGION"
echo "Node Type:       $NODE_TYPE$([ "$USE_SPOT" = true ] && echo " (spot instances)")"
echo "Scaling:         $MIN_NODES-$MAX_NODES nodes (auto-scaling)"
echo "K8s Version:     $K8S_VERSION"
echo ""

echo "Creating EKS cluster..."
eksctl create cluster --name=$CLUSTER_NAME \
    --nodegroup-name=spot-nodes \
    --nodes-min=$MIN_NODES \
    --nodes-max=$MAX_NODES \
    --node-type=$NODE_TYPE \
    $([ "$USE_SPOT" = true ] && echo "--spot") \
    --asg-access \
    --version=$K8S_VERSION \
    --node-private-networking

echo "Setting cluster to standard support (cost optimization)..."
aws eks update-cluster-config --name $CLUSTER_NAME \
    --upgrade-policy supportType=STANDARD

echo "Configuring kubectl access..."
aws eks update-kubeconfig --region $REGION --name $CLUSTER_NAME

echo "Setting up IAM OIDC provider for service accounts..."
eksctl utils associate-iam-oidc-provider \
    --region=$REGION \
    --cluster=$CLUSTER_NAME \
    --approve

echo "Setting up Cluster Autoscaler permissions..."
eksctl create iamserviceaccount \
    --name cluster-autoscaler \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn arn:aws:iam::aws:policy/AutoScalingFullAccess \
    --approve

echo "Installing Cluster Autoscaler..."
curl -o cluster-autoscaler-autodiscover.yaml \
    https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

kubectl apply -f cluster-autoscaler-autodiscover.yaml

# Wait for deployment to exist
sleep 10

echo "Configuring autoscaler for cluster: $CLUSTER_NAME"
sed "s/CLUSTER_NAME_PLACEHOLDER/$CLUSTER_NAME/g" cluster-autoscaler-patch.yaml | \
    kubectl patch deployment cluster-autoscaler -n kube-system --patch-file=/dev/stdin

echo "Waiting for autoscaler to be ready..."
kubectl rollout status deployment/cluster-autoscaler -n kube-system --timeout=120s

# Clean up temporary file
rm -f cluster-autoscaler-autodiscover.yaml

echo ""
echo "SUCCESS! EKS cluster '$CLUSTER_NAME' is ready"
echo "=========================================="
echo "Features enabled:"
echo "   - Auto-scaling: $MIN_NODES-$MAX_NODES nodes"
echo "   - Cost optimization: $([ "$USE_SPOT" = true ] && echo "Spot instances" || echo "On-demand instances")"
echo "   - Security: Nodes in private subnets (no public IPs)"
echo ""
echo "Verify with:"
echo "   kubectl get nodes"
echo "   kubectl get pods -A"
```

Key flags explained:

- **`--spot`** - Use spot instances for cost savings
- **`--asg-access`** - Grant nodes permission to manage Auto Scaling Groups (required for Cluster Autoscaler)
- **`--node-private-networking`** - Place nodes in private subnets only
- **`--nodes-min/max`** - Set auto-scaling bounds

## Step 2: Create the Autoscaler Patch

The Cluster Autoscaler needs configuration specific to your cluster. Create a patch file:

**cluster-autoscaler-patch.yaml**
```yaml
spec:
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    spec:
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.32.2
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/CLUSTER_NAME_PLACEHOLDER
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        env:
        - name: AWS_REGION
          value: us-east-1
```

Configuration explained:

- **`safe-to-evict: "false"`** - Prevents the autoscaler from being evicted during scale-down
- **`--expander=least-waste`** - Chooses nodes that minimize wasted resources
- **`--node-group-auto-discovery`** - Automatically discovers node groups by tags
- **`--balance-similar-node-groups`** - Distributes pods evenly across node groups

## Step 3: Create the Cleanup Script

For when you need to tear down the cluster:

**delete-cluster.sh**
```bash
#!/bin/bash

# Configuration (should match create-cluster.sh)
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "EKS Cluster Cleanup"
echo "==================="
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""
echo "This will remove:"
echo "  - EKS cluster and nodes"
echo "  - Cluster Autoscaler"
echo "  - IAM roles and policies"
echo ""

read -p "Are you sure? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "Cancelled"
    exit 0
fi

echo "Removing Cluster Autoscaler..."
kubectl delete -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml || echo "Already deleted"

echo "Removing autoscaler IAM permissions..."
eksctl delete iamserviceaccount \
    --name cluster-autoscaler \
    --namespace kube-system \
    --cluster $CLUSTER_NAME || echo "Already deleted"

echo "Removing OIDC provider..."
eksctl utils disassociate-iam-oidc-provider \
    --region=$REGION \
    --cluster=$CLUSTER_NAME \
    --approve || echo "Already deleted"

echo "Deleting EKS cluster..."
eksctl delete cluster --name=$CLUSTER_NAME

echo ""
echo "Cleanup complete!"
echo "Cluster '$CLUSTER_NAME' has been deleted."
```

## Step 4: Run the Setup

Execute the creation script:

```bash
chmod +x create-cluster.sh
./create-cluster.sh
```

This takes approximately 15-20 minutes. eksctl creates:
1. VPC with public and private subnets
2. EKS control plane
3. Node group with Auto Scaling Group
4. IAM roles for nodes and service accounts
5. Security groups for cluster communication

Expected output at completion:
```
SUCCESS! EKS cluster 'my-cluster' is ready
==========================================
Features enabled:
   - Auto-scaling: 1-5 nodes
   - Cost optimization: Spot instances
   - Security: Nodes in private subnets (no public IPs)

Verify with:
   kubectl get nodes
   kubectl get pods -A
```

## Step 5: Verify the Cluster

Check that everything is running:

```bash
# View nodes
kubectl get nodes
```

Expected output:
```
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-xx-xx.ec2.internal   Ready    <none>   5m    v1.31.x
```

Check system pods:
```bash
kubectl get pods -n kube-system
```

You should see the cluster-autoscaler pod running:
```
NAME                                  READY   STATUS    RESTARTS   AGE
cluster-autoscaler-xxxxxxxxx-xxxxx   1/1     Running   0          2m
coredns-xxxxxxxxx-xxxxx              1/1     Running   0          5m
coredns-xxxxxxxxx-xxxxx              1/1     Running   0          5m
kube-proxy-xxxxx                     1/1     Running   0          5m
```

## Step 6: Test Auto-Scaling

Deploy a workload that requests more resources than one node can provide:

```bash
# Create a deployment that will trigger scaling
kubectl create deployment nginx-scale-test \
    --image=nginx \
    --replicas=10

# Request more CPU per pod to trigger node scaling
kubectl set resources deployment nginx-scale-test \
    --requests=cpu=200m,memory=256Mi

# Watch nodes scale up
kubectl get nodes -w
```

The Cluster Autoscaler will detect pending pods and add nodes. Check autoscaler logs:

```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

Clean up the test:
```bash
kubectl delete deployment nginx-scale-test
```

After a few minutes of low utilization, the autoscaler will scale back down.

## How Auto-Scaling Works

Understanding the scaling mechanics helps you tune the cluster for your workloads.

### Node Resources

A `t3.medium` instance provides:

| Resource | Total | Allocatable | Reserved for System |
|----------|-------|-------------|---------------------|
| CPU | 2 vCPUs | ~1.9 vCPUs | ~100m for kubelet, OS |
| Memory | 4 GiB | ~3.5 GiB | ~500 MiB for system |

The "allocatable" capacity is what's available for your pods. Kubernetes reserves resources for the kubelet, container runtime, and OS processes.

### Scale-Up Decision

The Cluster Autoscaler checks for pending pods every 10 seconds. A pod is "pending" when the scheduler can't find a node with enough resources. The scale-up process:

1. **Detection** - Autoscaler finds pods in `Pending` state due to insufficient resources
2. **Simulation** - It simulates adding nodes from your node group to see if pods would fit
3. **Expansion** - If simulation succeeds, it increases the Auto Scaling Group's desired count
4. **Node startup** - AWS launches the instance (~2-3 minutes for EC2 to be ready)
5. **Node registration** - The new node joins the cluster and becomes `Ready`
6. **Scheduling** - Pending pods get scheduled to the new node

The `--expander=least-waste` flag we configured means the autoscaler picks the node type that would have the least unused resources after scheduling.

### Scale-Down Decision

Scale-down is more conservative to avoid thrashing. The autoscaler considers a node for removal when:

1. **Utilization threshold** - Node CPU and memory utilization is below 50% (default)
2. **Pod evictability** - All pods can be moved elsewhere (no local storage, not blocked by PodDisruptionBudget)
3. **Cool-down period** - The node has been underutilized for 10 minutes (default)

The scale-down process:

1. **Candidate selection** - Nodes below utilization threshold are marked as candidates
2. **Waiting period** - Must remain underutilized for the unneeded time (10 min default)
3. **Drain** - Pods are gracefully evicted with respect to PodDisruptionBudgets
4. **Termination** - The Auto Scaling Group terminates the instance

### Key Timings

| Event | Typical Duration |
|-------|------------------|
| Pending pod detection | 10 seconds (scan interval) |
| Scale-up decision | Immediate after detection |
| New node ready | 2-3 minutes |
| Scale-down consideration starts | When utilization drops below 50% |
| Scale-down waiting period | 10 minutes |
| Node drain and termination | 1-2 minutes |

### Configuration Flags We're Using

From the `cluster-autoscaler-patch.yaml`:

- **`--skip-nodes-with-local-storage=false`** - Allow scaling down nodes with emptyDir volumes
- **`--skip-nodes-with-system-pods=false`** - Allow scaling down nodes running kube-system pods (except DaemonSets)
- **`--balance-similar-node-groups`** - Keep node counts balanced across availability zones

### Monitoring Autoscaler Activity

Watch the autoscaler make decisions:

```bash
# Follow autoscaler logs
kubectl logs -f deployment/cluster-autoscaler -n kube-system

# Check current cluster state
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
CPU:.status.allocatable.cpu,\
MEMORY:.status.allocatable.memory
```

Look for log entries like:
- `Scale-up: setting group size` - Scaling up
- `Scale-down: removing empty node` - Removing underutilized node
- `Pod is unschedulable` - Trigger for scale-up consideration

### Modifying Node Group Limits

Need more than 5 nodes? You can update the scaling limits without recreating the cluster:

```bash
# Increase max nodes from 5 to 10
eksctl scale nodegroup --cluster=my-cluster \
    --name=spot-nodes \
    --nodes-min=1 \
    --nodes-max=10
```

This updates the Auto Scaling Group immediately. The autoscaler will now scale up to 10 nodes if workloads demand it.

### Changing Instance Types

To use larger instances (like `t3.large` instead of `t3.medium`), add a new node group and migrate workloads:

```bash
# Create new node group with larger instances
eksctl create nodegroup --cluster=my-cluster \
    --name=spot-nodes-large \
    --node-type=t3.large \
    --nodes-min=1 \
    --nodes-max=5 \
    --spot \
    --node-private-networking

# Drain and delete the old node group
eksctl delete nodegroup --cluster=my-cluster \
    --name=spot-nodes \
    --drain
```

The `--drain` flag gracefully moves pods to the new nodes before termination. You can also delete and recreate the node group directly if brief downtime is acceptable.

### Instance Type Comparison

| Type | vCPUs | Memory | Spot Price (approx) | Use Case |
|------|-------|--------|---------------------|----------|
| t3.medium | 2 | 4 GiB | ~$0.01/hr | Light workloads, testing |
| t3.large | 2 | 8 GiB | ~$0.02/hr | Memory-heavy apps |
| t3.xlarge | 4 | 16 GiB | ~$0.04/hr | Production workloads |
| m5.large | 2 | 8 GiB | ~$0.03/hr | Consistent performance |

Spot prices vary by region and availability. Check current prices with:

```bash
aws ec2 describe-spot-price-history \
    --instance-types t3.large \
    --product-descriptions "Linux/UNIX" \
    --query 'SpotPriceHistory[0].SpotPrice'
```

## Understanding the Architecture

Here's what eksctl creates with a single command:

<div class="mermaid">
graph LR
    subgraph VPC
        direction TB
        NAT[NAT Gateway]
        subgraph Private Subnets
            N1[Node 1]
            N2[Node 2]
        end
    end

    EKS[EKS Control Plane]
    ECR[ECR Registry]
    INT[Internet]

    EKS --> N1
    EKS --> N2
    N1 --> NAT
    N2 --> NAT
    NAT --> INT
    N1 -.-> ECR
    N2 -.-> ECR
</div>

All of this is created automatically by eksctl:

- **VPC + Subnets** - Public subnets (for NAT/load balancers), private subnets (for nodes)
- **NAT Gateway** - Allows nodes to pull images and reach AWS APIs without public IPs
- **EKS Control Plane** - Managed by AWS (API server, etcd, scheduler)
- **Worker Nodes** - EC2 instances in private subnets with Auto Scaling Group
- **IAM Roles** - Cluster role and node role with correct permissions
- **Security Groups** - Rules for control plane ↔ node communication

## Cost Breakdown

Approximate monthly costs for this configuration (us-east-1):

| Component | On-Demand | With Spot |
|-----------|-----------|-----------|
| EKS Control Plane | $72 | $72 |
| t3.medium node (1 node baseline) | $30 | ~$9 |
| NAT Gateway | ~$32 | ~$32 |
| **Total (idle)** | **~$134** | **~$113** |

Spot instances save ~70% on compute. The control plane cost is fixed regardless of node count.

**Cost optimization tips:**
- Scale to 0 nodes when not in use (requires manual intervention or scheduled scaling)
- Use smaller instances for development
- Consider NAT instance instead of NAT Gateway for further savings

## Troubleshooting

### Nodes Not Joining

Check the node group status:
```bash
eksctl get nodegroup --cluster my-cluster
```

Check for IAM issues:
```bash
aws eks describe-cluster --name my-cluster \
    --query 'cluster.identity.oidc.issuer'
```

### Autoscaler Not Scaling

Check autoscaler logs:
```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

Common issues:
- **ASG tags missing** - Ensure `--asg-access` was used during cluster creation
- **IAM permissions** - The autoscaler service account needs AutoScalingFullAccess

### Spot Instance Interruptions

Spot instances can be reclaimed by AWS. The cluster handles this gracefully:
1. AWS sends a 2-minute warning
2. Kubernetes marks the node as unschedulable
3. Pods are evicted and rescheduled to other nodes
4. Autoscaler provisions replacement capacity

For production workloads, consider a mixed instances policy with some on-demand nodes.

## What We've Accomplished

You now have:

- A running EKS cluster with managed control plane
- Auto-scaling from 1-5 nodes based on workload
- Spot instances for cost optimization
- Private networking for security
- Cluster Autoscaler for automatic capacity management
- IAM OIDC provider ready for service account authentication

## Next Steps

The base cluster is ready for add-ons. In the next article, we'll set up the AWS Load Balancer Controller for ingress, enabling external access to services with automatic SSL certificates.

**Next**: [1.3 EKS Ingress Controller - SSL and Load Balancing](/2026/01/17/eks-ingress-controller/)

---

*Questions about EKS cluster configuration? Reach out on socials.*
