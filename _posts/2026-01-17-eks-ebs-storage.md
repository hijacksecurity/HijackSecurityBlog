---
layout: post
title: "1.4 EKS: Persistent Storage with EBS CSI Driver"
date: 2026-01-17 14:00:00 -0000
categories: kubernetes eks infrastructure aws storage
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS", "Storage"]
series: "EKS Infrastructure Series"
series_part: "1.4"
---

Stateless applications are great, but most real workloads need persistent storage - databases, file uploads, caches. The EBS CSI driver lets Kubernetes dynamically provision and manage EBS volumes, so you can create persistent storage just by applying a PersistentVolumeClaim.

This is part 1.4 of the EKS Infrastructure Series. We're building on the cluster from [1.3 Ingress Controller](/2026/01/17/eks-ingress-controller/).

## What We're Building

By the end of this article, you'll have:

- **EBS CSI driver** installed as an EKS managed addon
- **GP3 default StorageClass** for automatic volume provisioning
- **IRSA permissions** for secure EBS API access
- **Encrypted volumes** by default

## What is EBS?

Amazon Elastic Block Store (EBS) is AWS's block storage service. Think of it like a virtual hard drive that attaches to EC2 instances (which is what EKS nodes are). It's fundamentally different from other AWS storage options:

| Storage Type | What It Is | Use Case |
|--------------|------------|----------|
| **EBS** | Block storage (virtual hard drive) | Databases, application data, anything needing filesystem access |
| **S3** | Object storage (files via HTTP API) | Static files, backups, data lakes |
| **EFS** | Network filesystem (NFS) | Shared storage across multiple pods/nodes |

Key characteristics of EBS:

- **Single-node attachment** - An EBS volume can only attach to one EC2 instance at a time (one node in Kubernetes terms)
- **AZ-bound** - Volumes exist in a single Availability Zone and can only attach to nodes in that AZ
- **Persistent** - Data survives pod restarts, node replacements, and cluster upgrades
- **Performant** - Low latency, consistent IOPS, suitable for databases

### Finding EBS Volumes in the AWS Console

When the CSI driver creates volumes, they appear in the EC2 console:

1. Go to **EC2** → **Elastic Block Store** → **Volumes**
2. Look for volumes tagged with `kubernetes.io/created-for/pvc/name`
3. The tags show which PVC and namespace the volume belongs to

You can also filter by:
- **Tag**: `kubernetes.io/cluster/my-cluster` = `owned`
- **State**: `in-use` (attached to a node) or `available` (detached)

The volume's **Attachment information** shows which EC2 instance (EKS node) it's currently attached to.

## Understanding CSI and the Component Layers

Before diving into installation, it helps to understand how the pieces fit together. There are several layers involved, each owned by different projects.

### What is CSI?

CSI (Container Storage Interface) is a standard API that lets Kubernetes work with any storage system. Before CSI, storage drivers were built into Kubernetes itself - adding support for a new storage system meant changing Kubernetes core code.

With CSI:
- Kubernetes defines the interface (what operations storage must support)
- Storage vendors implement drivers that speak this interface
- Drivers run as pods in your cluster, not as part of Kubernetes

This separation means AWS can update the EBS driver independently of Kubernetes releases.

### The Layer Cake

Here's what each layer provides:

<div class="mermaid">
graph TB
    subgraph "Your Application"
        APP[Pod with PVC]
    end
    subgraph "Kubernetes Core"
        PVC[PersistentVolumeClaim]
        PV[PersistentVolume]
        SC[StorageClass]
    end
    subgraph "CSI Standard"
        CSI[CSI Interface]
    end
    subgraph "AWS EBS CSI Driver"
        CTRL[Controller]
        NODE[Node Driver]
    end
    subgraph "AWS"
        EBS[EBS API]
        VOL[EBS Volume]
    end
    APP --> PVC
    PVC --> SC
    SC --> CSI
    CSI --> CTRL
    CTRL --> EBS
    EBS --> VOL
    NODE --> VOL
</div>

| Layer | Owner | What It Provides |
|-------|-------|------------------|
| **Kubernetes Core** | Kubernetes project | PVC, PV, StorageClass resources and scheduling |
| **CSI Spec** | Kubernetes SIG-Storage | Standard interface for storage operations |
| **EBS CSI Driver** | AWS (open source) | Implementation that talks to EBS APIs |
| **EBS** | AWS | Actual block storage volumes |

### Installation Methods

The EBS CSI driver can be installed multiple ways:

| Method | Managed By | Best For |
|--------|------------|----------|
| **EKS Addon** | AWS | Production EKS clusters (recommended) |
| **Helm Chart** | You | Custom configurations, non-EKS clusters |
| **Raw Manifests** | You | Learning, full control |

**EKS Addon** (what we use) - AWS manages the driver deployment. You get automatic updates, compatibility testing with your EKS version, and integration with eksctl. The addon is still the same open-source driver, just packaged and managed by AWS.

**Helm Chart** - Install from the `aws-ebs-csi-driver` Helm chart. Gives you more configuration options but you're responsible for upgrades.

**Raw Manifests** - Apply YAML files directly from the GitHub repo. Maximum control, maximum maintenance burden.

### What Gets Installed Where

When you install the EBS CSI driver as an EKS addon:

```
kube-system namespace:
├── ebs-csi-controller (Deployment, 2 replicas)
│   └── Manages volume create/delete/snapshot
├── ebs-csi-node (DaemonSet, runs on every node)
│   └── Handles attach/mount operations
└── ebs-csi-controller-sa (ServiceAccount)
    └── Has IAM role for EBS API access
```

The **controller** runs as a regular deployment - it doesn't need to be on every node because it just makes API calls to AWS.

The **node driver** runs as a DaemonSet because it needs to run on every node to handle mounting volumes into pods on that node.

### EKS Addons vs Helm

EKS addons are AWS's way of managing common Kubernetes components. Under the hood, an addon is typically the same software you'd install via Helm, but:

- AWS tests compatibility with each EKS version
- Updates are managed through `eksctl` or the AWS console
- Configuration options are more limited than Helm
- AWS provides support for addon issues

For the EBS CSI driver, the EKS addon is the recommended approach because:
1. AWS maintains compatibility with EKS versions
2. IAM role integration is simpler with `eksctl create addon`
3. Updates are one command: `eksctl update addon`

If you need advanced configuration (custom node selectors, tolerations, resource limits), you might prefer the Helm installation instead.

## Why EBS CSI Driver?

EKS doesn't include the EBS CSI driver by default. Without it, you can't use EBS volumes for persistent storage. The driver:

**Provisions volumes dynamically** - Create a PVC, get an EBS volume. No manual volume creation in the AWS console.

**Handles the lifecycle** - Attaches volumes when pods start, detaches when they stop, deletes when PVCs are removed.

**Supports volume snapshots** - Back up and restore data using EBS snapshots.

**Enables volume expansion** - Grow volumes without downtime (for supported filesystems).

## How It Works

When you create a PersistentVolumeClaim, here's what happens:

<div class="mermaid">
graph LR
    PVC[PVC Created] --> CSI[EBS CSI Driver]
    CSI --> EBS[Creates EBS Volume]
    EBS --> PV[Creates PV]
    PV --> Pod[Mounts to Pod]
</div>

1. **PVC created** - You apply a PersistentVolumeClaim requesting storage
2. **CSI driver provisions** - The driver calls AWS APIs to create an EBS volume
3. **PV created** - Kubernetes creates a PersistentVolume bound to the EBS volume
4. **Volume attached** - When a pod references the PVC, the volume is attached to the node
5. **Filesystem mounted** - The volume is formatted (if new) and mounted into the pod

## Prerequisites

Before installing:

```bash
# Base cluster running with OIDC provider (from 1.2)
kubectl get nodes
aws eks describe-cluster --name my-cluster \
    --query 'cluster.identity.oidc.issuer'

# AWS CLI configured
aws sts get-caller-identity
```

## Project Structure

```
ebs-csi/
├── install-ebs-csi.sh      # Installation script
├── delete-ebs-csi.sh       # Cleanup script
└── gp3-storageclass.yaml   # Default StorageClass definition
```

## Step 1: Create the StorageClass

First, define the default StorageClass. This tells Kubernetes how to provision EBS volumes:

**gp3-storageclass.yaml**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-default
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
```

Key settings explained:

| Setting | Value | Purpose |
|---------|-------|---------|
| `is-default-class` | `true` | Used when PVCs don't specify a StorageClass |
| `volumeBindingMode` | `WaitForFirstConsumer` | Creates volume in same AZ as the pod |
| `reclaimPolicy` | `Delete` | Deletes EBS volume when PVC is deleted |
| `allowVolumeExpansion` | `true` | Enables growing volumes without downtime |
| `type` | `gp3` | Uses GP3 volumes (better price/performance than GP2) |
| `encrypted` | `true` | Encrypts volumes at rest using AWS KMS |

### Why WaitForFirstConsumer?

The `WaitForFirstConsumer` binding mode is critical for multi-AZ clusters. Without it:

1. PVC is created
2. Volume is provisioned in a random AZ (say us-east-1a)
3. Pod gets scheduled to us-east-1b
4. Pod can't start - EBS volumes can only attach to nodes in the same AZ

With `WaitForFirstConsumer`, volume creation is delayed until the pod is scheduled. The driver then creates the volume in the same AZ as the node.

### Why GP3 Over GP2?

GP3 is the newer EBS volume type with better economics:

| Feature | GP2 | GP3 |
|---------|-----|-----|
| Baseline IOPS | 3 IOPS/GB (min 100) | 3,000 IOPS (fixed) |
| Baseline throughput | 128-250 MB/s | 125 MB/s |
| Cost (us-east-1) | $0.10/GB/month | $0.08/GB/month |
| IOPS scaling | Tied to size | Independent of size |

GP3 gives you 3,000 IOPS regardless of volume size. With GP2, you'd need a 1TB volume to get the same performance.

## Step 2: Create the Installation Script

**install-ebs-csi.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "Installing EBS CSI Driver for Persistent Storage"
echo "================================================="
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""

echo "Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "ERROR: Base cluster not found or kubectl not configured"
    exit 1
fi

# Check if OIDC provider exists
if ! aws eks describe-cluster --name $CLUSTER_NAME \
    --query 'cluster.identity.oidc.issuer' --output text | grep -q 'oidc'; then
    echo "ERROR: OIDC provider not configured for cluster"
    echo "   OIDC provider is required for service account IAM roles"
    exit 1
fi

echo "Prerequisites verified"
echo ""

echo "Configuring EBS CSI driver permissions..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create IAM role for the driver
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve \
    --role-only \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --region $REGION

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to create EBS CSI service account"
    exit 1
fi

echo "EBS CSI service account created with proper IAM role"
echo ""

echo "Installing EBS CSI driver addon..."
eksctl create addon \
    --name aws-ebs-csi-driver \
    --cluster $CLUSTER_NAME \
    --service-account-role-arn arn:aws:iam::$ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole \
    --force \
    --region $REGION

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to install EBS CSI driver addon"
    exit 1
fi

echo "Waiting for EBS CSI driver to be ready..."
kubectl wait --for=condition=Available --timeout=300s \
    deployment/ebs-csi-controller -n kube-system

echo ""
echo "Creating Default StorageClass..."

# Get the directory where this script is located
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
kubectl apply -f "$SCRIPT_DIR/gp3-storageclass.yaml"

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to create Default StorageClass"
    exit 1
fi

echo ""
echo "SUCCESS! EBS CSI Driver installed"
echo "================================="
echo "Components installed:"
echo "   - EBS CSI Controller (manages volume lifecycle)"
echo "   - EBS CSI Node Driver (mounts volumes on nodes)"
echo "   - IAM service account with EBS permissions"
echo "   - GP3 Default StorageClass"
echo ""
echo "Verify with:"
echo "   kubectl get pods -n kube-system -l app=ebs-csi-controller"
echo "   kubectl get storageclass"
```

## Understanding the Components

The EBS CSI driver installs several components:

### Controller (ebs-csi-controller)

Runs as a Deployment in kube-system. Handles:
- Volume provisioning (CreateVolume API)
- Volume deletion (DeleteVolume API)
- Volume snapshots
- Volume expansion

The controller needs IAM permissions to call EBS APIs. We use IRSA (IAM Roles for Service Accounts) so only the controller pods get these permissions.

### Node Driver (ebs-csi-node)

Runs as a DaemonSet on every node. Handles:
- Attaching volumes to the node (AttachVolume API)
- Mounting volumes into pods
- Detaching volumes when pods terminate

### IAM Policy

The `AmazonEBSCSIDriverPolicy` is an AWS-managed policy that grants:
- ec2:CreateVolume, DeleteVolume
- ec2:AttachVolume, DetachVolume
- ec2:CreateSnapshot, DeleteSnapshot
- ec2:DescribeVolumes, DescribeSnapshots
- kms:CreateGrant (for encrypted volumes)

## Step 3: Create the Cleanup Script

**delete-ebs-csi.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "EBS CSI Driver Removal"
echo "======================"
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""
echo "WARNING: This will remove:"
echo "   - All EBS volumes and persistent data"
echo "   - EBS CSI driver addon"
echo "   - Service accounts and IAM roles"
echo ""

read -p "Are you sure you want to continue? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "Operation cancelled"
    exit 0
fi

echo ""
echo "Checking for existing PVCs..."
PVC_COUNT=$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)
if [[ $PVC_COUNT -gt 0 ]]; then
    echo "Found $PVC_COUNT PersistentVolumeClaims:"
    kubectl get pvc -A
    echo ""
    read -p "Continue with PVC deletion? (yes/no): " pvc_confirm
    if [[ $pvc_confirm != "yes" ]]; then
        echo "Cannot proceed - PVCs must be deleted first"
        exit 1
    fi

    echo "Deleting all PVCs..."
    kubectl delete pvc -A --all --timeout=60s
fi

echo "Removing EBS CSI driver addon..."
eksctl delete addon --name aws-ebs-csi-driver --cluster $CLUSTER_NAME \
    2>/dev/null || true

echo "Removing EBS CSI service account and IAM role..."
eksctl delete iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME 2>/dev/null || true

echo "Removing custom storage class..."
kubectl delete storageclass gp3-default 2>/dev/null || true

echo ""
echo "Cleanup complete!"
echo "================="
echo "Removed:"
echo "   - EBS CSI driver addon"
echo "   - Service account and IAM role"
echo "   - GP3 default storage class"
```

The cleanup script warns about PVCs because deleting them with `reclaimPolicy: Delete` will permanently destroy the underlying EBS volumes and data.

## Step 4: Run the Installation

```bash
chmod +x install-ebs-csi.sh
./install-ebs-csi.sh
```

Expected output:
```
SUCCESS! EBS CSI Driver installed
=================================
Components installed:
   - EBS CSI Controller (manages volume lifecycle)
   - EBS CSI Node Driver (mounts volumes on nodes)
   - IAM service account with EBS permissions
   - GP3 Default StorageClass

Verify with:
   kubectl get pods -n kube-system -l app=ebs-csi-controller
   kubectl get storageclass
```

## Step 5: Verify Installation

Check the driver pods:

```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

Expected output:
```
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-xxxxxxxxx-xxxxx    6/6     Running   0          2m
ebs-csi-controller-xxxxxxxxx-xxxxx    6/6     Running   0          2m
```

Check the StorageClass:

```bash
kubectl get storageclass
```

Expected output:
```
NAME                  PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2                   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  1h
gp3-default (default) ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   2m
```

Note: `gp3-default` shows `(default)` indicating it's the default StorageClass.

## Step 6: Test with a PVC

Create a test PVC and pod:

```bash
# Create a PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Create a pod that uses it
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: test-pvc
EOF
```

Watch the volume get provisioned:

```bash
# Check PVC status
kubectl get pvc test-pvc -w
```

Initially shows `Pending` (waiting for pod), then `Bound` once the pod is scheduled:
```
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    AGE
test-pvc   Pending                                       gp3-default     5s
test-pvc   Bound     pvc-xxx  1Gi        RWO            gp3-default     30s
```

Verify the pod can write to the volume:

```bash
kubectl exec test-pod -- sh -c "echo 'Hello from EBS' > /data/test.txt"
kubectl exec test-pod -- cat /data/test.txt
```

Clean up:

```bash
kubectl delete pod test-pod
kubectl delete pvc test-pvc
```

## Common Use Cases

### Database Storage

PostgreSQL, MySQL, and other databases need persistent storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          value: "changeme"
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data
```

### High-Performance Storage

For workloads needing more IOPS, create a custom StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-iops
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  iops: "16000"
  throughput: "1000"
  encrypted: "true"
```

Then reference it in your PVC:

```yaml
spec:
  storageClassName: high-iops
  resources:
    requests:
      storage: 100Gi
```

## Volume Expansion

To grow a volume without downtime:

```bash
# Edit the PVC to increase storage
kubectl patch pvc test-pvc -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
```

The EBS volume will be expanded automatically. For ext4 filesystems, the filesystem is also expanded online.

## Cost Breakdown

EBS storage costs (us-east-1):

| Volume Type | Cost | Included IOPS | Included Throughput |
|-------------|------|---------------|---------------------|
| GP3 | $0.08/GB/month | 3,000 | 125 MB/s |
| GP2 | $0.10/GB/month | 3/GB (min 100) | 128-250 MB/s |
| io2 | $0.125/GB/month | Charged separately | 1,000 MB/s max |

For a typical 20GB database volume on GP3: ~$1.60/month

Additional IOPS on GP3 cost $0.005/IOPS/month beyond the included 3,000.

## Troubleshooting

### PVC Stuck in Pending

Check the PVC events:

```bash
kubectl describe pvc test-pvc
```

Common causes:
- **No default StorageClass** - Verify `gp3-default` exists with `kubectl get sc`
- **CSI driver not running** - Check `kubectl get pods -n kube-system -l app=ebs-csi-controller`
- **IAM permissions** - The driver role needs `AmazonEBSCSIDriverPolicy`

### Volume Won't Attach

EBS volumes can only attach to one node, and only in the same AZ. Check:

```bash
# See where the volume was created
kubectl get pv -o wide

# See where the pod is scheduled
kubectl get pod -o wide
```

If they're in different AZs, the volume can't attach. Use `WaitForFirstConsumer` to prevent this.

### Pod Can't Write to Volume

Check permissions inside the pod:

```bash
kubectl exec test-pod -- ls -la /data
```

If the volume is mounted as root-only, add a security context:

```yaml
spec:
  securityContext:
    fsGroup: 1000
  containers:
  - name: app
    securityContext:
      runAsUser: 1000
```

## What We've Accomplished

You now have:

- EBS CSI driver managing persistent volumes
- GP3 default StorageClass with encryption
- Dynamic volume provisioning from PVCs
- Support for volume expansion

## Next Steps

With storage working, pods can persist data. In the next article, we'll set up Pod Identity for secure AWS API access from your applications.

**Next**: [1.5 EKS Pod Identity - Secure AWS Access](/2026/01/17/eks-pod-identity/)

---

*Questions about EBS storage? Reach out on socials.*
