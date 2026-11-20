# SecDataGateway

[English](README.md) | [繁體中文](README.zh-Hant.md)

```text
請閱讀以下說明，在本機安裝最新版 SecDataGateway，打開授權／激活頁面後停止。只有系統權限提示時再叫我。不要向我索取、讀取或代為上傳 .lic：
https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.zh-Hant.md
```

**給使用者：** 本頁給本機 AI 助手用。流程是安裝 SecDataGateway、確認服務就緒、打開授權頁，並請你自己上傳供應商提供的 `.lic`。授權完成前，產品尚未完全可用。激活後的 IDE 接入等，請看應用內說明／產品手冊。

**成功標準：** 已為本機 OS／CPU 裝好（或確認正在跑）最新穩定版、服務就緒、已打開授權頁，並明確告知你自行上傳 `.lic`。助手不得代為激活。

請使用能執行本機終端機指令的助手；純網頁聊天只能口述。

---

## 給 AI 助手的步驟

請依序執行。優先用下列指令。**不要**讀取、索取、複製或上傳任何 `.lic`，**不要**替使用者完成激活，**不要**在本流程配置 IDE 或做脫敏測試。遇錯（缺資產、校驗失敗、無法啟動、健康檢查失敗）即停並回報；不要改用非官方下載源或關閉系統防護。不要 `sudo`。

本機預設介面：`http://127.0.0.1:18787`  
已核對的授權頁深鏈（產品 UI 的 hash 路由 `license`）：  
`http://127.0.0.1:18787/#license`

### 1) 只解析一次最新穩定版並鎖定 tag

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

`/releases/latest` 會排除草稿與預發佈。之後下載與 `checksums.txt` 都用同一 tag。資產請從該 tag 的 GitHub assets 列表精確挑選；沒有匹配項就**停止**。

| 電腦 | 資產檔名應以此結尾 |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| Apple 晶片 Mac | `darwin_arm64.tar.gz` |
| Intel Mac | `darwin_amd64.tar.gz` |
| Windows ARM64 | 目前無原生包 — **停止**（不可把 amd64 當原生） |

Mac 請辨識**原生** CPU（`uname -m`，必要時 `sysctl.proc_translated`／`hw.optional.arm64`），即使終端在 Rosetta 下，Apple 晶片仍選 `arm64`。

安裝根目錄（可用 `SDG_INSTALL_ROOT` 覆寫）：macOS `~/Applications/SecDataGateway`，Windows `%LOCALAPPDATA%\SecDataGateway`。使用**新建**的 `$SDG_INSTALL_ROOT/$SDG_TAG`；目錄已存在則拒絕覆蓋。

### 2) 若 18787 埠已有服務

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

將 `version` 與鎖定 tag 比對（可忽略前綴 `v`）。

- **版本相同且健康：** 沿用，跳過下載／啟動，直接第 4 步。不要宣稱剛完成新安裝。
- **版本不同：** 回報兩邊版本並**停止**。不要殺行程、不要自動覆蓋，也不要宣稱已裝上最新版。
- **埠被佔但健康失敗：** 回報衝突並停止。

檢查監聽：Mac `lsof -nP -iTCP:18787 -sTCP:LISTEN`。

### 3) 下載、校驗、解壓、啟動（僅在第 2 步判定需要新裝時）

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

若系統阻擋，須經使用者同意後再執行：`xattr -dr com.apple.quarantine ./SecDataGateway.app`（僅該 app）。不要關閉 Gatekeeper。

**Windows（PowerShell；撰寫環境未實跑）**  
解析 latest → 鎖 tag → 從該 tag assets 選唯一 `*windows_amd64.zip` → 下載並用同 tag `checksums.txt` 做 SHA-256 → 解壓到**新建** `%LOCALAPPDATA%\SecDataGateway\<tag>` → 啟動 `SecDataGateway.exe`。缺包或雜湊不符即停。

啟動後重複三個 GET，直到 `startup-status` 為 ready（否則回報失敗）。

### 4) 打開授權頁（本流程終點）

服務就緒後：

```bash
# macOS
open 'http://127.0.0.1:18787/#license'
```

```powershell
# Windows
Start-Process 'http://127.0.0.1:18787/#license'
```

依據：產品前端將 `license` 註冊為 hash 頁面；未激活時介面也會導向該頁；本機預設監聽 `127.0.0.1:18787`。`#license` 對應離線激活／選擇 `.lic` 的授權畫面。不要自動選檔或上傳。

然後用繁體中文回覆使用者（語意須等同）：

> 已完成安裝並開啟授權頁面，請自行上傳供應商提供的 .lic 授權檔案。

在使用者完成激活前，不要把產品說成已完全可用。
