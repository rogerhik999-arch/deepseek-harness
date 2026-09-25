# Linux 桌面版构建与使用指南

English | 简体中文（本文档为主文档）

本分支基于上游 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) v0.1.7-rc.2（master 同步自上游），包含一组使 DeepSeek Harness 桌面客户端与打包管线支持 Linux x64 的本地补丁。上游官方桌面产物仅覆盖 macOS 与 Windows；本文记录在本机（Ubuntu 24.04, linux-x64）从源码构建、日常使用到产出安装包的完整经验。所有 Linux 相关改动集中在本分支，未向上游提交。

## 环境要求

- Node.js `^22.19 || >=24`（本机使用 nvm 安装的 v22.23.3）
- pnpm 11.7.0（`npm install -g pnpm@11.7.0`）
- 构建工具链：gcc/g++、make、tar（native 附加组件与打包冒烟使用）
- Electron 及运行时载荷会从网络下载；GitHub Releases 不可达时可设置 `ELECTRON_MIRROR=https://npmmirror.com/mirrors/electron/`

## 从源码构建

```sh
pnpm install
pnpm run build
```

构建完成后即可运行 Web 客户端：`pnpm dsh web`（默认端口 3080）。Web 客户端无需任何补丁，上游源码原生支持 Linux。

## 桌面客户端 Linux 启用补丁

上游桌面客户端的构建目标白名单仅含 mac 与 win，启动时会在 `desktop build paths: unsupported target linux-x64` 处失败。本分支的最小补丁清单：

| 文件 | 改动 |
|---|---|
| `apps/desktop/scripts/desktop-build-paths.mjs` | 白名单加入 `linux-x64`/`linux-arm64`；`desktopTargetPlatform` 映射 `linux` 平台 |
| `apps/desktop/scripts/desktop-build-paths.d.mts` | 新增 `DesktopBuildTarget` 类型并同步签名 |
| `apps/desktop/scripts/development-project.ts` | 改用放宽后的目标类型 |
| `apps/desktop/tests/desktop-build-paths.spec.ts` | 断言 Linux 目标被接受，`freebsd`/`sunos` 仍被拒绝 |
| `apps/desktop/src/main.ts` | `focusPrimaryWindow` 在欢迎页状态下二次启动时重开欢迎窗口（Linux 无 Dock/托盘唤回途径） |

补丁后即可用开发模式运行桌面客户端：

```sh
pnpm run start:desktop   # 或 pnpm run dev:desktop（带构建）
```

首次运行会下载 linux-x64 主运行时（Node + Python + Office 库，数百 MB），并执行冒烟检查。

## Electron SUID 沙箱

Ubuntu 24.04 内核限制非特权 user namespaces（AppArmor），Electron 因此依赖 SUID 沙箱。Electron 44 的 `chrome-sandbox` 出厂权限为 755，直接启动会报：

```text
FATAL: The SUID sandbox helper binary was found, but is not configured correctly.
```

一次性修复（需要 sudo）：

```sh
ELECTRON_DIST=node_modules/.pnpm/electron@44.0.0/node_modules/electron/dist
sudo chown root:root "$ELECTRON_DIST/chrome-sandbox"
sudo chmod 4755 "$ELECTRON_DIST/chrome-sandbox"
```

注意两点：electron 升级后 `node_modules/.pnpm` 下的 dist 路径会变化，需对新路径重跑上述命令；终端环境中若存在 `ELECTRON_DISABLE_SANDBOX=1` 会掩盖此问题（沙箱被禁用后跳过检查），图形会话启动时没有该变量才会暴露。启动脚本 `~/.local/bin/dsh-desktop` 已显式 `unset ELECTRON_DISABLE_SANDBOX` 以保证行为一致。

## 快捷启动入口

`~/.local/bin/dsh-web` 与 `~/.local/bin/dsh-desktop` 两个启动脚本配合 `~/.local/share/applications/` 下的 `.desktop` 文件提供应用菜单入口，行为如下：

- `dsh-web`：3080 端口无服务时启动 Web 客户端（约 10-30 秒），从日志抓取带 token 的 URL 存入 `~/.cache/deepseek-harness/web.url` 并打开浏览器；已有服务时直接打开记录的地址。
- `dsh-desktop`：应用未运行时经 `pnpm run start:desktop` 完整启动（约 1 分钟，flock 防止启动期间重复点击）；已在运行时拉起一个瞬时第二 Electron 实例，其单实例锁竞争失败后触发运行中实例显示窗口——这是 Linux 上替代 Dock/托盘的窗口唤回途径。失败会写日志并弹桌面通知。
- 日志位置：`~/.cache/deepseek-harness/{web,desktop}.log`。

桌面客户端的窗口关闭语义是隐藏而非退出（Host 与后台任务继续运行），再次点击图标即唤回；彻底退出使用应用菜单的退出命令。

## 数据共享

桌面开发模式默认使用仓库内隔离的 home（`.desktop-build/development/home`），与 Web 客户端的 `~/.dsh` 互不相通。启动脚本通过 `export DSH_HOME=~/.dsh` 让两者共享同一份数据：

- 会话历史、工作区选择、模型设置、技能、附件：home 层共享
- 插件组合：各客户端独立（`~/.dsh/profiles/web` 与 `~/.dsh/profiles/desktop`）
- 桌面快捷键：Electron userData，与 DSH_HOME 无关

两个客户端可同时运行（SQLite 为 WAL 模式）；极端情况下两边同时写入可能偶发冲突报错，重试即可。

## 构建 Linux 安装包

本分支同时启用了 linux-x64 打包目标：

| 文件 | 改动 |
|---|---|
| `apps/desktop/scripts/package-target.ts` | `TARGETS` 加入 linux-x64；宿主校验；发布记录对 Linux 跳过 |
| `apps/desktop/scripts/prepare-runtime.ts` | Electron 下载平台映射与可执行路径支持 linux |
| `apps/desktop/scripts/prepare-dsh.ts` | Node 可执行路径按目标平台解析 |
| `apps/desktop/scripts/desktop-package-environment.mjs` | `.env.linux` 可选；Linux 跳过签名/更新通道/强制更新 policy 校验 |
| `apps/desktop/scripts/desktop-toolchain-preflight.ts` | 平台类型放宽 |
| `apps/desktop/scripts/electron-builder-config.mjs` | linux 段（deb+AppImage、图标、`executableName`、`syncDesktopName`）；Linux 跳过 update/policy；extraResources 使用原始图标 |
| `apps/desktop/scripts/smoke-packaged-runtime.ts` | `linux-unpacked` 产物布局 |
| `apps/desktop/scripts/desktop-upload-plan.ts` | 目标表补全 linux-x64 条目（上传仍限定 mac/win） |
| `apps/desktop/.env.linux.example` | Linux 打包环境模板（仅需 `DSH_DESKTOP_APP_ID`） |

打包命令与产物：

```sh
pnpm run package:desktop:linux:x64      # deb + AppImage + linux-unpacked
pnpm run package:desktop:linux:x64:dir  # 仅 linux-unpacked 目录
```

产物位于 `apps/desktop/.desktop-build/targets/linux-x64/artifacts/`：`deepseek-harness-<版本>-linux-amd64.deb`（约 300 MB）与 `deepseek-harness-<版本>-linux-x86_64.AppImage`（约 354 MB）。首次打包会下载 Electron linux 压缩包与构建缓存，耗时约 10-25 分钟。

## 目标机器安装

```sh
sudo dpkg -i deepseek-harness-<版本>-linux-amd64.deb
```

deb 的 postinst 会自动完成目标机的沙箱配置（检测 user namespaces → 以 root 设置 `chrome-sandbox` 权限 → 安装 AppArmor profile 授权），并注册 `/usr/bin/deepseek-harness` 命令、应用菜单图标与 mime 数据。AppImage 在 Ubuntu 24.04 上需要先安装 `libfuse2`，且不经过 postinst 的沙箱处理，作为分发格式弱于 deb。deb 基于本机 glibc 构建，目标机器建议 Ubuntu 24.04 及以上。

## 已知限制

- Office 文档转换在 Linux 使用 WASM 引擎回退（上游仅发布 darwin/win 原生引擎），性能低于原生；打包冒烟中 Host 进程内的转换检查会因 profile 解析器对缺失原生引擎的探测而报错，真实使用中转换运行在独立 PTC 子进程，不走该解析器。
- 自动更新体系仍限定 mac/win；Linux 包为本地构建，不含更新通道，升级需重新打包安装。
- 上游合并后若启动或打包报 `unsupported target linux-x64`，说明本分支补丁需要向上游新代码重放。
