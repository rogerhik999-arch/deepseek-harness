# Windows 本机构建笔记(本地,不入库)

2026-09-26 在本机完成的 Windows x64 桌面客户端构建记录。

## 产物

- NSIS 安装包:`apps/desktop/.desktop-build/targets/win-x64/unsigned-artifacts/deepseek-harness-0.1.7-rc.2-win-x64-unsigned.exe`(约 287 MB)
- 免安装目录:同目录下 `win-unpacked/DeepSeek Harness.exe`
- 打包运行日志:`apps/desktop/.desktop-build/packaging-runs/2026-09-25T18-48-37.037Z-GweAl2/`(result.json `"success": true`,含 packaged-runtime 冒烟:DOCX/XLSX/PPTX→PDF 全部通过)

未签名(unsigned):没有 EV 代码签名证书;未签名产物不能用于官方发布上传,SmartScreen 可能提示,属预期。

## 本机遇到的两个坑及解决方法

1. **GNU tar 把 `D:\...` 当远程主机**(`Cannot connect to D: resolve failed`)。
   仓库脚本把带盘符的绝对路径传给 tar,Git Bash 的 GNU tar 不支持。解决:打包时把 System32 的 bsdtar 提到 PATH 最前:
   `PATH="/c/Windows/System32:$PATH" pnpm package:desktop:win:x64:unsigned`

2. **构建路径超长导致 LibreOffice 冒烟崩溃**(退出码 0xC0000005,报 `services.rdb: no such file`,但文件其实存在)。
   本仓库根路径 `D:\AciLearn\DeepseekHarnessDesktop\deepseek-harness` 本身就很长,打包后 libreoffice-kit 内最深的文件路径达 284 字符,超过 Windows 260 上限,LibreOffice 引导即崩溃;同样的产物拷到短路径即可正常转换。`subst` 虚拟盘符无效(pnpm/Node 会 realpath 回真实路径)。
   解决:把仓库临时改名为短路径(如 `D:\dsh`)再打包,完成后移回:
   ```
   mv /d/AciLearn/DeepseekHarnessDesktop/deepseek-harness /d/dsh
   cd /d/dsh && CI=true PATH="/c/Windows/System32:$PATH" pnpm package:desktop:win:x64:unsigned
   mv /d/dsh /d/AciLearn/DeepseekHarnessDesktop/deepseek-harness
   ```
   移动后首次构建 pnpm 会要求清空重建 node_modules,非交互环境需 `CI=true`(依赖已在全局 store,重建只需重新链接)。

## 本地配置

- `apps/desktop/.env.windows`(Git 忽略)已按 `.env.windows.example` 创建:test 部署、占位更新源,`DSH_DESKTOP_NPM_REGISTRY=https://registry.npmmirror.com` 加速运行时安装。unsigned 构建不需要签名凭据。

## 常规重构建

```
pnpm install
pnpm run build
# 按"坑 2"的短路径方式打包
```
