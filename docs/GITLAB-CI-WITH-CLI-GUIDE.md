# GitLab CI Pipeline with ECR Mirror CLI

This GitLab CI pipeline uses the `ecr-mirror` CLI tool to mirror container images from public registries to your private AWS ECR.

## Pipeline Overview

The pipeline has **4 stages**:

```
┌──────────────┐
│ 1. list      │ ← List current ECR contents
└──────┬───────┘
       │
┌──────▼───────┐
│ 2. validate  │ ← Validate images.txt format
└──────┬───────┘
       │
┌──────▼───────┐
│ 3. prepare   │ ← Create ECR repos (AWS CLI)
└──────┬───────┘
       │
┌──────▼───────┐
│ 4. mirror    │ ← Mirror images (ecr-mirror CLI)
└──────────────┘
```

---

## Stage Details

### Stage 1: list
**Purpose:** Show what's currently in your ECR

**Image:** `amazon/aws-cli:latest`

**What it does:**
- Installs `ecr-mirror` CLI tool
- Runs `ecr-mirror list` to show all repositories and tags
- Helpful for seeing state before mirroring

**Output Example:**
```
==========================================
Current ECR Contents
==========================================
Repository: nginx
URI: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx
Images: 2 tagged
Tags:
  • 1.25.0
  • 1.25.3
```

**Notes:**
- `allow_failure: true` - Won't stop pipeline if ECR is empty
- Runs on: main branch, merge requests

---

### Stage 2: validate
**Purpose:** Validate `images.txt` configuration file

**Image:** `alpine:latest`

**What it does:**
- Checks if `images.txt` exists
- Validates each line format (`image:tag`)
- Reports any errors with line numbers

**Output Example:**
```
==========================================
Validating Configuration
==========================================
✓ Configuration file exists: images.txt

Checking image format...
✓ Line 1: nginx:1.25.3
✓ Line 2: public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0-eks-1-28-4
✓ Line 3: quay.io/prometheus/prometheus:v2.45.0

==========================================
Validation Summary
==========================================
✓ All images validated successfully
```

**Artifacts:**
- `images.txt` → Passed to next stage

---

### Stage 3: prepare-ecr
**Purpose:** Create ECR repositories (AWS tasks only)

**Image:** `amazon/aws-cli:latest`

**What it does:**
- Gets ECR login credentials
- Saves credentials to `ecr-password.txt`
- Creates ECR repositories if they don't exist
- Sets lifecycle policies (keep last 10 images)
- Enables image scanning

**Output Example:**
```
==========================================
Preparing ECR Repositories
==========================================
Region: us-east-1
Account: 123456789012

Getting ECR credentials...
✓ ECR credentials saved

Creating ECR repositories...
----------------------------------------
Processing: nginx:1.25.3
  ECR Repository: nginx
  ✓ Repository already exists

----------------------------------------
Processing: public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0-eks-1-28-4
  ECR Repository: kubernetes-csi-external-provisioner
  Creating repository...
  Setting lifecycle policy...
  ✓ Repository created successfully

==========================================
✓ All ECR repositories ready
✓ ECR credentials prepared for next stage
==========================================
```

**Artifacts:**
- `ecr-password.txt` → ECR credentials for mirror stage
- `images.txt` → Image list

---

### Stage 4: mirror
**Purpose:** Mirror images using `ecr-mirror` CLI

**Image:** `quay.io/skopeo/stable:latest`

**What it does:**
- Installs `ecr-mirror` CLI tool
- Reads artifacts from Stage 3
- Uses `ecr-mirror mirror` command to sync images
- Shows final ECR contents in `after_script`

**Output Example:**
```
==========================================
Mirroring Images to ECR
==========================================
Using ecr-mirror CLI tool

ℹ Verifying AWS credentials...
✓ Authenticated as: arn:aws:iam::123456789012:user/gitlab-ci
ℹ ECR Registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ℹ Processing: nginx:1.25.3
  Source Image: nginx:1.25.3
  ECR Destination: 123456789012.dkr.ecr.us-east-1.amazonaws.com/nginx:1.25.3

ℹ Repository exists: nginx
ℹ Copying image with skopeo...
✓ Successfully mirrored: nginx:1.25.3

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Successful: 3
Failed: 0
Skipped: 0

✓ All images mirrored successfully!

==========================================
Final ECR Contents
==========================================
Repository: nginx
Tags:
  • 1.25.0
  • 1.25.3
```

---

## Setup Instructions

### 1. Add the Pipeline File

```bash
# Copy the pipeline file
cp .gitlab-ci-with-cli.yml .gitlab-ci.yml

# Commit
git add .gitlab-ci.yml
git commit -m "Add ECR mirror pipeline with CLI"
git push
```

### 2. Upload ecr-mirror CLI

You need to make the `ecr-mirror` tool available. Options:

**Option A: Add to your repository**
```bash
# Add ecr-mirror to your repo
cp ecr-mirror your-repo/
git add ecr-mirror
git commit -m "Add ecr-mirror CLI tool"
git push
```

Then update pipeline to use:
```yaml
- cp ecr-mirror /usr/local/bin/ecr-mirror
```

**Option B: Host on GitLab**
Upload `ecr-mirror` to your GitLab repository and update the pipeline:
```yaml
- wget https://gitlab.com/your-org/your-repo/-/raw/main/ecr-mirror -O /usr/local/bin/ecr-mirror
```

**Option C: Use GitLab Package Registry**
Upload to package registry and download in pipeline.

### 3. Configure GitLab CI/CD Variables

Go to: **Settings → CI/CD → Variables**

Add these variables:

| Variable | Value | Type | Protected | Masked |
|----------|-------|------|-----------|--------|
| `AWS_ACCESS_KEY_ID` | `AKIA...` | Variable | ✓ | ✓ |
| `AWS_SECRET_ACCESS_KEY` | `xyz...` | Variable | ✓ | ✓ |
| `AWS_DEFAULT_REGION` | `us-east-1` | Variable | | |
| `AWS_ACCOUNT_ID` | `123456789012` | Variable | | |

### 4. Update Pipeline Variables

Edit `.gitlab-ci.yml`:

```yaml
variables:
  AWS_DEFAULT_REGION: us-east-1      # Your region
  AWS_ACCOUNT_ID: "123456789012"     # Your account ID
  IMAGE_CONFIG_FILE: "images.txt"
  ECR_MIRROR_VERSION: "1.0.0"
```

### 5. Create images.txt

```bash
cat > images.txt << 'EOF'
# Container images to mirror
nginx:1.25.3
public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner:v3.5.0-eks-1-28-4
quay.io/prometheus/prometheus:v2.45.0
EOF

git add images.txt
git commit -m "Add images to mirror"
git push
```

---

## Usage

### Manual Pipeline Run

1. Go to **CI/CD → Pipelines**
2. Click **Run Pipeline**
3. Select branch: `main`
4. Click **Run Pipeline**

### Automatic Triggers

The pipeline runs automatically on:
- ✅ Push to `main` branch
- ✅ Merge requests (for validation)
- ✅ Scheduled pipelines (for regular syncing)

### Set Up Scheduled Pipeline

**For daily syncing:**

1. Go to **CI/CD → Schedules**
2. Click **New Schedule**
3. Configure:
   - **Description:** Daily ECR Mirror Sync
   - **Interval Pattern:** `0 2 * * *` (2 AM daily)
   - **Target Branch:** `main`
   - **Active:** ✓
4. Save

---

## Pipeline Behavior

### First Run (Empty ECR)
```
1. list      → Shows "No ECR repositories found"
2. validate  → ✓ Validates images.txt
3. prepare   → Creates all ECR repos
4. mirror    → Mirrors all images
```

### Subsequent Runs (Existing Images)
```
1. list      → Shows existing images
2. validate  → ✓ Validates images.txt
3. prepare   → Repos exist, gets credentials
4. mirror    → Skips existing images, mirrors new ones
```

### After Adding New Image
```
1. Edit images.txt (add new image)
2. Push to main
3. Pipeline runs:
   - Validates new image
   - Creates new ECR repo if needed
   - Mirrors only new image
   - Skips existing images
```

---

## Advantages Over Manual Skopeo

| Aspect | This Pipeline | Manual Skopeo |
|--------|---------------|---------------|
| **Tool** | `ecr-mirror` CLI | Manual scripts |
| **Validation** | ✓ Built-in | Manual |
| **ECR Repos** | Auto-created | Manual |
| **Duplicate Check** | ✓ Automatic | Manual |
| **Lifecycle Policies** | ✓ Automatic | Manual |
| **Error Handling** | ✓ Built-in | Manual |
| **List ECR** | ✓ Before/after | Manual |
| **Readability** | ✓ High | Scripts |

---

## Customization

### Change Region

Edit `.gitlab-ci.yml`:
```yaml
variables:
  AWS_DEFAULT_REGION: us-west-2  # Change region
```

### Change Lifecycle Policy

Edit Stage 3 (`prepare-ecr`):
```yaml
--lifecycle-policy-text '{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep last 20 images",  # Changed from 10
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 20                    # Changed from 10
    },
    "action": { "type": "expire" }
  }]
}'
```

### Disable List Stage

Comment out the `list_ecr_contents` job:
```yaml
# list_ecr_contents:
#   stage: list
#   ...
```

### Add Notifications

Add to the end of `.gitlab-ci.yml`:
```yaml
notify_success:
  stage: .post
  image: alpine:latest
  script:
    - echo "ECR mirror completed successfully!"
    # Add Slack/email notification here
  when: on_success
```

---

## Troubleshooting

### Issue: "ecr-mirror: command not found"

**Cause:** CLI tool not downloaded properly

**Solution:**
```yaml
# Verify wget URL is correct
- wget https://your-correct-url/ecr-mirror -O /usr/local/bin/ecr-mirror
- chmod +x /usr/local/bin/ecr-mirror
- ecr-mirror version  # Test it works
```

### Issue: "AWS credentials not configured"

**Cause:** Missing GitLab CI/CD variables

**Solution:**
1. Go to Settings → CI/CD → Variables
2. Add `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
3. Make sure they're marked as **Protected** and **Masked**

### Issue: "ECR password not found"

**Cause:** Artifact not passed from Stage 3

**Solution:**
Verify `dependencies` in Stage 4:
```yaml
dependencies:
  - prepare_ecr_repositories
```

### Issue: Stage 3 creates repos but Stage 4 fails

**Cause:** ECR credentials expired (rare)

**Solution:** Pipeline auto-retries. If persistent:
```yaml
artifacts:
  expire_in: 30 minutes  # Reduce from 1 hour
```

---

## Monitoring

### View Pipeline Progress

**GitLab UI:**
1. Go to **CI/CD → Pipelines**
2. Click on running pipeline
3. Watch each stage execute

### View Stage Logs

1. Click on a stage (e.g., "mirror")
2. See real-time output from `ecr-mirror`

### Check ECR Contents

After pipeline completes:
```bash
# Use ecr-mirror locally
./ecr-mirror list --region us-east-1

# Or check AWS Console
# Go to ECR → Repositories
```

---

## File Structure

```
your-repo/
├── .gitlab-ci.yml              # This pipeline
├── ecr-mirror                  # CLI tool (optional, can host elsewhere)
├── images.txt                  # Images to mirror
└── README.md
```

---

## Comparison: This Pipeline vs Others

| Feature | This Pipeline | Direct Skopeo | Kaniko/Skopeo |
|---------|---------------|---------------|---------------|
| **Uses CLI** | ✓ `ecr-mirror` | Manual scripts | Manual scripts |
| **List ECR** | ✓ Built-in | Manual | Manual |
| **Validate** | ✓ Automatic | Manual | Manual |
| **Create Repos** | ✓ Automatic | Manual | Manual |
| **Readability** | ✓ High | Medium | Medium |
| **Maintainability** | ✓ Easy | Hard | Medium |
| **Debugging** | ✓ Easy | Medium | Medium |

---

## Example Workflow

### Day 1: Initial Setup
```bash
# 1. Add files
cp .gitlab-ci-with-cli.yml .gitlab-ci.yml
cp ecr-mirror .
cat > images.txt << 'EOF'
nginx:1.25.3
postgres:15-alpine
EOF

# 2. Configure GitLab variables
# (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, etc.)

# 3. Push
git add .
git commit -m "Initial ECR mirror setup"
git push

# 4. Pipeline runs:
#    - Validates images.txt
#    - Creates nginx and postgres repos in ECR
#    - Mirrors both images
```

### Day 30: Add New Image
```bash
# 1. Edit images.txt
echo "redis:7-alpine" >> images.txt

# 2. Push
git add images.txt
git commit -m "Add redis image"
git push

# 3. Pipeline runs:
#    - Validates images.txt
#    - Creates redis repo in ECR
#    - Mirrors only redis (skips nginx, postgres)
```

### Daily: Scheduled Sync
```
# Automatically at 2 AM:
- Checks for image updates
- Mirrors any new tags
- Skips existing images
```

---

## Summary

**This pipeline:**
- ✅ Uses `ecr-mirror` CLI for clean, maintainable code
- ✅ Shows ECR contents before/after mirroring
- ✅ Validates configuration automatically
- ✅ Handles all AWS tasks in separate stage
- ✅ Auto-creates ECR repos with best practices
- ✅ Skips existing images automatically
- ✅ Easy to read and debug
- ✅ Perfect for your AWS Public ECR use case

**Files:**
- [.gitlab-ci-with-cli.yml](computer:///mnt/user-data/outputs/.gitlab-ci-with-cli.yml) - The pipeline ⭐
- [ecr-mirror](computer:///mnt/user-data/outputs/ecr-mirror) - CLI tool
- [images.txt](computer:///mnt/user-data/outputs/example-images.txt) - Example config

**Ready to use!** 🚀
