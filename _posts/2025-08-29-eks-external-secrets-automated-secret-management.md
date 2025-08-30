---
layout: post
title: "1.5 EKS: External Secrets"
date: 2025-08-29 18:00:00 -0000
categories: kubernetes eks infrastructure security aws secrets
tags: ["Modern Full-Stack Development", "Infrastructure", "AWS", "Kubernetes", "Security"]
series: "Modern Full-Stack Development"
series_part: "1.5"
---

Your applications need secrets - API keys, database passwords, certificates. Storing them directly in Kubernetes as static secrets is a maintenance nightmare and security risk. The External Secrets Operator changes this by automatically syncing secrets from AWS Secrets Manager into your cluster, with automatic rotation and centralized management.

No more kubectl create secret commands. No more hardcoded secrets in YAML files. No more manual rotation processes. External Secrets watches AWS Secrets Manager and keeps your Kubernetes secrets synchronized automatically.

This is part 1.5 of our Modern Full-Stack Development series. We're adding automated secret management to the secure foundation we built in [1.1 EKS: Foundation](/2025/08/29/eks-foundation-building-modular-production-cluster/), [1.2 EKS: Ingress Controller](/2025/08/29/eks-ingress-controller-ssl-load-balancing/), [1.3 EKS: EBS Storage](/2025/08/29/eks-ebs-persistent-storage/), and [1.4 EKS: Pod Identity](/2025/08/29/eks-pod-identity-secure-aws-access/).

## What We're Building

By the end of this walkthrough, you'll have:

- **External Secrets Operator**: Automated secret synchronization from AWS Secrets Manager
- **ClusterSecretStore**: Secure connection to AWS Secrets Manager using service account authentication
- **Automatic Sync**: Secrets automatically update in Kubernetes when changed in AWS
- **Demo Application**: Complete example showing real-time secret synchronization
- **Production Patterns**: Best practices for secret lifecycle management

## Prerequisites Check

Before diving in, ensure you have:

1. **Base EKS cluster** from [part 1.1](/2025/08/29/eks-foundation-building-modular-production-cluster/) running
2. **AWS Load Balancer Controller** from [part 1.2](/2025/08/29/eks-ingress-controller-ssl-load-balancing/) installed
3. **Helm v3.12+** for installing the External Secrets Operator
4. **AWS Secrets Manager** access permissions

## The Architecture

Here's how External Secrets transforms secret management:

### Before: Manual Secret Management
```
Developer → kubectl create secret → Static K8s Secret
           ↓
Manual rotation, no sync, security risks
```

### After: Automated Secret Sync
```
AWS Secrets Manager → External Secrets Operator → Kubernetes Secret
                   ↓                            ↓
Centralized management      Auto-sync every 5min    Application access
```

The External Secrets Operator continuously watches AWS Secrets Manager and automatically creates or updates Kubernetes secrets based on ExternalSecret resources.

## Step 1: Install the External Secrets Operator

Create the installation script that sets up the operator with proper AWS permissions:

**File: `infra/test/eksctl/external-secrets/install-external-secrets.sh`**
```bash
#!/bin/bash

# External Secrets Operator Installation - Simple Setup
# Integrates AWS Secrets Manager with Kubernetes secrets

# Configuration
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"
NAMESPACE="external-secrets"

echo "🔐 Installing External Secrets Operator for AWS Secrets Manager"
echo "=============================================================="
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo "Namespace: $NAMESPACE"
echo ""

echo "🔍 Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "❌ ERROR: Base cluster not found or kubectl not configured"
    echo "   Run this first: cd ../base && ./create-cluster.sh"
    exit 1
fi

if ! command -v helm &> /dev/null; then
    echo "❌ ERROR: Helm is required but not installed"
    echo "   Install Helm: https://helm.sh/docs/intro/install/"
    exit 1
fi

echo "✅ Prerequisites verified"
echo ""

echo "📦 Creating namespace..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

echo "🔐 Creating IAM policy for Secrets Manager access..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::$ACCOUNT_ID:policy/ExternalSecretsOperatorPolicy"

# Check if policy exists, create only if it doesn't
if aws iam get-policy --policy-arn $POLICY_ARN &>/dev/null; then
    echo "✅ IAM policy already exists"
else
    aws iam create-policy \
        --policy-name ExternalSecretsOperatorPolicy \
        --policy-document file://external-secrets-policy.json
    
    if [[ $? -ne 0 ]]; then
        echo "❌ ERROR: Failed to create IAM policy"
        exit 1
    fi
    echo "✅ IAM policy created successfully"
fi

echo "👤 Creating IAM service account with IRSA..."
# Delete existing service account to ensure clean setup
eksctl delete iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=$NAMESPACE \
    --name=external-secrets \
    --wait 2>/dev/null || true

eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=$NAMESPACE \
    --name=external-secrets \
    --attach-policy-arn=$POLICY_ARN \
    --override-existing-serviceaccounts \
    --region=$REGION \
    --approve

if [[ $? -ne 0 ]]; then
    echo "❌ ERROR: Failed to create service account"
    exit 1
fi

echo "✅ Service account created with proper IAM role"
echo ""

echo "📦 Installing External Secrets Operator via Helm..."
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Uninstall existing version if present
helm uninstall external-secrets -n $NAMESPACE 2>/dev/null || true

helm install external-secrets external-secrets/external-secrets \
    -n $NAMESPACE \
    --set serviceAccount.create=false \
    --set serviceAccount.name=external-secrets \
    --wait

if [[ $? -ne 0 ]]; then
    echo "❌ ERROR: Failed to install External Secrets Operator"
    exit 1
fi

echo "⏳ Waiting for operator to be ready..."
kubectl wait --for=condition=Available --timeout=300s deployment/external-secrets -n $NAMESPACE
kubectl wait --for=condition=Available --timeout=300s deployment/external-secrets-webhook -n $NAMESPACE
kubectl wait --for=condition=Available --timeout=300s deployment/external-secrets-cert-controller -n $NAMESPACE

echo ""
echo "✅ External Secrets Operator installed successfully!"
echo "=================================================="
echo "🎯 Components installed:"
echo "   • External Secrets Operator"
echo "   • Webhook for validation"
echo "   • Certificate controller"
echo "   • IAM service account with Secrets Manager permissions"
```

## Step 2: Create the IAM Policy

Define the minimal permissions needed for AWS Secrets Manager access:

**File: `infra/test/eksctl/external-secrets/external-secrets-policy.json`**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:ListSecrets"
            ],
            "Resource": "*"
        }
    ]
}
```

This policy grants read-only access to AWS Secrets Manager - the operator can retrieve secrets but cannot modify them.

## Step 3: Run the Installation

Execute the installation script:

```bash
chmod +x install-external-secrets.sh
./install-external-secrets.sh
```

This process:
1. Creates the `external-secrets` namespace
2. Creates an IAM policy with Secrets Manager permissions
3. Creates a service account with IRSA (IAM Roles for Service Accounts)
4. Installs the External Secrets Operator via Helm
5. Verifies all components are running

```bash
# Verify the installation
kubectl get pods -n external-secrets
kubectl get sa external-secrets -n external-secrets -o yaml
```

## Step 4: Create the ClusterSecretStore

The ClusterSecretStore tells the operator how to connect to AWS Secrets Manager:

**File: `tests/external-secrets/secret-store.yaml`**
```yaml
# ClusterSecretStore: Connects External Secrets Operator to AWS Secrets Manager
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

Apply the ClusterSecretStore:

```bash
kubectl apply -f secret-store.yaml
```

## Step 5: Create a Secret in AWS Secrets Manager

Before we can sync secrets, we need to create one in AWS Secrets Manager:

```bash
# Create a test secret in AWS Secrets Manager
aws secretsmanager create-secret \
    --name "OPENAI_API_KEY-TEST" \
    --description "Demo secret for External Secrets testing" \
    --secret-string '{"OPENAI_API_KEY":"sk-test-demo-key-1234567890abcdef"}' \
    --region us-east-1

# Verify the secret was created
aws secretsmanager describe-secret \
    --secret-id "OPENAI_API_KEY-TEST" \
    --region us-east-1
```

## Step 6: Create an ExternalSecret Resource

Define how to sync the AWS secret to Kubernetes:

**File: `tests/external-secrets/demo-secret.yaml`**
```yaml
# External Secret Demo: Syncs a demo secret from AWS Secrets Manager
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: demo-secret
  namespace: default
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: demo-app-secret     # Kubernetes secret name
    creationPolicy: Owner
  data:
  - secretKey: openai-api-key
    remoteRef:
      key: "OPENAI_API_KEY-TEST"     # AWS Secrets Manager secret name
      property: OPENAI_API_KEY       # Property within the secret
  refreshInterval: 5m                # Sync every 5 minutes for demo
```

Apply the ExternalSecret:

```bash
kubectl apply -f demo-secret.yaml

# Watch the secret get created
kubectl get externalsecret demo-secret -w
kubectl get secret demo-app-secret
```

## Step 7: Deploy the Demo Application

Create a comprehensive demo that shows the synchronized secret in action:

**File: `tests/external-secrets/demo-app.yaml`** (key sections)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: default
spec:
  replicas: 1
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
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
        - name: secret-content
          mountPath: /usr/share/nginx/html/secret
          readOnly: true  # Mount the secret as read-only
      volumes:
      - name: html-content
        configMap:
          name: external-secrets-demo-html
      - name: secret-content
        secret:
          secretName: demo-app-secret  # The secret created by ExternalSecret

---
# Web interface showing the synchronized secret
apiVersion: v1
kind: ConfigMap
metadata:
  name: external-secrets-demo-html
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>External Secrets Demo</title>
        <style>
            /* Modern styling similar to other demos */
            body {
                font-family: 'Segoe UI', sans-serif;
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                color: white;
                min-height: 100vh;
                display: flex;
                align-items: center;
                justify-content: center;
            }
            .container {
                background: rgba(255, 255, 255, 0.1);
                backdrop-filter: blur(10px);
                border-radius: 20px;
                padding: 40px;
                max-width: 800px;
                text-align: center;
            }
            .secret-card {
                background: rgba(255, 255, 255, 0.15);
                padding: 25px;
                border-radius: 15px;
                margin: 20px 0;
                text-align: left;
            }
            .secret-value {
                font-family: 'Courier New', monospace;
                background: rgba(0, 0, 0, 0.4);
                padding: 20px;
                border-radius: 8px;
                color: #FFD700;
                word-break: break-all;
            }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🔐 External Secrets Demo</h1>
            <div class="secret-card">
                <div>🤖 OpenAI API Key from AWS Secrets Manager</div>
                <div class="secret-value" id="secret-value">Loading...</div>
            </div>
        </div>
        <script>
            fetch('/secret/openai-api-key')
                .then(response => response.text())
                .then(data => {
                    document.getElementById('secret-value').textContent = data;
                });
        </script>
    </body>
    </html>
```

## Step 8: Deploy and Test the Complete Demo

Run the deployment script that sets everything up:

**File: `tests/external-secrets/deploy-demo.sh`** (truncated)
```bash
#!/bin/bash

# External Secrets Demo Deployment
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"
SECRET_NAME="OPENAI_API_KEY-TEST"

echo "🔐 Deploying External Secrets Demo"
echo "=================================="

# Check prerequisites
if ! kubectl get pods -n external-secrets &>/dev/null; then
    echo "❌ ERROR: External Secrets Operator not installed"
    exit 1
fi

# Verify AWS secret exists
if ! aws secretsmanager describe-secret --secret-id "$SECRET_NAME" --region $REGION &>/dev/null; then
    echo "❌ ERROR: Secret '$SECRET_NAME' not found in AWS Secrets Manager"
    exit 1
fi

# Deploy resources
kubectl apply -f secret-store.yaml
kubectl apply -f demo-secret.yaml
kubectl apply -f demo-app.yaml

echo "⏳ Waiting for secret to be synced..."
sleep 10

# Verify secret sync
if kubectl get secret demo-app-secret &>/dev/null; then
    echo "✅ Kubernetes secret created successfully"
else
    echo "⚠️  Secret not yet synced, checking status..."
    kubectl describe externalsecret demo-secret
fi

echo "✅ External Secrets Demo deployed successfully!"
```

## Step 9: Verify Secret Synchronization

Test that secrets are properly synchronized:

```bash
# Check ExternalSecret status
kubectl get externalsecret demo-secret
kubectl describe externalsecret demo-secret

# Verify the Kubernetes secret was created
kubectl get secret demo-app-secret
kubectl describe secret demo-app-secret

# Check the secret value (base64 encoded)
kubectl get secret demo-app-secret -o jsonpath='{.data.openai-api-key}' | base64 -d

# Test that the application can access the secret
kubectl exec deployment/demo-app -- cat /usr/share/nginx/html/secret/openai-api-key
```

## Step 10: Test Automatic Synchronization

Update the secret in AWS and watch it sync automatically:

```bash
# Update the secret in AWS Secrets Manager
aws secretsmanager update-secret \
    --secret-id "OPENAI_API_KEY-TEST" \
    --secret-string '{"OPENAI_API_KEY":"sk-updated-demo-key-9876543210zyxwvu"}' \
    --region us-east-1

# Wait for sync (up to 5 minutes based on refreshInterval)
echo "Waiting for secret sync... (up to 5 minutes)"
sleep 30

# Check if the secret updated in Kubernetes
kubectl get secret demo-app-secret -o jsonpath='{.data.openai-api-key}' | base64 -d
```

## Production Patterns

### Multiple Secret Sources
```yaml
# ExternalSecret with multiple keys from the same AWS secret
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-config
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-config-secret
  data:
  - secretKey: database-url
    remoteRef:
      key: "production-app-config"
      property: DATABASE_URL
  - secretKey: api-key
    remoteRef:
      key: "production-app-config"
      property: API_KEY
  - secretKey: jwt-secret
    remoteRef:
      key: "production-app-config"
      property: JWT_SECRET
```

### Namespace-specific SecretStores
```yaml
# SecretStore (not ClusterSecretStore) for namespace isolation
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: production-secrets
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

### Secret Rotation with Refresh Intervals
```yaml
# Different refresh intervals for different secret types
spec:
  refreshInterval: 1h     # Database passwords - hourly
  # or
  refreshInterval: 15m    # API keys - every 15 minutes
  # or  
  refreshInterval: 24h    # Certificates - daily
```

### Template-based Secret Creation
```yaml
# Create structured secrets using templates
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: database-config
spec:
  target:
    name: postgres-config
    template:
      type: Opaque
      data:
        connection-string: |
          postgresql://{{ .username }}:{{ .password }}@{{ .host }}:5432/{{ .database }}
  data:
  - secretKey: username
    remoteRef:
      key: "postgres-credentials"
      property: username
  - secretKey: password
    remoteRef:
      key: "postgres-credentials"
      property: password
```

## Monitoring and Troubleshooting

### Check Operator Status
```bash
# External Secrets Operator logs
kubectl logs -f deployment/external-secrets -n external-secrets

# Check webhook and cert controller
kubectl logs -f deployment/external-secrets-webhook -n external-secrets
kubectl logs -f deployment/external-secrets-cert-controller -n external-secrets
```

### Debug Secret Synchronization
```bash
# Check ExternalSecret status
kubectl describe externalsecret demo-secret

# Common status conditions:
# - SecretSynced: True/False
# - Ready: True/False  
# - Error messages in events

# Check events for troubleshooting
kubectl get events --sort-by='.lastTimestamp' | grep -i external
```

### Verify AWS Permissions
```bash
# Test AWS Secrets Manager access manually
aws secretsmanager get-secret-value \
    --secret-id "OPENAI_API_KEY-TEST" \
    --region us-east-1

# Check service account annotations
kubectl get sa external-secrets -n external-secrets -o yaml
```

## Security Best Practices

### Least Privilege IAM Policies
```json
{
    "Effect": "Allow",
    "Action": [
        "secretsmanager:GetSecretValue"
    ],
    "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:production/*",
        "arn:aws:secretsmanager:us-east-1:123456789012:secret:staging/*"
    ]
}
```

### Network Security
- Use VPC endpoints for Secrets Manager to keep traffic private
- Configure security groups to restrict External Secrets Operator network access
- Enable VPC Flow Logs for audit trails

### Secret Rotation Strategy
- Enable automatic rotation in AWS Secrets Manager
- Set appropriate `refreshInterval` based on secret sensitivity
- Monitor secret usage and access patterns

## What We've Accomplished

Your EKS cluster now has:

- **Automated secret management**: No more manual kubectl secret creation
- **Centralized secret storage**: All secrets managed in AWS Secrets Manager
- **Automatic synchronization**: Secrets update automatically when changed in AWS
- **Secure authentication**: Service accounts with minimal required permissions
- **Production-ready patterns**: Templates, namespacing, and monitoring capabilities

## Cleanup Commands

To remove the test resources:

```bash
# Delete demo application and ExternalSecret
kubectl delete -f demo-app.yaml
kubectl delete -f demo-secret.yaml
kubectl delete -f secret-store.yaml

# Delete AWS secret
aws secretsmanager delete-secret \
    --secret-id "OPENAI_API_KEY-TEST" \
    --force-delete-without-recovery \
    --region us-east-1

# Keep External Secrets Operator installed for future use
```

## Next Steps

Your cluster now provides automated, secure secret management integrated with AWS Secrets Manager. This completes the core infrastructure foundation of our Modern Full-Stack Development series.

Coming up in Phase 2:
- DevSecOps pipeline integration
- CI/CD automation with GitHub Actions
- Container image scanning and vulnerability management
- Automated testing and deployment workflows

The automated secret management foundation we built today eliminates manual secret operations and provides a secure, auditable approach to credential management.

---

**Coming Next**: Phase 2 - DevSecOps and CI/CD pipeline automation for the complete development workflow.

*Questions about External Secrets configuration or secret management patterns? Reach out on socials.*