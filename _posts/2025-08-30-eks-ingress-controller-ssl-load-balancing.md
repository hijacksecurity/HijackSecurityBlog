---
layout: post
title: "1.2 EKS: Ingress Controller with SSL and Load Balancing"
date: 2025-08-29 12:00:00 -0000
categories: kubernetes eks infrastructure security aws
tags: ["Modern Full-Stack Development", "Infrastructure", "AWS", "Kubernetes", "Security"]
series: "Modern Full-Stack Development"
series_part: "1.2"
---

You've got your EKS cluster running with auto-scaling nodes. Great. But it's still isolated from the world - your applications are trapped inside, accessible only through kubectl port-forwarding or ugly NodePort services. Time to fix that with proper ingress, SSL certificates, and intelligent load balancing.

The AWS Load Balancer Controller is the bridge between your Kubernetes services and the AWS ecosystem. It creates Application Load Balancers (ALBs) and Network Load Balancers (NLBs) that integrate seamlessly with AWS Certificate Manager, WAF, and Route 53. No more manual load balancer configuration or certificate rotation nightmares.

This is part 1.2 of our Modern Full-Stack Development series. We're adding production-grade ingress to the foundation we built in [1.1 EKS: Kubernetes Base and Node Autoscaling](/2025/08/29/eks-foundation-building-modular-production-cluster/).

## What We're Building

By the end of this walkthrough, you'll have:

- **AWS Load Balancer Controller**: Latest version (1.13.0) with proper IAM configuration
- **SSL/TLS Termination**: Automatic certificate management with AWS Certificate Manager
- **Intelligent Routing**: Path-based and host-based routing with ALB
- **Security Features**: IP whitelisting, security headers, WAF integration
- **Health Checks**: Automatic target health monitoring and failover
- **Cost Optimization**: Shared ALBs across multiple services

## Prerequisites Check

Before diving in, ensure you have:

1. **Base EKS cluster** from [part 1.1](/2025/08/29/eks-foundation-building-modular-production-cluster/) running
2. **Helm** installed (v3.12+ recommended)
3. **Domain name** for SSL certificates (we'll use AWS Certificate Manager)
4. **AWS CLI** configured with appropriate permissions

## The Architecture

Here's how the AWS Load Balancer Controller transforms your cluster:

### Before: Isolated Services
```
Internet → ❌ (No access)
         ↓
    EKS Cluster
    ├── Service A (ClusterIP)
    ├── Service B (ClusterIP)
    └── Service C (ClusterIP)
```

### After: Production-Ready Ingress
```
Internet → Route 53 → ALB (with SSL)
                       ├── /api → Service A
                       ├── /app → Service B
                       └── /    → Service C
```

The controller watches for Ingress resources and automatically provisions ALBs with the right configuration. No manual AWS console work required.

## Step 1: Verify Subnet Tagging

ALBs need to know which subnets to use. The controller discovers them through tags:

```bash
# Get your cluster's VPC ID
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"
VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION \
    --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Check public subnet tags (for internet-facing ALBs)
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    "Name=tag:kubernetes.io/role/elb,Values=1" \
    --region $REGION --query "Subnets[].SubnetId"

# Check private subnet tags (for internal ALBs)
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
    --region $REGION --query "Subnets[].SubnetId"
```

If subnets aren't tagged, the controller can't create load balancers. The installation script handles this automatically.

## Step 2: Create the Installation Script

Create the AWS Load Balancer Controller installation script:

**File: `infra/test/eksctl/ingress-controller/install-alb-controller.sh`**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"
CONTROLLER_VERSION="1.13.0"  # Latest stable version
POLICY_VERSION="v2.13.3"     # Latest IAM policy version

echo "🔒 Installing AWS Load Balancer Controller"
echo "=========================================="
echo "Cluster: $CLUSTER_NAME"
echo "Controller Version: $CONTROLLER_VERSION"

# Download latest IAM policy
curl -fsSL -o iam-policy.json \
    https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/$POLICY_VERSION/docs/install/iam_policy.json

# Create IAM policy
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy"

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

# Create IAM service account with IRSA
eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=$POLICY_ARN \
    --override-existing-serviceaccounts \
    --region=$REGION \
    --approve

# Install controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

VPC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --region $REGION \
    --query "cluster.resourcesVpcConfig.vpcId" --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=$CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set region=$REGION \
    --set vpcId=$VPC_ID \
    --version $CONTROLLER_VERSION \
    --wait

# Verify installation
kubectl wait --for=condition=Available --timeout=300s \
    deployment/aws-load-balancer-controller -n kube-system

echo "✅ AWS Load Balancer Controller installed successfully!"
```

## Step 3: Run the Installation

Execute the installation script:

```bash
chmod +x install-alb-controller.sh
./install-alb-controller.sh
```

This process:
1. Downloads the latest IAM policy with all required permissions
2. Creates an IAM role for the controller (no hardcoded credentials!)
3. Installs the controller using Helm with production settings
4. Waits for the controller to be fully operational

![AWS Load Balancer Controller Installation](/assets/images/full-stack/1.2install.png)
*AWS Load Balancer Controller successfully installed with all production features enabled*

## Step 4: Request an SSL Certificate

Before creating ingress resources, set up SSL with AWS Certificate Manager:

```bash
# Request a wildcard certificate for your domain
DOMAIN="*.your-domain.com"
CERT_ARN=$(aws acm request-certificate \
    --domain-name "$DOMAIN" \
    --validation-method DNS \
    --region us-east-1 \
    --query 'CertificateArn' \
    --output text)

echo "Certificate ARN: $CERT_ARN"

# Get DNS validation records
aws acm describe-certificate \
    --certificate-arn "$CERT_ARN" \
    --region us-east-1 \
    --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

Add the CNAME record to your DNS provider and wait for validation (usually 5-30 minutes).

![DNS CNAME Configuration](/assets/images/full-stack/1.2cname.png)
*DNS CNAME record configuration for ACM certificate validation*

## Step 5: Deploy a Demo Application

Let's deploy a demo app with SSL ingress:

**File: `tests/ingress/demo-app.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
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
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: default
  annotations:
    # ALB Configuration
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: my-apps
    
    # SSL Configuration
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/certificate-arn: "YOUR_CERTIFICATE_ARN"
    
    # Security - IP Whitelisting
    alb.ingress.kubernetes.io/inbound-cidrs: "0.0.0.0/0"  # Restrict this!
    
    # Health Check Configuration
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/success-codes: '200'
spec:
  ingressClassName: alb
  rules:
  - host: demo.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-app-service
            port:
              number: 80
```

Deploy the application:

```bash
# Replace with your certificate ARN
sed -i "s|YOUR_CERTIFICATE_ARN|$CERT_ARN|g" demo-app.yaml

# Deploy
kubectl apply -f demo-app.yaml

# Watch the ALB creation
kubectl get ingress demo-app-ingress -w
```

## Step 6: Configure DNS

Once the ALB is created, point your domain to it:

```bash
# Get the ALB DNS name
ALB_DNS=$(kubectl get ingress demo-app-ingress \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "ALB DNS: $ALB_DNS"
```

Create a Route 53 CNAME record:
- Name: `demo.your-domain.com`
- Type: CNAME
- Value: The ALB DNS name

![Load Balancer DNS Configuration](/assets/images/full-stack/1.2cname-lb.png)
*CNAME record (can be a wildcard) pointing your domain to the Application Load Balancer*

## Step 7: Verify SSL and Access

Test your secure endpoint:

```bash
# Check SSL certificate
curl -I https://demo.your-domain.com

# Should see:
# HTTP/2 200
# strict-transport-security: max-age=31536000

# Check health status
aws elbv2 describe-target-health \
    --target-group-arn $(aws elbv2 describe-target-groups \
        --region us-east-1 \
        --query "TargetGroups[?contains(TargetGroupName, 'k8s')].TargetGroupArn" \
        --output text | head -1)
```

![ALB Network Mapping](/assets/images/full-stack/1.2networkmap.png)
*Application Load Balancer showing healthy targets and routing configuration*

When everything is working correctly, you should see your demo application:

![Working Demo Application](/assets/images/full-stack/1.2workingdemo.png)
*Demo application successfully deployed with SSL/TLS, load balancing, and security features*

## Advanced Ingress Patterns

### Pattern 1: Path-Based Routing
```yaml
spec:
  rules:
  - host: api.your-domain.com
    http:
      paths:
      - path: /users
        backend:
          service:
            name: user-service
      - path: /orders
        backend:
          service:
            name: order-service
      - path: /
        backend:
          service:
            name: frontend-service
```

### Pattern 2: Host-Based Routing
```yaml
spec:
  rules:
  - host: api.your-domain.com
    http:
      paths:
      - backend:
          service:
            name: api-service
  - host: app.your-domain.com
    http:
      paths:
      - backend:
          service:
            name: app-service
```

### Pattern 3: Shared ALB (Cost Optimization)
```yaml
annotations:
  # Same group name = same ALB
  alb.ingress.kubernetes.io/group.name: shared-alb
  alb.ingress.kubernetes.io/group.order: '10'
```

## Security Best Practices

### IP Whitelisting
```yaml
# Only allow specific IPs
alb.ingress.kubernetes.io/inbound-cidrs: "203.0.113.0/24,198.51.100.0/24"
```

### Security Headers
```yaml
alb.ingress.kubernetes.io/actions.response-headers: |
  {
    "X-Frame-Options": "DENY",
    "X-Content-Type-Options": "nosniff",
    "Strict-Transport-Security": "max-age=31536000"
  }
```

### WAF Integration
```yaml
# Attach AWS WAF Web ACL
alb.ingress.kubernetes.io/wafv2-acl-arn: "arn:aws:wafv2:..."
```

## Cost Optimization Tips

1. **Share ALBs**: Use `group.name` to share ALBs across services (~$16/month per ALB saved)
2. **Target type IP**: More efficient than instance mode for pod-to-pod communication
3. **Deregistration delay**: Set to 30s for faster deployments
4. **Idle timeout**: Adjust based on your longest requests (default 60s)

## Troubleshooting Guide

**ALB not creating?**
```bash
# Check controller logs
kubectl logs -f deployment/aws-load-balancer-controller -n kube-system

# Common issues:
# - Missing subnet tags
# - IAM permissions
# - Invalid certificate ARN
```

**Targets unhealthy?**
```bash
# Check pod readiness
kubectl get pods -o wide
kubectl describe pod <pod-name>

# Check security groups
# Ensure node security group allows traffic from ALB
```

**SSL not working?**
```bash
# Verify certificate status
aws acm describe-certificate --certificate-arn $CERT_ARN

# Check ingress annotations
kubectl describe ingress demo-app-ingress
```

## What We've Accomplished

Your EKS cluster now has:

- **Production-grade ingress**: AWS Load Balancer Controller managing ALBs automatically
- **SSL/TLS everywhere**: ACM certificates with automatic renewal
- **Intelligent routing**: Path and host-based routing with health checks
- **Security hardening**: IP whitelisting, security headers, TLS 1.3
- **Cost optimization**: Shared ALBs and efficient target routing

## Cleanup Commands

To remove the test resources:

```bash
# Delete demo app and ingress
kubectl delete -f demo-app.yaml

# ALB will be automatically deleted (may take 2-3 minutes)

# Keep the controller installed for future use
```

## Next Steps

Your cluster can now expose services securely to the internet. In the next article, we'll add persistent storage with the EBS CSI driver, enabling stateful applications like databases.

Coming up:
- Dynamic volume provisioning with EBS
- Storage classes for different performance tiers
- Snapshots and backup strategies
- StatefulSet patterns

The ingress foundation we built today makes your cluster truly production-ready for real-world applications.

---

**Coming Next**: [1.3 EKS: EBS Persistent Storage](/2025/08/29/eks-ebs-persistent-storage/) - Enable stateful workloads with dynamic volume provisioning.

*Found an issue? Have questions about the ingress configuration? Reach out on socials.*