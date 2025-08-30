---
layout: post
title: "1.1 EKS: Kubernetes Base and Node Autoscaling"
date: 2025-08-29 11:00:00 -0000
categories: kubernetes eks infrastructure security aws
tags: ["Modern Full-Stack Development", "EKS"]
series: "Modern Full-Stack Development"
series_part: "1.1"
---

Every senior engineer has lived this nightmare: you inherit an EKS cluster that "works perfectly" until you need to add SSL, persistent storage, or proper secrets management. What started as a 30-minute demo setup becomes weeks of untangling hardcoded configurations, fixing security gaps, and debugging mysterious networking issues.

The problem isn't EKS itself - it's the "move fast and break things" approach to infrastructure. Most tutorials teach you to `eksctl create cluster` with all the defaults, then bolt on security as an afterthought. The result? Clusters that work in isolation but crumble under production constraints.

Today we're building the foundation: a base EKS cluster with essential security and scaling configured properly from the start. This is part 1.1 of our Modern Full-Stack Development series - we'll build on this foundation throughout Phase 1 by adding ingress, storage, secrets management, and other components as separate, modular pieces.

## What We're Building

By the end of this walkthrough, you'll have the base EKS foundation that serves as both a test environment and a blueprint for production deployment:

- **Base Cluster**: EKS with spot instances (t3.medium) scaling 1-5 nodes
- **Security Foundation**: Private subnets, OIDC provider, proper IAM setup
- **Auto-Scaling**: Cluster Autoscaler with intelligent node management
- **Cost Optimized**: Spot instances, standard support tier
- **Modular Design**: Clean foundation ready for addons (ingress, storage, secrets, etc.)

This base cluster is just the foundation - throughout Section 1.1, we'll add ingress with SSL, EBS storage, external secrets management, and Pod Identity as separate, modular components.

## Our Infrastructure Tool: eksctl

Before we dive into the architecture, let's talk about the primary tool we'll use for cluster management: `eksctl`. This is AWS's official CLI tool for creating and managing EKS clusters, and it's what powers most of the infrastructure automation in this series.

`eksctl` simplifies the complex process of EKS cluster creation by handling the underlying CloudFormation stacks, IAM roles, and networking configuration. Instead of manually configuring dozens of AWS resources, we can define our cluster declaratively and let `eksctl` handle the implementation details.


### Why eksctl Over Other Options?

While you could use Terraform, CDK, or raw CloudFormation, `eksctl` offers several advantages for EKS:

- **EKS-specific optimization**: Built specifically for EKS workflows
- **Best practices by default**: Handles IAM roles, security groups, and networking sensibly
- **Incremental updates**: Add and remove components without recreating the entire cluster
- **Active maintenance**: Kept up-to-date with latest EKS features and Kubernetes versions


### Prerequisites

Before we start building, make sure you have:

```bash
# Required tools
aws --version        # AWS CLI v2
eksctl version      # eksctl for EKS management  
kubectl version     # kubectl for cluster interaction
```

You'll also need AWS credentials configured with appropriate permissions for EKS cluster creation. The [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html) covers the detailed setup steps.

## The Modular Approach

Here's what most EKS tutorials get wrong: they install everything at once. ALB controllers, storage drivers, secrets management - all bundled together in one massive script. When something breaks, good luck figuring out which component caused the issue.

Our architecture separates concerns cleanly:

### Base Cluster (`create-cluster.sh`)
```bash
# The foundation - only essentials
- EKS control plane (latest version)
- Spot instance node group (t3.medium, 1-5 nodes)
- Cluster Autoscaler v1.32.2 with proper IAM
- OIDC provider for service account authentication
- Private networking with NAT gateway
```

### Optional Addons (install as needed)
```bash
# Each addon is self-contained
./install-alb-controller.sh    # SSL/TLS ingress with ACM
./install-ebs-csi.sh          # Persistent storage (GP3/GP2/IO2)
./install-external-secrets.sh # AWS Secrets Manager integration
./install-pod-identity.sh     # Credential-less AWS API access
```

This separation gives you:
- **Faster debugging**: Know exactly which component failed
- **Selective deployment**: Production might not need all addons
- **Independent updates**: Update storage without touching networking
- **Clear dependencies**: Base cluster works standalone, addons build on it

## Security Architecture

Security isn't an add-on feature - it's baked into every architectural decision. Here's how we implement defense-in-depth:

### Network Isolation
```bash
# Worker nodes get ZERO public IPs
--node-private-networking
```
- All EC2 instances in private subnets
- Internet access through NAT gateway only
- Load balancers in public subnets, applications in private
- Subnet tagging for proper ALB/NLB placement

The `--node-private-networking` is a **a-must** security control because it removes your worker nodes from the internet

### IAM Without Secrets
```bash
# OIDC provider enables service account authentication
eksctl utils associate-iam-oidc-provider --cluster=$CLUSTER_NAME --approve
```
- No AWS access keys stored in pods or containers
- Each service gets minimal IAM permissions via service accounts
- Cluster Autoscaler, ALB Controller use IRSA (IAM Roles for Service Accounts)
- Pod Identity addon for even more granular AWS access

### Production Security Defaults
- EKS API server accessible from authorized networks only
- Cluster logging to CloudWatch for audit trails
- Standard support tier (avoids extended support vulnerabilities)
- Ready for network policies and Pod Security Standards

## Cost Optimization Strategy

Production-ready doesn't mean burning cash. Our optimization strategy delivers ~$0.18/hour minimum cost:

### Real Cost Breakdown
```
EKS Control Plane (Standard): $0.10/hour
t3.medium spot (1 node min):  $0.01/hour
NAT Gateway:                  $0.045/hour
ALB (when deployed):          $0.025/hour
────────────────────────────────────────
Total minimum:                $0.18/hour = ~$130/month
```

### Spot Instance Intelligence
```bash
# 90% cost savings over on-demand
USE_SPOT=true
NODE_TYPE="t3.medium"  # Right-sized for most workloads
MIN_NODES=1           # Reduced from typical 2-3 minimum
```
- Cluster Autoscaler handles spot interruptions gracefully
- Mix of AZs reduces interruption probability
- `--balance-similar-node-groups` for better spot availability

### Smart Operational Choices
- **Standard support tier**: No extended support fees
- **Single NAT gateway**: Multi-AZ traffic but shared egress
- **Managed EKS**: No control plane maintenance costs
- **Auto-scaling to 1**: Most clusters sit idle 80% of the time

## Implementation Walkthrough

Time to build this thing. Follow these numbered steps to get your base EKS cluster running with auto-scaling and security configured properly.

### Step 1: Verify Prerequisites

First, make sure you have the required tools installed:

```bash
# Check your versions
aws --version        # AWS CLI v2 (required for EKS)
eksctl version       # eksctl 0.150+ recommended
kubectl version      # kubectl 1.29+ for EKS compatibility
helm version         # v3.12+ for addon installations
```

AWS credentials need EKS, EC2, and IAM permissions. The `AdministratorAccess` policy works for testing, but production should use more restrictive permissions.

![Prerequisites Check](/assets/images/full-stack/1.1step1.png)
*Step 1: Verifying all required tools are installed and ready*

### Step 2: Create the Base Cluster Script

Create your cluster creation script with the optimized configuration:

**File: `infra/test/eksctl/base/create-cluster.sh`**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"
NODE_TYPE="t3.medium"
MIN_NODES=1
MAX_NODES=5
K8S_VERSION="latest"
USE_SPOT=true

# Create EKS cluster with optimized configuration
eksctl create cluster --name=$CLUSTER_NAME \
    --nodegroup-name=spot-nodes \
    --nodes-min=$MIN_NODES \
    --nodes-max=$MAX_NODES \
    --node-type=$NODE_TYPE \
    $([ "$USE_SPOT" = true ] && echo "--spot") \
    --asg-access \
    --version=$K8S_VERSION \
    --node-private-networking

# Set cluster to standard support (cost optimization)
aws eks update-cluster-config --name $CLUSTER_NAME \
    --upgrade-policy supportType=STANDARD

# Configure kubectl access
aws eks update-kubeconfig --region $REGION --name $CLUSTER_NAME

# Set up IAM OIDC provider for service accounts
eksctl utils associate-iam-oidc-provider \
    --region=$REGION --cluster=$CLUSTER_NAME --approve
```

**Key configuration decisions:**
- **`--node-private-networking`**: Workers get no public IPs, all traffic routes through NAT Gateway
- **`--spot`**: 70% cost savings using spot instances 
- **`--asg-access`**: Enables Cluster Autoscaler to manage node groups
- **`supportType=STANDARD`**: Avoids extended support costs ($50/month savings)

### Step 3: Create the Autoscaler Patch Configuration

Create the patch file for customizing the Cluster Autoscaler:

**File: `infra/test/eksctl/base/cluster-autoscaler-patch.yaml`**
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

**Critical settings explained:**
- **`v1.32.2`**: Specific version that works with EKS 1.32+
- **`expander=least-waste`**: Chooses most cost-effective scaling option
- **`balance-similar-node-groups`**: Distributes pods across availability zones
- **`safe-to-evict: false`**: Prevents autoscaler from scaling itself down

### Step 4: Run the Cluster Creation

Execute the script to create your base cluster:

```bash
chmod +x create-cluster.sh
./create-cluster.sh
```

This will:
1. Create the EKS cluster with spot instances and private networking
2. Configure standard support tier for cost savings
3. Set up kubectl access to your new cluster
4. Enable OIDC provider for secure service account authentication

The cluster creation takes 15-20 minutes. You'll see progress output from eksctl as it creates the underlying CloudFormation stacks.

### Step 5: Configure Cluster Autoscaler

Set up the IAM service account and install the Cluster Autoscaler:

```bash
# Create IAM service account for autoscaler
eksctl create iamserviceaccount \
    --name cluster-autoscaler \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn arn:aws:iam::aws:policy/AutoScalingFullAccess \
    --approve

# Download and apply autoscaler manifest
curl -o cluster-autoscaler-autodiscover.yaml \
    https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

**Security win**: No AWS keys in pods. The Cluster Autoscaler gets temporary credentials through the OIDC provider.

### Step 6: Apply Autoscaler Optimizations

Patch the autoscaler deployment with our optimized configuration:

```bash
# Apply our cluster-specific patch
sed "s/CLUSTER_NAME_PLACEHOLDER/$CLUSTER_NAME/g" \
    cluster-autoscaler-patch.yaml | \
    kubectl patch deployment cluster-autoscaler -n kube-system --patch-file=/dev/stdin
```

This configures:
- **Image version**: `v1.32.2` (matches EKS version compatibility)
- **Expander strategy**: `least-waste` (cost optimization)
- **Balance similar node groups**: Better spot instance availability

### Step 7: Validate Your Cluster

Verify everything is working correctly:

```bash
# Check cluster status
kubectl get nodes
kubectl get pods -A

# Verify autoscaler is running
kubectl get deployment cluster-autoscaler -n kube-system
kubectl logs -f deployment/cluster-autoscaler -n kube-system

# Check OIDC provider
aws eks describe-cluster --name your-cluster-name \
    --query "cluster.identity.oidc.issuer"
```

If you see nodes in `Ready` state and the autoscaler pod running, you're good to go.

![Cluster Validation](/assets/images/full-stack/1.1step7.png)
*Step 7: Successful cluster validation showing Ready nodes and running autoscaler*

### Step 8: Test Your Cluster (Optional)

Let's validate that everything actually works by deploying a test application and demonstrating the autoscaling functionality.

Deploy a simple test application:

**File: `tests/base/hello-world-nginx.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-nginx
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world-nginx
  template:
    metadata:
      labels:
        app: hello-world-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-content
        configMap:
          name: hello-world-html

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-world-html
  namespace: default
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <title>EKS Test Cluster</title>
        <style>
            body { font-family: Arial; text-align: center; margin-top: 100px; background: #f0f0f0; }
            .container { background: white; padding: 40px; border-radius: 10px; display: inline-block; box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🚀 Hello from EKS Test Cluster!</h1>
            <p><strong>✅ Cluster Autoscaler working</strong></p>
            <p><strong>✅ Spot instances running</strong></p>
            <p><strong>✅ Private networking enabled</strong></p>
        </div>
    </body>
    </html>

---
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
  namespace: default
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/load-balancer-source-ranges: "0.0.0.0/0"  # Restrict this to your IP range
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: hello-world-nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello-world-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-world-nginx
  minReplicas: 1
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```
The `service.beta.kubernetes.io/aws-load-balancer...` just gives you the AWS network load balancer specific for this test app (so no ingress controller yet).

Apply the test deployment:

```bash
kubectl apply -f hello-world-nginx.yaml

# Check deployment status
kubectl get pods
kubectl get deployment hello-world-nginx
```

Confirm it works!

![Test Application Running](/assets/images/full-stack/1.1step8.png)
*Step 8: Test application successfully deployed and accessible via Network Load Balancer*

You can always get your LB's domain name with `kubectl get svc hello-world-service`.


### Step 9: Test Autoscaling Functionality (Optional)

Now let's prove the Cluster Autoscaler actually works by triggering scale events:

**File: `tests/base/test-autoscaling.sh`**
```bash
#!/bin/bash

echo "🧪 Testing Cluster Autoscaler"
echo "============================"
echo "This script will test automatic node scaling up and down"

echo "📊 Current cluster state:"
kubectl get nodes
echo "Pod count: $(kubectl get pods --all-namespaces --no-headers | wc -l)"

echo "⬆️  PHASE 1: Testing scale UP..."
echo "Creating high resource demand to trigger node addition..."

# Create a deployment that will require more nodes
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: autoscale-test
spec:
  replicas: 20
  selector:
    matchLabels:
      app: autoscale-test
  template:
    metadata:
      labels:
        app: autoscale-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
EOF

echo "⏳ Waiting for pods to be scheduled and autoscaler to react..."
sleep 30

echo "📊 Checking for pending pods (should trigger scale-up)..."
kubectl get pods --field-selector=status.phase=Pending -o wide

echo "📈 Checking autoscaler logs for scale-up events..."
kubectl logs deployment/cluster-autoscaler -n kube-system --tail=10 | grep -i "triggeredscaleup" || echo "Scale-up in progress..."

echo "⏳ Waiting 2 minutes for node provisioning..."
sleep 120

echo "📊 Cluster state after scale-up:"
kubectl get nodes
echo "Total nodes: $(kubectl get nodes --no-headers | wc -l)"

echo "⬇️  PHASE 2: Testing scale DOWN..."
echo "Removing resource demand to trigger node removal..."

kubectl delete deployment autoscale-test

echo "✅ Autoscaling test completed!"
echo "💡 If you see 3+ nodes, scale-up worked!"
echo "💡 Nodes will automatically scale down in ~10 minutes when unneeded"
```

Run the autoscaling test:

```bash
chmod +x test-autoscaling.sh
./test-autoscaling.sh
```

**What this test proves:**
- Cluster Autoscaler detects resource pressure and adds nodes
- Spot instances are successfully provisioned
- Node scale-down happens automatically (with 10-minute delay for stability)
- Your OIDC and IAM service account configuration works correctly

Monitor the autoscaler in real-time:

```bash
# Watch autoscaler decisions
kubectl logs deployment/cluster-autoscaler -n kube-system -f

# Watch node changes
kubectl get nodes -w
```

*Multiple pods are pending -> Nodes scale from 1 to 2 -> Nodes are able to complete the requested resources*
![Autoscaling Demonstration](/assets/images/full-stack/1.1step9.png)

## What We've Accomplished

You now have a production-grade EKS foundation that:

- **Scales intelligently**: 1-5 t3.medium spot instances with Cluster Autoscaler v1.32.2
- **Stays secure**: Private networking, OIDC provider, zero long-lived credentials
- **Costs ~$130/month minimum**: Spot instances + standard support optimization (extended is unnecessary expense)
- **Ready for addons**: Modular architecture supports SSL ingress, storage, secrets

To delete the testing resources, run:
```bash
kubectl delete deploy hello-world-nginx
kubectl delete horizontalpodautoscaler.autoscaling/hello-world-hpa
kubectl delete service hello-world-service

# Confirm all testing resources are deleted
kubectl get all
```

## Quick Troubleshooting

**Nodes stuck in NotReady?**
```bash
kubectl describe node <node-name>
# Usually: CNI plugin issues or instance launch failures
```

**Autoscaler not scaling?**
```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
# Look for IAM permission errors or ASG tag issues
```

**Spot instances getting terminated too often?**
```bash
# Check interruption rate in your region/AZ
aws ec2 describe-spot-price-history --instance-types t3.medium
# Consider mixed instance types in production
```

## Production Hardening Checklist

Now keep in mind that although this cluster is in good shape, I am considering this my "TEST cluster" that will have all public apps limited to access from my IPs only, primarily due to uknown security state of applications
as they are entering non-production environment. 

For an actual production use, consider adding these:

- **API server access**: Restrict `--endpoint-access` to your networks
- **Logging**: Enable CloudWatch logging for audit/diagnostic
- **Network policies**: Implement microsegmentation between namespaces
- **Pod security standards**: Enforce restricted security policies
- **Monitoring**: CloudWatch Container Insights or Prometheus stack
- **Backup strategy**: Velero for cluster state backup

## Next Steps

Your cluster is running, but it's still isolated from the world. In the next article, we'll install the AWS Load Balancer Controller and set up SSL/TLS termination with AWS Certificate Manager.

Here's what's coming:
- SSL certificate automation with ACM
- Intelligent traffic routing with ALB
- Production-ready ingress patterns
- Real applications accessible at HTTPS endpoints

The modular foundation we built today makes adding these capabilities straightforward - just run the install script from the ingress addon directory and you're ready for SSL.

---

**Coming Next**: [EKS Ingress: Modern Load Balancing and SSL Management](/posts/coming-soon) - Transform your private cluster into a secure, public-facing platform.

*Questions about the implementation? Spotted an issue with the configuration? Reach out to me on socials.*