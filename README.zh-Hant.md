# SecDataGateway

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-Hant.md)

本機 AI 資料閘道。安裝後會在電腦上運行，並開啟瀏覽器介面；完整使用前需要匯入授權檔。

**支援的安裝包：** Windows x64 · macOS Apple 晶片 · macOS Intel  
**暫不提供：** Windows ARM64 原生包

## 完整操作手冊

Windows／Mac 圖文說明（安裝、接入本機 IDE、授權、脫敏驗收）：

- 手冊首頁：[docs/](docs/index.md)（啟用 GitHub Pages 後亦可線上瀏覽）
- [Windows 操作手冊](docs/windows/index.md)
- [Mac 操作手冊](docs/mac/index.md)

下方短提示詞僅涵蓋「用 AI 安裝到授權頁」；完整產品操作請看上手冊。

## 用 AI 助手安裝

1. 開啟 Cursor、Claude Code、WorkBuddy，或其他能在**這部電腦**執行指令的助手。
2. 貼上：

```text
請閱讀 https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.zh-Hant.md ，在本機安裝最新版 SecDataGateway。授權頁打開後停止，並提醒我自行上傳供應商提供的 .lic。只有系統權限提示時再叫我。不要向我索取或代為上傳 .lic。
```

3. 助手說授權頁已打開後，請在該頁面自行上傳供應商提供的 `.lic`。

完成授權後，IDE 接入等功能請看應用內說明或產品手冊。

---

## AI 安裝說明

目標：為這部電腦安裝最新穩定版、確認服務就緒、打開授權頁，然後停止。

規則：

- 只使用 [SecDataGateway-releases](https://github.com/lbxdorz/SecDataGateway-releases/releases) 的官方資產。
- 不要改用自行猜測的鏡像站。
- 不要使用 `sudo`／系統管理員權限。
- 不要結束已在運行的 SecDataGateway。
- 不要讀取、索取、複製或上傳任何 `.lic`。
- 本流程不要配置 IDE，也不要做脫敏測試。
- 若缺少資產、校驗失敗、無法啟動或健康檢查失敗：停止並回報錯誤。

就緒後要打開的授權頁：`http://127.0.0.1:18787/#license`

### 1. 解析最新穩定版一次並鎖定

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

之後下載安裝包與 `checksums.txt` 都使用同一 tag。請從該 tag 的 GitHub assets 列表選擇檔案；沒有匹配項就停止。

| 電腦 | 資產檔名結尾 |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| macOS Apple 晶片 | `darwin_arm64.tar.gz` |
| macOS Intel | `darwin_amd64.tar.gz` |
| Windows ARM64 | 停止 — 無原生包 |

在 macOS 上用 `uname -m`，必要時再加上 `sysctl.proc_translated`／`hw.optional.arm64`，辨識**原生** CPU，避免 Rosetta 終端誤選 Intel 包。

預設安裝根目錄（可用 `SDG_INSTALL_ROOT` 覆寫）：

- macOS：`~/Applications/SecDataGateway`
- Windows：`%LOCALAPPDATA%\SecDataGateway`

使用新建目錄 `$SDG_INSTALL_ROOT/$SDG_TAG`。若目錄已存在，停止，不要覆蓋。

### 2. 若 18787 埠已有服務

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

將 `version` 與鎖定的 tag 比對（可忽略前綴 `v`）。

| 結果 | 做法 |
| --- | --- |
| 版本相同且健康 | 沿用；跳過下載／啟動；前往第 4 步 |
| 版本不同 | 回報兩邊版本並停止；不要替換正在運行的安裝 |
| 埠被佔用但不健康 | 回報衝突並停止 |

macOS 可選檢查：`lsof -nP -iTCP:18787 -sTCP:LISTEN`

### 3. 下載、校驗、解壓並啟動

僅在第 2 步判定需要新安裝時執行。

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

若 macOS 阻擋開啟，先詢問使用者，再只執行：

```bash
xattr -dr com.apple.quarantine ./SecDataGateway.app
```

不要關閉整個系統的 Gatekeeper。

**Windows（PowerShell）**

1. 解析 `/releases/latest` 並鎖定 tag。  
2. 從該 tag 的 assets 選出唯一以 `windows_amd64.zip` 結尾的檔名。  
3. 下載該 zip 與同 tag 的 `checksums.txt`。  
4. 用 `Get-FileHash -Algorithm SHA256` 核對。  
5. 解壓到新建的 `%LOCALAPPDATA%\SecDataGateway\<tag>`。  
6. 啟動該目錄中的 `SecDataGateway.exe`。  

缺資產或雜湊不符就停止。

啟動後重複三個健康檢查，直到 `startup-status` 顯示 ready；一直未就緒則停止。

### 4. 打開授權頁並結束

```bash
# macOS
open 'http://127.0.0.1:18787/#license'
```

```powershell
# Windows
Start-Process 'http://127.0.0.1:18787/#license'
```

然後回覆使用者：

> 已完成安裝並開啟授權頁面，請自行上傳供應商提供的 .lic 授權檔案。

本安裝流程到此結束，不要繼續代為激活或配置其他功能。
