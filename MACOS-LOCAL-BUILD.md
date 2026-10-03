# macOS ARM64 本地构建与运行笔记

> 本文档记录在本机（Apple Silicon，macOS，Node 24 / pnpm 11）从源码构建、运行 DeepSeek Harness 桌面客户端并配置双 Git 远程的完整经验，最初基于 `dsh 0.1.7-rc.2`（commit `477b4f4`）；自 2026-10-04 起日常发布走第 9 节的 GitHub Actions 线上构建（当前基线 `dsh 0.2.1-alpha.1`）。上游可能有破坏性变更，升级后请复核本文命令是否仍然适用。

## 0. 环境前提

| 项 | 值 |
|---|---|
| 系统 | macOS（Apple Silicon / arm64） |
| Node.js | `^22.19.0 \|\| >=24.0.0`（本机 24.18.1，nvm） |
| pnpm | 11.x（仓库 pin `pnpm@11.7.0`） |
| 网络 | 系统代理 `127.0.0.1:7897`（Clash 系）；shell 环境默认不带代理变量 |

## 1. 克隆与依赖安装

```sh
git clone https://github.com/rogerhik999-arch/deepseek-harness.git
cd deepseek-harness
CI=true pnpm install
```

两个关键坑：

1. **lefthook postinstall 与全局 `core.hooksPath` 冲突**。仓库的 `scripts/install-lefthook.mjs` 检测到 git 全局配置里 `core.hooksPath` 指向用户自有钩子目录（如 `~/.codex/git-hooks`）时会拒绝执行并使 `pnpm install` 整体失败。该脚本在 `CI=true` 时直接跳过（`scripts/install-lefthook.mjs` 开头的短路逻辑），git hooks 对构建没有影响。
2. **pnpm 运行任意脚本前会做依赖状态校验**（`verify-deps-before-run`），必要时触发一次**嵌套的** `pnpm install`，该嵌套进程不会继承你手工加的跳过手段。因此本机上**所有** `pnpm install` / `pnpm run <script>` 都应统一加 `CI=true` 前缀：

```sh
CI=true pnpm install
CI=true pnpm run build
CI=true pnpm run start:desktop
```

安装日志中若出现 `Failed to create bin at ... node_modules/.bin/dsh`，是 `@deepseek-ai/dsh` 的 bin 尚未构建的预期警告，构建后消失。

## 2. 补装 Electron 二进制

pnpm 默认跳过依赖的构建脚本时，Electron 只装了 JS 包、没有下载二进制（`node_modules/.pnpm/electron@44.0.0/node_modules/electron/dist` 缺失），启动时报 `Electron failed to install correctly`。手动补装（npmmirror 镜像对国内网络更快）：

```sh
cd node_modules/.pnpm/electron@44.0.0/node_modules/electron
ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/" node install.js
```

## 3. 完整构建

```sh
CI=true pnpm run build
```

依次执行 Host tsc+tsdown、Client tsdown、Web 前端 Vite 构建、Desktop 主进程 bundle。成功后关键产物：

- `apps/desktop/lib/main.js` —— Electron 主进程
- `apps/desktop-host/lib/index.js` —— 桌面私有 Host
- `apps/cli/lib/bin.js` —— dsh CLI 入口
- `apps/web/dist/` —— Web 前端产物

## 4. 桌面主运行时与首次启动

```sh
CI=true pnpm run start:desktop
```

启动器（`apps/desktop/scripts/dev.ts --skip-build`）先准备主运行时再拉起 Electron：

- 运行时落地于 `apps/desktop/.desktop-build/targets/mac-arm64/runtime/primary-runtime`（约 389MB：独立 Node 24.21、Python 3.12.14、numpy/pandas/Pillow/lxml 等办公库），来源为 nodejs.org、GitHub（python-build-standalone）与 files.pythonhosted.org；
- GitHub 下载可能瞬时失败（日志只有一行 `fetch failed`），重试即可；Node 的 `fetch` 默认不走代理，需要时：

```sh
CI=true HTTPS_PROXY=http://127.0.0.1:7897 HTTP_PROXY=http://127.0.0.1:7897 NODE_USE_ENV_PROXY=1 \
  pnpm run start:desktop
```

（`NODE_USE_ENV_PROXY=1` 是 Node ≥24 让内置 fetch 读取代理环境变量的开关。）

启动成功的判据：

- 日志出现 `dsh web: http://127.0.0.1:19387/?token=...`（Desktop 专用端口 19387，Web 模式才是 3080）；
- `Office runtime versions and document round trips passed.`（运行时冒烟）；
- 裸 `curl http://127.0.0.1:19387/` 返回 **401 是正常的**（令牌鉴权，应用窗口内自动携带）；
- 首次运行会有 `sandbox_extension_issue_file failed ... Operation not permitted`，可忽略。

调试端口：主进程 9229、渲染层 9222、Host 9230（`DSH_DESKTOP_*_PORT` 可改）。

## 5. 配置继承（~/.dsh）

`start:desktop` / `dev:desktop` 属于开发启动，**有意隔离**全部 Harness 状态到 `apps/desktop/.desktop-build/development/home`（README "Develop" 节：sessions/settings/credentials/package links/browser data 都不落真实主目录）。要继承本机原有 `~/.dsh`（设置、凭据 `.credentials.yaml`、会话等），启动前设置：

```sh
export DSH_HOME="$HOME/.dsh"
```

继承后桌面端会创建并独占 `~/.dsh/profiles/desktop`（与 CLI/Web 的 `profiles/web` 并列，互不冲突）。

**坑：直接双击 `Harness Dev.app` 不继承配置。** 该 .app 内生成的启动脚本（`development-app.ts` 生成）会无条件 `export DSH_HOME='…/development/home'`，覆盖外部传入值——与 README "An explicit `DSH_HOME` replaces only the development Harness home" 的语义不一致（命令行走 `dev.ts` 时继承正常，LaunchServices 冷启动被硬编码覆盖，属上游开发预览版的缺口）。另外它还缺少未打包启动必需的 `DSH_DESKTOP_PRIMARY_RUNTIME_DIR`（`apps/desktop/src/main.ts` 中强制校验，缺失时进程静默退出，无崩溃报告）。

本机采用的方案是不改上游源码，在启动器里运行时用 awk 修补 `DSH_HOME` 一行并补齐缺失变量（见第 6 节）。

验证手段：

```sh
ps eww <主进程pid> | tr ' ' '\n' | grep '^DSH_HOME='   # 确认进程环境变量
ls ~/.dsh/profiles/                                     # 应出现 desktop
```

## 6. 快捷启动（/Applications 包装 App）

仓库深处的 `Harness Dev.app` 不应直接双击（见第 5 节）。做法是做一个薄包装 App 放进「应用程序」：`Contents/MacOS/dsh-launcher` 为 shell 脚本——设 `DSH_HOME` 与 `DSH_DESKTOP_PRIMARY_RUNTIME_DIR`、awk 替换 `HarnessDev` 脚本里的 `DSH_HOME` 行后 exec 之；`Info.plist` 配名称/标识符，图标用 `apps/desktop/resources/icon-macos.png` 经 `sips` + `iconutil -c icns` 生成，最后 `codesign --force --deep --sign -`（ad-hoc）+ `lsregister -f` 注册。本机已装于：

```
/Applications/DeepSeek Harness.app
```

之后即可 Spotlight（`⌘Space` 搜 "Harness"）、启动台、Dock 拖拽启动。注意菜单栏程序名可能显示 "Electron / Harness Dev"（开发版外观，正常）。单实例锁生效：已运行时再次启动只会聚焦窗口。

仓库外另有一个等价的双击脚本 `../启动 DeepSeek Harness.command`（工作区根目录），会陪跑一个终端窗口。

## 7. 双 Git 远程

```sh
git remote rename origin upstream                          # upstream → deepseek-ai/deepseek-harness
git remote add origin https://github.com/rogerhik999-arch/deepseek-harness.git
git fetch upstream --unshallow                             # 浅克隆需补全历史后才能推送（约 250MB，20k+ 提交）
git push -u origin master && git push origin --tags
```

- 推送/建仓凭据来自 macOS Keychain（`git credential fill` 可非交互取用；经 API `GET /user` 确认账号身份与 `repo` scope）；
- 目标仓库不存在时用 API 创建：`POST /user/repos`（本仓库建为 **private**，改公开：GitHub Settings → General → Danger Zone → Change visibility）；
- 日常：`git pull upstream master` 同步官方，`git push` 推回自己仓库；本地 master 跟踪 `origin/master`，会领先上游若干本地文档提交，属预期。

## 8. 官方签名打包路径（需要 Apple 开发者凭据，本机未走）

> 2026-10-04 起本节的「本地无证书路线」已被第 10 节的 CI 线上构建取代；本节保留用于拿到真实 Apple 证书后的正式打包。

发布级签名打包 `CI=true pnpm run package:mac:arm64` 需要准备 `apps/desktop/.env.macos`（复制 `.env.macos.example`），其中要求真实的 Developer ID Application 证书（`CSC_LINK` p12）、Team ID 与 Apple 公证凭据；连 `--prepare-only` 也会校验这些项。本机无 Apple 开发者凭据，故日常使用开发版客户端（功能等价，差异是无签名、版本号一致、数据目录规则见第 5 节）。


## 9. CI 线上构建与发布（2026-10-04 起，当前采用）

仓库为公开仓库，GitHub 的 `macos-14`（Apple Silicon）runner 免费且不限时长。`.github/workflows/desktop-mac-local.yml` 用 `workflow_dispatch` 触发（Actions 页面 → desktop-mac-local → Run workflow），输入 `attach_release_tag` 指定要附加 DMG 的 Release 标签；构建完成后用 `gh release upload --clobber` 覆盖同名资产。全流程约 25 分钟，产出约 434MB 的 DMG（本机 hdiutil 同方法约 459MB，属压缩差异）。与 Windows 本机构建 + 手动上传、Linux 的 `desktop-linux-*` 标签触发 workflow 三者并存。

workflow 流程 = 第 8 节本地链的 CI 版：runner 上现场生成自签证书（CN 必须照抄 `Developer ID Application: …` 前缀）+ 专用钥匙串 → 官方构建链（`build:official` → tarball → `prepare:runtime/packages/dsh`）→ electron-builder `--dir` 免公证 → `hdiutil` 制作 DMG → `codesign --verify --deep --strict` 校验 → 附加到 Release。两处本地补丁（Authority 断言放宽、跳过主运行时重签）在 workflow 内以补丁步骤形式应用，构建后 `git checkout --` 还原。

CI 环境独有的四个坑（本机不出现，排障时先查这里）：

1. **runner 的 LibreSSL 不支持 `openssl pkcs12 -legacy`**，导出 p12 需要降级分支（LibreSSL 默认导出本就是钥匙串兼容格式）；
2. **`security set-keychain-settings -t 0` 会让钥匙串立即自动上锁**，此后任何无头 `security import`/codesign 操作都会永久等待解锁 UI（表现为步骤挂起数十分钟）——用 `-t 86400` 并在 unlock 之后设置。本机当时这条命令恰好被系统拒绝未生效，所以本地从未暴露此问题；
3. **electron-builder 要求证书在信任库中有效**，否则报 `CSSMERR_TP_NOT_TRUSTED`（0 valid identities）——需要 `sudo security add-trusted-cert -d -r trustRoot -p codeSign -k /Library/Keychains/System.keychain`（admin 域，无头安全、秒过）；
4. **签名缓存探针**：`prepare:dsh` 的原生签名依赖 `DSH_DESKTOP_MACOS_SIGNING_PROBE` 指向一个用当前证书签过的 Mach-O 探针文件（CI 里签一份 `/bin/echo` 即可），漏设会以空路径调用 codesign 报「verification failed」。

观测技巧：Actions 任务进行中时日志 API 返回 BlobNotFound 拿不到，把可疑步骤按命令拆成多个微步骤，用 jobs API 的 step 状态实时定位卡点。

发布约定沿用 `desktop-dev-YYYYMMDD`（亚洲/上海日期）：先建标签与 Release，再触发 workflow 附挂 DMG；同一 Release 内资产可被 `--clobber` 原子替换。

## 10. 故障速查

| 症状 | 原因 | 处理 |
|---|---|---|
| `pnpm install` 报 `[install-lefthook] refusing to replace user-owned core.hooksPath` | 全局 git 钩子路径与仓库安装器冲突 | 命令加 `CI=true` 前缀 |
| 任何 `pnpm run` 前突然跑 `pnpm install` 并因 lefthook 失败 | 依赖状态校验触发嵌套安装 | 同上，统一 `CI=true` |
| `Electron failed to install correctly` | pnpm 跳过了 electron 的安装脚本 | 第 2 节手动补装 |
| 启动期一行 `fetch failed` | 主运行时下载（GitHub）瞬时失败 | 重试，或加 `HTTPS_PROXY` + `NODE_USE_ENV_PROXY=1` |
| 双击 `Harness Dev.app` 秒退、无崩溃报告 | 生成脚本硬编码 `DSH_HOME` 且缺 `DSH_DESKTOP_PRIMARY_RUNTIME_DIR` | 用第 6 节包装 App 启动 |
| 想确认客户端用的是哪个主目录 | — | `ps eww <pid> | grep DSH_HOME` |
| `curl 127.0.0.1:19387` 返回 401 | 令牌鉴权，正常 | 从启动日志取带 token 的 URL |
