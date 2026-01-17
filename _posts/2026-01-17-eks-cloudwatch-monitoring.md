---
layout: post
title: "1.7 EKS: CloudWatch Observability for Monitoring"
date: 2026-01-17 17:00:00 -0000
categories: kubernetes eks infrastructure aws monitoring
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "EKS", "Monitoring", "CloudWatch"]
series: "EKS Infrastructure Series"
series_part: "1.7"
---

A cluster without monitoring is flying blind. CloudWatch Observability gives you ready-to-use dashboards for your EKS cluster - CPU, memory, network, and logs - with zero configuration.

This is part 1.7 of the EKS Infrastructure Series. We're building on the cluster from [1.6 External Secrets](/2026/01/17/eks-external-secrets/).

## What We're Building

By the end of this article, you'll have:

- **CloudWatch Observability addon** installed via EKS
- **Container Insights dashboards** for cluster, node, and pod metrics
- **Log collection** via Fluent Bit
- **Pod Identity authentication** for secure AWS access

## Why CloudWatch Observability?

EKS doesn't include monitoring by default. Without it, you can't answer basic questions:

- Is the cluster healthy?
- Which pods are using the most memory?
- Why did that deployment fail?
- Are nodes running out of resources?

CloudWatch Observability solves this with:

**Zero-configuration dashboards** - Pre-built views for cluster, node, pod, and container performance. No Grafana setup, no PromQL queries to write.

**Integrated logging** - Fluent Bit collects container logs and ships them to CloudWatch Logs. Search across all pods from one place.

**AWS-native** - Uses the same CloudWatch you already know. Alarms, dashboards, and log insights all work together.

**Pod Identity integration** - Uses the Pod Identity we set up in 1.5 for authentication. No IRSA configuration needed.

## What Gets Installed

The CloudWatch Observability addon installs three components:

| Component | Purpose | Runs As |
|-----------|---------|---------|
| **CloudWatch Agent** | Collects metrics (CPU, memory, network, disk) | DaemonSet |
| **Fluent Bit** | Collects and ships container logs | DaemonSet |
| **ADOT Collector** | Prometheus metrics support | Deployment |

All components run in the `amazon-cloudwatch` namespace.

## Container Insights Dashboards

Once installed, you get pre-built dashboards in the CloudWatch console:

**Cluster Overview**
- Total CPU/memory utilization
- Node count and status
- Pod count by namespace
- Network throughput

**Node Performance**
- Per-node CPU and memory
- Disk I/O and usage
- Network in/out per node
- Node conditions (ready, disk pressure, memory pressure)

**Pod Performance**
- Per-pod resource usage
- Container restarts
- Pod status by namespace
- Resource requests vs actual usage

**Container Performance**
- Individual container metrics
- CPU throttling
- Memory limits and OOM events

## Prerequisites

CloudWatch Observability requires **Pod Identity** (from 1.5). The addon uses Pod Identity to authenticate with CloudWatch APIs - no static credentials or IRSA configuration needed.

Verify Pod Identity is installed:

```bash
eksctl get addon --name eks-pod-identity-agent --cluster my-cluster
```

If not installed, go back to [1.5 Pod Identity](/2026/01/17/eks-pod-identity/) first.

## Project Structure

```
monitoring/
├── install-cloudwatch-observability.sh    # Installation script
├── delete-cloudwatch-observability.sh     # Cleanup script
└── cloudwatch-trust-policy.json           # IAM trust policy for Pod Identity
```

## Step 1: Create the IAM Trust Policy

The CloudWatch agent needs an IAM role to write metrics and logs. This trust policy allows Pod Identity to assume the role:

**cloudwatch-trust-policy.json**
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

This is the same trust policy pattern we used for Pod Identity testing - trust `pods.eks.amazonaws.com`.

## Step 2: Create the Installation Script

**install-cloudwatch-observability.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "Installing CloudWatch Observability"
echo "===================================="
echo "Cluster: $CLUSTER_NAME"
echo "Region:  $REGION"
echo ""

echo "Checking prerequisites..."
if ! kubectl get nodes &>/dev/null; then
    echo "ERROR: Cluster not found or kubectl not configured"
    exit 1
fi

# Check if Pod Identity is installed
if ! eksctl get addon --name eks-pod-identity-agent --cluster $CLUSTER_NAME 2>/dev/null | grep -q "ACTIVE"; then
    echo "ERROR: Pod Identity Agent is required but not installed"
    echo "   Install it first: see 1.5 Pod Identity article"
    exit 1
fi

echo "Prerequisites verified"
echo ""

# Get account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Get script directory for finding trust policy
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

echo "Creating IAM role for CloudWatch Agent..."
aws iam create-role \
    --role-name CloudWatchObservabilityRole \
    --assume-role-policy-document file://$SCRIPT_DIR/cloudwatch-trust-policy.json \
    2>/dev/null || echo "Role already exists"

# CloudWatchAgentServerPolicy covers metrics
aws iam attach-role-policy \
    --role-name CloudWatchObservabilityRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    2>/dev/null || true

# AmazonEBSCSIDriverPolicy is NOT needed - that was a copy-paste error
# CloudWatchLogsFullAccess allows Fluent Bit to create log groups and write logs
aws iam attach-role-policy \
    --role-name CloudWatchObservabilityRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
    2>/dev/null || true

echo "IAM role created with metrics and logs permissions"
echo ""

echo "Installing CloudWatch Observability addon with Pod Identity..."
aws eks create-addon \
    --addon-name amazon-cloudwatch-observability \
    --cluster-name $CLUSTER_NAME \
    --pod-identity-associations serviceAccount=cloudwatch-agent,roleArn=arn:aws:iam::$ACCOUNT_ID:role/CloudWatchObservabilityRole \
    --region $REGION

if [[ $? -ne 0 ]]; then
    echo "ERROR: Failed to install CloudWatch Observability addon"
    exit 1
fi

echo "Waiting for CloudWatch Observability to be ready..."
sleep 30

kubectl wait --for=condition=Ready pods -n amazon-cloudwatch \
    -l app.kubernetes.io/name=cloudwatch-agent --timeout=300s
kubectl wait --for=condition=Ready pods -n amazon-cloudwatch \
    -l app.kubernetes.io/name=fluent-bit --timeout=300s

echo ""
echo "SUCCESS! CloudWatch Observability installed"
echo "============================================"
echo "Components installed:"
echo "   - CloudWatch Agent (metrics)"
echo "   - Fluent Bit (logs)"
echo "   - ADOT Collector (Prometheus metrics)"
echo "   - Container Insights dashboards"
echo ""
echo "Access dashboards:"
echo "   https://console.aws.amazon.com/cloudwatch/home?region=$REGION#container-insights:performance/EKS:Cluster"
echo ""
echo "Verify with:"
echo "   kubectl get pods -n amazon-cloudwatch"
```

## Step 3: Create the Cleanup Script

**delete-cloudwatch-observability.sh**
```bash
#!/bin/bash

# Configuration
CLUSTER_NAME="my-cluster"
REGION="us-east-1"

echo "Removing CloudWatch Observability"
echo "=================================="
echo "Cluster: $CLUSTER_NAME"
echo ""
echo "WARNING: This will remove:"
echo "   - CloudWatch Observability addon"
echo "   - All monitoring dashboards"
echo "   - Log collection"
echo ""

read -p "Are you sure? (yes/no): " confirm
if [[ $confirm != "yes" ]]; then
    echo "Operation cancelled"
    exit 0
fi

echo ""
echo "Removing Pod Identity associations..."
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Delete Pod Identity associations for amazon-cloudwatch namespace
aws eks list-pod-identity-associations \
    --cluster-name $CLUSTER_NAME \
    --region $REGION \
    --query 'associations[?namespace==`amazon-cloudwatch`].associationId' \
    --output text | tr '\t' '\n' | while read -r association_id; do
    if [[ -n "$association_id" ]]; then
        echo "Deleting association: $association_id"
        aws eks delete-pod-identity-association \
            --cluster-name $CLUSTER_NAME \
            --association-id "$association_id" \
            --region $REGION 2>/dev/null || true
    fi
done

echo "Removing CloudWatch Observability addon..."
eksctl delete addon \
    --name amazon-cloudwatch-observability \
    --cluster $CLUSTER_NAME 2>/dev/null || true

echo "Cleaning up namespace..."
kubectl delete namespace amazon-cloudwatch --ignore-not-found=true

echo "Cleaning up IAM role..."
aws iam detach-role-policy \
    --role-name CloudWatchObservabilityRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    2>/dev/null || true

aws iam detach-role-policy \
    --role-name CloudWatchObservabilityRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
    2>/dev/null || true

aws iam delete-role \
    --role-name CloudWatchObservabilityRole \
    2>/dev/null || true

echo "Waiting for cleanup..."
sleep 15

echo ""
echo "Cleanup complete!"
echo "================="
echo "Removed:"
echo "   - CloudWatch Observability addon"
echo "   - CloudWatch Agent and Fluent Bit"
echo "   - IAM role"
echo ""
echo "Note: Historical CloudWatch data is preserved"
echo "      (logs and metrics remain in CloudWatch)"
echo ""
echo "To delete log groups (removes all historical logs):"
echo "   aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/application"
echo "   aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/dataplane"
echo "   aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/host"
echo "   aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/performance"
```

## Step 4: Run the Installation

```bash
chmod +x install-cloudwatch-observability.sh
./install-cloudwatch-observability.sh
```

Expected output:
```
SUCCESS! CloudWatch Observability installed
============================================
Components installed:
   - CloudWatch Agent (metrics)
   - Fluent Bit (logs)
   - ADOT Collector (Prometheus metrics)
   - Container Insights dashboards

Access dashboards:
   https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#container-insights:performance/EKS:Cluster

Verify with:
   kubectl get pods -n amazon-cloudwatch
```

## Step 5: Verify Installation

Check the pods in the amazon-cloudwatch namespace:

```bash
kubectl get pods -n amazon-cloudwatch
```

Expected output:
```
NAME                                  READY   STATUS    RESTARTS   AGE
cloudwatch-agent-xxxxx                1/1     Running   0          2m
cloudwatch-agent-yyyyy                1/1     Running   0          2m
fluent-bit-xxxxx                      1/1     Running   0          2m
fluent-bit-yyyyy                      1/1     Running   0          2m
```

You should see one cloudwatch-agent and one fluent-bit pod per node (they're DaemonSets).

Check the addon status:

```bash
eksctl get addon --name amazon-cloudwatch-observability --cluster my-cluster
```

Expected output:
```
NAME                                VERSION                 STATUS  ISSUES
amazon-cloudwatch-observability     v2.1.0-eksbuild.1       ACTIVE  0
```

## Step 6: Access the Dashboards

Open the CloudWatch Container Insights console:

```
https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#container-insights:performance
```

Or navigate manually:
1. Go to **CloudWatch** in the AWS Console
2. Click **Container Insights** in the left navigation
3. Select **Performance monitoring**
4. Choose your cluster from the dropdown

Dashboards take 5-10 minutes to populate with data after installation.

## Step 7: Verify Log Groups Were Created

Before testing, verify the log groups exist. Fluent Bit creates these automatically:

```bash
aws logs describe-log-groups \
    --log-group-name-prefix /aws/containerinsights/my-cluster \
    --query 'logGroups[*].logGroupName'
```

Expected output:
```json
[
    "/aws/containerinsights/my-cluster/application",
    "/aws/containerinsights/my-cluster/dataplane",
    "/aws/containerinsights/my-cluster/host",
    "/aws/containerinsights/my-cluster/performance"
]
```

If the log groups don't exist, check that:
1. Fluent Bit pods are running: `kubectl get pods -n amazon-cloudwatch -l app.kubernetes.io/name=fluent-bit`
2. The IAM role has `CloudWatchLogsFullAccess` attached
3. Fluent Bit logs for errors: `kubectl logs -n amazon-cloudwatch -l app.kubernetes.io/name=fluent-bit`

## Step 8: Test Log Collection

Now let's verify logs flow from pods to CloudWatch.

### A. Create a Test Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: log-generator
  namespace: default
spec:
  containers:
  - name: logger
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Log generator started"
      while true; do
        echo "Test log message at \$(date)"
        sleep 5
      done
EOF
```

### B. Wait for the Pod and Verify It's Logging

```bash
kubectl wait --for=condition=Ready pod/log-generator --timeout=60s

# Verify the pod is generating logs locally
kubectl logs log-generator --tail=3
```

You should see:
```
Log generator started
Test log message at Fri Jan 17 12:00:00 UTC 2026
Test log message at Fri Jan 17 12:00:05 UTC 2026
```

### C. Wait for Logs to Appear in CloudWatch

Fluent Bit batches logs, so wait 1-2 minutes for logs to appear in CloudWatch.

```bash
# Wait for log delivery
sleep 120
```

### D. Verify Logs in CloudWatch

Check if logs arrived:

```bash
aws logs filter-log-events \
    --log-group-name /aws/containerinsights/my-cluster/application \
    --filter-pattern "log-generator" \
    --start-time $(($(date +%s) - 300))000 \
    --limit 5 \
    --query 'events[*].message'
```

Expected output (JSON array of log messages):
```json
[
    "... \"log\":\"Test log message at Fri Jan 17 12:00:00 UTC 2026\\n\" ...",
    "... \"log\":\"Test log message at Fri Jan 17 12:00:05 UTC 2026\\n\" ..."
]
```

If you get an empty result `[]`, the logs haven't arrived yet. Common causes:
- **Log group doesn't exist** - Check Step 7 above
- **IAM permissions** - Verify `CloudWatchLogsFullAccess` is attached to the role
- **Fluent Bit errors** - Check `kubectl logs -n amazon-cloudwatch -l app.kubernetes.io/name=fluent-bit`

### E. View Logs in the Console

1. Go to **CloudWatch** in the AWS Console
2. Click **Log groups** in the left navigation
3. Click `/aws/containerinsights/my-cluster/application`
4. Click a recent log stream (named after the node)
5. Search for `log-generator`

### F. Clean Up Test Resources

```bash
kubectl delete pod log-generator
```

## Understanding the Metrics

CloudWatch Observability collects metrics at multiple levels:

### Cluster Metrics

| Metric | Description |
|--------|-------------|
| `cluster_failed_node_count` | Nodes in NotReady state |
| `cluster_node_count` | Total node count |
| `namespace_number_of_running_pods` | Pods per namespace |

### Node Metrics

| Metric | Description |
|--------|-------------|
| `node_cpu_utilization` | CPU usage percentage |
| `node_memory_utilization` | Memory usage percentage |
| `node_filesystem_utilization` | Disk usage percentage |
| `node_network_total_bytes` | Network throughput |

### Pod Metrics

| Metric | Description |
|--------|-------------|
| `pod_cpu_utilization` | CPU vs requested |
| `pod_memory_utilization` | Memory vs requested |
| `pod_network_rx_bytes` | Network received |
| `pod_network_tx_bytes` | Network transmitted |

### Container Metrics

| Metric | Description |
|--------|-------------|
| `container_cpu_utilization` | Per-container CPU |
| `container_memory_utilization` | Per-container memory |
| `container_restart_count` | Restart count |

## Creating Alarms

Use CloudWatch Alarms to get notified of issues:

### High CPU Alarm Example

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name "EKS-HighCPU" \
    --alarm-description "Alert when cluster CPU exceeds 80%" \
    --metric-name node_cpu_utilization \
    --namespace ContainerInsights \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=ClusterName,Value=my-cluster \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:alerts
```

### Node Not Ready Alarm

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name "EKS-NodeNotReady" \
    --alarm-description "Alert when nodes are not ready" \
    --metric-name cluster_failed_node_count \
    --namespace ContainerInsights \
    --statistic Maximum \
    --period 60 \
    --threshold 0 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=ClusterName,Value=my-cluster \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:alerts
```

### Delete Alarms

To remove alarms you've created:

```bash
aws cloudwatch delete-alarms --alarm-names "EKS-HighCPU" "EKS-NodeNotReady"
```

## Cost Considerations

CloudWatch Observability has associated costs:

| Component | Cost |
|-----------|------|
| **Custom metrics** | $0.30 per metric per month (first 10k) |
| **Log ingestion** | $0.50 per GB |
| **Log storage** | $0.03 per GB per month |
| **Dashboard** | Free (Container Insights dashboards) |

For a small cluster (3 nodes, 20 pods), expect roughly $15-30/month. Costs scale with:
- Number of nodes and pods (more metrics)
- Log volume (application log verbosity)
- Retention period for logs

### Reducing Costs

1. **Set log retention** - Don't keep logs forever:
```bash
aws logs put-retention-policy \
    --log-group-name /aws/containerinsights/my-cluster/application \
    --retention-in-days 7
```

2. **Filter noisy logs** - Exclude high-volume, low-value logs in Fluent Bit config

3. **Use metric filters** - Create alarms from logs instead of storing everything

## Troubleshooting

### Pods Stuck in Pending

Check if the DaemonSet can schedule:

```bash
kubectl describe pod -n amazon-cloudwatch -l app.kubernetes.io/name=cloudwatch-agent
```

Common causes:
- Node taints preventing scheduling
- Resource constraints

### No Metrics in Dashboard

Check if the agent can reach CloudWatch:

```bash
kubectl logs -n amazon-cloudwatch -l app.kubernetes.io/name=cloudwatch-agent
```

Look for authentication errors - usually means Pod Identity association is missing.

### Logs Not Appearing

Check Fluent Bit:

```bash
kubectl logs -n amazon-cloudwatch -l app.kubernetes.io/name=fluent-bit
```

Common issues:
- Log group doesn't exist (Fluent Bit creates it, but check permissions)
- IAM policy doesn't allow log writes

## Complete Cleanup

To remove everything created in this article:

### A. Remove Test Resources

```bash
# Delete test pod (if still running)
kubectl delete pod log-generator --ignore-not-found=true

# Delete any alarms you created
aws cloudwatch delete-alarms --alarm-names "EKS-HighCPU" "EKS-NodeNotReady" 2>/dev/null || true
```

### B. Run the Delete Script

```bash
./delete-cloudwatch-observability.sh
```

This removes the addon, IAM role, and Pod Identity associations.

### C. Delete Log Groups (Optional)

The delete script preserves historical logs. To remove them completely:

```bash
aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/application 2>/dev/null || true
aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/dataplane 2>/dev/null || true
aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/host 2>/dev/null || true
aws logs delete-log-group --log-group-name /aws/containerinsights/my-cluster/performance 2>/dev/null || true
```

## What We've Accomplished

You now have:

- CloudWatch Observability collecting metrics and logs
- Pre-built dashboards for cluster visibility
- Fluent Bit shipping container logs to CloudWatch
- Pod Identity providing secure authentication

## What's Next

The infrastructure is built. The final article covers how to use it - operational commands, production checklists, and architecture patterns for different application types.

**Next**: [1.8 Production Architecture and Operations Guide](/2026/01/17/eks-production-architecture/)

---

*Questions about monitoring? Reach out on socials.*
