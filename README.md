# SecDataGateway

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-Hant.md)

Local data gateway for AI tools. It runs on your computer, opens a browser UI, and asks for a license file before full use.

**Supported packages:** Windows x64 · macOS Apple silicon · macOS Intel  
**Not available:** Windows ARM64 native builds

## Install with an AI assistant

1. Open Cursor, Claude Code, WorkBuddy, or another assistant that can run commands on **this computer**.
2. Paste:

```text
Read https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.md and install the latest SecDataGateway for this computer. When the license page is open, stop and tell me to upload the vendor .lic myself. Ask me only for OS permission prompts. Do not request or upload my .lic.
```

3. When the assistant says the license page is open, upload the `.lic` file from your vendor in that page.

After licensing, use the in-app help for IDE connection and other features.

---

## Agent install guide

Goal: install the latest stable release for this machine, confirm the service is ready, open the license page, then stop.

Rules:

- Use only packages from [SecDataGateway-releases](https://github.com/lbxdorz/SecDataGateway-releases/releases).
- Do not download from mirrors you invent.
- Do not use `sudo` / Administrator elevation.
- Do not kill an existing SecDataGateway process.
- Do not read, request, copy, or upload any `.lic`.
- Do not configure IDEs or run redaction tests in this flow.
- If an asset is missing, a checksum fails, startup fails, or health checks fail: stop and report the error.

License page to open when ready: `http://127.0.0.1:18787/#license`

### 1. Resolve the latest stable release once

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

Keep this tag for the package and for `checksums.txt`. Choose the asset from that tag’s GitHub assets list. If no matching asset exists, stop.

| Machine | Asset name ends with |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| macOS Apple silicon | `darwin_arm64.tar.gz` |
| macOS Intel | `darwin_amd64.tar.gz` |
| Windows ARM64 | stop — no native package |

On macOS, detect native CPU with `uname -m` and, if needed, `sysctl.proc_translated` / `hw.optional.arm64`, so an Apple silicon Mac still uses `arm64` under Rosetta.

Default install roots (override with `SDG_INSTALL_ROOT`):

- macOS: `~/Applications/SecDataGateway`
- Windows: `%LOCALAPPDATA%\SecDataGateway`

Use a new folder `$SDG_INSTALL_ROOT/$SDG_TAG`. If that folder already exists, stop instead of overwriting it.

### 2. Reuse a healthy service already on port 18787

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

Compare `version` to the locked tag (ignore a leading `v`).

| Result | Action |
| --- | --- |
| Same version and healthy | Reuse it. Skip download/start. Go to step 4. |
| Different version | Report both versions and stop. Do not replace the running install. |
| Port in use but unhealthy | Report the conflict and stop. |

Optional listener check on macOS: `lsof -nP -iTCP:18787 -sTCP:LISTEN`

### 3. Download, verify, extract, and start

Run this only when step 2 says a new install is needed.

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

If macOS blocks the app, ask the user, then run only:

```bash
xattr -dr com.apple.quarantine ./SecDataGateway.app
```

Do not turn off Gatekeeper for the whole system.

**Windows (PowerShell)**

1. Resolve `/releases/latest` and lock the tag.  
2. From that tag’s assets, take the single name ending in `windows_amd64.zip`.  
3. Download that zip and the same tag’s `checksums.txt`.  
4. Verify with `Get-FileHash -Algorithm SHA256`.  
5. Expand into a new `%LOCALAPPDATA%\SecDataGateway\<tag>` folder.  
6. Start `SecDataGateway.exe` from that folder.  

If the asset is missing or the hash does not match, stop.

After start, repeat the three health checks until `startup-status` reports ready. If it never becomes ready, stop.

### 4. Open the license page and stop

```bash
# macOS
open 'http://127.0.0.1:18787/#license'
```

```powershell
# Windows
Start-Process 'http://127.0.0.1:18787/#license'
```

Then tell the user:

> Installation is complete and the license page is open. Please upload the `.lic` file from your vendor yourself.

If the user prefers Chinese, use:

> 已完成安裝並開啟授權頁面，請自行上傳供應商提供的 .lic 授權檔案。

Do not continue past this point in this install flow.
