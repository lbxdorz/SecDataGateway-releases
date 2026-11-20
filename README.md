# SecDataGateway

[English](README.md) | [繁體中文](README.zh-Hant.md)

```text
Read the following instructions and help me install, configure, and verify SecDataGateway on this computer. Ask me when you need me to sign in, provide a license, or confirm a permission:
https://raw.githubusercontent.com/lbxdorz/SecDataGateway-releases/main/README.md
```

Use an AI assistant that can run local shell/file commands. Browser or desktop control helps when the OS or a provider asks you to confirm something. A web chat without local access can only guide you.

SecDataGateway is a local AI gateway with document redaction. This guide verifies the **target IDE → gateway → provider** path. Steps the assistant can finish with verified commands or the product’s own local HTTP calls are below; passwords, license files, SSO, and OS security prompts stay with you; anything whose contract does not match this running build is **not** finished automation—say so and stop.

## 1. Machine, latest tag, and any service already running

Default listen address for local installs is `http://127.0.0.1:18787`. Install into a user folder (macOS `~/Applications/SecDataGateway`, Windows `%LOCALAPPDATA%\SecDataGateway`, override with `SDG_INSTALL_ROOT`). No `sudo` / Administrator shell. Do not kill an existing SecDataGateway. Do not overwrite unrelated IDE/MCP settings.

**Always resolve the latest stable release once**, then keep that tag for every later download and checksum:

```bash
SDG_REPO='https://github.com/lbxdorz/SecDataGateway-releases'
SDG_RELEASE_URL=$(curl -fLsS -o /dev/null -w '%{url_effective}' "$SDG_REPO/releases/latest")
SDG_TAG=${SDG_RELEASE_URL##*/}
case "$SDG_TAG" in v[0-9]*) ;; *) echo 'Unexpected release tag'; exit 1;; esac
echo "Locked tag: $SDG_TAG"
```

GitHub `/releases/latest` excludes drafts and prereleases. Prefer selecting the asset from that tag’s real asset list (API or release page), not a guessed URL. If the matching asset is missing, **stop**.

**If something already listens on `18787`:**

1. Read-only probes (do not POST yet):

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/health'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/version'
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/startup-status'
```

2. Compare `version` to the locked tag (ignore a leading `v`).  
   - **Same version and health OK:** reuse this service; skip download/start; continue at licensing / IDE. Do not claim a new install.  
   - **Different version:** report both versions and **stop**. Do not kill the process, do not auto-overwrite the old install, and do **not** claim the latest build is installed.  
   - **Port busy but health fails:** report the conflict and stop.

Listener checks: Mac `lsof -nP -iTCP:18787 -sTCP:LISTEN`; PowerShell `Get-NetTCPConnection -State Listen -LocalPort 18787` (PowerShell path not executed on this documentation host).

**CPU / package selection**

| Machine | Choose an asset whose name contains |
| --- | --- |
| Windows x64 | `windows_amd64.zip` |
| Apple silicon Mac | `darwin_arm64.tar.gz` |
| Intel Mac | `darwin_amd64.tar.gz` |
| Windows ARM64 | No native package in current releases — **stop** (do not treat amd64 as native) |

On Mac, detect **native** architecture (`uname -m`, and if needed `sysctl.proc_translated` / `hw.optional.arm64`) so a Rosetta shell still picks `arm64` when the Mac is Apple silicon.

## 2. Download, verify, extract, start (only when section 1 says to install)

After the tag is locked, pick the exact asset name from that release’s assets, download **the same tag’s** `checksums.txt`, verify SHA-256, extract into a **new** `$SDG_INSTALL_ROOT/$SDG_TAG` folder (refuse if the folder already exists), then start.

**macOS (Terminal)**

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

If macOS blocks the app, ask the user before running only:  
`xattr -dr com.apple.quarantine ./SecDataGateway.app`  
from the folder that contains the app. Do not disable Gatekeeper globally.

**Windows (PowerShell)** — syntax documented; not executed on the Mac that authored this file:

```powershell
$ErrorActionPreference = 'Stop'
$repo = 'https://github.com/lbxdorz/SecDataGateway-releases'
$latest = Invoke-WebRequest -UseBasicParsing -Uri "$repo/releases/latest" -MaximumRedirection 0 -ErrorAction SilentlyContinue
if (-not $latest -and $_.Exception.Response) { $latest = $_.Exception.Response }
# Prefer: resolve Location from /releases/latest, lock $tag, then
# GET https://api.github.com/repos/lbxdorz/SecDataGateway-releases/releases/tags/<tag>
# and pick the single asset whose name ends with windows_amd64.zip.
# Download that asset + checksums.txt from the same tag, verify with Get-FileHash -Algorithm SHA256,
# Expand-Archive into $env:LOCALAPPDATA\SecDataGateway\<tag> (must be new), then
# Start-Process the extracted SecDataGateway.exe with WorkingDirectory set to that folder.
# If the asset is missing or the hash mismatches, stop. Do not use an amd64 zip on Windows ARM64 as "native".
```

After start, repeat the three GET checks in section 1 until `startup-status` shows ready (or report failure).

## 3. License

Read-only:

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/api/license'
curl --noproxy '*' -fSs -D - 'http://127.0.0.1:18787/api/license/request' -o SecDataGateway-device-request.json
```

Keep the device-request file private; send it only through the vendor’s licensing channel. When the user gives a **device-bound** `.lic` path, import with the product’s multipart field name `license` (do not invent JSON `activation_code`):

```bash
curl --noproxy '*' -fSs -H 'Origin: http://127.0.0.1:18787' \
  -F "license=@/absolute/path/to/user.lic" \
  'http://127.0.0.1:18787/api/license'
```

Getting the `.lic` is always a human step. Do not paste license or device contents into chat. If this POST returns an unexpected status/body versus a working UI import on the **same running build**, stop and report **product interface pending confirmation; automation not complete**—then use the in-app offline activation page.

## 4. IDE scan, models, connect (local product HTTP — not a public stable CLI)

These paths are what the official UI already calls under `/magpie/api/…` on local deployments. They are **not** a documented public CLI. Before any changing call: backup IDE/MCP config you might touch; `GET /api/version`; `GET /magpie/api/agents/page`. If routes 404 or JSON shape differs from below, **stop** with **product interface pending confirmation; automation not complete** and finish those steps in the UI.

```bash
# List / refresh detected IDEs (rescan is POST with empty body)
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/magpie/api/agents/page'
curl --noproxy '*' -fSs -X POST 'http://127.0.0.1:18787/magpie/api/agents/page/rescan'

# Connect only an id returned above (empty body)
curl --noproxy '*' -fSs -X POST "http://127.0.0.1:18787/magpie/api/agents/page/connect/<id-from-page>"
```

Provider login / SSO / API keys: open the **official** sign-in flow the product starts (`/magpie/api/signin` and related UI). Never scrape cookies or tokens.

When the user is already authorized and `GET /magpie/api/providers` returns the catalog, you may save **only ids/options from that response** (do not invent model names):

```bash
curl --noproxy '*' -fSs -H 'Content-Type: application/json' \
  -d '{"id":"<provider-id>","models":["<model-id-from-catalog>"],"noModels":false,"bestInFamily":false,"unlisted":false,"hiddenDrawers":[]}' \
  'http://127.0.0.1:18787/magpie/api/provider/save'
```

Then select the gateway-exposed model inside the IDE. Prefer not changing an already-good selection. Disconnect only if the user asks.

## 5. Acceptance (fake data through the target IDE)

1. `GET /api/rules` and `GET /api/rule-groups` — note **enabled** rules.  
2. Choose a **synthetic** string those rules should match (for example a made-up email). Do **not** send text that is already a placeholder such as `PERSON_001`. Do not use real customer data.  
3. Send that string from the **target IDE** using the gateway model (no substitute document-upload path).  
4. Inspect records:

```bash
curl --noproxy '*' -fSs 'http://127.0.0.1:18787/magpie/api/masking/records?limit=40'
curl --noproxy '*' -fSs --get --data-urlencode 'key=<key-from-records>' \
  'http://127.0.0.1:18787/magpie/api/masking/record'
```

**PASS** only if that record’s outbound / vendor-visible detail (for example `sent`) shows the synthetic value replaced and absent as raw text, and the request was not merely blocked. List counts alone are not enough. If detail fields are missing, report **「連線已驗證，脫敏效果未完整驗證」** / “Connection verified; redaction effect not fully verified” and **do not** claim overall PASS.

## 6. Scope reminder

This document does **not** promise a fully GUI-free install. Automated pieces above depend on local commands plus matching front-end HTTP on **this** running build. Human steps remain for license issuance, credentials/SSO, and OS prompts. Unverified or mismatched interfaces must be reported as incomplete automation—not as success.
