---
layout: post
title: "1.6 EKS: External Secrets with AWS Secrets Manager"
date: 2026-01-17 16:00:00 -0000
categories: kubernetes eks infrastructure aws security
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS", "Security", "Secrets"]
series: "EKS Infrastructure Series"
series_part: "1.6"
---

Hardcoding secrets in Kubernetes manifests is a security anti-pattern. The External Secrets Operator syncs secrets from AWS Secrets Manager into Kubernetes, giving you centralized secret management with automatic rotation support.

This is part 1.6 of the EKS Infrastructure Series. We're building on the cluster from [1.5 Pod Identity](/2026/01/17/eks-pod-identity/).

## What We're Building

By the end of this article, you'll have:

- **External Secrets Operator** installed via Helm
- **ClusterSecretStore** connected to AWS Secrets Manager
- **IRSA permissions** for secure Secrets Manager access
- **Automatic sync** of AWS secrets to Kubernetes secrets

## The Problem with Kubernetes Secrets

Kubernetes has a built-in Secret resource, but it has limitations:

**Secrets are just base64-encoded** - Not encrypted by default. Anyone with cluster access can decode them.

**No audit trail** - Kubernetes doesn't track who accessed which secrets or when.

**Manual rotation** - Changing a secret means updating the Kubernetes resource and restarting pods.

**Scattered management** - Secrets live in the cluster, separate from your other infrastructure secrets.

AWS Secrets Manager solves these problems:
- Secrets are encrypted at rest with KMS
- CloudTrail logs all access
- Automatic rotation with Lambda functions
- Central location for all secrets (not just Kubernetes)

## How External Secrets Works

The External Secrets Operator watches for `ExternalSecret` resources and syncs them to Kubernetes:

<div class="mermaid">
graph LR
    SM[AWS Secrets Manager] --> ESO[External Secrets Operator]
    ESO --> K8S[Kubernetes Secret]
    K8S --> Pod[Pod]
</div>

1. **You create a secret** in AWS Secrets Manager
2. **You create an ExternalSecret** resource in Kubernetes referencing it
3. **Operator syncs** the value from AWS to a Kubernetes Secret
4. **Pods use the secret** like any normal Kubernetes Secret
5. **Operator keeps it synced** - changes in AWS propagate to Kubernetes

## What is AWS Secrets Manager?

AWS Secrets Manager is a managed service for storing sensitive data - database passwords, API keys, tokens. It's different from other AWS "secret" options:

| Service | Use Case | Rotation | Pricing |
|---------|----------|----------|---------|
| **Secrets Manager** | Application secrets, DB credentials | Built-in automatic | $0.40/secret/month + API calls |
| **Parameter Store (Standard)** | Config values, non-sensitive data | Manual | Free (up to 10k params) |
| **Parameter Store (Advanced)** | Large configs, policies | Optional | $0.05/param/month |

Use Secrets Manager when you need:
- Automatic rotation
- Cross-account sharing
- Fine-grained IAM policies per secret
- Audit logging of access

### Finding Secrets in the AWS Console

1. Go to **Secrets Manager** in the AWS Console
2. Click **Secrets** in the left navigation
3. Secrets are organized by name (using paths like `EKS/prod/database`)

The console shows:
- Secret name and description
- Last accessed date
- Rotation status
- Tags for organization

## Naming Convention Summary

Before we dive in, here's how all the names connect. This is critical - mismatched names are the most common cause of sync failures:

| Component | Name Used | Purpose |
|-----------|-----------|---------|
| **Namespace** | `external-secrets` | Where the operator runs |
| **Service Account** | `external-secrets` | Kubernetes identity for the operator |
| **IAM Policy** | `ExternalSecretsOperatorPolicy` | Grants Secrets Manager access |
| **IAM Policy Resource** | `EKS/*` | Only secrets under this prefix are accessible |
| **ClusterSecretStore** | `aws-secrets-manager` | Referenced by ExternalSecrets |
| **AWS Secret Path** | `EKS/test/demo-secret` | Must match IAM policy prefix |

All ExternalSecrets must reference `aws-secrets-manager` as the store and use paths starting with `EKS/`.

## Understanding the Components

External Secrets has several moving parts:

### External Secrets Operator

The operator runs in your cluster and does the actual syncing. It's a standard Kubernetes operator pattern:
- Watches for ExternalSecret resources
- Fetches values from external providers (Secrets Manager, Vault, etc.)
- Creates/updates Kubernetes Secrets

Installed via Helm, not as an EKS addon (it's not AWS-specific).

### SecretStore / ClusterSecretStore

Defines how to connect to the external secret provider:

- **SecretStore** - Namespaced, only ExternalSecrets in that namespace can use it
- **ClusterSecretStore** - Cluster-wide, any namespace can reference it

We use ClusterSecretStore so any namespace can access AWS Secrets Manager.

### ExternalSecret

The resource you create to sync a specific secret:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-app-secrets  # Name of the K8s Secret to create
  data:
  - secretKey: database-password    # Key in the K8s Secret
    remoteRef:
      key: EKS/prod/database        # Path in Secrets Manager (must match IAM policy prefix)
      property: password            # JSON property (if secret is JSON)
```

## Project Structure

```
external-secrets/
├── install-external-secrets.sh      # Installation script
├── delete-external-secrets.sh       # Cleanup script
├── cluster-secret-store.yaml        # ClusterSecretStore definition
└── external-secrets-policy.json     # IAM policy for Secrets Manager access
```

## Step 1: Create the IAM Policy

The operator needs permissions to read secrets from Secrets Manager. Create a minimal policy:

**external-secrets-policy.json**
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
            "Resource": "arn:aws:secretsmanager:*:*:secret:EKS/*"
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

This policy:
- Allows reading secrets under the `EKS/` prefix only (adjust for your naming convention)
- Allows listing secrets (needed for discovery)
- Denies access to secrets outside the prefix

### Principle of Least Privilege

The resource restriction (`EKS/*`) is important. Without it, the operator could read any secret in your AWS account. Scope it to only what the cluster needs:

- `EKS/prod/*` - Production cluster secrets only
- `EKS/dev/*` - Development cluster secrets only
- `EKS/myapp/*` - Application-specific secrets (still under the EKS prefix)

## Step 2: Create the ClusterSecretStore

This connects the operator to AWS Secrets Manager:

**cluster-secret-store.yaml**
```yaml
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

The `jwt` auth method uses IRSA - the service account's IAM role provides credentials. No static AWS credentials needed.

## Step 3: Create the Installation Script

**install-external-secrets.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"
NAMESPACE="external-secrets"

echo "Installing External Secrets Operator"
echo "====================================="
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo "Namespace: $NAMESPACE"
echo ""

echo "Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "ERROR: Base cluster not found or kubectl not configured"
    exit 1
fi

if ! command -v helm &> /dev/null; then
    echo "ERROR: Helm is required but not installed"
    exit 1
fi

echo "Prerequisites verified"
echo ""

echo "Creating namespace..."
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

echo "Creating IAM policy for Secrets Manager access..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::$ACCOUNT_ID:policy/ExternalSecretsOperatorPolicy"

# Get the directory where this script is located
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

# Check if policy exists, create only if it doesn't
if aws iam get-policy --policy-arn $POLICY_ARN &>/dev/null; then
    echo "IAM policy already exists"
else
    aws iam create-policy \
        --policy-name ExternalSecretsOperatorPolicy \
        --policy-document file://$SCRIPT_DIR/external-secrets-policy.json

    if [[ $? -ne 0 ]]; then
        echo "ERROR: Failed to create IAM policy"
        exit 1
    fi
    echo "IAM policy created successfully"
fi
echo ""

echo "Creating IAM service account with IRSA..."
eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=$NAMESPACE \
    --name=external-secrets \
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

echo "Installing External Secrets Operator via Helm..."
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
    -n $NAMESPACE \
    --set serviceAccount.create=false \
    --set serviceAccount.name=external-secrets \
    --wait

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to install External Secrets Operator"
    exit 1
fi

echo "Waiting for operator to be ready..."
kubectl wait --for=condition=Available --timeout=300s \
    deployment/external-secrets -n $NAMESPACE
kubectl wait --for=condition=Available --timeout=300s \
    deployment/external-secrets-webhook -n $NAMESPACE
kubectl wait --for=condition=Available --timeout=300s \
    deployment/external-secrets-cert-controller -n $NAMESPACE

echo "Setting up ClusterSecretStore for AWS Secrets Manager..."
kubectl apply -f "$SCRIPT_DIR/cluster-secret-store.yaml"

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to create ClusterSecretStore"
    exit 1
fi

echo ""
echo "SUCCESS! External Secrets Operator installed"
echo "============================================="
echo "Components installed:"
echo "   - External Secrets Operator"
echo "   - Webhook for validation"
echo "   - Certificate controller"
echo "   - IAM service account with Secrets Manager permissions"
echo "   - ClusterSecretStore for AWS Secrets Manager"
echo ""
echo "Verify with:"
echo "   kubectl get pods -n $NAMESPACE"
echo "   kubectl get clustersecretstores"
```

## Step 4: Create the Cleanup Script

**delete-external-secrets.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"
NAMESPACE="external-secrets"

echo "Removing External Secrets Operator"
echo "==================================="
echo "Cluster: $CLUSTER_NAME"
echo "Namespace: $NAMESPACE"
echo ""

read -p "Are you sure? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "Operation cancelled"
    exit 0
fi

echo "Checking for existing ExternalSecrets..."
EXTERNAL_SECRETS=$(kubectl get externalsecrets -A --no-headers 2>/dev/null | wc -l)
if [ "$EXTERNAL_SECRETS" -gt 0 ]; then
    echo "WARNING: Found $EXTERNAL_SECRETS ExternalSecret resource(s)"
    echo "   These will be orphaned. Kubernetes secrets they created will remain."
    kubectl get externalsecrets -A
    echo ""
fi

SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

echo "Deleting ClusterSecretStore..."
kubectl delete -f "$SCRIPT_DIR/cluster-secret-store.yaml" --ignore-not-found=true

echo "Uninstalling External Secrets Operator..."
helm uninstall external-secrets -n $NAMESPACE 2>/dev/null || true

echo "Removing service account and IAM role..."
eksctl delete iamserviceaccount \
    --name=external-secrets \
    --namespace=$NAMESPACE \
    --cluster=$CLUSTER_NAME || true

echo "Removing IAM policy..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam delete-policy \
    --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/ExternalSecretsOperatorPolicy \
    2>/dev/null || true

echo "Removing namespace and CRDs..."
kubectl delete namespace $NAMESPACE --ignore-not-found=true
kubectl delete crd secretstores.external-secrets.io --ignore-not-found=true
kubectl delete crd externalsecrets.external-secrets.io --ignore-not-found=true
kubectl delete crd clustersecretstores.external-secrets.io --ignore-not-found=true

echo ""
echo "Cleanup complete!"
echo "================="
echo "Removed:"
echo "   - External Secrets Operator"
echo "   - IAM service account and role"
echo "   - IAM policy"
echo "   - Namespace and CRDs"
```

## Step 5: Run the Installation

```bash
chmod +x install-external-secrets.sh
./install-external-secrets.sh
```

Expected output:
```
SUCCESS! External Secrets Operator installed
=============================================
Components installed:
   - External Secrets Operator
   - Webhook for validation
   - Certificate controller
   - IAM service account with Secrets Manager permissions
   - ClusterSecretStore for AWS Secrets Manager

Verify with:
   kubectl get pods -n external-secrets
   kubectl get clustersecretstores
```

## Step 6: Verify Installation

Check the operator pods:

```bash
kubectl get pods -n external-secrets
```

Expected output:
```
NAME                                         READY   STATUS    RESTARTS   AGE
external-secrets-xxxxxxxxx-xxxxx             1/1     Running   0          2m
external-secrets-cert-controller-xxx-xxxxx   1/1     Running   0          2m
external-secrets-webhook-xxxxxxxxx-xxxxx     1/1     Running   0          2m
```

Check the ClusterSecretStore:

```bash
kubectl get clustersecretstores
```

Expected output:
```
NAME                    AGE    STATUS   CAPABILITIES   READY
aws-secrets-manager     2m     Valid    ReadWrite      True
```

If STATUS shows `Valid` and READY is `True`, the connection to AWS Secrets Manager is working.

## Step 7: Test with a Real Secret

Let's create a test secret and sync it to Kubernetes. Pay attention to the naming - the secret path in AWS Secrets Manager must match the IAM policy prefix (`EKS/*`).

### A. Create a Secret in AWS Secrets Manager

```bash
aws secretsmanager create-secret \
    --name EKS/test/demo-secret \
    --secret-string '{"username":"admin","password":"supersecret123"}'
```

The path `EKS/test/demo-secret` works because our IAM policy allows `EKS/*`.

### B. Create an ExternalSecret

Now create an ExternalSecret that references our ClusterSecretStore (`aws-secrets-manager`) and the AWS secret path (`EKS/test/demo-secret`):

```bash
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: demo-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager       # Must match ClusterSecretStore name
    kind: ClusterSecretStore
  target:
    name: demo-secret               # Name of the K8s Secret to create
  data:
  - secretKey: username             # Key in the K8s Secret
    remoteRef:
      key: EKS/test/demo-secret     # Path in AWS Secrets Manager
      property: username            # JSON property to extract
  - secretKey: password
    remoteRef:
      key: EKS/test/demo-secret
      property: password
EOF
```

Key naming relationships:
- `secretStoreRef.name: aws-secrets-manager` → matches the ClusterSecretStore we created
- `remoteRef.key: EKS/test/demo-secret` → matches the AWS secret path (and IAM policy prefix)
- `target.name: demo-secret` → the Kubernetes Secret that will be created

### C. Verify the Sync

```bash
# Check ExternalSecret status
kubectl get externalsecret demo-secret

# Check the created Kubernetes Secret
kubectl get secret demo-secret -o jsonpath='{.data.password}' | base64 -d
```

You should see `supersecret123`.

### D. Clean Up Test Resources

```bash
kubectl delete externalsecret demo-secret
aws secretsmanager delete-secret --secret-id EKS/test/demo-secret --force-delete-without-recovery
```

## Common Patterns

### Database Credentials

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: postgres-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: postgres-credentials
  data:
  - secretKey: POSTGRES_USER
    remoteRef:
      key: EKS/prod/postgres
      property: username
  - secretKey: POSTGRES_PASSWORD
    remoteRef:
      key: EKS/prod/postgres
      property: password
```

### API Keys

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: api-keys
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: api-keys
  data:
  - secretKey: STRIPE_API_KEY
    remoteRef:
      key: EKS/prod/stripe
      property: api_key
  - secretKey: SENDGRID_API_KEY
    remoteRef:
      key: EKS/prod/sendgrid
      property: api_key
```

### Entire Secret as JSON

If you want to sync the entire secret without specifying properties:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-config
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-config
  dataFrom:
  - extract:
      key: EKS/prod/app-config
```

This creates a Kubernetes Secret with all JSON keys from the AWS secret.

## Refresh Intervals and Rotation

The `refreshInterval` controls how often the operator checks for changes:

| Interval | Use Case |
|----------|----------|
| `1h` | Standard secrets, low change frequency |
| `15m` | Secrets that might rotate |
| `5m` | High-security secrets, frequent rotation |
| `0` | One-time sync, no refresh |

When AWS rotates a secret, the operator picks up the new value on the next refresh. Pods using the secret as environment variables need a restart to see the new value. Pods using it as a mounted file see updates automatically.

## Cost Breakdown

| Component | Cost |
|-----------|------|
| External Secrets Operator | Free (open source) |
| Secrets Manager secrets | $0.40/secret/month |
| Secrets Manager API calls | $0.05/10,000 calls |

For 10 secrets refreshing hourly: ~$4/month + negligible API costs.

## Troubleshooting

### ExternalSecret Shows "SecretSyncedError"

Check the ExternalSecret status:

```bash
kubectl describe externalsecret demo-secret
```

Common causes:
- **IAM permissions** - The service account role can't access the secret
- **Secret doesn't exist** - Check the path in Secrets Manager
- **Wrong property name** - The JSON key doesn't exist in the secret

### ClusterSecretStore Shows "InvalidProviderConfig"

Check the store status:

```bash
kubectl describe clustersecretstore aws-secrets-manager
```

Common causes:
- **Service account not found** - Check the namespace and name match
- **IRSA not configured** - Verify the service account has the IAM role annotation
- **Region mismatch** - The region in the store must match where secrets are stored

### Secrets Not Updating

Check the refresh interval and operator logs:

```bash
kubectl logs -f deployment/external-secrets -n external-secrets
```

The operator logs show when it syncs and any errors encountered.

## What We've Accomplished

You now have:

- External Secrets Operator managing secret sync
- ClusterSecretStore connected to AWS Secrets Manager
- IRSA providing secure, scoped access
- Automatic refresh keeping secrets up to date

## Next Steps

With secrets management in place, you need visibility into your cluster. The next article covers CloudWatch Observability - pre-built dashboards for monitoring cluster health, node performance, and container logs.

**Next**: [1.7 CloudWatch Monitoring - Container Insights and Logs](/2026/01/17/eks-cloudwatch-monitoring/)

---

*Questions about secrets management? Reach out on socials.*
