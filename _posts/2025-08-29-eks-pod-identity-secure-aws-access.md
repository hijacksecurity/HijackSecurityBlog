---
layout: post
title: "1.4 EKS: Pod Identity - Secure AWS Service Access Without Secrets"
date: 2025-08-29 16:00:00 -0000
categories: kubernetes eks infrastructure security aws
tags: ["Modern Full-Stack Development", "Infrastructure", "AWS", "Kubernetes", "Security"]
series: "Modern Full-Stack Development"
series_part: "1.4"
---

Your applications running in Kubernetes need to access AWS services like S3, RDS, and SQS. Traditional approaches involve storing AWS access keys as secrets in your cluster - a security nightmare. EKS Pod Identity eliminates this completely by providing secure, temporary AWS credentials to your pods without any secrets.

No more hardcoded AWS keys. No more credential rotation headaches. No more security vulnerabilities from exposed secrets. Pod Identity uses AWS's native token exchange to provide temporary, auto-rotating credentials based on the pod's service account.

This is part 1.4 of our Modern Full-Stack Development series. We're adding secure AWS service access to the cluster we built in [1.1 EKS: Foundation](/2025/08/29/eks-foundation-building-modular-production-cluster/), [1.2 EKS: Ingress Controller](/2025/08/29/eks-ingress-controller-ssl-load-balancing/), and [1.3 EKS: EBS Storage](/2025/08/29/eks-ebs-persistent-storage/).

## What We're Building

By the end of this walkthrough, you'll have:

- **EKS Pod Identity Agent**: Native AWS credential provider for pods
- **Zero Secrets**: No AWS keys, tokens, or passwords stored in your cluster
- **Secure Token Exchange**: Kubernetes service accounts linked to IAM roles
- **Automatic Rotation**: AWS credentials refresh automatically every hour
- **S3 Demo Application**: Complete example showing secure S3 access without secrets

## Prerequisites Check

Before diving in, ensure you have:

1. **Base EKS cluster** from [part 1.1](/2025/08/29/eks-foundation-building-modular-production-cluster/) running
2. **AWS Load Balancer Controller** from [part 1.2](/2025/08/29/eks-ingress-controller-ssl-load-balancing/) installed
3. **AWS CLI** configured with IAM permissions for Pod Identity
4. **S3 bucket** or permission to create one for testing

## The Architecture

Here's how EKS Pod Identity transforms AWS access:

### Before: Secrets-Based Authentication
```
Pod → AWS Access Key (secret) → AWS API
     ↓
Security Risk: Keys can be extracted, don't rotate
```

### After: Pod Identity Authentication
```
Pod → K8s Service Account → Pod Identity Agent → AWS STS → Temporary Credentials
                         ↓
Kubernetes Token → AWS Token Exchange → Auto-rotating AWS credentials
```

The Pod Identity Agent runs as a DaemonSet and handles all credential exchange, providing AWS SDKs with temporary credentials that are automatically refreshed.

## Step 1: Install the EKS Pod Identity Agent

Create the installation script that sets up the Pod Identity Agent:

**File: `infra/test/eksctl/pod-identity/install-pod-identity.sh`**
```bash
#!/bin/bash

# EKS Pod Identity - Secure AWS access without secrets
# Provides IAM roles for service accounts using native EKS Pod Identity

# Configuration
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"

echo "🔐 Installing EKS Pod Identity for Secure AWS Access"
echo "=================================================="
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""

echo "🔍 Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "❌ ERROR: Base cluster not found or kubectl not configured"
    echo "   Run this first: cd ../base && ./create-cluster.sh"
    exit 1
fi

echo "✅ Prerequisites verified"
echo ""

echo "🔧 Installing EKS Pod Identity Agent..."
# Install the EKS Pod Identity Agent addon
eksctl create addon \
    --name eks-pod-identity-agent \
    --cluster $CLUSTER_NAME \
    --region $REGION

if [[ $? -ne 0 ]]; then
    echo "❌ ERROR: Failed to install Pod Identity Agent"
    exit 1
fi

echo "⏳ Waiting for Pod Identity Agent to be ready..."
kubectl wait --for=condition=Ready --timeout=300s pod -l app.kubernetes.io/name=eks-pod-identity-agent -n kube-system

echo ""
echo "✅ EKS Pod Identity installed successfully!"
echo "=========================================="
echo "🎯 Components installed:"
echo "   • EKS Pod Identity Agent (manages AWS credentials)"
echo "   • Native EKS integration (no IRSA needed)"
echo "   • Secure token exchange for pod authentication"
echo ""
echo "🔍 Verify installation:"
echo "   kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent"
echo "   kubectl get daemonset -n kube-system eks-pod-identity-agent"
echo ""
echo "📝 Next steps:"
echo "   1. Create IAM roles with trust policy for Pod Identity"
echo "   2. Associate roles with service accounts"
echo "   3. Deploy applications using the service accounts"
```

## Step 2: Run the Installation

Execute the installation script:

```bash
chmod +x install-pod-identity.sh
./install-pod-identity.sh
```

This installs the Pod Identity Agent as an EKS addon, which runs as a DaemonSet on every node to provide AWS credentials to pods.

```bash
# Verify the installation
kubectl get daemonset -n kube-system eks-pod-identity-agent
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

## Step 3: Create IAM Trust Policy

Pod Identity requires a special trust policy that allows the EKS Pod Identity service to assume the role:

**File: `tests/pod-identity/pod-identity-trust-policy.json`**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

This trust policy is different from traditional IAM roles - it specifically trusts the EKS Pod Identity service.

## Step 4: Deploy the S3 Demo Application

Let's create a comprehensive demo that shows secure S3 access without any secrets:

**File: `tests/pod-identity/s3-demo-policy.json`**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::REPLACE_WITH_S3_BUCKET",
                "arn:aws:s3:::REPLACE_WITH_S3_BUCKET/*"
            ]
        }
    ]
}
```

**File: `tests/pod-identity/deploy-demo.sh`** (truncated for brevity)
```bash
#!/bin/bash

# Pod Identity S3 Demo Deployment
# Demonstrates secure AWS S3 access without secrets using EKS Pod Identity

# Configuration
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"
S3_BUCKET="your-test-bucket-$(date +%s)"

echo "🔐 Deploying Pod Identity S3 Demo"
echo "================================="

# Create S3 bucket for demo
aws s3 mb s3://$S3_BUCKET --region $REGION
echo "Demo file created on $(date)" > /tmp/demo-file.txt
aws s3 cp /tmp/demo-file.txt s3://$S3_BUCKET/demo-file.txt

# Create IAM role with Pod Identity trust policy
ROLE_NAME="EKSPodIdentityS3DemoRole"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-role \
    --role-name $ROLE_NAME \
    --assume-role-policy-document file://pod-identity-trust-policy.json \
    --description "Pod Identity role for S3 demo access"

# Apply S3 policy to role
sed "s/REPLACE_WITH_S3_BUCKET/$S3_BUCKET/g" s3-demo-policy.json | \
aws iam put-role-policy \
    --role-name $ROLE_NAME \
    --policy-name S3DemoAccess \
    --policy-document file:///dev/stdin

# Deploy the demo application
kubectl apply -f demo-app.yaml

# Create pod identity association
eksctl create podidentityassociation \
    --cluster $CLUSTER_NAME \
    --namespace default \
    --service-account-name s3-demo-sa \
    --role-arn arn:aws:iam::$ACCOUNT_ID:role/$ROLE_NAME

echo "✅ Pod Identity S3 Demo deployed successfully!"
```

## Step 5: Create the Demo Application

The demo app includes two containers - one running nginx to serve a web interface, and another running AWS CLI to demonstrate S3 access:

**File: `tests/pod-identity/demo-app.yaml`** (key sections)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-demo-sa
  namespace: default
  labels:
    app: s3-demo

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-demo-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-demo-app
  template:
    metadata:
      labels:
        app: s3-demo-app
    spec:
      serviceAccountName: s3-demo-sa  # This links to the IAM role
      containers:
      - name: s3-demo
        image: nginx:alpine
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
        - name: shared-data
          mountPath: /usr/share/nginx/html/shared
      - name: aws-cli
        image: amazon/aws-cli:latest
        env:
        - name: S3_BUCKET
          value: "your-s3-bucket"
        - name: AWS_REGION
          value: "us-east-1"
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "Pod Identity S3 Demo - $(date)" > /shared/s3-test.log
            echo "AWS S3 List Bucket:" >> /shared/s3-test.log
            aws s3 ls s3://$S3_BUCKET/ >> /shared/s3-test.log 2>&1
            echo "AWS Identity:" >> /shared/s3-test.log
            aws sts get-caller-identity >> /shared/s3-test.log 2>&1
            sleep 30
          done
      volumes:
      - name: shared-data
        emptyDir: {}
```

## Step 6: Deploy and Test

Run the complete deployment:

```bash
chmod +x deploy-demo.sh
./deploy-demo.sh
```

This script:
1. Creates an S3 bucket with demo content
2. Creates an IAM role with the Pod Identity trust policy
3. Attaches S3 permissions to the role
4. Deploys the demo application
5. Associates the service account with the IAM role

## Step 7: Verify Secure AWS Access

Test that your application can access AWS services without any stored secrets:

```bash
# Check pod identity associations
eksctl get podidentityassociation --cluster your-cluster-name

# Verify no AWS credentials in pod environment
kubectl exec deployment/s3-demo-app -c aws-cli -- env | grep AWS
# Should show NO AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY

# Test S3 access
kubectl exec deployment/s3-demo-app -c aws-cli -- aws s3 ls s3://your-bucket/

# Check the AWS identity being used
kubectl exec deployment/s3-demo-app -c aws-cli -- aws sts get-caller-identity
# Should show the assumed role ARN
```

The key insight: your pod can access AWS services but has zero AWS credentials stored anywhere.

## Step 8: Understanding Pod Identity Flow

Here's what happens when your pod makes an AWS API call:

1. **Pod Startup**: Pod Identity Agent detects pods with associated service accounts
2. **Token Request**: AWS SDK requests credentials from metadata endpoint
3. **Token Exchange**: Agent exchanges Kubernetes token for AWS credentials
4. **API Call**: Pod uses temporary AWS credentials for S3 access
5. **Auto Refresh**: Credentials automatically refresh before expiration

## Security Benefits

### Zero Secrets Stored
```bash
# No AWS credentials anywhere in the cluster
kubectl get secrets --all-namespaces | grep -i aws
# Should return nothing related to AWS keys

# No environment variables with credentials
kubectl exec deployment/s3-demo-app -- env | grep -i secret
# Should show no AWS secrets
```

### Automatic Credential Rotation
```bash
# Credentials rotate automatically every hour
kubectl logs deployment/s3-demo-app -c aws-cli | grep "Expir"
# Shows credential expiration times
```

### Least Privilege Access
```json
{
    "Effect": "Allow",
    "Action": [
        "s3:GetObject",
        "s3:ListBucket"
    ],
    "Resource": "arn:aws:s3:::specific-bucket/*"
}
```

## Production Patterns

### Multiple Service Accounts for Different Permissions
```bash
# Frontend app - read-only S3 access
eksctl create podidentityassociation \
    --cluster $CLUSTER_NAME \
    --service-account-name frontend-sa \
    --role-arn arn:aws:iam::$ACCOUNT_ID:role/FrontendS3ReadOnlyRole

# Backend API - full database access
eksctl create podidentityassociation \
    --cluster $CLUSTER_NAME \
    --service-account-name backend-sa \
    --role-arn arn:aws:iam::$ACCOUNT_ID:role/BackendRDSFullAccessRole
```

### Cross-Account Access
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "aws:SourceAccount": "123456789012"
                }
            }
        }
    ]
}
```

### Monitoring Pod Identity Usage
```bash
# Check which pods are using Pod Identity
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceAccountName}{"\n"}{end}' | grep -v default

# Monitor Pod Identity Agent logs
kubectl logs -f daemonset/eks-pod-identity-agent -n kube-system
```

## Troubleshooting Guide

### Pod Identity Association Not Working
```bash
# Check association exists
eksctl get podidentityassociation --cluster $CLUSTER_NAME

# Verify service account in pod spec
kubectl get deployment s3-demo-app -o yaml | grep serviceAccountName

# Check Pod Identity Agent status
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

### AWS API Calls Failing
```bash
# Check if role can be assumed
aws sts assume-role \
    --role-arn arn:aws:iam::$ACCOUNT_ID:role/EKSPodIdentityS3DemoRole \
    --role-session-name test-session

# Verify IAM policy permissions
aws iam get-role-policy \
    --role-name EKSPodIdentityS3DemoRole \
    --policy-name S3DemoAccess
```

### Pod Identity Agent Issues
```bash
# Check agent logs
kubectl logs -f daemonset/eks-pod-identity-agent -n kube-system

# Verify addon status
eksctl get addon --cluster $CLUSTER_NAME --name eks-pod-identity-agent
```

## Pod Identity vs IRSA

**EKS Pod Identity (Recommended)**:
- Native EKS integration
- Simpler setup (no OIDC provider management)
- Better performance (fewer network calls)
- Automatic credential refresh

**IRSA (Legacy)**:
- Requires OIDC provider configuration
- More complex token exchange
- Still works but Pod Identity is preferred

## What We've Accomplished

Your EKS cluster now has:

- **Zero-secrets AWS access**: No AWS credentials stored anywhere in your cluster
- **Automatic security**: Credentials rotate every hour without intervention
- **Fine-grained permissions**: Different service accounts can have different AWS permissions
- **Production-ready patterns**: Secure, scalable AWS service integration
- **Comprehensive monitoring**: Full visibility into credential usage and rotation

## Cleanup Commands

To remove the test resources:

```bash
# Delete demo app
kubectl delete -f demo-app.yaml

# Delete pod identity association
eksctl delete podidentityassociation \
    --cluster $CLUSTER_NAME \
    --service-account-name s3-demo-sa

# Clean up AWS resources
aws s3 rb s3://your-bucket --force
aws iam delete-role-policy --role-name EKSPodIdentityS3DemoRole --policy-name S3DemoAccess
aws iam delete-role --role-name EKSPodIdentityS3DemoRole

# Keep Pod Identity Agent installed for future use
```

## Next Steps

Your cluster now provides secure, credential-free AWS service access for all your applications. In the next article, we'll add external secrets management with AWS Secrets Manager integration.

Coming up:
- External Secrets Operator with AWS Secrets Manager
- Automated secret rotation and management
- Database credentials and API key management
- Complete DevSecOps pipeline integration

The secure AWS access foundation we built today eliminates one of the biggest security risks in Kubernetes deployments - stored AWS credentials.

---

**Coming Next**: [1.5 EKS: External Secrets](/2025/08/29/eks-external-secrets-automated-secret-management/) - Integrate AWS Secrets Manager for automated secret rotation and management.

*Questions about Pod Identity setup or AWS security patterns? Reach out on socials.*