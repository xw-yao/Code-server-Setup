# code-server setup (NFS-safe, local disk, zero-CDN)

A practical, copy-paste setup that makes **VS Code in the browser** work reliably behind VPNs and on NFS home directories. It:

- Installs `code-server` in your home (no sudo)  
- Forces **webviews/notebooks** to use **local assets** (not `vscode-cdn.net`)  
- Stores caches on **local disk** (e.g., `/tmp`) to avoid NFS weirdness  
- Adds handy **aliases** on both the remote server and your laptop  

---

## Table of contents

- [Prereqs](#prereqs)
- [Install on the remote server](#install-on-the-remote-server)
- [Prefer local disk over NFS](#prefer-local-disk-over-nfs)
- [Aliases & helpers](#aliases--helpers)
  - [Remote server (`~/.bashrc`)](#remote-server-bashrc)
  - [Local machine (`~/.bashrc` or `~/.zshrc`)](#local-machine-bashrc-or-zshrc)
- [Harden notebooks: disable the CDN](#harden-notebooks-disable-the-cdn)
- [First run checklist](#first-run-checklist)

---

## Prereqs

- You can SSH into the **remote** Linux server.
- You have a user-writable directory like `~/apps`.
- Your browser can connect to `http://127.0.0.1:8680` via an SSH tunnel.

> Tip: If your `$HOME` is on **NFS**, we’ll keep volatile data in `/tmp/$USER/...` to avoid the classic `.nfsXXXX` “file busy” ghosts.

---

## Install on the remote server

> **No sudo.** We install under `~/apps/code-server`.

```bash
# choose the version you want (4.103.2 used here)
VER=4.103.2

mkdir -p ~/apps && cd ~/apps
# use the direct release URL (avoid “latest” redirect issues on VPNs)
curl -L -o cs.tgz   "https://github.com/coder/code-server/releases/download/v${VER}/code-server-${VER}-linux-amd64.tar.gz"

# sanity check
ls -lh cs.tgz
file cs.tgz   # should say: gzip compressed data

# extract & place
tar -xzf cs.tgz
mv "code-server-${VER}-linux-amd64" code-server

# confirm
~/apps/code-server/bin/code-server --version
```

---

## Prefer local disk over NFS

Run code-server using **local** directories for data & extensions (fast, stable).  
If your machine has `/scratch/$USER`, use that; otherwise `/tmp/$USER` is fine.

**Option A: fully local/ephemeral** (fastest; cleared on reboot)
```bash
DATA_BASE="/tmp/$USER"
mkdir -p "$DATA_BASE/code-server-data" "$DATA_BASE/code-server-exts"
```

**Option B: hybrid** (keep extensions persistent in `$HOME`, speed up caches)
```bash
DATA_BASE="/tmp/$USER"
mkdir -p "$DATA_BASE/code-server-data" "$HOME/.local/share/code-server-extensions"
```

We’ll wire these into the alias below.

---

## Aliases & helpers

### Remote server (`~/.bashrc`)

Add these to `~/.bashrc` and `source ~/.bashrc` afterwards.

```bash
# Pick where to store data/exts (A = both in /tmp; B = hybrid)
# --- Option A (both ephemeral) ---
CS_DATA_BASE="/tmp/$USER/"
CS_EXT_DIR="/tmp/$USER/code-server-exts"

# --- Option B (hybrid: persistent extensions in $HOME) ---
# CS_DATA_BASE="/tmp/$USER"
# CS_EXT_DIR="$HOME/.local/share/code-server-extensions"

mkdir -p "$CS_DATA_BASE/code-server-data" "$CS_EXT_DIR"

# Start code-server in background (quiet)
alias cs-up='nohup "$HOME/apps/code-server/bin/code-server"   --bind-addr 127.0.0.1:8680   --auth none   --log error   --user-data-dir "'"$CS_DATA_BASE"'/code-server-data"   --extensions-dir "'"$CS_EXT_DIR"'"   >/dev/null 2>&1 &'

# Foreground with debug logs (Ctrl+C to stop)
alias cs-debug='"$HOME/apps/code-server/bin/code-server"   --bind-addr 127.0.0.1:8680   --auth none   --log debug   --user-data-dir "'"$CS_DATA_BASE"'/code-server-data"   --extensions-dir "'"$CS_EXT_DIR"'"'

# Status / Stop
alias cs-status='ps -fu "$USER" | grep -i "[c]ode-server" || echo "no code-server for $USER"'
alias cs-stop='pkill -u "$USER" -f "code-server" || true'

# (Optional) environment guardrails some builds respect
export WEBVIEW_EXTERNAL_ENDPOINT=""
export VSCODE_WEBVIEW_EXTERNAL_ENDPOINT=""
```

### Local machine (`~/.bashrc` or `~/.zshrc`)

SSH tunnel to the remote’s 127.0.0.1:8680:

```bash
# Forward localhost:8680 → remote 127.0.0.1:8680
# Replace HOST_ALIAS with your ~/.ssh/config alias or user@host
alias cstunnel='ssh -fN   -L 8680:127.0.0.1:8680   -o ExitOnForwardFailure=yes   -o ServerAliveInterval=30   -o ServerAliveCountMax=3   HOST_ALIAS'
```

Usage:

```bash
# on remote:
cs-up      # or cs-debug

# on local:
cstunnel
# Then open: http://127.0.0.1:8680
```

---

## Harden notebooks: disable the CDN

We blank the webview CDN endpoints so notebooks load assets **locally**.

```bash
VSC=~/apps/code-server/lib/vscode/product.json

# Patch the three keys (safe to re-run after upgrades)
sed -i   -e 's/"webviewContentExternalBaseUrlTemplate": *"[^"]*"/"webviewContentExternalBaseUrlTemplate": ""/'   -e 's/"webviewExternalEndpoint": *"[^"]*"/"webviewExternalEndpoint": ""/'   -e 's/"webviewExternalEndpointTemplate": *"[^"]*"/"webviewExternalEndpointTemplate": ""/'   "$VSC" || true

# Verify
grep -n '"webview.*External' "$VSC"
# Expect empty values for all three keys
```

In VS Code **User Settings (JSON)** (on the code-server UI: gear → Settings → open as JSON), add:

```json
{
  "webview.useExternalEndpoint": false,
  "webview.experimental.useExternalEndpoint": false,
  "security.workspace.trust.untrustedFiles": "open"
}
```

---


## First run checklist

1. Start server (`cs-up` or `cs-debug`), then start your SSH tunnel.
2. In the browser, open **DevTools → Application → Service Workers**:  
   - **Unregister** any existing service worker  
   - **Storage → Clear site data** (all boxes)  
   - **Hard reload** (Ctrl/Cmd+Shift+R)
3. Open a notebook and run:
   ```python
   from IPython.display import JSON, HTML, display
   display(JSON({"ok": True}))
   display(HTML("<b>Hello</b>"))
   ```
4. In **DevTools → Network**, filter `vscode-cdn.net` — it should be **empty**.  
   (Renderer assets should load from `http://127.0.0.1:8680/...`)

---

### License

MIT for this guide. Use freely.
