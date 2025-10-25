# ECR Mirror - Automated Container Image Mirroring

Automatically mirror container images from public registries (Docker Hub, Quay.io, AWS Public ECR, etc.) to your private AWS Elastic Container Registry (ECR).

---

## 🎯 Goals

This solution provides:

1. **Automated Image Mirroring** - Copy public container images to your private AWS ECR
2. **No Privileged Mode Required** - Works on GitLab runners without privileged access
3. **CI/CD Integration** - Fully automated GitLab CI/CD pipeline
4. **Command-Line Tool** - Standalone CLI for local use and scripting
5. **Cost Optimization** - Reduce data transfer costs and improve image pull speeds
6. **Security** - Keep images in your private registry with vulnerability scanning enabled
7. **Reliability** - No dependency on public registry availability during deployments

### Why Mirror Images?

**Problem:**
- Your Kubernetes/EKS clusters pull images from public registries (Docker Hub, AWS Public ECR, etc.)
- Rate limits on public registries can break deployments
- Data transfer costs for pulling from public registries
- Slower image pulls from public internet
- Dependency on external registry availability

**Solution:**
- Mirror images to your private ECR once
- Pull from your private ECR (same region, no rate limits)
- Faster deployments, lower costs, better reliability

### Use Cases

- **EKS Deployments** - Mirror AWS Public ECR images (EKS Distro, CSI drivers, etc.)
- **Air-Gapped Environments** - Deploy without public internet access
- **Cost Savings** - Eliminate data transfer fees for repeated pulls
- **Rate Limit Avoidance** - Bypass Docker Hub rate limits
- **Version Control** - Pin image versions in your private registry

---

## 📋 Table of Contents

- [How It Works](#-how-it-works)
- [Quick Start](#-quick-start)
- [Using the CLI Tool](#-using-the-cli-tool)
- [Using GitLab CI Pipeline](#-using-gitlab-ci-pipeline)
- [Configuration](#-configuration)
- [Examples](#-examples)
- [Troubleshooting](#-troubleshooting)
- [Advanced Usage](#-advanced-usage)

---

## 🔧 How It Works

### Architecture

```
┌─────────────────────┐
│  Public Registry    │
│  - Docker Hub       │
│  - Quay.io          │
│  - AWS Public ECR   │
└──────────┬──────────┘
           │
           │ Pull
           ▼
┌─────────────────────┐
│   ecr-mirror CLI    │
│   or                │
│   GitLab Pipeline   │
└──────────┬──────────┘
           │
           │ Push
           ▼
┌─────────────────────┐
│  Private AWS ECR    │
│  - Your Account     │
│  - Your Region      │
└─────────────────────┘
```

### Components

#### 1. **ecr-mirror CLI Tool**

A bash script that:
- Lists and searches your ECR repositories
- Validates image configuration
- Creates ECR repositories automatically
- Mirrors images using Skopeo (no Docker daemon required)
- Skips duplicate images
- Provides detailed progress and error reporting

#### 2. **GitLab CI Pipeline**

A 4-stage pipeline that:
1. **list** - Shows current ECR contents
2. **validate** - Validates `images.txt` configuration
3. **prepare-ecr** - Creates ECR repositories with AWS CLI
4. **mirror** - Mirrors images using the ecr-mirror CLI tool

### Technology Stack

- **Skopeo** - Container image copying tool (no Docker daemon needed)
- **AWS CLI** - Manages ECR repositories and authentication
- **Bash** - Portable scripting for the CLI tool
- **GitLab CI** - Automation and scheduling

### Why Skopeo?

- ✅ No Docker daemon required (works without privileged mode)
- ✅ Faster than Docker (direct registry-to-registry copy)
- ✅ Works on minimal base images
- ✅ Built for image mirroring
- ✅ Supports all major registries

---

## 🚀 Quick Start

### Prerequisites

- AWS account with ECR access
- AWS CLI configured with credentials
- Skopeo installed (for CLI tool)
- GitLab repository (for CI/CD)

### Option 1: Using the CLI Tool (Local/Manual)

```bash
# 1. Download the tool
wget https://raw.githubusercontent.com/your-org/your-repo/main/ecr-mirror
chmod +x ecr-mirror

# 2. Install dependencies
# macOS
brew install skopeo awscli

# Ubuntu/Debian
sudo apt-get install skopeo awscli

# Amazon Linux
sudo yum install skopeo awscli

# 3. Configure AWS credentials
aws configure

# 4. Create images.txt
cat > images.txt << 'EOF'
nginx:1.25.3
redis:7.2-alpine
postgres:15-alpine
EOF

# 5. Mirror images
./ecr-mirror mirror images.txt --account YOUR_AWS_ACCOUNT_ID

# Done! Images are now in your ECR
```

### Option 2: Using GitLab CI/CD (Automated)

```bash
# 1. Add files to your repository
cp .gitlab-ci-with-cli.yml .gitlab-ci.yml
cp ecr-mirror .
cat > images.txt << 'EOF'
nginx:1.25.3
redis:7.2-alpine
EOF

# 2. Configure GitLab CI/CD variables
# Go to: Settings → CI/CD → Variables
# Add:
#   AWS_ACCESS_KEY_ID (masked, protected)
#   AWS_SECRET_ACCESS_KEY (masked, protected)
#   AWS_DEFAULT_REGION (e.g., us-east-1)
#   AWS_ACCOUNT_ID (your 12-digit account ID)

# 3. Commit and push
git add .gitlab-ci.yml ecr-mirror images.txt
git commit -m "Add ECR mirror automation"
git push

# 4. Pipeline runs automatically!
# Go to: CI/CD → Pipelines
```

---

## 💻 Using the CLI Tool

### Installation

```bash
# Download
wget https://raw.githubusercontent.com/your-org/your-repo/main/ecr-mirror
chmod +x ecr-mirror

# Or if you have it in your repo
cp ecr-mirror /usr/local/bin/
chmod +x /usr/local/bin/ecr-mirror
```

### Commands

#### List ECR Contents

Shows all repositories and their tags:

```bash
ecr-mirror list
```

**Output:**
```
Repository: nginx
URI: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx
Images: 2 tagged
Tags:
  • 1.25.0
  • 1.25.3

Repository: redis
URI: 123456789012.dkr.ecr.us-east-1.amazonaws.com/redis
Images: 1 tagged
Tags:
  • 7.2-alpine
```

#### Search ECR

Find images by repository name or tag:

```bash
# Search by repository name
ecr-mirror search nginx

# Search by tag
ecr-mirror search 7.2-alpine

# Search by partial match
ecr-mirror search eks-1-28
```

**Output:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Repository: nginx [MATCH]
URI: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx

All Tags: (2)
  • 1.25.0
  • 1.25.3

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SEARCH RESULTS
Repositories matched: 1
Tags matched: 0
Total matches: 1
```

#### Mirror Images

Copy images from public registries to ECR:

```bash
# Mirror from images.txt
ecr-mirror mirror images.txt --account 123456789012

# Dry run (preview without making changes)
ecr-mirror mirror images.txt --account 123456789012 --dry-run

# Specify region
ecr-mirror mirror images.txt --account 123456789012 --region us-west-2
```

**Output:**
```
ℹ ECR Registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
ℹ Getting ECR credentials...
✓ ECR credentials obtained

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ℹ Processing: nginx:1.25.3

  Source Image: nginx:1.25.3
  ECR Repository: nginx
  Destination: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3

ℹ Creating repository: nginx
✓ Created repository: nginx
ℹ Copying image with skopeo...
✓ Successfully mirrored: nginx:1.25.3

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Successful: 1
  Failed: 0
  Skipped: 0

✓ All images mirrored successfully!
```

### CLI Options

```
ecr-mirror <command> [options]

Commands:
  list                    List all ECR repositories and their images
  search <term>          Search ECR by repository, image, or tag
  mirror <file>          Mirror images from file to ECR
  version                Show version
  help                   Show help

Options:
  --region <region>      AWS region (default: $AWS_REGION)
  --account <id>         AWS account ID (required for mirror)
  --dry-run              Preview without making changes

Examples:
  ecr-mirror list
  ecr-mirror search nginx
  ecr-mirror mirror images.txt --account 123456789012
  ecr-mirror mirror images.txt --account 123456789012 --dry-run
  ecr-mirror mirror images.txt --account 123456789012 --region us-west-2
```

---

## 🔄 Using GitLab CI Pipeline

### Pipeline Stages

The GitLab CI pipeline has 4 stages:

```
┌──────────────┐
│ 1. list      │  Show current ECR contents
└──────┬───────┘
       │
┌──────▼───────┐
│ 2. validate  │  Validate images.txt format
└──────┬───────┘
       │
┌──────▼───────┐
│ 3. prepare   │  Create ECR repositories (AWS CLI)
└──────┬───────┘
       │
┌──────▼───────┐
│ 4. mirror    │  Mirror images (ecr-mirror CLI)
└──────────────┘
```

### Stage Details

#### Stage 1: list
- **Purpose:** Display current ECR state
- **Image:** `amazon/aws-cli:latest`
- **What it does:**
  - Installs ecr-mirror CLI
  - Runs `ecr-mirror list` to show all repositories and tags
  - Helps visualize state before mirroring
- **Note:** `allow_failure: true` - Won't stop pipeline if ECR is empty

#### Stage 2: validate
- **Purpose:** Validate configuration file
- **Image:** `alpine:latest`
- **What it does:**
  - Checks if `images.txt` exists
  - Validates each line format (`image:tag`)
  - Reports errors with line numbers
- **Artifacts:** Passes `images.txt` to next stages

#### Stage 3: prepare-ecr
- **Purpose:** Prepare AWS ECR (AWS tasks only)
- **Image:** `amazon/aws-cli:latest`
- **What it does:**
  - Gets ECR login credentials
  - Creates ECR repositories if they don't exist
  - Sets lifecycle policies (keeps last 10 images)
  - Enables vulnerability scanning
  - Tags repositories appropriately
- **Artifacts:** Passes ECR credentials to mirror stage

#### Stage 4: mirror
- **Purpose:** Mirror images using ecr-mirror CLI
- **Image:** `amazon/aws-cli:latest` (with Skopeo installed)
- **What it does:**
  - Uses `ecr-mirror mirror` command
  - Copies images from public registries to ECR
  - Skips existing images automatically
  - Shows final ECR state in `after_script`

### Pipeline Configuration

**File:** `.gitlab-ci-with-cli.yml`

```yaml
stages:
  - list
  - validate
  - prepare-ecr
  - mirror

variables:
  AWS_DEFAULT_REGION: us-east-1      # Your AWS region
  AWS_ACCOUNT_ID: "123456789012"     # Your AWS account ID
  IMAGE_CONFIG_FILE: "images.txt"    # Image list file
```

### Triggers

The pipeline runs on:
- ✅ Push to `main` branch
- ✅ Merge requests (validation only)
- ✅ Scheduled runs (daily sync)

### Setting Up Scheduled Pipeline

For automated daily syncing:

1. Go to **CI/CD → Schedules**
2. Click **New Schedule**
3. Configure:
   - **Description:** Daily ECR Mirror Sync
   - **Interval Pattern:** `0 2 * * *` (2 AM daily)
   - **Cron Timezone:** Your timezone
   - **Target Branch:** `main`
   - **Active:** ✓ Check
4. Click **Save pipeline schedule**

---

## ⚙️ Configuration

### images.txt Format

The `images.txt` file lists images to mirror, one per line:

```txt
# Format: image:tag
# Comments start with #

# Basic Docker Hub images
nginx:1.25.3
redis:7.2-alpine
postgres:15-alpine

# Images from other registries
quay.io/prometheus/prometheus:v2.45.0
gcr.io/cadvisor/cadvisor:v0.47.0

# AWS Public ECR (EKS Distro)
public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0-eks-1-28-4
public.ecr.aws/eks-distro/kubernetes-csi/livenessprobe:v2.10.0-eks-1-28-4
```

**Rules:**
- One image per line in format `image:tag`
- Lines starting with `#` are comments
- Empty lines are ignored
- Docker Hub images: `nginx:1.25.3` or `library/nginx:1.25.3`
- Other registries: Include full path (e.g., `quay.io/prometheus/prometheus:v2.45.0`)

### AWS IAM Permissions

Your AWS user/role needs these ECR permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:CreateRepository",
        "ecr:DescribeRepositories",
        "ecr:DescribeImages",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutLifecyclePolicy",
        "ecr:PutImageScanningConfiguration"
      ],
      "Resource": "*"
    }
  ]
}
```

### GitLab CI/CD Variables

Go to **Settings → CI/CD → Variables** and add:

| Variable | Value | Type | Protected | Masked |
|----------|-------|------|-----------|--------|
| `AWS_ACCESS_KEY_ID` | Your AWS access key | Variable | ✓ | ✓ |
| `AWS_SECRET_ACCESS_KEY` | Your AWS secret key | Variable | ✓ | ✓ |
| `AWS_DEFAULT_REGION` | Your AWS region (e.g., `us-east-1`) | Variable | | |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | Variable | | |

### Repository Naming

ECR repository names are automatically derived from image names:

| Source Image | ECR Repository Name |
|--------------|---------------------|
| `nginx:1.25.3` | `nginx` |
| `library/nginx:1.25.3` | `nginx` |
| `quay.io/prometheus/prometheus:v2.45.0` | `prometheus-prometheus` |
| `public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0` | `kubernetes-csi-external-provisioner` |

**Rules:**
- Slashes (`/`) are replaced with hyphens (`-`)
- Registry domains are removed
- Original tags are preserved

---

## 📚 Examples

### Example 1: Mirror Docker Hub Images

**images.txt:**
```txt
nginx:1.25.3
redis:7.2-alpine
postgres:15-alpine
```

**Command:**
```bash
./ecr-mirror mirror images.txt --account 123456789012
```

**Result:**
```
Your Private ECR:
- 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3
- 123456789012.dkr.ecr.us-east-1.amazonaws.com/redis:7.2-alpine
- 123456789012.dkr.ecr.us-east-1.amazonaws.com/postgres:15-alpine
```

**Use in Kubernetes:**
```yaml
# Before
image: nginx:1.25.3

# After
image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3
```

### Example 2: Mirror AWS Public ECR (EKS Distro)

**images.txt:**
```txt
public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0-eks-1-28-4
public.ecr.aws/eks-distro/kubernetes-csi/csi-node-driver-registrar:v2.8.0-eks-1-28-4
public.ecr.aws/eks-distro/kubernetes-csi/livenessprobe:v2.10.0-eks-1-28-4
```

**Command:**
```bash
./ecr-mirror mirror images.txt --account 123456789012
```

**Result:**
```
Your Private ECR:
- 123456789012.dkr.ecr.us-east-1.amazonaws.com/kubernetes-csi-external-provisioner:v3.5.0-eks-1-28-4
- 123456789012.dkr.ecr.us-east-1.amazonaws.com/kubernetes-csi-csi-node-driver-registrar:v2.8.0-eks-1-28-4
- 123456789012.dkr.ecr.us-east-1.amazonaws.com/kubernetes-csi-livenessprobe:v2.10.0-eks-1-28-4
```

### Example 3: Mirror Multiple Registries

**images.txt:**
```txt
# Docker Hub
nginx:1.25.3

# Quay.io
quay.io/prometheus/prometheus:v2.45.0
quay.io/prometheus/node-exporter:v1.6.0

# AWS Public ECR
public.ecr.aws/eks-distro/kubernetes/pause:v1.28.4-eks-1-28-8

# Google Container Registry
gcr.io/cadvisor/cadvisor:v0.47.0
```

**Command:**
```bash
./ecr-mirror mirror images.txt --account 123456789012
```

All images are mirrored to your private ECR!

### Example 4: Dry Run (Preview)

Test without making changes:

```bash
./ecr-mirror mirror images.txt --account 123456789012 --dry-run
```

**Output:**
```
⚠ DRY-RUN MODE: No changes will be made

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ℹ Processing: nginx:1.25.3

⚠ [DRY-RUN] Would copy:
  From: docker://docker.io/nginx:1.25.3
  To:   docker://123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3
```

### Example 5: Search and Audit

Find what's in your ECR:

```bash
# List everything
./ecr-mirror list

# Find all nginx images
./ecr-mirror search nginx

# Find all EKS 1.28 images
./ecr-mirror search eks-1-28

# Find specific version
./ecr-mirror search v3.5.0
```

---

## 🔍 Troubleshooting

### Issue: "AWS credentials not configured"

**Error:**
```
✗ AWS credentials are not configured
```

**Solution:**
```bash
# Configure AWS CLI
aws configure

# Or set environment variables
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-east-1
```

### Issue: "Skopeo command not found"

**Error:**
```
✗ Skopeo is not installed
```

**Solution:**
```bash
# macOS
brew install skopeo

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install skopeo

# Amazon Linux / RHEL
sudo yum install skopeo

# Alpine (Docker/GitLab CI)
apk add skopeo
```

### Issue: "Error loading trust policy: open /etc/containers/policy.json"

**Error:**
```
Error loading trust policy: open /etc/containers/policy.json: no such file or directory
```

**Solution:**

The tool auto-creates this file. If it still fails:

```bash
# Create policy manually
sudo mkdir -p /etc/containers
sudo cat > /etc/containers/policy.json << 'EOF'
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ]
}
EOF
```

### Issue: "Image already exists in ECR, skipping"

**Not an error!** The tool skips images that already exist to save time.

**To force re-mirror:**
1. Delete the image from ECR
2. Run mirror command again

### Issue: GitLab Pipeline Fails on Mirror Stage

**Check:**
1. ✅ CI/CD variables are set correctly
2. ✅ `ecr-mirror` file is in your repository
3. ✅ AWS credentials have correct permissions
4. ✅ `images.txt` format is valid

**Debug:**
```bash
# Test locally first
./ecr-mirror mirror images.txt --account 123456789012 --dry-run
```

### Issue: "Failed to mirror" for Specific Image

**Possible causes:**
- Image doesn't exist in source registry
- Network timeout
- Rate limiting

**Solution:**
```bash
# Test image exists
skopeo inspect docker://docker.io/nginx:1.25.3

# Check ECR login
aws ecr get-login-password --region us-east-1

# Try manual mirror
skopeo copy \
  docker://docker.io/nginx:1.25.3 \
  docker://123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3
```

---

## 🚀 Advanced Usage

### Custom Lifecycle Policies

Edit the `prepare-ecr` stage in `.gitlab-ci-with-cli.yml`:

```yaml
aws ecr put-lifecycle-policy \
  --repository-name $REPO_NAME \
  --region $AWS_DEFAULT_REGION \
  --lifecycle-policy-text '{
    "rules": [{
      "rulePriority": 1,
      "description": "Keep last 20 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": { "type": "expire" }
    }]
  }'
```

### Multi-Region Mirroring

Mirror to multiple regions:

```bash
# Region 1
./ecr-mirror mirror images.txt --account 123456789012 --region us-east-1

# Region 2
./ecr-mirror mirror images.txt --account 123456789012 --region us-west-2

# Region 3
./ecr-mirror mirror images.txt --account 123456789012 --region eu-west-1
```

### Scripted Updates

Automate adding new images:

```bash
#!/bin/bash
# add-image.sh

IMAGE=$1
TAG=$2

echo "${IMAGE}:${TAG}" >> images.txt
./ecr-mirror mirror images.txt --account 123456789012

echo "Image ${IMAGE}:${TAG} added and mirrored!"
```

Usage:
```bash
./add-image.sh nginx 1.25.4
```

### Integration with Other Tools

#### Terraform
```hcl
# Use mirrored images in EKS
resource "kubernetes_deployment" "app" {
  spec {
    template {
      spec {
        container {
          image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3"
        }
      }
    }
  }
}
```

#### Helm
```yaml
# values.yaml
image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx
  tag: 1.25.3
```

#### ArgoCD
```yaml
# application.yaml
spec:
  source:
    helm:
      parameters:
        - name: image.repository
          value: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx
```

---

## 📊 Benefits

### Cost Savings
- ✅ No data transfer fees for repeated pulls
- ✅ Reduced bandwidth usage
- ✅ Lower ECR costs (lifecycle policies)

### Performance
- ✅ Faster image pulls (same region)
- ✅ Reduced deployment times
- ✅ Less network overhead

### Reliability
- ✅ No dependency on public registry availability
- ✅ Bypass rate limits
- ✅ Works in air-gapped environments
- ✅ Better disaster recovery

### Security
- ✅ Images in your private registry
- ✅ Automatic vulnerability scanning
- ✅ Better compliance
- ✅ Audit trail

---

## 🎉 Summary

You now have a complete solution for:
- ✅ Automated image mirroring to ECR
- ✅ CLI tool for local use
- ✅ GitLab CI/CD pipeline
- ✅ Search and audit capabilities
- ✅ Production-ready configuration

**Get started:**
```bash
./ecr-mirror mirror images.txt --account 123456789012
```

**Happy mirroring!** 🚀
