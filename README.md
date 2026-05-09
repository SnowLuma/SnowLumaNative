# SnowLumaNative

构建并缓存 SnowLuma 的原生产物（Linux x64/arm64 + Windows x64，共 6 个文件），由 [`SnowLuma/nnphook`](https://github.com/SnowLuma/nnphook) 触发，构建完成后将产物发布到本仓库 Release，并向 [`SnowLuma/SnowLuma`](https://github.com/SnowLuma/SnowLuma) 自动提交 PR 替换 `packages/runtime/native/` 下的对应文件。

## 产物映射

| 产物文件 | 构建源 |
| --- | --- |
| `snowluma-linux-{x64,arm64}.node` | nnphook 根 CMake 项目，`snowluma_linux_addon` target（Ninja Release） |
| `snowluma-linux-{x64,arm64}.so` | nnphook 根 CMake 项目，`snowluma_linux_hook` target（Ninja Release） |
| `snowluma-win32-x64.dll` | nnphook 根 CMake 项目，`qq_hook_dll` target（VS17 Release）→ 输出 `qq_hook.dll` 后改名 |
| `snowluma-win32-x64.node` | `examples/injector_addon` cmake-js 项目（VS17 Release）→ 输出 `injector_addon.node` 后改名 |

## 流水线

```
nnphook (本地仓库)
   │  workflow_dispatch / push tag v* / commit msg [build-native]
   ▼
nnphook/.github/workflows/dispatch-native-build.yml
   │  repository_dispatch  event_type = nnphook-native-build
   ▼
SnowLumaNative/.github/workflows/build-native.yml
   ├─ build-native-linux  (matrix: ubuntu-22.04 / ubuntu-22.04-arm)
   │     cmake -G Ninja Release → snowluma-linux-{x64,arm64}.{node,so}
   │     verify stripped: 无 .debug/.symtab/.strtab
   ├─ build-native-windows (windows-2022, x64)
   │     cmake -G "Visual Studio 17 2022" --target qq_hook_dll → qq_hook.dll
   │     cmake-js rebuild (examples/injector_addon)            → injector_addon.node
   │     rename → snowluma-win32-x64.{dll,node}
   │     verify: no sibling .pdb
   ├─ release  (publishes 6 binaries + SHA256SUMS.txt)         (GITHUB_TOKEN)
   └─ open-snowluma-pr (if open_pr=true)                       (SNOWLUMA_TOKEN)
         分支: native/auto-update-<release_tag>
         替换:
           packages/runtime/native/snowluma-linux-x64.node
           packages/runtime/native/snowluma-linux-x64.so
           packages/runtime/native/snowluma-linux-arm64.node
           packages/runtime/native/snowluma-linux-arm64.so
           packages/runtime/native/snowluma-win32-x64.node
           packages/runtime/native/snowluma-win32-x64.dll
```

## 触发方式（在 nnphook 仓库一侧）

1. **手动**：Actions → `Dispatch SnowLuma Native Build` → `Run workflow`，可填 ref / release_tag / open_pr。
2. **打 tag**：`git tag vX.Y.Z && git push origin vX.Y.Z`，自动以 tag 名作为 release_tag。
3. **提交信息标记**：向 `main` 推送的 commit message 中包含 `[build-native]` 或 `[release-native]` 即触发，release_tag 自动取 `native-<short_sha>`。

## 所需 Secrets

PR 创建者要显示成 bot，必须用 **GitHub App installation token**（PAT 不行）。所以仓库现在分两路 secret：

| 仓库 | Secret | 类型 | 用途 |
| --- | --- | --- | --- |
| `SnowLuma/nnphook` | `SNOWLUMA_TOKEN` | PAT | (a) 调用 API 在 SnowLumaNative 上触发 `repository_dispatch`；(b) `release-docker.yml` 跨仓 checkout SnowLuma + Docker 框架 |
| `SnowLuma/SnowLumaNative` | `SNOWLUMA_TOKEN` | PAT | `build-native-{linux,windows}` 跨仓 checkout 私有 nnphook 源码（只读） |
| `SnowLuma/SnowLumaNative` | `SNOWLUMA_BOT_APP_ID` | App | GitHub App「Snowluma Bot」的 App ID |
| `SnowLuma/SnowLumaNative` | `SNOWLUMA_BOT_PRIVATE_KEY` | App | 同 App 的 PEM 私钥（包含 `-----BEGIN ... PRIVATE KEY-----` 头尾） |

### GitHub App 设置一次性步骤

1. 在 SnowLuma 组织设置里 **New GitHub App**：
   - **Name**：`Snowluma Bot`（PR 创建者将显示为 `snowluma-bot[bot]`，slug 即 App 名小写连字符）
   - **Homepage URL**：随便填一个
   - **Webhook**：取消勾选 Active
   - **Repository permissions**：
     - Contents: Read and write
     - Pull requests: Read and write
     - Metadata: Read-only（默认）
   - **Where can this GitHub App be installed?**：Only on this account
2. 创建完成后页面上：
   - 记下 **App ID**（数字）→ 填到 `SNOWLUMA_BOT_APP_ID`
   - 滚到底部 → **Generate a private key** → 下载 `.pem` → 整个文件内容（包括头尾）填到 `SNOWLUMA_BOT_PRIVATE_KEY`
3. 左侧 **Install App** → 选 SnowLuma 组织 → 至少勾选 `SnowLuma/SnowLuma`（PR 目标仓库），可顺手勾选 `SnowLumaNative`。

### 备注

- 最简 PAT 方案：classic PAT（`repo` scope，覆盖 nnphook + SnowLuma + SnowLumaNative）即可同时填进两个仓库的 `SNOWLUMA_TOKEN`。
- `GITHUB_TOKEN` 已在 `release` job 通过 `permissions: contents: write` 授权创建 Release，无需额外 secret。
- Docker Hub 凭据 `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` 与本流水线无关，仅 `release-docker.yml` 使用，保持不变。

## 手动跑一次（不通过 nnphook）

进入本仓库 Actions → `Build SnowLuma Native` → `Run workflow`，填好 inputs 即可，等价于 nnphook 的 dispatch。
