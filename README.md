# ci-templates

GitHub Actions 可复用 workflow 模板。应用源码仓一行 `uses` 即可获得与平台一致的流水线：

**build(GHCR) → trivy 扫描（CRITICAL 阻断）→ cosign keyless 签名 → 自动改 k8s-platform 的 `newTag`**

> 为什么单独建公开仓：GitHub 限制——私有仓的 reusable workflow 不能被其他私有仓调用，
> 所以本仓必须 public，各应用仓才能引用。

## 应用仓接入

在应用源码仓 `.github/workflows/ci.yml`：

```yaml
name: ci
on:
  push:
    branches: [main]
permissions:
  contents: read
  packages: write
  id-token: write
jobs:
  deploy:
    uses: byte-marvel/ci-templates/.github/workflows/app-ci.yml@main
    with:
      app: <应用名>            # = k8s-platform 的 apps/<app> 目录名 = GHCR 包名
      # context: .            # 构建上下文，缺省仓库根
      # dockerfile: Dockerfile # 相对构建上下文，缺省 Dockerfile
    secrets:
      DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}   # 跨仓改 newTag 的令牌
```

## 输入

| input | 必填 | 说明 |
|---|---|---|
| `app` | ✅ | 应用名，须与 k8s-platform `apps/<app>` 目录名、GHCR 包名一致 |
| `image` | — | 完整镜像名（不含 tag），缺省 `ghcr.io/byte-marvel/<app>` |
| `context` | — | 构建上下文路径，缺省仓库根 |
| `dockerfile` | — | Dockerfile 路径（相对 context），缺省 `Dockerfile` |
| `tcr_namespace` | — | TCR 个人版命名空间；配 secrets `TCR_USER`/`TCR_PASSWORD` 启用 TCR 推送 |
| `runs_on` | — | `ubuntu-latest`（默认，境外）或 `self-hosted`（国内节点，只推 TCR） |
| `push_ghcr` | — | 是否推 GHCR（默认 true；self-hosted 模式应设 false） |
| `buildx_builder` | — | self-hosted 模式复用的持久 buildx builder 名 |

## Secret

| secret | 说明 |
|---|---|
| `DEPLOY_TOKEN` | fine-grained PAT，只授权 `byte-marvel/k8s-platform` 仓 Contents Read/Write。不传时镜像照常发布（`:latest`），仅跳过自动改 `newTag` |
| `TCR_USER` / `TCR_PASSWORD` | 腾讯云 TCR 个人版访问凭证。配合 input `tcr_namespace` 启用 TCR 推送（集群拉 GHCR 跨境极慢）。self-hosted 模式必传 |
