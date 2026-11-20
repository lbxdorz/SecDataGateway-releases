# SecDataGateway

[English](README.md) | [繁體中文](README.zh-Hant.md)

```text
Read the following instructions and install the latest SecDataGateway for this computer, then open the license/activation page and stop. Ask me only for OS permission prompts. Do not upload or request my .lic file:
https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.md
```

**For people:** This page is for a local AI assistant. It installs SecDataGateway, checks that the service is ready, opens the license page, and asks you to upload the `.lic` file from your vendor yourself. It does not finish product setup after licensing.

**Success means:** latest stable build for this OS/CPU is running, the license page is open, and the assistant told you to upload the vendor `.lic` yourself. Unlicensed software is **not** fully usable yet. After you activate, use the in-app help / product manuals for IDE connection and other features.

Use an assistant with local shell access. A web chat without local control can only guide you.

---

## Instructions for the AI assistant

Do these steps in order. Prefer commands below. Do **not** read, request, copy, or upload any `.lic`. Do **not** activate licensing for the user. Do **not** configure IDEs or run redaction tests in this flow. On errors (missing asset, bad checksum, start failure, health failure), **stop** and report—do not invent mirrors or disable OS security. No `sudo`.

Default local UI base: `http://127.0.0.1:18787`.  
Verified license page deep link (SPA hash route `license` in the product UI):  
`http://127.0.0.1:18787/#license`

### 1) Lock the latest stable release once

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

GitHub `/releases/latest` excludes drafts and prereleases. Keep this tag for every download and for `checksums.txt`. Prefer picking the asset from that tag’s real GitHub assets list; if none matches, **stop**.

| Machine | Asset name must end with |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| Apple silicon Mac | `darwin_arm64.tar.gz` |
| Intel Mac | `darwin_amd64.tar.gz` |
| Windows ARM64 | No native package — **stop** (do not treat amd64 as native) |

On Mac, detect **native** CPU (`uname -m`, and if needed `sysctl.proc_translated` / `hw.optional.arm64`) so a Rosetta shell still selects `arm64` on Apple silicon.

Install root (overridable with `SDG_INSTALL_ROOT`): macOS `~/Applications/SecDataGateway`, Windows `%LOCALAPPDATA%\SecDataGateway`. Use a **new** `$SDG_INSTALL_ROOT/$SDG_TAG` folder; refuse to overwrite an existing one.

### 2) If something already listens on port 18787

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

Compare `version` to the locked tag (ignore a leading `v`).

- **Same version and healthy:** reuse it; skip download/start; go to step 4. Do not claim a new install.
- **Different version:** report both versions and **stop**. Do not kill the process, do not auto-overwrite, do not claim the latest build is installed.
- **Port busy but unhealthy:** report conflict and stop.

Listener check: Mac `lsof -nP -iTCP:18787 -sTCP:LISTEN`.

### 3) Download, verify, extract, start (only when step 2 says to install)

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

If macOS blocks the app, ask before: `xattr -dr com.apple.quarantine ./SecDataGateway.app` (only that app). Do not disable Gatekeeper.

**Windows (PowerShell — not executed on the Mac that authored this doc)**  
Resolve `/releases/latest` → lock tag → from that tag’s assets pick the sole `*windows_amd64.zip` → download it and same-tag `checksums.txt` → `Get-FileHash -Algorithm SHA256` → `Expand-Archive` into a **new** `%LOCALAPPDATA%\SecDataGateway\<tag>` → `Start-Process` `SecDataGateway.exe`. Missing asset or hash mismatch → stop.

After start, repeat the three GET checks until `startup-status` is ready (or stop with failure).

### 4) Open the license page (required end of this flow)

When health/version/startup look good:

```bash
# macOS
open 'http://127.0.0.1:18787/#license'
```

```powershell
# Windows
Start-Process 'http://127.0.0.1:18787/#license'
```

Evidence for this URL: the product UI registers page id `license` as a hash route, and when license status is not active the app navigates to that page. Local default listen address is `127.0.0.1:18787`. Opening `#license` lands on the offline activation / license file UI (export device request + choose `.lic` in the browser). Do not automate file pick or upload.

Then reply to the user **exactly in this sense** (Traditional Chinese OK if they use that language):

> 已完成安裝並開啟授權頁面，請自行上傳供應商提供的 .lic 授權檔案。

English equivalent: *Installation is done and the license page is open. Please upload the vendor-provided .lic file yourself.*

Do not say the product is fully ready before they activate.
