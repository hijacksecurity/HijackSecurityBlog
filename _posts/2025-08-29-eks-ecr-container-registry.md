---
layout: post
title: "1.1 ECR: Setting Up Your Container Registry"
date: 2025-08-29 11:00:00 -0000
categories: kubernetes eks infrastructure aws containers
tags: ["EKS Infrastructure Series", "Infrastructure", "AWS", "Kubernetes", "Containers", "ECR"]
series: "EKS Infrastructure Series"
series_part: "1.1"
---

Before you can deploy applications to Kubernetes, you need somewhere to store your container images. Amazon Elastic Container Registry (ECR) provides a private registry that integrates directly with AWS IAM - no separate authentication system to manage.

This is part 1.1 of the EKS Infrastructure Series. We're setting up ECR first because every subsequent article will use it to store and deploy container images.

## What We're Building

By the end of this article, you'll have:

- **Private ECR repositories** configured via a reusable script
- **Vulnerability scanning** enabled to catch security issues before deployment
- **Lifecycle policies** to automatically clean up old images and control costs
- **Immutable tags** preventing accidental overwrites in production
- **Configuration-driven setup** that's easy to extend

## Why ECR?

A few reasons to use ECR over alternatives like Docker Hub or GitHub Container Registry:

**Native IAM integration** - No separate registry credentials. The same IAM roles that access other AWS services can push and pull images.

**Built-in security scanning** - ECR scans images for known vulnerabilities using the same database as Amazon Inspector.

**No egress costs within AWS** - Pulling images from ECR to EKS in the same region is free. External registries charge for outbound data transfer.

**Integrated with EKS** - Works seamlessly with EKS Pod Identity and IRSA for secure, credential-free image pulls.

## Prerequisites

Make sure you have:

```bash
# Verify AWS CLI is installed and configured
aws --version
aws sts get-caller-identity

# jq for JSON parsing
jq --version

# Docker for building and pushing images
docker --version
```

Your AWS user or role needs these permissions:
- `ecr:CreateRepository`
- `ecr:DeleteRepository`
- `ecr:DescribeRepositories`
- `ecr:GetAuthorizationToken`
- `ecr:PutLifecyclePolicy`
- `ecr:BatchCheckLayerAvailability`
- `ecr:PutImage`
- `ecr:InitiateLayerUpload`
- `ecr:UploadLayerPart`
- `ecr:CompleteLayerUpload`

## Project Structure

We'll organize the ECR setup with configuration files and scripts:

```
ecr/
├── repositories.json      # Repository definitions
├── lifecycle-policy.json  # Image retention rules
├── create-ecr.sh         # Creation script
└── delete-ecr.sh         # Cleanup script
```

This structure keeps configuration separate from scripts, making it easy to add repositories without modifying code.

## Step 1: Define Your Repositories

Create a configuration file that defines your ECR repositories:

**repositories.json**
```json
{
  "project": "my-project",
  "repositories": [
    {
      "name": "my-app",
      "description": "Main application repository"
    },
    {
      "name": "my-app-worker",
      "description": "Background worker service"
    }
  ]
}
```

This configuration-driven approach means adding a new repository is just adding an entry to the JSON file.

## Step 2: Configure Lifecycle Policies

Define image retention rules to control costs:

**lifecycle-policy.json**
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

This policy keeps the 10 most recent images and automatically deletes older ones. Adjust `countNumber` based on your deployment frequency and rollback needs.

For more complex scenarios, you can create multiple rules:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep tagged production images for 90 days",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod-", "release-"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 90
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep only 5 untagged images",
      "selection": {
        "tagStatus": "untagged",
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

## Step 3: Create the Setup Script

This script reads the configuration and creates repositories with security best practices:

**create-ecr.sh**
```bash
#!/bin/bash

# Configuration
REGION="us-east-1"
CONFIG_FILE="repositories.json"
LIFECYCLE_FILE="lifecycle-policy.json"

echo "ECR Repository Setup"
echo "===================="

# Get AWS account ID dynamically
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account: $ACCOUNT_ID"
echo "Region:  $REGION"
echo ""

# Read project name from configuration
PROJECT=$(jq -r '.project' $CONFIG_FILE)
echo "Project: $PROJECT"
echo ""

# Create repositories from configuration
echo "Creating ECR repositories..."
jq -r '.repositories[] | @base64' $CONFIG_FILE | while read -r repo_data; do
    # Decode JSON for each repository
    repo_info=$(echo "$repo_data" | base64 --decode)
    REPO_NAME=$(echo "$repo_info" | jq -r '.name')
    DESCRIPTION=$(echo "$repo_info" | jq -r '.description')

    echo "Creating: $REPO_NAME"
    echo "  Description: $DESCRIPTION"

    # Check if repository already exists (idempotent)
    if aws ecr describe-repositories --repository-names "$REPO_NAME" --region $REGION >/dev/null 2>&1; then
        echo "  Already exists"
    else
        # Create repository with security best practices
        aws ecr create-repository \
            --repository-name "$REPO_NAME" \
            --region $REGION \
            --image-tag-mutability IMMUTABLE \
            --image-scanning-configuration scanOnPush=true \
            >/dev/null
        echo "  Created"
    fi

    # Apply lifecycle policy
    aws ecr put-lifecycle-policy \
        --repository-name "$REPO_NAME" \
        --lifecycle-policy-text file://$LIFECYCLE_FILE \
        --region $REGION \
        >/dev/null 2>&1
    echo "  Lifecycle policy applied"
    echo ""
done

echo "ECR repositories ready!"
echo ""
echo "Registry: $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"
echo ""
echo "Login command:"
echo "  aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"
echo ""
echo "Example usage:"
echo "  docker tag myimage:latest $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/my-app:latest"
echo "  docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/my-app:latest"
```

Key features of this script:

- **Idempotent** - Safe to run multiple times; won't fail if repositories exist
- **Configuration-driven** - Add repositories by editing JSON, not the script
- **Security by default** - Immutable tags and vulnerability scanning enabled
- **Dynamic account ID** - No hardcoded AWS account numbers

## Step 4: Create the Cleanup Script

For completeness, here's a deletion script with safety confirmations:

**delete-ecr.sh**
```bash
#!/bin/bash

# Configuration
REGION="us-east-1"
CONFIG_FILE="repositories.json"

echo "ECR Repository Deletion"
echo "======================="

# Get AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account: $ACCOUNT_ID"
echo "Region:  $REGION"
echo ""

# Read project name
PROJECT=$(jq -r '.project' $CONFIG_FILE)
echo "Project: $PROJECT"
echo ""

# Show what will be deleted
echo "WARNING: This will delete the following repositories:"
jq -r '.repositories[].name' $CONFIG_FILE | while read -r repo_name; do
    echo "  - $repo_name"
done
echo ""

# Require explicit confirmation
read -p "Are you sure? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "Cancelled"
    exit 0
fi

echo ""

# Delete repositories
jq -r '.repositories[].name' $CONFIG_FILE | while read -r repo_name; do
    echo "Deleting: $repo_name"

    if aws ecr describe-repositories --repository-names $repo_name --region $REGION >/dev/null 2>&1; then
        # Force delete removes all images
        aws ecr delete-repository \
            --repository-name $repo_name \
            --region $REGION \
            --force \
            >/dev/null
        echo "  Deleted"
    else
        echo "  Not found"
    fi
done

echo ""
echo "ECR cleanup complete!"
```

## Step 5: Run the Setup

Execute the creation script:

```bash
chmod +x create-ecr.sh
./create-ecr.sh
```

Expected output:
```
ECR Repository Setup
====================
Account: ************
Region:  us-east-1

Project: my-project

Creating ECR repositories...
Creating: my-app
  Description: Main application repository
  Created
  Lifecycle policy applied

Creating: my-app-worker
  Description: Background worker service
  Created
  Lifecycle policy applied

ECR repositories ready!

Registry: ************.dkr.ecr.us-east-1.amazonaws.com
...
```

## Step 6: Build and Push an Image

Test the setup by pushing an image:

```bash
# Authenticate Docker with ECR
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"

aws ecr get-login-password --region $REGION | \
    docker login --username AWS --password-stdin \
    $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

Create a simple test image:

```dockerfile
# Dockerfile
FROM nginx:alpine
RUN echo "<h1>ECR Test</h1>" > /usr/share/nginx/html/index.html
EXPOSE 80
```

Build and push:

```bash
# Build
docker build -t my-app:1.0.0 .

# Tag for ECR
ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/my-app"
docker tag my-app:1.0.0 $ECR_URI:1.0.0

# Push
docker push $ECR_URI:1.0.0
```

## Step 7: Verify the Push and Scan Results

Check that the image was pushed:

```bash
aws ecr describe-images \
    --repository-name my-app \
    --region $REGION \
    --query 'imageDetails[*].[imageTags,imagePushedAt]' \
    --output table
```

Check vulnerability scan results:

```bash
aws ecr describe-image-scan-findings \
    --repository-name my-app \
    --image-id imageTag=1.0.0 \
    --region $REGION
```

## Why Immutable Tags?

The `--image-tag-mutability IMMUTABLE` flag prevents overwriting existing tags. This matters because:

- **Predictable deployments** - Tag `1.0.0` always refers to the same image
- **Audit trail** - You can trace exactly what was deployed
- **No accidental overwrites** - CI/CD can't accidentally push over a production tag

If you need to update an image, create a new tag (e.g., `1.0.1`).

## IAM Policies

### For CI/CD Pipelines (Push Access)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:PutImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload"
            ],
            "Resource": "arn:aws:ecr:us-east-1:*:repository/my-app*"
        }
    ]
}
```

### For EKS Nodes (Pull Access Only)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage"
            ],
            "Resource": "arn:aws:ecr:us-east-1:*:repository/*"
        }
    ]
}
```

## Cost Considerations

ECR pricing is straightforward:

| Component | Cost |
|-----------|------|
| Storage | $0.10 per GB/month |
| Data transfer (same region to EKS) | Free |
| Data transfer (cross-region or internet) | Standard AWS data transfer rates |

For a typical setup with 10 images averaging 200MB each:
- Storage: ~2GB = $0.20/month
- Pulls within the same region: Free

Lifecycle policies are your main cost control mechanism.

## Troubleshooting

### "no basic auth credentials" Error

The Docker token has expired (12-hour validity). Re-authenticate:

```bash
aws ecr get-login-password --region $REGION | \
    docker login --username AWS --password-stdin \
    $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

### "repository does not exist" Error

Check the repository name and region:

```bash
aws ecr describe-repositories --region $REGION
```

### Scan Shows "SCAN_FAILED"

Some minimal base images (scratch, distroless) can't be scanned. The image still works; you just won't get vulnerability data.

### "tag invalid" with Immutable Tags

You're trying to push to an existing tag. Create a new tag instead - this is the intended behavior for production safety.

## What We've Accomplished

You now have:

- Configuration-driven ECR repository management
- Automatic vulnerability scanning on every push
- Lifecycle policies controlling storage costs
- Immutable tags preventing accidental overwrites
- Reusable scripts for your infrastructure

## Next Steps

With ECR configured, we can build the EKS cluster that will pull images from this registry. In the next article, we'll create the base cluster with auto-scaling nodes and proper security configuration.

**Next**: [1.2 EKS Base Cluster - Kubernetes Foundation with Auto-scaling](/2025/08/29/eks-base-cluster/) *(Coming Soon)*

---

*Questions about ECR configuration? Reach out on socials.*
