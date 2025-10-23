# AutoDevOps Pipeline

**Automated CI/CD pipeline for containerized applications with semantic versioning, Docker builds, and AWS ECS deployment.**

---

## 🚀 Quick Start

Add this to `.github/workflows/ci-cd.yml` in your repository:

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  pipeline:
    uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@main
    permissions:
      contents: write
      packages: write
      id-token: write
      issues: write
      pull-requests: write
      attestations: write
```

**That's it!** 🎉 Your repo now has:
- ✅ Automatic semantic versioning
- ✅ Docker image builds and publishing to GHCR
- ✅ Automated releases and changelogs

---

## 📋 What It Does

The pipeline automatically:

1. **Analyzes your commits** using [Conventional Commits](https://www.conventionalcommits.org/)
2. **Determines version bump** (major, minor, or patch)
3. **Creates Git tags** and GitHub releases
4. **Generates CHANGELOG.md**
5. **Builds Docker images** (if Dockerfile exists)
6. **Pushes to GitHub Container Registry** (ghcr.io)
7. **Optionally deploys to AWS ECS** (if enabled)

---

## 🎯 Usage Examples

### Basic Usage (Docker Build Only)

No configuration needed! Just add the workflow file above.

**Requirements:**
- A `Dockerfile` in your repository root

**What you get:**
- Images pushed to `ghcr.io/your-username/your-repo:version`
- Automatic tagging: `1.2.3`, `1.2`, `1`, `latest`

---

### With AWS ECS Deployment

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  pipeline:
    uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@main
    permissions:
      contents: write
      packages: write
      id-token: write
      issues: write
      pull-requests: write
      attestations: write
    with:
      # Enable AWS deployment
      deploy_to_aws: true
      
      # AWS Configuration
      aws_region: us-east-1
      ecr_repository: my-app
      ecs_cluster: production-cluster
      ecs_service: my-app-service
      ecs_task_definition: .aws/task-definition.json
      container_name: my-app-container
      
      # Authentication (OIDC recommended)
      aws_role_arn: ${{ vars.AWS_ROLE_ARN }}
```

**Additional Requirements:**
- ECS task definition file (e.g., `.aws/task-definition.json`)
- AWS credentials configured (see [AWS Setup](#-aws-setup))

---

### Custom Docker Configuration

```yaml
jobs:
  pipeline:
    uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@main
    permissions:
      contents: write
      packages: write
      id-token: write
      issues: write
      pull-requests: write
      attestations: write
    with:
      dockerfile_path: docker/Dockerfile  # Custom Dockerfile location
      platforms: linux/amd64              # Build for specific platform only
```

---

## 📝 Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/) to trigger releases:

### Patch Release (1.0.0 → 1.0.1)
```bash
git commit -m "fix: resolve authentication bug"
git commit -m "docs: update installation guide"
git commit -m "perf: optimize database queries"
```

### Minor Release (1.0.0 → 1.1.0)
```bash
git commit -m "feat: add user profile page"
git commit -m "feat(api): implement search endpoint"
```

### Major Release (1.0.0 → 2.0.0)
```bash
# Option 1: Using exclamation mark
git commit -m "feat!: remove legacy API support"

# Option 2: Using BREAKING CHANGE footer
git commit -m "feat: redesign authentication flow

BREAKING CHANGE: JWT tokens are now required for all endpoints"
```

### No Release
```bash
git commit -m "chore: update dependencies"
git commit -m "ci: fix workflow syntax"
git commit -m "test: add unit tests"
git commit -m "refactor: simplify error handling"
```

---

## ⚙️ Configuration Options

### All Available Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `deploy_to_aws` | Enable AWS ECS deployment | No | `false` |
| `dockerfile_path` | Path to Dockerfile | No | `Dockerfile` |
| `platforms` | Docker platforms (comma-separated) | No | `linux/amd64,linux/arm64` |
| `aws_region` | AWS region | If deploying | - |
| `ecr_repository` | ECR repository name | If deploying | - |
| `ecs_cluster` | ECS cluster name | If deploying | - |
| `ecs_service` | ECS service name | If deploying | - |
| `ecs_task_definition` | Path to task definition JSON | If deploying | `.aws/task-definition.json` |
| `container_name` | Container name in task definition | If deploying | - |
| `aws_role_arn` | AWS IAM role ARN for OIDC | If using OIDC | - |

### Outputs

| Output | Description |
|--------|-------------|
| `version` | The semantic version that was released (e.g., `1.2.3`) |
| `image_tags` | Docker image tags that were created |
| `deployed` | Whether AWS deployment was successful |

---

## 🔐 AWS Setup

### Option A: OIDC (Recommended) ⭐

More secure - no long-lived credentials needed.

#### 1. Create OIDC Provider in AWS

```bash
# In AWS Console: IAM → Identity Providers → Add Provider
Provider URL: https://token.actions.githubusercontent.com
Audience: sts.amazonaws.com
```

#### 2. Create IAM Role

**Trust Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

**Permissions Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeServices",
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskExecutionRole"
    }
  ]
}
```

#### 3. Add Role ARN to Repository

```bash
# GitHub: Settings → Secrets and variables → Actions → Variables
# Add new variable:
Name: AWS_ROLE_ARN
Value: arn:aws:iam::123456789012:role/GitHubActionsRole
```

#### 4. Use in Workflow

```yaml
with:
  aws_role_arn: ${{ vars.AWS_ROLE_ARN }}
```

---

### Option B: Access Keys

Simpler but less secure.

#### 1. Create IAM User

Create IAM user with the same permissions policy as above.

#### 2. Generate Access Keys

In AWS Console: IAM → Users → Security Credentials → Create Access Key

#### 3. Add to Repository Secrets

```bash
# GitHub: Settings → Secrets and variables → Actions → Secrets
# Add two secrets:
AWS_ACCESS_KEY_ID: AKIA...
AWS_SECRET_ACCESS_KEY: wJalrXUtn...
```

#### 4. Use in Workflow

```yaml
with:
  deploy_to_aws: true
  aws_region: us-east-1
  # ... other AWS settings ...
secrets:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## 🏗️ AWS Infrastructure Setup

### 1. Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name my-app \
  --region us-east-1
```

### 2. Create ECS Task Definition

Save as `.aws/task-definition.json`:

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "my-app-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole"
}
```

### 3. Create ECS Cluster and Service

```bash
# Create cluster
aws ecs create-cluster \
  --cluster-name production-cluster \
  --region us-east-1

# Create service
aws ecs create-service \
  --cluster production-cluster \
  --service-name my-app-service \
  --task-definition my-app \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"
```

---

## 🔍 Troubleshooting

### No Release Created

**Problem:** Push to main but no release created.

**Solution:** Check your commit messages follow [Conventional Commits](#-commit-message-format). Use `feat:`, `fix:`, or `BREAKING CHANGE:`.

### Docker Build Failed

**Problem:** `Dockerfile not found`

**Solution:** Ensure you have a `Dockerfile` in your repository root, or specify `dockerfile_path` input.

### AWS Deployment Failed

**Problem:** Permission denied or authentication failed.

**Solutions:**
- Verify IAM role/user has correct permissions
- Check `aws_role_arn` is correct
- Ensure OIDC provider is configured properly
- Verify task definition file exists at specified path

### Image Not Found in ECR

**Problem:** ECS deployment fails with "image not found"

**Solution:** The workflow automatically pulls from GHCR and pushes to ECR. Ensure:
- Docker build succeeded
- ECR repository exists
- AWS credentials are valid

---

## 📊 Workflow Outputs

Access outputs in subsequent jobs:

```yaml
jobs:
  pipeline:
    uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@main
    # ... configuration ...

  notify:
    needs: [pipeline]
    runs-on: ubuntu-latest
    steps:
      - name: Send notification
        run: |
          echo "Deployed version: ${{ needs.pipeline.outputs.version }}"
          echo "Deployment status: ${{ needs.pipeline.outputs.deployed }}"
```

---

## 🎓 Best Practices

### 1. Version Pinning

Pin to a specific version instead of `@main` for production:

```yaml
uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@v1.0.0
```

### 2. Branch Protection

Enable branch protection on `main`:
- Require pull request reviews
- Require status checks to pass
- Require signed commits (optional)

### 3. Environment Secrets

Use GitHub Environments for different deployment targets:

```yaml
jobs:
  pipeline:
    environment: production  # Requires approval
    uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@main
    # ... rest of config ...
```

### 4. Multi-Environment Deployments

Create separate workflows for different environments:

```yaml
# .github/workflows/deploy-staging.yml
on:
  push:
    branches: [develop]

jobs:
  pipeline:
    uses: jasperbruin/aws-ml-deployment/.github/workflows/main-pipeline.yml@main
    with:
      deploy_to_aws: true
      aws_region: us-east-1
      ecs_cluster: staging-cluster
      ecs_service: my-app-staging
      # ... other staging config ...
```

---

## 🆘 Support

- **Issues:** [GitHub Issues](https://github.com/jasperbruin/aws-ml-deployment/issues)
- **Discussions:** [GitHub Discussions](https://github.com/jasperbruin/aws-ml-deployment/discussions)
- **Documentation:** [Conventional Commits](https://www.conventionalcommits.org/)

---

## 📄 License

This project is available for use by all developers.

---

## 🙏 Acknowledgments

Built with:
- [semantic-release](https://github.com/semantic-release/semantic-release)
- [Docker Buildx](https://github.com/docker/buildx)
- [GitHub Actions](https://github.com/features/actions)
- [AWS Actions](https://github.com/aws-actions)