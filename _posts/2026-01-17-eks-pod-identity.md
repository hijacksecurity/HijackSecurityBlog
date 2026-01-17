---
layout: post
title: "1.5 EKS: Pod Identity for Secure AWS Access"
date: 2026-01-17 15:00:00 -0000
categories: kubernetes eks infrastructure aws security
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS", "Security", "IAM"]
series: "EKS Infrastructure Series"
series_part: "1.5"
---

Pods need AWS access - to read from S3, write to DynamoDB, or fetch secrets. The old way was storing AWS credentials in environment variables. That's a security nightmare. EKS Pod Identity lets pods assume IAM roles with no stored credentials.

This is part 1.5 of the EKS Infrastructure Series. We're building on the cluster from [1.4 EBS Storage](/2026/01/17/eks-ebs-storage/).

## What We're Building

By the end of this article, you'll have:

- **EKS Pod Identity Agent** installed as an EKS addon
- **Understanding** of how Pod Identity differs from IRSA
- **Pattern** for granting pods secure AWS access

## The Credential Problem

Kubernetes pods often need to call AWS APIs:

- Read configuration from S3
- Write logs to CloudWatch
- Access databases via RDS IAM auth
- Fetch secrets from Secrets Manager

The naive approach - embedding AWS access keys in environment variables or mounted secrets - has serious problems:

**Credentials can leak** - Environment variables show up in logs, crash dumps, and debugging tools.

**No automatic rotation** - Keys stay valid until manually revoked, even if compromised.

**Blast radius** - Long-lived credentials can be exfiltrated and used from anywhere.

**Audit difficulty** - Hard to trace which pod made which AWS API call.

## Two Solutions: IRSA vs Pod Identity

AWS provides two ways for pods to get IAM credentials without stored secrets:

### IRSA (IAM Roles for Service Accounts)

IRSA has been around since 2019. It uses OIDC (OpenID Connect) federation:

1. EKS exposes an OIDC provider endpoint
2. Service accounts get annotated with an IAM role ARN
3. Pods get a projected service account token
4. AWS STS exchanges the token for temporary credentials

IRSA works but has limitations:
- Requires setting up OIDC provider per cluster
- IAM trust policies reference OIDC provider URLs (can get complex)
- Token refresh handled by AWS SDK (older SDKs had issues)

### Pod Identity (Recommended)

Pod Identity launched in late 2023 as the simpler approach:

1. EKS Pod Identity Agent runs on each node
2. You create pod identity associations (role + service account)
3. Agent intercepts credential requests from pods
4. Agent returns temporary credentials from the associated role

**Why Pod Identity is better:**
- No OIDC setup required
- Simpler IAM trust policies
- Associations managed via EKS API (not annotations)
- Works with any AWS SDK version

## How Pod Identity Works

<div class="mermaid">
sequenceDiagram
    participant Pod
    participant Agent as Pod Identity Agent
    participant EKS as EKS Control Plane
    participant STS as AWS STS
    participant S3 as AWS Service

    Pod->>Agent: Request credentials (SDK call)
    Agent->>EKS: Validate pod identity
    EKS->>Agent: Return association (role ARN)
    Agent->>STS: AssumeRole
    STS->>Agent: Temporary credentials
    Agent->>Pod: Return credentials
    Pod->>S3: API call with credentials
</div>

The Pod Identity Agent runs as a DaemonSet, meaning one pod runs on every node. When a pod's AWS SDK requests credentials:

1. The SDK's credential chain calls the container credential endpoint
2. The agent intercepts this request
3. The agent checks EKS for a pod identity association matching the pod's service account
4. If found, the agent assumes the IAM role and returns temporary credentials
5. Credentials auto-refresh before expiration

No credentials are ever stored - they're generated on demand and cached briefly.

## Project Structure

```
pod-identity/
├── install-pod-identity.sh    # Installation script
└── delete-pod-identity.sh     # Cleanup script
```

Pod Identity is lightweight - just an EKS addon. The IAM configuration happens when you create associations for specific workloads.

## Step 1: Create the Installation Script

**install-pod-identity.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "Installing EKS Pod Identity"
echo "============================"
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""

echo "Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "ERROR: Cluster not found or kubectl not configured"
    exit 1
fi

echo "Prerequisites verified"
echo ""

echo "Installing EKS Pod Identity Agent addon..."
eksctl create addon \
    --name eks-pod-identity-agent \
    --cluster $CLUSTER_NAME \
    --region $REGION

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to install Pod Identity Agent"
    exit 1
fi

echo "Waiting for Pod Identity Agent to be ready..."
kubectl wait --for=condition=Ready --timeout=300s \
    pod -l app.kubernetes.io/name=eks-pod-identity-agent -n kube-system

echo ""
echo "SUCCESS! EKS Pod Identity installed"
echo "===================================="
echo "Components installed:"
echo "   - EKS Pod Identity Agent (DaemonSet)"
echo "   - Credential provider for pods"
echo ""
echo "Verify with:"
echo "   kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent"
echo "   kubectl get daemonset -n kube-system eks-pod-identity-agent"
```

## Step 2: Create the Cleanup Script

**delete-pod-identity.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "Removing EKS Pod Identity"
echo "========================="
echo "Cluster: $CLUSTER_NAME"
echo ""
echo "WARNING: This will remove:"
echo "   - Pod Identity Agent addon"
echo "   - Pods will lose AWS access via Pod Identity"
echo ""

read -p "Are you sure? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "Operation cancelled"
    exit 0
fi

echo ""
echo "Removing EKS Pod Identity Agent addon..."
eksctl delete addon \
    --name eks-pod-identity-agent \
    --cluster $CLUSTER_NAME \
    --region $REGION 2>/dev/null || echo "Addon not found"

echo "Waiting for cleanup..."
sleep 10

echo ""
echo "Cleanup complete!"
echo "================="
echo "Removed:"
echo "   - EKS Pod Identity Agent"
echo ""
echo "Note: IAM roles and pod identity associations remain"
echo "      (manual cleanup via AWS console or CLI)"
```

## Step 3: Run the Installation

```bash
chmod +x install-pod-identity.sh
./install-pod-identity.sh
```

Expected output:
```
SUCCESS! EKS Pod Identity installed
====================================
Components installed:
   - EKS Pod Identity Agent (DaemonSet)
   - Credential provider for pods

Verify with:
   kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
   kubectl get daemonset -n kube-system eks-pod-identity-agent
```

## Step 4: Verify Installation

Check the DaemonSet:

```bash
kubectl get daemonset -n kube-system eks-pod-identity-agent
```

Expected output:
```
NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
eks-pod-identity-agent    2         2         2       2            2           <none>          2m
```

The DESIRED count matches your node count - one agent per node.

Check the pods:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

Expected output:
```
NAME                           READY   STATUS    RESTARTS   AGE
eks-pod-identity-agent-abc12   1/1     Running   0          2m
eks-pod-identity-agent-def34   1/1     Running   0          2m
```

## Step 5: Test Pod Identity

Let's verify Pod Identity works by creating a pod that can call AWS APIs without any stored credentials. We'll use a minimal role that only allows `sts:GetCallerIdentity` - no access to actual AWS resources.

### A. Create the IAM Trust Policy

```bash
cat > pod-identity-trust-policy.json << 'EOF'
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
EOF
```

This trust policy is simpler than IRSA - just trust `pods.eks.amazonaws.com`. No OIDC URLs to manage.

### B. Create a Minimal Test Policy

This policy grants zero access to AWS resources - it only allows the pod to verify its own identity:

```bash
cat > pod-identity-test-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:GetCallerIdentity",
            "Resource": "*"
        }
    ]
}
EOF
```

### C. Create the IAM Role

```bash
aws iam create-role \
    --role-name PodIdentityTestRole \
    --assume-role-policy-document file://pod-identity-trust-policy.json

aws iam put-role-policy \
    --role-name PodIdentityTestRole \
    --policy-name PodIdentityTestPolicy \
    --policy-document file://pod-identity-test-policy.json
```

### D. Create a Service Account

```bash
kubectl create serviceaccount pod-identity-test-sa
```

### E. Create the Pod Identity Association

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create podidentityassociation \
    --cluster my-cluster \
    --namespace default \
    --service-account-name pod-identity-test-sa \
    --role-arn arn:aws:iam::$ACCOUNT_ID:role/PodIdentityTestRole
```

This tells EKS: "Pods using service account `pod-identity-test-sa` in namespace `default` can assume role `PodIdentityTestRole`."

### F. Deploy a Test Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-identity-test
spec:
  serviceAccountName: pod-identity-test-sa
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep", "infinity"]
EOF
```

### G. Wait for the Pod

```bash
kubectl wait --for=condition=Ready pod/pod-identity-test --timeout=120s
```

### H. Test AWS Access

```bash
kubectl exec -it pod-identity-test -- aws sts get-caller-identity
```

Expected output:
```json
{
    "UserId": "AROA...:eks-my-cluster-pod-ident...",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/PodIdentityTestRole/eks-my-cluster-pod-identity-test-sa-..."
}
```

The pod assumed the IAM role with no credentials stored in its configuration. The `Arn` field confirms it's using `PodIdentityTestRole`.

Verify the role has no real permissions by trying to list S3 buckets (this should fail):

```bash
kubectl exec -it pod-identity-test -- aws s3 ls
```

Expected error:
```
An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

This confirms the principle of least privilege - the pod can authenticate but has no access to AWS resources beyond what we explicitly granted.

### I. Clean Up Test Resources

```bash
# Delete the test pod
kubectl delete pod pod-identity-test

# Delete the service account
kubectl delete serviceaccount pod-identity-test-sa

# Delete the pod identity association
eksctl delete podidentityassociation \
    --cluster my-cluster \
    --namespace default \
    --service-account-name pod-identity-test-sa

# Delete the inline policy and role
aws iam delete-role-policy \
    --role-name PodIdentityTestRole \
    --policy-name PodIdentityTestPolicy

aws iam delete-role --role-name PodIdentityTestRole

# Remove the policy files
rm pod-identity-trust-policy.json pod-identity-test-policy.json
```

## Pod Identity vs IRSA: When to Use Which

| Aspect | Pod Identity | IRSA |
|--------|--------------|------|
| **Setup complexity** | Simpler | Requires OIDC provider |
| **IAM trust policy** | Generic (pods.eks.amazonaws.com) | Cluster-specific OIDC URL |
| **SDK compatibility** | Any version | Needs recent SDK for best refresh |
| **Management** | EKS API / eksctl | Annotations + IAM |
| **Recommended for** | New clusters | Existing IRSA setups |

**Use Pod Identity for:**
- New EKS clusters
- Simpler IAM trust policy management
- Environments with many service accounts

**Use IRSA for:**
- Existing clusters already using IRSA
- Cross-account access patterns you've already built
- Clusters running older EKS versions (Pod Identity requires 1.24+)

## Important: Existing IRSA Still Works

If you've already set up IRSA for components like:
- External Secrets Operator
- AWS Load Balancer Controller
- EBS CSI Driver

Those continue to work. Pod Identity doesn't replace IRSA - it's an alternative. Many teams use both:
- IRSA for EKS addons (often set up via eksctl)
- Pod Identity for application workloads

## Security Considerations

**Least privilege** - Create specific roles for each workload. Don't share roles across unrelated applications.

**Namespace isolation** - Pod identity associations are namespace-scoped. A role associated with `prod/my-app-sa` can't be assumed by `dev/my-app-sa`.

**Audit trail** - CloudTrail logs show which role was assumed and from which pod. The session tags include cluster, namespace, and service account.

**Token lifetime** - Credentials are short-lived (typically 15 minutes) and auto-refresh. Even if intercepted, they expire quickly.

## Troubleshooting

### Pod Can't Get Credentials

Check if the association exists:

```bash
eksctl get podidentityassociation --cluster my-cluster
```

Verify the service account matches:

```bash
kubectl get pod POD_NAME -o jsonpath='{.spec.serviceAccountName}'
```

Check agent logs:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent
```

### "No credentials found" Error

The AWS SDK in the pod isn't finding the credential provider. Common causes:

- Pod not using the correct service account
- No pod identity association for that service account
- Agent not running on the node

### Role Can't Be Assumed

Check the IAM role trust policy includes:

```json
{
    "Principal": {
        "Service": "pods.eks.amazonaws.com"
    },
    "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
    ]
}
```

Both actions are required - `TagSession` is needed for the session tags.

## What We've Accomplished

You now have:

- EKS Pod Identity Agent running on all nodes
- Understanding of credential-free AWS access
- Pattern for creating pod identity associations

## Next Steps

With Pod Identity in place, your pods can securely access AWS services. The next article covers External Secrets - which can use Pod Identity (or IRSA) to sync secrets from AWS Secrets Manager into Kubernetes.

**Next**: [1.6 External Secrets - Automated Secret Sync from AWS](/2026/01/17/eks-external-secrets/)

---

*Questions about Pod Identity? Reach out on socials.*
