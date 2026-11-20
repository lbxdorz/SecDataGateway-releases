# SecDataGateway

[English](README.md) | [繁體中文](README.zh-Hant.md)

```text
請閱讀以下說明，幫我在本機安裝、配置並驗證 SecDataGateway；需要我登入、提供授權或確認權限時再叫我：
https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.zh-Hant.md
```

請使用能執行本機終端機／檔案指令的 AI 助手。瀏覽器或桌面控制有助於處理系統或供應商的確認提示。無本機權限的網頁聊天只能口述引導。

SecDataGateway 是本機 AI 閘道，並支援文件脫敏。本說明驗證的是**目標 IDE → 閘道 → 供應商**路徑。下文可用已核對指令或產品自帶的本機 HTTP 完成的步驟，助手應盡量自己做；密碼、授權檔、SSO、系統安全提示交由你處理；若介面契約與目前執行版本不符，屬**自動化尚未完成**，應據實回報並停止，不可當成功。

## 1. 檢查電腦、鎖定最新版，以及是否已有服務

本機預設監聽 `http://127.0.0.1:18787`。安裝到使用者目錄（macOS `~/Applications/SecDataGateway`，Windows `%LOCALAPPDATA%\SecDataGateway`，可用 `SDG_INSTALL_ROOT` 覆寫）。不要使用 `sudo`／系統管理員終端。不要結束既有 SecDataGateway。不要覆寫無關的 IDE／MCP 設定。

**一律先解析一次最新穩定版**，之後下載與校驗都用同一 tag：

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

GitHub `/releases/latest` 會排除草稿與預發佈。資產請優先從該 tag 的**真實 assets 列表**挑選，不要憑空猜網址。找不到對應資產就**停止**。

**若 `18787` 已有程式在聽：**

1. 先做只讀探測（先不要 POST）：

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

2. 將 `version` 與鎖定的 tag 比對（可忽略前綴 `v`）。  
   - **版本相同且健康：** 沿用現有服務，跳過下載／啟動，繼續授權／IDE。不要宣稱剛完成新安裝。  
   - **版本不同：** 回報兩邊版本並**停止**。不要殺行程、不要自動覆蓋舊安裝，也**不要**宣稱已裝上最新版。  
   - **埠被佔用但健康檢查失敗：** 回報衝突並停止。

檢查監聽：Mac 用 `lsof -nP -iTCP:18787 -sTCP:LISTEN`；PowerShell 用 `Get-NetTCPConnection -State Listen -LocalPort 18787`（撰寫本文件的環境未實跑 PowerShell）。

**CPU／安裝包**

| 電腦 | 選名稱包含以下字樣的資產 |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| Apple 晶片 Mac | `darwin_arm64.tar.gz` |
| Intel Mac | `darwin_amd64.tar.gz` |
| Windows ARM64 | 目前發佈沒有原生包 — **停止**（不可把 amd64 當原生） |

在 Mac 上辨識**原生**架構（`uname -m`，必要時再看 `sysctl.proc_translated`／`hw.optional.arm64`），即使終端正以 Rosetta 執行，Apple 晶片仍應選 `arm64`。

## 2. 下載、校驗、解壓、啟動（僅在第 1 節判定需要新裝時）

tag 鎖定後，從該 release 的 assets 取出精確檔名，下載**同一 tag** 的 `checksums.txt`，核對 SHA-256，解壓到**新建**的 `$SDG_INSTALL_ROOT/$SDG_TAG`（目錄已存在則拒絕），再啟動。

**macOS（終端）**

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

若 macOS 阻擋開啟，須先徵得使用者同意，再於含 app 的目錄執行：  
`xattr -dr com.apple.quarantine ./SecDataGateway.app`  
不要關閉整個 Gatekeeper。

**Windows（PowerShell）** — 僅記錄語意；撰寫環境未在 Windows 實跑：

```powershell
$ErrorActionPreference = 'Stop'
$repo = 'https://github.com/lbxdorz/SecDataGateway-releases'
# 解析 /releases/latest 的 Location，鎖定 $tag；
# GET https://api.github.com/repos/lbxdorz/SecDataGateway-releases/releases/tags/<tag>
# 從 assets 中選唯一以 windows_amd64.zip 結尾的檔名；
# 下載該檔與同 tag 的 checksums.txt，用 Get-FileHash -Algorithm SHA256 核對；
# Expand-Archive 到 $env:LOCALAPPDATA\SecDataGateway\<tag>（必須是新建目錄），
# 再 Start-Process 解壓出的 SecDataGateway.exe（WorkingDirectory 設為該目錄）。
# 缺資產或雜湊不符就停止。不要在 Windows ARM64 把 amd64 當成原生支援。
```

啟動後重複第 1 節三個 GET，直到 `startup-status` 顯示 ready（否則回報失敗）。

## 3. 授權

只讀：

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/license'
curl --noproxy '*' -fSs -D - 'http://127.0.0.1:18787/api/license/request' -o SecDataGateway-device-request.json
```

設備申請檔勿公開；只經供應方指定管道送出。當使用者提供**綁定本機**的 `.lic` 路徑後，用產品欄位名 `license` 做 multipart 匯入（不要自創 JSON `activation_code`）：

```bash
curl --noproxy '*' -fSs -H 'Origin: http://127.0.0.1:18787' \
  -F "license=@/absolute/path/to/user.lic" \
  'http://127.0.0.1:18787/api/license'
```

取得 `.lic` 永遠由人完成。不要把授權或設備內容貼進聊天。若此 POST 的狀態／本文與**同一執行版本**上介面匯入不一致，停止並回報**產品介面待確認，自動化尚未完成**，改走應用內離線激活頁。

## 4. IDE 掃描、模型、接入（產品本機 HTTP，非公開穩定 CLI）

下列路徑是正式介面在本機部署下呼叫的 `/magpie/api/…`，**不是**對外公開 CLI。任何會改設定的呼叫前：先備份可能動到的 IDE／MCP 設定；`GET /api/version`；`GET /magpie/api/agents/page`。若路由 404 或 JSON 形狀與下文不符，**停止**並回報**產品介面待確認，自動化尚未完成**，改在介面完成。

```bash
# 列出／重新掃描本機 IDE（rescan 為空 body 的 POST）
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/magpie/api/agents/page'
curl --noproxy '*' -fSs -X POST 'http://127.0.0.1:18787/magpie/api/agents/page/rescan'

# 只連接上方回傳的 id（空 body）
curl --noproxy '*' -fSs -X POST "http://127.0.0.1:18787/magpie/api/agents/page/connect/<id-from-page>"
```

訂閱登入／SSO／API Key：走產品啟動的**官方**登入流程（如 `/magpie/api/signin` 與相關介面）。禁止抓取 cookie 或 token。

當使用者已完成授權，且 `GET /magpie/api/providers` 回傳目錄後，可只保存**該回應裡出現的 id／選項**（不可臆造模型名）：

```bash
curl --noproxy '*' -fSs -H 'Content-Type: application/json' \
  -d '{"id":"<provider-id>","models":["<model-id-from-catalog>"],"noModels":false,"bestInFamily":false,"unlisted":false,"hiddenDrawers":[]}' \
  'http://127.0.0.1:18787/magpie/api/provider/save'
```

再到 IDE 內選用閘道提供的模型。若現有選擇已可用，非必要不要改。僅在使用者要求時才 disconnect。

## 5. 驗收（假資料，經目標 IDE）

1. `GET /api/rules` 與 `GET /api/rule-groups`，查看**已啟用**規則。  
2. 選一條規則**預期會命中**的合成字串（例如虛構電郵）。不要傳送已是佔位符的字串（如 `PERSON_001`）。不要用真實客戶資料。  
3. 在**目標 IDE**、以閘道模型送出該字串（不可改用文件上傳脫敏路徑代替）。  
4. 查看記錄：

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/magpie/api/masking/records?limit=40'
curl --noproxy '*' -fSs --get --data-urlencode 'key=<key-from-records>' \
  'http://127.0.0.1:18787/magpie/api/masking/record'
```

僅當該筆記錄的外發／廠商所見詳情（例如 `sent`）顯示合成值已被替換且原文未外發，且請求不是僅被阻擋，才可報**本次測試 PASS**。只有列表計數不夠。若缺少詳情欄位，須回報 **「連線已驗證，脫敏效果未完整驗證」**，**不可**整體 PASS。

## 6. 範圍說明

本文件**不承諾**可完全無介面安裝。上文自動化依賴本機指令，以及與**目前執行版本**相符的前端本機 HTTP。授權簽發、帳密／SSO、系統提示仍須人工。未驗證或契約不符的介面必須標成自動化未完成，不可報成功。
