# aws-ml-deployment

Reusable Docker build workflow examples for consumer repositories using your-org/autodevops.

Prerequisites
- Your reusable workflow lives at your-org/autodevops/.github/workflows/docker-build-reusable.yml.
- Target repos have GitHub Actions enabled and permission to push to your registry (e.g., GHCR).
- Required job-level permissions are set (see examples below).

Option A: Simple usage (all defaults)
- File: .github/workflows/docker.yml in the consumer repository

```yaml
name: Docker Build
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
jobs:
  docker:
    uses: your-org/autodevops/.github/workflows/docker-build-reusable.yml@main
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
```

Option B: Advanced usage with custom inputs
- File: .github/workflows/docker.yml in the consumer repository

```yaml
name: Docker Build (Custom)
on:
  push:
    branches: [main, staging, production]
  pull_request:
  workflow_dispatch:
jobs:
  docker:
    uses: your-org/autodevops/.github/workflows/docker-build-reusable.yml@main
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    with:
      dockerfile_path: 'docker/Dockerfile'
      build_context: '.'
      platforms: 'linux/amd64'
      custom_tags: |
        type=raw,value=stable,enable=${{ github.ref == 'refs/heads/production' }}
  post-build:
    needs: docker
    runs-on: ubuntu-latest
    if: success()
    steps:
      - name: Display build info
        run: |
          echo "Image Tags: ${{ needs.docker.outputs.image_tags }}"
          echo "Image Digest: ${{ needs.docker.outputs.image_digest }}"
```

Version pinning example
- Best practice: pin to a specific version/tag of the reusable workflow instead of @main

```yaml
name: Docker Build (Pinned)
on:
  push:
    branches: [main]
jobs:
  docker:
    uses: your-org/autodevops/.github/workflows/docker-build-reusable.yml@v1.0.0
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
```

Notes
- Replace your-org/autodevops with the actual owner/repo hosting the reusable workflow.
- Ensure the reusable workflow defines outputs image_tags and image_digest if you plan to consume them in downstream jobs.
- If your builds run on pull_request and should not push, ensure the reusable workflow conditionally disables push (commonly implemented via push_enabled and event checks).
- Consider pinning to specific action SHAs for supply-chain hardening in the reusable workflow itself.
