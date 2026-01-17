---
layout: post
title: "1.3 EKS: Ingress Controller with SSL and Load Balancing"
date: 2026-01-17 13:00:00 -0000
categories: kubernetes eks infrastructure aws
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS", "Networking"]
series: "EKS Infrastructure Series"
series_part: "1.3"
---

With a running EKS cluster, the next step is exposing services to the internet. The AWS Load Balancer Controller creates and manages Application Load Balancers (ALB) directly from Kubernetes Ingress resources - no manual ALB configuration required.

This is part 1.3 of the EKS Infrastructure Series. We're building on the base cluster from [1.2 EKS Base Cluster](/2026/01/17/eks-base-cluster/).

## What We're Building

By the end of this article, you'll have:

- **AWS Load Balancer Controller** managing ALBs from Kubernetes
- **Automatic SSL/TLS** via AWS Certificate Manager integration
- **Subnet auto-discovery** so ALBs deploy to the right subnets
- **IAM Roles for Service Accounts (IRSA)** for secure AWS API access

## Why AWS Load Balancer Controller?

Kubernetes has a built-in Ingress resource type, but it needs a controller to actually do anything. You have several options:

| Controller | Pros | Cons |
|------------|------|------|
| AWS Load Balancer Controller | Native ALB/NLB, ACM integration, WAF support | AWS-specific |
| nginx-ingress | Portable, feature-rich | Extra hop, no native ALB |
| Traefik | Auto-discovery, middleware | Extra hop, learning curve |

The AWS Load Balancer Controller wins for EKS because:

**Native ALB integration** - Each Ingress creates an actual AWS ALB. No proxy layer, no extra hops. Traffic goes directly from ALB to pods.

**ACM certificates** - Reference certificates by ARN. No cert-manager, no secrets to manage, automatic renewal handled by AWS.

**Target type: IP** - ALB routes directly to pod IPs, bypassing NodePort. Lower latency, better load distribution.

**AWS WAF integration** - Attach WAF rules to ALBs for DDoS protection and request filtering.

## How It Works

When you create a Kubernetes Ingress with the right annotations, here's what happens:

<div class="mermaid">
graph LR
    I[Ingress Resource] --> C[LB Controller]
    C --> ALB[Creates ALB]
    C --> TG[Creates Target Group]
    C --> R[Creates Listener Rules]
    ALB --> P1[Pod 1]
    ALB --> P2[Pod 2]
</div>

The controller watches for Ingress resources and translates them into AWS infrastructure:

1. **Ingress created** - You apply an Ingress manifest with ALB annotations
2. **Controller detects** - Watches the Kubernetes API for Ingress events
3. **ALB provisioned** - Creates an Application Load Balancer in your VPC
4. **Target group created** - Registers pod IPs as targets (not node IPs)
5. **Listeners configured** - Sets up HTTP/HTTPS listeners with your rules
6. **DNS available** - ALB gets a DNS name you can point your domain to

## Prerequisites

Before installing:

```bash
# Base cluster running (from 1.2)
kubectl get nodes

# Helm v3 for installation
helm version

# AWS CLI configured
aws sts get-caller-identity
```

## Project Structure

```
ingress-controller/
├── install-alb-controller.sh    # Installation script
└── delete-alb-controller.sh     # Cleanup script
```

## Step 1: Create the Installation Script

This script handles the complete setup including IAM permissions and subnet tagging:

**install-alb-controller.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"
CONTROLLER_VERSION="1.17.1"
POLICY_VERSION="v2.17.1"

echo "Installing AWS Load Balancer Controller"
echo "========================================"
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo "Controller Version: $CONTROLLER_VERSION"
echo ""

echo "Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "ERROR: Base cluster not found or kubectl not configured"
    echo "   Run this first: cd ../base && ./create-cluster.sh"
    exit 1
fi

if ! command -v helm &> /dev/null; then
    echo "ERROR: Helm is required but not installed"
    echo "   Install Helm: https://helm.sh/docs/intro/install/"
    exit 1
fi

echo "Prerequisites verified"
echo ""

echo "Checking cluster subnet tags for load balancer discovery..."
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION \
    --query "cluster.resourcesVpcConfig.vpcId" --output text)

PUBLIC_SUBNETS=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
              "Name=tag:kubernetes.io/role/elb,Values=1" \
    --region $REGION --query "Subnets[].SubnetId" --output text)

PRIVATE_SUBNETS=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
              "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
    --region $REGION --query "Subnets[].SubnetId" --output text)

if [[ -z "$PUBLIC_SUBNETS" ]]; then
    echo "WARNING: Public subnets missing load balancer tags"
    echo "   Adding required tags for ALB discovery..."
    aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" \
        --region $REGION \
        --query "Subnets[?MapPublicIpOnLaunch==\`true\`].SubnetId" \
        --output text | xargs -n1 -I {} \
        aws ec2 create-tags --resources {} \
            --tags Key=kubernetes.io/role/elb,Value=1
fi

if [[ -z "$PRIVATE_SUBNETS" ]]; then
    echo "WARNING: Private subnets missing load balancer tags"
    echo "   Adding required tags for internal ALB discovery..."
    aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" \
        --region $REGION \
        --query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId" \
        --output text | xargs -n1 -I {} \
        aws ec2 create-tags --resources {} \
            --tags Key=kubernetes.io/role/internal-elb,Value=1
fi

echo "Subnet tags verified"
echo ""

echo "Downloading latest IAM policy ($POLICY_VERSION)..."
curl -fsSL -o iam-policy.json \
    https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/$POLICY_VERSION/docs/install/iam_policy.json

if [[ ! -f iam-policy.json ]]; then
    echo "ERROR: Failed to download IAM policy"
    exit 1
fi

echo "Creating/updating IAM policy..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy"

# Delete existing policy if it exists (to update with latest permissions)
aws iam delete-policy --policy-arn $POLICY_ARN 2>/dev/null || true

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to create IAM policy"
    exit 1
fi

echo "IAM policy created successfully"
echo ""

echo "Creating IAM service account with IRSA..."
# Delete existing service account to ensure clean setup
eksctl delete iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --wait 2>/dev/null || true

eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=$POLICY_ARN \
    --override-existing-serviceaccounts \
    --region=$REGION \
    --approve

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to create service account"
    exit 1
fi

echo "Service account created with proper IAM role"
echo ""

echo "Installing AWS Load Balancer Controller via Helm..."
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Uninstall existing version if present
helm uninstall aws-load-balancer-controller -n kube-system 2>/dev/null || true

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=$CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set region=$REGION \
    --set vpcId=$VPC_ID \
    --version $CONTROLLER_VERSION \
    --wait

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to install controller"
    exit 1
fi

echo "Waiting for controller to be ready..."
kubectl wait --for=condition=Available --timeout=300s \
    deployment/aws-load-balancer-controller -n kube-system

# Clean up temporary file
rm -f iam-policy.json

echo ""
echo "SUCCESS! AWS Load Balancer Controller installed"
echo "================================================"
echo "Features enabled:"
echo "   - Application Load Balancer (ALB) support"
echo "   - Network Load Balancer (NLB) support"
echo "   - SSL/TLS with AWS Certificate Manager"
echo "   - Advanced routing and traffic management"
echo ""
echo "Verify with:"
echo "   kubectl get deployment -n kube-system aws-load-balancer-controller"
```

## Understanding Subnet Tags

The ALB controller needs to know which subnets to use. It discovers them via tags:

| Tag | Value | Purpose |
|-----|-------|---------|
| `kubernetes.io/role/elb` | `1` | Public subnets for internet-facing ALBs |
| `kubernetes.io/role/internal-elb` | `1` | Private subnets for internal ALBs |

eksctl usually adds these tags automatically, but the script checks and adds them if missing. Without these tags, the controller can't create load balancers.

## Understanding IRSA (IAM Roles for Service Accounts)

The controller needs AWS permissions to create ALBs, target groups, and security groups. Instead of using node-level IAM roles (which would give all pods on that node the same permissions), we use IRSA:

<div class="mermaid">
graph LR
    SA[Service Account] --> OIDC[OIDC Provider]
    OIDC --> IAM[IAM Role]
    IAM --> ALB[ALB Permissions]
    Pod --> SA
</div>

How IRSA works:

1. **OIDC provider** - EKS cluster has an OpenID Connect provider (we set this up in 1.2)
2. **IAM role** - Created with a trust policy that only allows the specific service account to assume it
3. **Service account annotation** - Kubernetes service account has an annotation pointing to the IAM role ARN
4. **Token injection** - EKS injects a web identity token into pods using this service account
5. **Credential exchange** - AWS SDK exchanges the token for temporary IAM credentials

The result: only pods running as `aws-load-balancer-controller` service account get ALB management permissions. No credentials stored, no overly permissive node roles.

## Step 2: Create the Cleanup Script

**delete-alb-controller.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "Removing AWS Load Balancer Controller"
echo "======================================"
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""

echo "Checking for existing ingresses that use ALB..."
INGRESSES=$(kubectl get ingress -A --no-headers 2>/dev/null | wc -l)
if [ "$INGRESSES" -gt 0 ]; then
    echo "WARNING: Found $INGRESSES ingress resource(s)"
    echo "   Delete these first to avoid orphaned ALBs:"
    kubectl get ingress -A
    echo ""
    read -p "Continue deletion anyway? (y/N): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        echo "Deletion cancelled"
        exit 1
    fi
fi

echo "Uninstalling AWS Load Balancer Controller..."
helm uninstall aws-load-balancer-controller -n kube-system 2>/dev/null || true

echo "Removing service account and IAM role..."
eksctl delete iamserviceaccount \
    --name=aws-load-balancer-controller \
    --namespace=kube-system \
    --cluster=$CLUSTER_NAME || true

echo "Removing IAM policy..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam delete-policy \
    --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    2>/dev/null || true

echo "Removing leftover webhook configurations..."
kubectl delete validatingwebhookconfiguration aws-load-balancer-webhook 2>/dev/null || true
kubectl delete mutatingwebhookconfiguration aws-load-balancer-webhook 2>/dev/null || true

echo ""
echo "Cleanup complete!"
echo "================="
echo "Removed:"
echo "   - AWS Load Balancer Controller deployment"
echo "   - IAM service account and role"
echo "   - IAM policy"
echo ""
echo "Manual cleanup may be required for:"
echo "   - Orphaned ALB/NLB resources (check AWS Console)"
echo "   - Security groups created by the controller"
```

The warning about orphaned ALBs is important. If you delete the controller while Ingress resources still exist, the ALBs remain in AWS but nothing manages them. Always delete Ingress resources first.

## Step 3: Run the Installation

```bash
chmod +x install-alb-controller.sh
./install-alb-controller.sh
```

Expected output at completion:
```
SUCCESS! AWS Load Balancer Controller installed
================================================
Features enabled:
   - Application Load Balancer (ALB) support
   - Network Load Balancer (NLB) support
   - SSL/TLS with AWS Certificate Manager
   - Advanced routing and traffic management

Verify with:
   kubectl get deployment -n kube-system aws-load-balancer-controller
```

## Step 4: Verify Installation

Check the controller is running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected output:
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           2m
```

Check the controller logs:
```bash
kubectl logs -f deployment/aws-load-balancer-controller -n kube-system
```

Look for:
```
{"level":"info","msg":"version","GitVersion":"v2.7.1"}
{"level":"info","msg":"Starting controller"}
```

## Step 5: Create a Test Ingress

Let's verify the controller works by creating an Ingress that exposes a simple nginx deployment:

```bash
# Create a test deployment
kubectl create deployment nginx-test --image=nginx --port=80
kubectl expose deployment nginx-test --port=80 --target-port=80

# Create an Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-test
            port:
              number: 80
EOF
```

Key annotations:

| Annotation | Value | Purpose |
|------------|-------|---------|
| `kubernetes.io/ingress.class` | `alb` | Use the AWS Load Balancer Controller |
| `scheme` | `internet-facing` | Public ALB (use `internal` for private) |
| `target-type` | `ip` | Route directly to pod IPs (recommended) |

### Watch the ALB Get Created

```bash
kubectl get ingress nginx-test-ingress -w
```

After a few minutes, you'll see an ADDRESS:
```
NAME                 CLASS   HOSTS   ADDRESS                                                      PORTS   AGE
nginx-test-ingress   <none>  *       k8s-default-nginxtes-xxxxx.us-east-1.elb.amazonaws.com       80      3m
```

### Test the ALB

```bash
# Get the ALB URL
ALB_URL=$(kubectl get ingress nginx-test-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test it
curl http://$ALB_URL
```

You should see the nginx welcome page.

### Clean Up Test Resources

```bash
kubectl delete ingress nginx-test-ingress
kubectl delete service nginx-test
kubectl delete deployment nginx-test
```

## Adding HTTPS (Requires Custom Domain)

The ALB DNS name (`k8s-xxx.elb.amazonaws.com`) is owned by AWS, so you can't get an SSL certificate for it. HTTPS requires a domain you own.

**Why?** ACM (AWS Certificate Manager) validates that you control the domain before issuing a certificate. Since AWS owns the `.elb.amazonaws.com` domain, you can't prove ownership.

If you have a custom domain, here's how to enable HTTPS:

### Request an ACM Certificate

```bash
# Request a certificate for your domain
aws acm request-certificate \
    --domain-name "*.example.com" \
    --subject-alternative-names "example.com" \
    --validation-method DNS \
    --region us-east-1
```

Complete DNS validation by adding the CNAME record ACM provides to your domain's DNS.

### Create an HTTPS Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

HTTPS-specific annotations:

| Annotation | Value | Purpose |
|------------|-------|---------|
| `listen-ports` | `[{"HTTP":80},{"HTTPS":443}]` | Accept both HTTP and HTTPS |
| `ssl-redirect` | `443` | Redirect HTTP to HTTPS automatically |
| `certificate-arn` | Your ACM cert ARN | TLS certificate for HTTPS |
| `ssl-policy` | `ELBSecurityPolicy-TLS13-1-2-2021-06` | Modern TLS 1.2/1.3 only |

### Point Your Domain to the ALB

Create a CNAME or Route53 alias record pointing your domain to the ALB DNS name.

### SSL Architecture

With HTTPS configured, the ALB handles SSL termination:

<div class="mermaid">
graph LR
    U[User] -->|HTTPS 443| ALB[ALB]
    ALB -->|HTTP 80| P1[Pod]
    ALB -->|HTTP 80| P2[Pod]
</div>

This is the recommended pattern because:

- **Traffic inside VPC is isolated** - Pods are in private subnets with no public access
- **No TLS overhead on pods** - Your application doesn't need to handle certificates
- **Simpler certificate management** - ACM handles renewal automatically
- **Better performance** - ALB is optimized for TLS termination

### SSL Best Practices

When you do add HTTPS:

- **Use modern TLS policies** - `ELBSecurityPolicy-TLS13-1-2-2021-06` supports TLS 1.2 and 1.3 only
- **Always redirect HTTP to HTTPS** - The `ssl-redirect: '443'` annotation handles this
- **Include apex domain in certificate** - Wildcard certs (`*.example.com`) don't cover `example.com`, so add it as a SAN

## Common Ingress Annotations

| Annotation | Purpose | Example |
|------------|---------|---------|
| `alb.ingress.kubernetes.io/scheme` | Public or internal | `internet-facing` or `internal` |
| `alb.ingress.kubernetes.io/target-type` | Route to node or pod | `ip` (recommended) or `instance` |
| `alb.ingress.kubernetes.io/healthcheck-path` | Health check endpoint | `/health` |
| `alb.ingress.kubernetes.io/success-codes` | Healthy response codes | `200-299` |
| `alb.ingress.kubernetes.io/load-balancer-attributes` | ALB settings | `idle_timeout.timeout_seconds=60` |

## Cost Breakdown

Each ALB adds to your monthly bill:

| Component | Cost (approx) |
|-----------|---------------|
| ALB base cost | $16.20/month |
| LCU (Load Balancer Capacity Units) | ~$5.84/month per LCU |

For light traffic, expect ~$20-25/month per ALB. Consider sharing one ALB across multiple services using path-based or host-based routing.

## Troubleshooting

### Ingress Stuck Without Address

Check controller logs:
```bash
kubectl logs -f deployment/aws-load-balancer-controller -n kube-system
```

Common causes:
- **Subnet tags missing** - Controller can't find subnets for ALB
- **IAM permissions** - Service account role missing required policies
- **Security group limits** - Reached AWS security group quota

### ALB Not Routing Traffic

Check target group health:
```bash
# Get target group ARN from ALB
aws elbv2 describe-target-groups --names k8s-default-myapp-xxx

# Check target health
aws elbv2 describe-target-health --target-group-arn TARGET_GROUP_ARN
```

If targets are unhealthy:
- Verify health check path returns 200
- Check security groups allow traffic from ALB to pods
- Ensure pods are actually running and ready

### Certificate Not Working

- Verify certificate is in the same region as the cluster
- Check certificate status is "Issued" (not pending validation)
- Ensure the domain matches your Ingress host

## What We've Accomplished

You now have:

- AWS Load Balancer Controller managing ALBs from Kubernetes
- IRSA providing secure AWS credentials to the controller
- Subnet auto-discovery for ALB placement
- SSL/TLS support via ACM integration
- A working pattern for exposing services to the internet

## Next Steps

With ingress working, the cluster needs persistent storage for stateful workloads. In the next article, we'll set up the EBS CSI driver for dynamic volume provisioning.

**Next**: [1.4 EKS EBS Storage - Persistent Volumes for Stateful Apps](/2026/01/17/eks-ebs-storage/)

---

*Questions about ingress configuration? Reach out on socials.*
