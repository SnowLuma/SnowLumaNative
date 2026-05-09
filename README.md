# SnowLumaNative

构建并缓存 SnowLuma 的 Linux 原生产物（`snowluma-linux-{x64,arm64}.{node,so}`），由 [`SnowLuma/nnphook`](https://github.com/SnowLuma/nnphook) 触发，构建完成后将产物发布到本仓库 Release，并向 [`SnowLuma/SnowLuma`](https://github.com/SnowLuma/SnowLuma) 自动提交 PR 替换 `packages/runtime/native/` 下的对应文件。

## 流水线

```
nnphook (本地仓库)
   │  workflow_dispatch / push tag v* / commit msg [build-native]
   ▼
nnphook/.github/workflows/dispatch-native-build.yml
   │  repository_dispatch  event_type = nnphook-native-build
   ▼
SnowLumaNative/.github/workflows/build-native.yml
   ├─ checkout SnowLuma/nnphook @ ref            (SNOWLUMA_TOKEN)
   ├─ cmake -G Ninja Release  →  matrix x64 + arm64
   ├─ verify stripped: 无 .debug/.symtab/.strtab
   ├─ publish Release on SnowLumaNative          (GITHUB_TOKEN)
   │     - snowluma-linux-x64.{node,so}
   │     - snowluma-linux-arm64.{node,so}
   │     - SHA256SUMS.txt
   └─ open PR on SnowLuma/SnowLuma               (SNOWLUMA_TOKEN)
         分支: native/auto-update-<release_tag>
         替换:
           packages/runtime/native/snowluma-linux-x64.node
           packages/runtime/native/snowluma-linux-x64.so
           packages/runtime/native/snowluma-linux-arm64.node
           packages/runtime/native/snowluma-linux-arm64.so
```

## 触发方式（在 nnphook 仓库一侧）

1. **手动**：Actions → `Dispatch SnowLuma Native Build` → `Run workflow`，可填 ref / release_tag / open_pr。
2. **打 tag**：`git tag vX.Y.Z && git push origin vX.Y.Z`，自动以 tag 名作为 release_tag。
3. **提交信息标记**：向 `main` 推送的 commit message 中包含 `[build-native]` 或 `[release-native]` 即触发，release_tag 自动取 `native-<short_sha>`。

## 所需 Secrets

所有 GitHub PAT 统一命名为 **`SNOWLUMA_TOKEN`**，需要在以下三个仓库各自配置同名 secret（值可以是同一个 PAT，也可以是分别签发的 fine-grained PAT，只要权限覆盖到对应用途即可）：

| 仓库 | 用途 | 该仓库下 `SNOWLUMA_TOKEN` 需要的权限 |
| --- | --- | --- |
| `SnowLuma/nnphook` | 调用 API 在 SnowLumaNative 上触发 `repository_dispatch`；`release-docker.yml` 跨仓 checkout SnowLuma + Docker 框架 | 对 `SnowLumaNative`：`contents:write`；对 `SnowLuma/SnowLuma` 和 Docker 框架仓库：`contents:read` |
| `SnowLuma/SnowLumaNative` | checkout 私有 nnphook 源码；在 SnowLuma 上推分支 + 开 PR | 对 `nnphook`：`contents:read`；对 `SnowLuma/SnowLuma`：`contents:write` + `pull-requests:write` |
| `SnowLuma/SnowLuma`（如另有需要）| 由其他流水线引用 | 按需 |

> 最简方案：签发一枚 classic PAT（`repo` scope，覆盖三个仓库），分别复制到三个仓库的 `SNOWLUMA_TOKEN` secret 即可。
> `GITHUB_TOKEN` 已在 `release` job 通过 `permissions: contents: write` 授权创建 Release，无需额外 secret。
> Docker Hub 凭据 `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` 与本流水线无关，仅 `release-docker.yml` 使用，保持不变。

## 手动跑一次（不通过 nnphook）

进入本仓库 Actions → `Build SnowLuma Native` → `Run workflow`，填好 inputs 即可，等价于 nnphook 的 dispatch。
