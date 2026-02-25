# merge-docker-digest

用於將多架構 Docker 映像 digest 合併為單一 manifest 的 GitHub Action。

## 概述

在為多種架構（例如 amd64、arm64）建置 Docker 映像時，每次建置都會產生獨立的 digest。此 action 會將這些 digest 合併為統一的多架構 manifest，讓使用者執行 `docker pull` 時能自動取得對應其架構的映像。

## 功能特點

- 支援多種容器倉庫（Docker Hub、GHCR 及其他）
- 將平行建置的 digest 合併為單一 manifest
- 可為合併後的映像套用多個標籤

## 使用方式

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

### 輸入參數

| 名稱 | 必填 | 預設值 | 說明 |
|------|------|--------|------|
| `image-name` | 是 | - | 完整映像名稱（例如 `ghcr.io/user/repo` 或 Docker Hub 的 `user/repo`） |
| `artifact-pattern` | 是 | - | 用於匹配 digest artifact 的 pattern（例如 `digest-*`） |
| `tags` | 是 | - | 要套用的標籤，以換行分隔 |
| `timezone` | 否 | `UTC` | 日期標籤的時區 |
| `registry` | 否 | `''` | 容器倉庫 URL（例如 `ghcr.io`、`docker.io`）。Docker Hub 請留空 |
| `registry-username` | 是 | - | 倉庫使用者名稱 |
| `registry-password` | 是 | - | 倉庫密碼或存取權杖 |

## 範例

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

## 運作原理

1. **下載 Digests**：取得所有符合指定 pattern 的 digest artifacts
2. **設定 Buildx**：配置 Docker Buildx 以進行 manifest 操作
3. **登入**：向指定的容器倉庫進行身份驗證
4. **建立 Manifest**：使用 `docker buildx imagetools create` 將所有 digest 合併為多架構 manifest，並套用指定的標籤
5. **檢查**：驗證已建立的 manifest

## 授權條款

詳見 [LICENSE](LICENSE)。
