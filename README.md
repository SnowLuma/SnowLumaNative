# SnowLumaNative

SnowLuma 原生产物（Linux x64/arm64 + Windows x64，共 6 个文件）的自动构建与发布仓库。构建完成后将产物发布到本仓库 Release，并可自动向 [`SnowLuma/SnowLuma`](https://github.com/SnowLuma/SnowLuma) 提交 PR，替换 `packages/runtime/native/` 下的对应文件。

## 产物

| 文件 | 平台 |
| --- | --- |
| `snowluma-linux-x64.{node,so}`   | Linux x64 |
| `snowluma-linux-arm64.{node,so}` | Linux arm64 |
| `snowluma-win32-x64.{dll,node}`  | Windows x64 |

发布的 Release 还附带 `SHA256SUMS.txt` 校验清单。

## 流水线

`Build SnowLuma Native`（`.github/workflows/build-native.yml`）：

- **build-native-linux** — `ubuntu-22.04` / `ubuntu-22.04-arm`，CMake + Ninja（Release），产出 `snowluma-linux-{x64,arm64}.{node,so}`，并校验已剥离调试符号。
- **build-native-windows** — `windows-2022` x64，产出 `snowluma-win32-x64.{dll,node}`，并校验无 PDB / 调试目录。
- **release** — 发布 6 个二进制 + `SHA256SUMS.txt` 到本仓库 Release。
- **open-snowluma-pr** — 可选，向 `SnowLuma/SnowLuma` 提交替换 `packages/runtime/native/` 的 PR。

## 运行

由上游事件（`repository_dispatch`）自动触发，或在本仓库 **Actions → Build SnowLuma Native → Run workflow** 手动触发。手动运行时可填写源 ref、`release_tag`、是否开 PR 等参数。

> 构建所需的访问令牌等凭据已在组织 / 仓库层面配置，不在此公开。
