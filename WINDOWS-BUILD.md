# Windows 构建经验总结(Windows Build Notes)

> 在 Windows 11(10.0.26200 x64)+ Git Bash 上,从源码完整构建 DeepSeek Harness 桌面客户端(0.1.7-rc.2)的经验记录。按本文操作最终产出通过全部打包冒烟检查的 NSIS 安装包与免安装目录。上游权威文档见 [apps/desktop/README.md](apps/desktop/README.md);本文记录的是那套文档之外、本平台实际会踩到的坑与解法。

## 1. 环境要求(实测版本)

| 组件 | 要求 | 实测 |
|---|---|---|
| Node.js | `^22.19.0 \|\| >=24.0.0` | v24.13.0(nvm4w) |
| pnpm | 与根 package.json `packageManager` 字段一致 | 11.7.0(必须精确匹配) |
| Git | Git for Windows(Git Bash) | 2.52.0 |
| gh CLI | 可选,建仓/推送用 | 2.87.2 |

## 2. 构建流程(五步)

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install                 # 约 1 分钟;bin 警告属预期(lib 未构建)
pnpm run build               # native-system → lib(host+client) → web,数分钟
cp apps/desktop/.env.windows.example apps/desktop/.env.windows   # 按第 6 节填写
pnpm package:desktop:win:x64:unsigned                            # 按第 3、4 节加前置修复,约 7 分钟
```

产物在 `apps/desktop/.desktop-build/targets/win-x64/unsigned-artifacts/`:

- `deepseek-harness-<version>-win-x64-unsigned.exe` — NSIS 安装包(约 287 MB)
- `win-unpacked/DeepSeek Harness.exe` — 免安装目录版
- 打包运行日志与 `result.json` 在 `apps/desktop/.desktop-build/packaging-runs/<run>/`

## 3. 坑一:GNU tar 把盘符解析成远程主机

**现象**(打包进行到 `prepare:packages` 阶段):

```
tar -xOzf D:\...\deepseek-ai-dsh-0.1.7-rc.2.tgz package/package.json
tar (child): Cannot connect to D: resolve failed
```

**根因**:打包脚本把带盘符的绝对路径传给 tar;Git Bash 的 GNU tar 把 `D:` 当作 `host:file` 远程语法。上游 toolchain preflight 的 tar 探测只用相对路径(在归档旁运行),所以测不出这个问题。

**解法**:打包时把 System32 自带的 bsdtar 提到 PATH 最前(bsdtar 支持盘符路径):

```sh
PATH="/c/Windows/System32:$PATH" pnpm package:desktop:win:x64:unsigned
```

## 4. 坑二:构建路径超长导致 LibreOffice 冒烟崩溃

**现象**(electron-builder 与 NSIS 都已成功,最后 `smoke-packaged-runtime` 失败):

```
Error: desktop runtime: docx conversion failed ...
Bootstrapping exception 'file:///.../libreoffice-kit-win32-x64/program/program/../program/services.rdb: no such file'
exit 3221225477  (0xC0000005, STATUS_ACCESS_VIOLATION)
```

`services.rdb` 实际存在(8702 字节),打包副本与暂存副本 730 个文件逐一对比完全一致——这不是缺文件,是崩溃后 LibreOffice 的误导性报错。

**根因**:Windows `MAX_PATH` 260。本例仓库根路径本身就长(`D:\AciLearn\DeepseekHarnessDesktop\deepseek-harness`),叠加打包树后 libreoffice-kit 内最深路径达 **284 字符**;LibreOffice 原生引导代码按 ANSI API 访问路径,超限即崩溃。暂存目录比打包目录浅约 74 字符,所以 `prepare:runtime` 阶段的冒烟能过、打包后的冒烟必挂。

**三组对照实验**(直接调用 helper 复现,可作通用诊断手法):

| 测试 | kit 前缀长度 | 结果 |
|---|---|---|
| A. 暂存位置 | 152 字符 | ✅ 正常出 PDF |
| B. 打包位置 | 212 字符 | ❌ 同样崩溃 |
| C. 打包副本拷到 `D:\kit-c`(34 字符) | — | ✅ 正常出 PDF |

helper 直调命令(输入文档可用 `apps/desktop/tests/fixtures/office-conversion-inputs.py` 配合内置 Python 生成):

```sh
<kit>/bin/libreoffice-kit.exe --program-directory <kit>/program/program \
  --input-path input.docx --output-path out.pdf --profile-directory <临时目录> \
  --max-output-bytes 10485760 --max-image-resolution 300 --format pdf --recalculate false
```

**解法**:使用固定的短路径构建树 `D:\dsh`(推荐,2026-09-28 起):

```sh
# 一次性建立:从主检出本地克隆(对象硬链接,秒级),配置远端与本地配置
git clone /d/AciLearn/DeepseekHarnessDesktop/deepseek-harness /d/dsh
cd /d/dsh && git remote set-url origin https://github.com/rogerhik999-arch/deepseek-harness.git
git remote add upstream https://github.com/deepseek-ai/deepseek-harness.git
cp /d/AciLearn/DeepseekHarnessDesktop/deepseek-harness/apps/desktop/.env.windows apps/desktop/

# 每次重打包:同步最新提交后直接打包(产物留在 D:\dsh,不回传主检出)
cd /d/dsh && git pull origin master
CI=true PATH="/c/Windows/System32:$PATH" pnpm package:desktop:win:x64:unsigned
```

曾经用过的"临时改名移动法"(`mv` 仓库到 `D:\dsh` 打包再移回)已被弃用:主检出位于 ZCode 工作区内,构建产物落盘后常被 Defender/索引器长期占用目录锁,`mv` 频繁报 `Device or resource busy`。克隆出的构建树是一次性成本,之后完全免移动。

**为什么 `subst` 虚拟盘符不行**:pnpm 与 Node 启动时会 realpath(`GetFinalPathNameByHandle`)还原真实路径,错误信息里路径仍是原始长路径,等于没做。junction 同理。

**影响范围**:仅构建主机上的自检。产物本身与路径无关,安装到 `C:\Program Files\DeepSeek Harness\` 后 kit 路径约 130 字符,完全正常。

## 5. 坑三:移动工作区后 pnpm 依赖状态检查

**现象**(改名打包后,或移回原位后首次 pnpm 调用):

```
[ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY] Aborted removal of modules directory due to no TTY
```

**根因**:`node_modules/.modules.yaml` 记录的绝对路径与新位置不符,pnpm 要求清空重建 node_modules;非交互环境下默认中止。pre-push 钩子(见第 6 节)里的 deps-status-check 也会因此失败,表现为推送被拒,易误判为类型错误。

**解法**:

```sh
CI=true pnpm install    # 约 36 秒,依赖已在全局 store,只需重新链接
```

## 6. Git 钩子:pre-push 全量 typecheck

lefthook 的 `pre-push` 运行 `pnpm run typecheck`(`build:lib:host` + `tsc -b tsconfig.client.json`),实测约 40 秒。推送"卡住"或失败时先看是不是第 5 节的 deps 检查问题,而不是代码问题。pre-commit 只做 whitespace、vendor manifest 等轻量检查,普通 markdown 文件无额外门禁。

## 7. `apps/desktop/.env.windows`(本地打包配置)

从 `.env.windows.example` 复制,Git 已忽略。unsigned 构建的最小可用配置:

```
DSH_DESKTOP_APP_ID=com.deepseek.harness
DSH_DESKTOP_AUTO_UPDATE_ENV=test
DOWNLOAD_TEST_RELEASE_ID=<node crypto randomBytes(16) 的 32 位 hex>
DSH_DESKTOP_MANDATORY_UPDATE_TEST_ORIGIN=https://download-test.deepseek.com
DSH_DESKTOP_MANDATORY_UPDATE_CONFIG='{"allowedAuthOrigins":["https://login.example.com"]}'
DSH_DESKTOP_NPM_REGISTRY=https://registry.npmmirror.com
```

说明:

- 签名四项(`DSH_DESKTOP_WINDOWS_CER_FILE/SIGNTOOL/KEY_CONTAINER/TOKEN_PIN`)留空即可,`--unsigned` 跳过签名校验,不访问硬件令牌。
- 更新源是策略校验必填项(HTTPS 源、不带路径);unsigned 构建的更新器配置本就不生成,占位值无运行时影响。
- `DSH_DESKTOP_NPM_REGISTRY` 指到 npmmirror 可显著加速打包内嵌运行时的安装。

## 8. 错误速查表

| 报错 | 根因 | 解法 |
|---|---|---|
| `tar (child): Cannot connect to D: resolve failed` | GNU tar 把盘符当远程主机 | PATH 前置 System32(bsdtar),见第 3 节 |
| `Bootstrapping exception '...services.rdb: no such file'` + exit `3221225477` | 构建路径超 MAX_PATH 260 | 在短路径构建树 `D:\dsh` 打包,见第 4 节 |
| `[ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY]` | 工作区移动后依赖元数据过期 | `CI=true pnpm install`,见第 5 节 |
| `desktop package: cannot read .env.windows` | 缺本地打包配置 | 从 example 复制并按第 7 节填写 |
| pre-push typecheck 失败但未改代码 | deps 状态检查先挂 | 见第 5 节 |
| `desktop package: unsupported setting ... in .env.windows` | 配置键不在平台模板白名单 | 严格按 `.env.windows.example` 的键名 |

## 9. unsigned 产物须知

- 无 EV 证书时用 `unsigned` 模式;产物**不能用于官方发布上传**(发布要求签名)。
- 首次运行 SmartScreen 可能提示"未知发布者",属预期;选"仍要运行"即可。
- 桌面端默认与既有 dsh 共享 `~/.dsh`(设置、凭据、会话自动继承),自有插件目录 `profiles/desktop`,端口 19387(与 Web 的 3080 不冲突)。
