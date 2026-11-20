# SecDataGateway

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-Hant.md)

本机 AI 数据网关。安装后会在本机运行并打开浏览器界面；完整使用前需要导入授权文件。

**支持的安装包：** Windows x64 · macOS Apple 芯片 · macOS Intel  
**暂不提供：** Windows ARM64 原生包

## 用 AI 助手安装

1. 打开 Cursor、Claude Code、WorkBuddy，或其他能在**这台电脑**执行命令的助手。
2. 粘贴：

```text
请阅读 https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.zh-CN.md ，在本机安装最新版 SecDataGateway。授权页打开后停止，并提醒我自行上传供应商提供的 .lic。只有系统权限弹窗时再叫我。不要向我索取或代为上传 .lic。
```

3. 助手提示授权页已打开后，请在该页面自行上传供应商提供的 `.lic`。

完成授权后，IDE 接入等功能请查看应用内说明或产品手册。

---

## AI 安装说明

目标：为这台电脑安装最新稳定版、确认服务就绪、打开授权页，然后停止。

规则：

- 只使用 [SecDataGateway-releases](https://github.com/lbxdorz/SecDataGateway-releases/releases) 的官方资源。
- 不要改用自行猜测的镜像站。
- 不要使用 `sudo`／系统管理员权限。
- 不要结束已在运行的 SecDataGateway。
- 不要读取、索取、复制或上传任何 `.lic`。
- 本流程不要配置 IDE，也不要做脱敏测试。
- 若缺少资源、校验失败、无法启动或健康检查失败：停止并报告错误。

就绪后要打开的授权页：`http://127.0.0.1:18787/#license`

### 1. 解析最新稳定版一次并锁定

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

之后下载安装包与 `checksums.txt` 都使用同一 tag。请从该 tag 的 GitHub assets 列表选择文件；没有匹配项就停止。

| 电脑 | 资源文件名结尾 |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| macOS Apple 芯片 | `darwin_arm64.tar.gz` |
| macOS Intel | `darwin_amd64.tar.gz` |
| Windows ARM64 | 停止 — 无原生包 |

在 macOS 上用 `uname -m`，必要时再加上 `sysctl.proc_translated`／`hw.optional.arm64`，识别**原生** CPU，避免 Rosetta 终端误选 Intel 包。

默认安装根目录（可用 `SDG_INSTALL_ROOT` 覆盖）：

- macOS：`~/Applications/SecDataGateway`
- Windows：`%LOCALAPPDATA%\SecDataGateway`

使用新建目录 `$SDG_INSTALL_ROOT/$SDG_TAG`。若目录已存在，停止，不要覆盖。

### 2. 若 18787 端口已有服务

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

将 `version` 与锁定的 tag 比对（可忽略前缀 `v`）。

| 结果 | 做法 |
| --- | --- |
| 版本相同且健康 | 沿用；跳过下载／启动；前往第 4 步 |
| 版本不同 | 报告两边版本并停止；不要替换正在运行的安装 |
| 端口被占用但不健康 | 报告冲突并停止 |

macOS 可选检查：`lsof -nP -iTCP:18787 -sTCP:LISTEN`

### 3. 下载、校验、解压并启动

仅在第 2 步判定需要新安装时执行。

**macOS**

```bash
bash <<'SH'
set -eu
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
SDG_MACHINE=$(uname -m)
SDG_TRANSLATED=$(sysctl -in sysctl.proc_translated 2>/dev/null || true)
SDG_ARM=$(sysctl -n hw.optional.arm64 2>/dev/null || true)
if [ "$SDG_MACHINE" = arm64 ] || [ "$SDG_TRANSLATED" = 1 ] || [ "$SDG_ARM" = 1 ]; then SDG_ARCH=arm64
elif [ "$SDG_MACHINE" = x86_64 ]; then SDG_ARCH=amd64
else echo 'Unsupported CPU'; exit 1; fi
SDG_INSTALL_ROOT="${SDG_INSTALL_ROOT:-$HOME/Applications/SecDataGateway}"
SDG_DIR="$SDG_INSTALL_ROOT/$SDG_TAG"
SDG_SUFFIX="darwin_${SDG_ARCH}.tar.gz"
export SDG_SUFFIX
SDG_API="https://api.github.com/repos/lbxdorz/SecDataGateway-releases/releases/tags/${SDG_TAG}"
SDG_FILE=$(curl -fLsS "$SDG_API" | python3 -c 'import json,sys,os; s=os.environ["SDG_SUFFIX"]; d=json.load(sys.stdin); names=[a["name"] for a in d.get("assets") or [] if a.get("name","").endswith(s)];
assert len(names)==1, names; print(names[0])')
SDG_DOWNLOAD="$SDG_REPO/releases/download/$SDG_TAG"
mkdir -p "$SDG_INSTALL_ROOT"
mkdir "$SDG_DIR"
cd "$SDG_DIR"
curl -fLsS "$SDG_DOWNLOAD/$SDG_FILE" -o "$SDG_FILE"
curl -fLsS "$SDG_DOWNLOAD/checksums.txt" -o checksums.txt
awk -v f="$SDG_FILE" '{sub(/\r$/, "")} $2 == f {print; n++} END {if(n != 1) exit 1}' checksums.txt > selected.sha256
shasum -a 256 -c selected.sha256
tar -xzf "$SDG_FILE"
test -f SecDataGateway.app/Contents/MacOS/SecDataGateway
open "$SDG_DIR/SecDataGateway.app"
printf 'Tag: %s\nFolder: %s\nAsset: %s\n' "$SDG_TAG" "$SDG_DIR" "$SDG_FILE"
SH
```

若 macOS 阻止打开，先询问用户，再只执行：

```bash
xattr -dr com.apple.quarantine ./SecDataGateway.app
```

不要关闭整个系统的 Gatekeeper。

**Windows（PowerShell）**

1. 解析 `/releases/latest` 并锁定 tag。  
2. 从该 tag 的 assets 选出唯一以 `windows_amd64.zip` 结尾的文件名。  
3. 下载该 zip 与同 tag 的 `checksums.txt`。  
4. 用 `Get-FileHash -Algorithm SHA256` 核对。  
5. 解压到新建的 `%LOCALAPPDATA%\SecDataGateway\<tag>`。  
6. 启动该目录中的 `SecDataGateway.exe`。  

缺少资源或哈希不符就停止。

启动后重复三个健康检查，直到 `startup-status` 显示 ready；一直未就绪则停止。

### 4. 打开授权页并结束

```bash
# macOS
open 'http://127.0.0.1:18787/#license'
```

```powershell
# Windows
Start-Process 'http://127.0.0.1:18787/#license'
```

然后回复用户：

> 已完成安装并打开授权页面，请自行上传供应商提供的 .lic 授权文件。

本安装流程到此结束，不要继续代为激活或配置其他功能。
