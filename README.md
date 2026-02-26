# merge-docker-digest

GitHub Action for merging multi-architecture Docker image digests into a single manifest.

## Overview

When building Docker images for multiple architectures (e.g., amd64, arm64), each build produces a separate digest. This action merges those digests into a unified multi-arch manifest, allowing users to `docker pull` and automatically receive the correct image for their architecture.

## Features

- Supports multiple container registries (Docker Hub, GHCR, and others)
- Merges digests from parallel builds into a single manifest
- Applies multiple tags to the merged image

## Usage

```yaml
- name: Merge Manifest
  uses: thanatosdi/merge-docker-digest@main
  with:
    image-name: myuser/myrepo
    artifact-pattern: digest-*
    tags: |
      latest
      1.0.0
    registry-username: ${{ secrets.DOCKERHUB_USERNAME }}
    registry-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `image-name` | Yes | - | Full image name (e.g., `ghcr.io/user/repo` or `user/repo` for Docker Hub) |
| `artifact-pattern` | Yes | - | Pattern to match digest artifacts (e.g., `digest-*`) |
| `tags` | Yes | - | Tags to apply, separated by newlines |
| `timezone` | No | `UTC` | Timezone for date-based tags |
| `registry` | No | `''` | Container registry URL (e.g., `ghcr.io`, `docker.io`). Leave empty for Docker Hub |
| `registry-username` | Yes | - | Registry username (⚠️ Always pass via secrets! For GHCR, use `github.actor`) |
| `registry-password` | Yes | - | Registry password/token (⚠️ Always pass via secrets! For GHCR, use `secrets.GITHUB_TOKEN`) |

## Examples

### Docker Hub

```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [linux/amd64, linux/arm64]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push by digest
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: ${{ matrix.platform }}
          outputs: type=image,name=myuser/myrepo,push-by-digest=true,push=true

      - name: Export digest
        run: |
          mkdir -p /tmp/digests
          digest="${{ steps.build.outputs.digest }}"
          echo "${digest#sha256:}" > "/tmp/digests/${digest#sha256:}"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digest-${{ matrix.platform == 'linux/amd64' && 'amd64' || 'arm64' }}
          path: /tmp/digests/*

  merge:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Merge Manifest
        uses: thanatosdi/merge-docker-digest@main
        with:
          image-name: myuser/myrepo
          artifact-pattern: digest-*
          tags: |
            latest
            1.0.0
          registry-username: ${{ secrets.DOCKERHUB_USERNAME }}
          registry-password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### GitHub Container Registry (GHCR)

```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        platform: [linux/amd64, linux/arm64]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push by digest
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: ${{ matrix.platform }}
          outputs: type=image,name=ghcr.io/${{ github.repository }},push-by-digest=true,push=true

      - name: Export digest
        run: |
          mkdir -p /tmp/digests
          digest="${{ steps.build.outputs.digest }}"
          echo "${digest#sha256:}" > "/tmp/digests/${digest#sha256:}"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digest-${{ matrix.platform == 'linux/amd64' && 'amd64' || 'arm64' }}
          path: /tmp/digests/*

  merge:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Merge Manifest
        uses: thanatosdi/merge-docker-digest@main
        with:
          image-name: ghcr.io/${{ github.repository }}
          artifact-pattern: digest-*
          tags: |
            latest
            ${{ github.sha }}
          registry: ghcr.io
          registry-username: ${{ github.actor }}
          registry-password: ${{ secrets.GITHUB_TOKEN }}
```

## How It Works

1. **Download Digests**: Fetches all digest artifacts matching the specified pattern
2. **Setup Buildx**: Configures Docker Buildx for manifest manipulation
3. **Login**: Authenticates with the specified container registry
4. **Create Manifest**: Uses `docker buildx imagetools create` to merge all digests into a multi-arch manifest with the specified tags
5. **Inspect**: Verifies the created manifest

## License

See [LICENSE](LICENSE) for details.

---

> Translated with AI

