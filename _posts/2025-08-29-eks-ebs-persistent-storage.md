---
layout: post
title: "1.3 EKS: EBS Persistent Storage - Dynamic Volume Provisioning"
date: 2025-08-29 14:00:00 -0000
categories: kubernetes eks infrastructure storage aws
tags: ["Modern Full-Stack Development", "Infrastructure", "AWS", "Kubernetes"]
series: "Modern Full-Stack Development"
series_part: "1.3"
---

Your applications need more than just compute and networking - they need persistent storage that survives pod restarts, scaling events, and node failures. The EBS CSI driver transforms your EKS cluster into a dynamic storage platform that automatically provisions and manages AWS EBS volumes based on your application requirements.

No more manual EBS volume creation. No more static volume attachments. The CSI driver watches for PersistentVolumeClaims and creates the right storage automatically - GP2 for cost-sensitive workloads, GP3 for balanced performance, IO2 for high-performance databases.

This is part 1.3 of our Modern Full-Stack Development series. We're adding production-grade persistent storage to the cluster we built in [1.1 EKS: Kubernetes Foundation](/2025/08/29/eks-foundation-building-modular-production-cluster/) and exposed with [1.2 EKS: Ingress Controller](/2025/08/29/eks-ingress-controller-ssl-load-balancing/).

## What We're Building

By the end of this walkthrough, you'll have:

- **EBS CSI Driver**: Latest driver with proper IAM configuration
- **Dynamic Provisioning**: Automatic EBS volume creation based on storage classes
- **Multiple Volume Types**: GP2, GP3, and IO2 storage classes for different performance needs
- **Production Patterns**: Encryption, snapshots, and volume expansion capabilities
- **Demo Application**: Complete example showing persistent storage across pod lifecycle

## Prerequisites Check

Before diving in, ensure you have:

1. **Base EKS cluster** from [part 1.1](/2025/08/29/eks-foundation-building-modular-production-cluster/) running
2. **AWS Load Balancer Controller** from [part 1.2](/2025/08/29/eks-ingress-controller-ssl-load-balancing/) installed
3. **OIDC provider** configured for your cluster (required for service account IAM roles)
4. **AWS CLI** configured with EBS permissions

## The Architecture

Here's how the EBS CSI driver integrates with your cluster:

### Before: No Persistent Storage
```
Pod Restart → Data Lost
Node Failure → Data Lost
Scale Down → Data Lost
```

### After: EBS-Backed Persistence
```
PersistentVolumeClaim → StorageClass → EBS Volume
                     ↓
Pod → PersistentVolume → EBS Volume (survives all failures)
```

The CSI driver acts as the bridge between Kubernetes storage requests and AWS EBS volumes, handling the entire lifecycle automatically.

## Step 1: Install the EBS CSI Driver

Create the installation script that sets up the driver with proper permissions:

**File: `infra/test/eksctl/ebs-csi/install-ebs-csi.sh`**
```bash
#!/bin/bash

# EBS CSI Driver Installation - Production-ready persistent storage
# Provides dynamic EBS volume provisioning for Kubernetes

# Configuration
CLUSTER_NAME="your-cluster-name"
REGION="us-east-1"

echo "📀 Installing EBS CSI Driver for Persistent Storage"
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

# Check if OIDC provider exists
if ! aws eks describe-cluster --name $CLUSTER_NAME --query 'cluster.identity.oidc.issuer' --output text | grep -q 'oidc'; then
    echo "❌ ERROR: OIDC provider not configured for cluster"
    echo "   OIDC provider is required for service account IAM roles"
    exit 1
fi

echo "✅ Prerequisites verified"
echo ""

echo "📀 Configuring EBS CSI driver permissions..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Check if service account already exists
if kubectl get serviceaccount ebs-csi-controller-sa -n kube-system &>/dev/null; then
    echo "⚠️  EBS CSI service account already exists, updating..."
    eksctl delete iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster $CLUSTER_NAME --wait
fi

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
    echo "❌ ERROR: Failed to create EBS CSI service account"
    exit 1
fi

echo "✅ EBS CSI service account created with proper IAM role"
echo ""

echo "📦 Installing EBS CSI driver addon..."
# Remove existing addon if present
eksctl delete addon --name aws-ebs-csi-driver --cluster $CLUSTER_NAME 2>/dev/null || true
sleep 5

eksctl create addon \
    --name aws-ebs-csi-driver \
    --cluster $CLUSTER_NAME \
    --service-account-role-arn arn:aws:iam::$ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole \
    --force \
    --region $REGION

if [[ $? -ne 0 ]]; then
    echo "❌ ERROR: Failed to install EBS CSI driver addon"
    exit 1
fi

echo "⏳ Waiting for EBS CSI driver to be ready..."
kubectl wait --for=condition=Available --timeout=300s deployment/ebs-csi-controller -n kube-system

echo ""
echo "✅ EBS CSI Driver installed successfully!"
echo "========================================"
echo "🎯 Components installed:"
echo "   • EBS CSI Controller (manages volume lifecycle)"
echo "   • EBS CSI Node Driver (mounts volumes on nodes)"
echo "   • IAM service account with EBS permissions"
echo "   • Default gp2 storage class (from EKS)"
echo ""
echo "🔍 Verify installation:"
echo "   kubectl get pods -n kube-system -l app=ebs-csi-controller"
echo "   kubectl get storageclass"
echo "   kubectl get csidriver"
```

## Step 2: Run the Installation

Execute the installation script:

```bash
chmod +x install-ebs-csi.sh
./install-ebs-csi.sh
```

This process:
1. Verifies your cluster has an OIDC provider (required for service account authentication)
2. Creates an IAM service account with the EBS CSI policy attached
3. Installs the EBS CSI driver as an EKS addon
4. Waits for the controller to become operational

```bash
# Verify the installation
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get storageclass
kubectl get csidriver ebs.csi.aws.com
```

The default `gp2` storage class is automatically created by EKS, but we'll add more storage classes for different performance requirements.

## Step 3: Create Advanced Storage Classes

Define storage classes for different performance and cost requirements:

**File: `tests/ebs-csi/demo-pvc.yaml`**
```yaml
# EBS Storage Demo: PersistentVolumeClaim Examples
# Demonstrates different EBS volume types and configurations

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-gp3-claim
  namespace: default
  labels:
    app: ebs-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-gp2-claim
  namespace: default
  labels:
    app: ebs-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 5Gi

---
# Custom StorageClass for high-performance volumes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-io2-fast
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "1000"
  fsType: ext4
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-io2-claim
  namespace: default
  labels:
    app: ebs-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-io2-fast
  resources:
    requests:
      storage: 20Gi
```

Apply the storage classes and PVCs:

```bash
kubectl apply -f demo-pvc.yaml
```

## Step 4: Deploy the Storage Demo Application

Create a comprehensive demo that showcases persistent storage across different volume types:

**File: `tests/ebs-csi/demo-app.yaml`** (truncated for brevity - see full version with UI in the source)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ebs-demo-app
  namespace: default
  labels:
    app: ebs-demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ebs-demo-app
  template:
    metadata:
      labels:
        app: ebs-demo-app
    spec:
      containers:
      - name: storage-demo
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: gp3-storage
          mountPath: /data/gp3
        - name: gp2-storage
          mountPath: /data/gp2
        - name: io2-storage
          mountPath: /data/io2
        - name: html-content
          mountPath: /usr/share/nginx/html
        # Initialize storage demonstration
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo "EBS GP3 Volume Test Data - $(date)" > /data/gp3/test.txt
                echo "EBS GP2 Volume Test Data - $(date)" > /data/gp2/test.txt
                echo "EBS IO2 High-Performance Test Data - $(date)" > /data/io2/test.txt
                df -h /data/* > /data/gp3/disk-usage.txt
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      volumes:
      - name: gp3-storage
        persistentVolumeClaim:
          claimName: ebs-gp3-claim
      - name: gp2-storage
        persistentVolumeClaim:
          claimName: ebs-gp2-claim
      - name: io2-storage
        persistentVolumeClaim:
          claimName: ebs-io2-claim
      - name: html-content
        configMap:
          name: ebs-demo-html

---
apiVersion: v1
kind: Service
metadata:
  name: ebs-demo-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: ebs-demo-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ebs-demo-ingress
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: hijacksecurity-test
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/certificate-arn: "your-certificate-arn"
    alb.ingress.kubernetes.io/inbound-cidrs: "your-ip/32"
spec:
  ingressClassName: alb
  rules:
  - host: storage-demo.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ebs-demo-service
            port:
              number: 80
```

Deploy the complete demo:

```bash
kubectl apply -f demo-app.yaml
```

## Step 5: Test Storage Persistence

The real test of persistent storage is whether data survives pod and node failures. Let's verify this:

```bash
# Check that all volumes were created and bound
kubectl get pv,pvc

# Verify the demo app is running
kubectl get pods -l app=ebs-demo-app

# Check what data was written to the volumes
kubectl exec -it deployment/ebs-demo-app -- ls -la /data/*/
kubectl exec -it deployment/ebs-demo-app -- cat /data/gp3/test.txt

# Test persistence by deleting the pod
kubectl delete pod -l app=ebs-demo-app

# Wait for new pod to start
kubectl wait --for=condition=Ready pod -l app=ebs-demo-app

# Verify data persisted across pod restart
kubectl exec -it deployment/ebs-demo-app -- cat /data/gp3/test.txt
# Should show the same timestamp from before pod deletion
```

## Step 6: Understanding EBS Volume Types

### GP2 (General Purpose, Previous Generation)
- **Cost**: Lowest cost option
- **Performance**: Baseline 100 IOPS, bursts to 3,000 IOPS
- **Use case**: Development, testing, light workloads
- **Limitations**: Performance tied to volume size

### GP3 (General Purpose, Latest Generation)
- **Cost**: 20% cheaper than GP2 for same storage
- **Performance**: Baseline 3,000 IOPS, configurable up to 16,000
- **Use case**: Most production workloads
- **Benefits**: Predictable performance independent of size

### IO2 (Provisioned IOPS SSD)
- **Cost**: Higher cost but predictable performance
- **Performance**: Up to 64,000 IOPS with sub-millisecond latency
- **Use case**: Databases, critical applications
- **Benefits**: Guaranteed IOPS, 99.999% durability

## Storage Class Best Practices

### Encryption by Default
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  fsType: ext4
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Volume Expansion Example
```bash
# Edit PVC to increase size
kubectl patch pvc ebs-gp3-claim -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Check expansion status
kubectl describe pvc ebs-gp3-claim
```

### Snapshot Configuration
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tagSpecification_1: "Name=ebs-snapshot"
  tagSpecification_2: "Environment=production"
```

## Production Patterns

### StatefulSet with Persistent Storage
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  template:
    # ... pod template
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ebs-io2-fast"
      resources:
        requests:
          storage: 100Gi
```

### Multi-AZ Storage Topology
```yaml
# StorageClass with topology constraints
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: multi-az-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - us-east-1a
    - us-east-1b
    - us-east-1c
volumeBindingMode: WaitForFirstConsumer
```

## Monitoring and Troubleshooting

### Check EBS CSI Driver Status
```bash
# Controller pods
kubectl get pods -n kube-system -l app=ebs-csi-controller

# Node daemon set
kubectl get daemonset -n kube-system ebs-csi-node

# Driver registration
kubectl get csinode
```

### Volume Troubleshooting
```bash
# Check PVC status
kubectl describe pvc ebs-gp3-claim

# Check actual EBS volumes in AWS
aws ec2 describe-volumes --filters "Name=tag:kubernetes.io/cluster/your-cluster,Values=owned"

# CSI driver logs
kubectl logs -f deployment/ebs-csi-controller -n kube-system -c ebs-plugin
```

### Common Issues and Solutions

**PVC stuck in Pending state:**
- Check storage class exists
- Verify CSI driver is running
- Ensure node has permissions to create EBS volumes

**Volume mount failures:**
- Check node security groups allow EBS attachment
- Verify the EBS volume is in the same AZ as the node
- Check filesystem type compatibility

## Cost Optimization Tips

1. **Choose the right volume type**: GP3 for most workloads, GP2 only for cost-sensitive development
2. **Size appropriately**: Start smaller with volume expansion enabled
3. **Use lifecycle policies**: Delete old snapshots automatically
4. **Monitor unused volumes**: Set up alerts for unattached EBS volumes

## What We've Accomplished

Your EKS cluster now has:

- **Dynamic storage provisioning**: Automatic EBS volume creation based on application needs
- **Multiple performance tiers**: GP2, GP3, and IO2 storage classes for different requirements
- **Production features**: Encryption, volume expansion, and snapshot capabilities
- **Persistent data**: Applications can safely restart without data loss
- **Cost optimization**: Right-sized volumes with expansion capabilities

## Cleanup Commands

To remove the test resources:

```bash
# Delete demo app (this preserves the PVCs and data)
kubectl delete -f demo-app.yaml

# Delete PVCs (this will delete the EBS volumes)
kubectl delete pvc --all

# Keep the EBS CSI driver installed for future use
```

## Next Steps

Your cluster now provides persistent storage for stateful applications like databases, file systems, and data processing workloads. In the next article, we'll add secure AWS service access using EKS Pod Identity.

Coming up:
- EKS Pod Identity for secure AWS API access
- External Secrets integration with AWS Secrets Manager
- Complete CI/CD pipeline setup
- Application deployment patterns

The persistent storage foundation we built today enables your cluster to handle real-world stateful workloads with confidence.

---

**Coming Next**: [1.4 EKS: Pod Identity - Secure AWS Service Access](/posts/coming-soon) - Enable secure, credential-free AWS service access for your applications.

*Questions about EBS configuration or storage patterns? Reach out on socials.*