# Self-hosting from scratch: Mac server, Windows work PC, Android, Tailscale

This runbook builds the whole setup from zero:

```text
Android / Web / phone browser
        │  HTTPS or Tailscale
        ▼
Mac mini  ── Docker: PostgreSQL + Redis + migrate + Server + Web
        ▲
        │  outbound WebSocket
        │
Windows PC ── Desktop + managed Connector ── Codex / Claude / DSH / Command Code
```

The Server stores accounts, devices, sessions, timelines, and attachments. Agent work happens on the Windows PC. Clients never connect to the Windows PC directly; the Connector dials out to the Server.

> **Version note.** The current mainline is **2.2.0**, which is not published yet. If you need Command Code, use the 2.2.0 code and builds described here. If you only need the released 2.1.0 line, use the `v2.1.0` tag and skip the Command Code sections. 2.1.0 clients can still sign in and continue existing Codex/Claude/DSH sessions against a 2.2.0 Server, but they cannot create 2.2.0 workflow or Command Code sessions. Do not expect 2.2.0-only features from a 2.1.0 Server or client.

## 0. Decide the address first

Pick one Server URL and use it on every device. Changing it later means re-entering the address on every client and re-pairing the Windows Connector.

| Placeholder | Meaning |
| --- | --- |
| `<SERVER_LAN_IP>` | Mac's address on the home LAN, for example `192.168.x.x` |
| `<SERVER_TAILSCALE_IP>` | Mac's Tailscale address (`100.x.y.z`) |
| `<WINDOWS_TAILSCALE_IP>` | Windows PC's Tailscale address |
| `<ANDROID_TAILSCALE_IP>` | Android phone's Tailscale address |
| `<SERVER_PORT>` | Published Web/API port, default `5174` |

Replace every `<...>` placeholder before running a command. Angle brackets are shell redirection characters in `zsh` and PowerShell; a literal `<SERVER_TAILSCALE_IP>` will not work.

Two valid choices:

| Choice | Server URL | When |
| --- | --- | --- |
| LAN only | `http://<SERVER_LAN_IP>:<SERVER_PORT>` | Every client is always on the same Wi-Fi/LAN |
| Tailscale | `http://<SERVER_TAILSCALE_IP>:<SERVER_PORT>` | Phone or laptop works from anywhere; recommended here |

Use the **Tailscale URL** if you want the Android phone to work away from home. The phone must have Tailscale enabled. Android allows plain HTTP for self-hosted addresses; iOS does not allow raw `100.64.0.0/10` HTTP, so iOS needs the HTTPS/MagicDNS option in the Tailscale section.

Set `AGENT_SERVER_PUBLIC_ORIGIN` to the URL you choose. It is used for OAuth redirects and share links; it does not restrict who can reach the Server.

## 1. Mac mini as the Server

### Prerequisites

- macOS with Docker Desktop installed and running.
- `git`, `curl`, and enough disk for PostgreSQL, Redis AOF, and attachments.
- The Mac must stay awake and reachable. A sleeping Mac means every client is offline.

### Checkout

```zsh
cd ~/Documents
git clone https://github.com/anywhere-labs/Agents-Anywhere.git aa-server
cd aa-server
git checkout main          # 2.2.0 line
git log -1 --oneline
```

For a released line use `git checkout v2.1.0`. The Server needs the 2.2.0 line for ACP sessions and Command Code; the Command Code provider itself is in the Windows Connector build below, and the Android build is the 2.2.0 client.

### Deployment env file

Keep secrets outside the repository:

```zsh
mkdir -p ~/agents-anywhere
cd ~/agents-anywhere
umask 077
cat > server.env <<'EOF'
POSTGRES_PASSWORD=replace-with-openssl-rand-hex-24
AGENT_SERVER_SECRET=replace-with-openssl-rand-hex-32
AGENTS_ANYWHERE_WEB_PORT=5174
# Use the LAN URL instead if every client stays on the LAN.
AGENT_SERVER_PUBLIC_ORIGIN=http://<SERVER_TAILSCALE_IP>:5174
REDIS_MAXMEMORY=256mb
EOF
chmod 600 server.env
```

Generate the two secrets with `openssl rand -hex 24` and `openssl rand -hex 32`. Do not append `/api/v2` and do not add a trailing slash to `AGENT_SERVER_PUBLIC_ORIGIN`.

### Start the stack

`zsh` does not split a string variable into words. Use a function, not `COMPOSE="docker compose ..."`:

```zsh
compose() {
  docker compose --env-file "$HOME/agents-anywhere/server.env" \
    -f docker/docker-compose.postgres.yml "$@"
}

cd ~/Documents/aa-server
compose up -d --build
compose logs migrate-next | tail -20
compose ps
```

The `migrate-next` service upgrades the database before `server-next` starts. Do not start a second Server version against the same database.

### Health checks

```zsh
curl -s http://<SERVER_TAILSCALE_IP>:5174/api/v2/health
curl -s http://<SERVER_TAILSCALE_IP>:5174/api/v2/health/ready
```

Expected:

- `/health` → HTTP 200, `"status":"ok"`, and `"version"` matches the checked-out line (`2.2.0` for `main`, `2.1.0` for `v2.1.0`).
- `/health/ready` → database, Redis and realtime all `"ok"`.
- `schemaVersion` matches the migration head of the checked-out code. At the time of writing it is `v2_55`; read the value from `/health/ready` and from `migrate-next` logs instead of guessing.

### Create the first admin

On an empty database the Server prints a one-time setup token:

```zsh
compose logs server-next | grep -A 6 'first-run setup required'
```

Open `http://<SERVER_TAILSCALE_IP>:5174`, choose the self-hosted/bootstrap path, and create the first account. That account becomes admin and registration is disabled afterward. The token expires after 15 minutes. Reading the log again does not mint a new token; reload the sign-in page (which calls `GET /api/v2/auth/config`) and then re-read the log, or restart the Server.

Add any extra accounts from the admin Team page.

### Keep it running

- Docker Desktop → Settings → General → **Start Docker Desktop when you sign in**.
- Docker Desktop needs a logged-in GUI session. Automatic login is blocked while FileVault is enabled: either turn FileVault off and enable automatic login, or accept that after a reboot the stack stays down until you log in and unlock the Mac.
- Prevent sleep:

```zsh
sudo pmset -a sleep 0 disksleep 0 womp 1
pmset -g
```

- No router port forward and no public tunnel. Tailscale is the remote path.
- Allow Docker in the macOS firewall.

### Backups

Back up all three stateful pieces. PostgreSQL holds the database, the files volume holds uploads, and the Redis AOF volume can contain accepted timeline writes that have not been flushed.

```zsh
#!/bin/bash
set -euo pipefail
BACKUP_DIR="$HOME/agents-anywhere/backups"
STAMP="$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"
cd "$HOME/Documents/aa-server"

docker compose --env-file "$HOME/agents-anywhere/server.env" \
  -f docker/docker-compose.postgres.yml \
  exec -T postgres-next pg_dump -U agents_anywhere agents_anywhere \
  | gzip > "$BACKUP_DIR/pg-$STAMP.sql.gz"

docker run --rm \
  -v agents-anywhere_agents-anywhere-files-next:/data:ro \
  -v "$BACKUP_DIR":/out \
  alpine tar czf "/out/files-$STAMP.tar.gz" -C /data .

docker run --rm \
  -v agents-anywhere_agents-anywhere-redis-next:/data:ro \
  -v "$BACKUP_DIR":/out \
  alpine tar czf "/out/redis-$STAMP.tar.gz" -C /data .

find "$BACKUP_DIR" -maxdepth 1 -type f -mtime +14 -delete
```

Save this as `~/agents-anywhere/backup.sh` and make it executable (`chmod +x ~/agents-anywhere/backup.sh`). Check volume names with `docker volume ls` if your Compose project name differs. The timestamp keeps a second run on the same day from overwriting the day's only backup. The `find` line is the 14-day retention; adjust it if you want a different window. Schedule the script with `launchd` or `cron` and test a restore into a scratch database at least once. A backup you have never restored is not a backup.

### Upgrade

```zsh
~/agents-anywhere/backup.sh
cd ~/Documents/aa-server
git fetch --tags
git checkout <new-tag-or-main>
compose up -d --build
compose logs migrate-next | tail -20
curl -s http://<SERVER_TAILSCALE_IP>:5174/api/v2/health/ready
```

Upgrade the Windows Desktop and Android client to the same release line. Never run an older Server against a migrated database; restore the backup if you need to go back.

## 2. Tailscale

Install Tailscale on the Mac, the Windows PC, and the Android phone. Sign all three into the **same tailnet**.

### Mac

```zsh
# Use the standalone build; the sandboxed App Store build may not expose the
# CLI, tailscale serve, or subnet routing.
brew install tailscale
sudo brew services start tailscale
# Alternative: sudo tailscaled install-system-daemon
sudo tailscale up
sudo tailscale set --operator=$USER
tailscale status
tailscale ip -4
```

If `tailscale up` reports `failed to connect to local Tailscale service`, the daemon is not running. Start it with `sudo brew services start tailscale` (or install the system daemon) and retry. If you use the Mac App Store build, `tailscale` may not be on `PATH`; use the app's command or add the binary to `PATH`.

### Windows

Install Tailscale from <https://tailscale.com/download/windows>, sign in, and confirm:

```powershell
tailscale status
tailscale ip -4
```

### Android

Install Tailscale from Google Play, sign in, and enable the VPN toggle. The status bar shows a VPN icon.

### Verify the path

From Windows:

```powershell
curl.exe http://<SERVER_TAILSCALE_IP>:5174/api/v2/health
```

From the Android browser, open the same URL. It should return JSON, not a LAN timeout. If it fails, run `tailscale status` on both devices and confirm they show each other in the same tailnet.

### Use Tailscale names instead of IPs

MagicDNS can give you a stable name such as `http://peters-mac-mini:5174`. Enable MagicDNS in the tailnet settings. If MagicDNS is not enabled, use the `100.x.y.z` address. The IPs are stable per device but can change after re-authentication.

### Optional: make the LAN IP work over Tailscale

You do **not** need this if you use the Tailscale URL. If you want `192.168.x.x` to work from the phone, advertise the LAN subnet from the Mac (replace `192.168.1.0/24` with your LAN CIDR):

```zsh
sudo tailscale up --advertise-routes=192.168.1.0/24
```

Approve the route in the Tailscale admin console, then enable **Use Tailscale subnets** in the Android Tailscale app. Subnet routing requires the standalone `tailscaled` build; the sandboxed Mac App Store build may not support it. This is optional; the direct Tailscale IP is simpler.

### iOS caveat

iOS ATS blocks plain HTTP to `100.64.0.0/10`. Android does not have that restriction. If you need iOS or a secure browser context, terminate HTTPS with Tailscale.

First enable HTTPS certificates for the tailnet in the admin console:

<https://login.tailscale.com/admin/dns> → **HTTPS Certificates** → **Enable**.

Without that toggle, `tailscale serve` can hang and `tailscale cert` reports that HTTPS cert support is not enabled. Current Tailscale uses the port-only target form; the old `https:443 http://127.0.0.1:5174` mount syntax is gone:

```zsh
tailscale serve --bg 5174
# explicit form:
tailscale serve --bg --https=443 5174
tailscale serve status
```

Replace `5174` with the published `AGENTS_ANYWHERE_WEB_PORT` if you changed it.

`tailscale serve` also requires the standalone `tailscaled` build. Then set `AGENT_SERVER_PUBLIC_ORIGIN=https://<host>.<tailnet>.ts.net` in `server.env`, restart `server-next`, and use the HTTPS URL on every client.

## 3. Windows work PC

### Prerequisites

- Windows 10/11 x64.
- Node.js 22 LTS and Corepack/Yarn 4 (needed to build Desktop).
- Git for Windows (Claude Code's bash tool and general shell work want it on `PATH`).
- Codex and Claude Code CLIs:

```powershell
winget install OpenJS.NodeJS.LTS
winget install Git.Git
corepack enable
npm i -g @openai/codex @anthropic-ai/claude-code
```

Run `codex` and `claude` once in a normal terminal to sign in. The Connector discovers them from `PATH`; restart the Desktop after installing or signing in if they do not appear.

- Command Code: install the **Command Code Desktop** app, which bundles its CLI, or install the CLI globally:

```powershell
npm i -g command-code@latest
cmdc login
```

The Connector can also use the CLI bundled inside the Command Code Desktop app, so a separate global install is optional. `cmdc login` and `cmdc status` are Command Code CLI commands; see [the Command Code headless docs](https://commandcode.ai/docs/headless) for their authoritative flags. The Connector passes its environment to the CLI, so `COMMAND_CODE_API_KEY` is an alternative authentication method documented by Command Code.

- Android build tools only if you build the APK on this PC: JDK 17 and the Android SDK with platform 36 and build-tools 36.0.0. Set `ANDROID_HOME` or `android/local.properties`.

### Build the Desktop installer

This guide is published on the branch that already contains Command Code. A fresh checkout of that branch needs no extra step; if you already have it, stay on it:

```powershell
git clone --branch <branch-with-command-code> https://github.com/anywhere-labs/Agents-Anywhere.git D:\path\to\Agents-Anywhere
# or: cd D:\path\to\Agents-Anywhere   (the checkout you already have)
corepack enable
cd desktop-workbench
yarn install
yarn dist:win
```

Skip the clone if the directory already exists. If you are starting from `upstream/main` before PR #611 is merged, fetch the PR branch instead:

```powershell
git fetch origin pull/611/head:command-code
git checkout command-code
```

If `origin` is your fork instead of `anywhere-labs`, add the base remote and fetch from it:

```powershell
git remote add upstream https://github.com/anywhere-labs/Agents-Anywhere.git
git fetch upstream pull/611/head:command-code
git checkout command-code
```

After PR #611 is merged, `main` works directly.

The installer is `desktop-workbench\release\Agents Anywhere Setup 2.2.0.exe`. The filename follows the version in `desktop-workbench/package.json`; adjust it after a version bump. It is unsigned, so Windows SmartScreen will ask for confirmation.

If `yarn dist:win` fails with `EBUSY ... default_app.asar`, a file watcher or antivirus scan is holding the unpacked Electron files. Build to a directory outside the repository:

```powershell
corepack yarn electron-builder --win nsis --x64 --publish never `
  --config.directories.output="$env:TEMP\aa-dist"
```

Then copy `Agents Anywhere Setup 2.2.0.exe` from `$env:TEMP\aa-dist` into `desktop-workbench\release`.

A faster development loop, once the installer exists, is `yarn dev` with the repo Connector:

```powershell
cd ..\connector
uv sync
$env:WORKBENCH_CONNECTOR_CLI="D:\path\to\Agents-Anywhere\connector\.venv\Scripts\anywhere-cli.exe"
cd ..\desktop-workbench
yarn dev
```

Quit the installed Desktop first; both share `%APPDATA%\Agents Anywhere` and the single-instance lock.

### Install and connect

1. Quit any installed **Agents Anywhere** from the tray. Do not run two Connectors for the same Windows user.
2. Run the installer and launch the app.
3. On the login screen choose **Connect to your own server instance** and enter the chosen URL:
   `http://<SERVER_TAILSCALE_IP>:5174` (or the LAN URL if every client is on the LAN).
4. Wait for the Connector environment to finish downloading Python and dependencies. The first launch can take several minutes.
5. Wait until the device shows **online**.

If the app was previously signed in with the LAN URL, sign out and sign in again with the Tailscale URL. The Connector stores the Server origin separately and must reconnect.

### Add runtimes

On the device page:

- **Codex** and **Claude**: add them if you installed and signed in to their CLIs. If a runtime is missing, restart the Desktop so it picks up the updated `PATH`.
- **DeepSeek Harness (DSH)**: optional; install the DSH Desktop plugin and follow [DSH Bridge Next](../dsh-bridge-next/README.md).
- **Command Code**: see the next section.

Keep the Desktop running in the tray. Closing the window hides it; quitting the tray process stops the Connector and the device goes offline. Do not additionally start a CLI Connector on the same Windows user.

Logs:

- Desktop: `%APPDATA%\Agents Anywhere\logs`
- Connector: `%USERPROFILE%\.agents-anywhere\logs`

## 4. Command Code

Command Code runs through the ACP runtime with the local `cmdc acp` entry.

### Login

```powershell
cmdc login
cmdc status
```

If `cmdc` is not on `PATH`, use the CLI bundled with Command Code Desktop:

```text
C:\Users\<you>\AppData\Local\Programs\Command Code\resources\app\node_modules\command-code\dist\index.mjs
```

This is the default Windows install path; if yours differs, locate `command-code/dist/index.mjs` under the Command Code installation. Alternatively, set `COMMAND_CODE_API_KEY` in the Connector's environment; the Connector passes its environment to the CLI. Whether the CLI accepts that variable is part of Command Code's contract (see the headless docs). The Connector and the Command Code Desktop app share the same user account files.

### Add the runtime

1. Device page → **Agent Runtimes → Available to add → Command Code**.
2. **Quick add**. The Connector auto-detects the CLI from `PATH`, the npm global package, or the Command Code Desktop bundle.
3. If it reports `commandcode_cli_missing`, open **Configure** and set `cliPath` to one of:
   - `C:\Users\<you>\AppData\Local\Programs\Command Code\resources\app\node_modules\command-code\dist\index.mjs`
   - `%APPDATA%\npm\cmdc.cmd` after a global npm install
4. Save and wait for the runtime to show **running**.

The Connector never launches `cmdc.cmd` directly. It resolves the npm shim to Node plus `dist/index.mjs`; on Windows it deliberately does not search `cmd`, so it cannot accidentally pick `C:\Windows\System32\cmd.exe`.

### Use it

- Create a session in a project folder and choose a model.
- The model and effort selectors come from Command Code's ACP catalog.
- The permission picker has five modes: Standard, Accept edits, Plan, Don't ask, Bypass.
- Tool approvals, attachments, session recovery, and "continue task" work through the normal session UI.
- Command Code commands are exposed when the CLI advertises them.
- Mid-turn steering is not supported. Wait for the turn or interrupt it.
- The native plan updates are not yet rendered as AA plan cards.

The Connector passes `--no-auto-update` so a Command Code background update cannot replace a live ACP process. Update the CLI manually when you want a new version.

## 5. Android

### Which build to install

The Play Store may still offer **2.1.0**. 2.2.0 is not published yet. Use one of:

- **2.2.0 debug APK** built from the repo, or
- the **2.2.0 Web UI** in the phone browser.

The native 2.1.0 app can open and continue existing sessions, but it cannot create 2.2.0 workflow sessions or Command Code sessions.

### Build and install the APK

Build the 2.2.0 app from the branch you are on. If you started from `main` before PR #611 is merged, use the PR branch shown in the Windows section so you also get the localized Command Code error strings.

```powershell
cd D:\path\to\Agents-Anywhere\android
.\gradlew.bat assembleDebug "-PagentsAnywhere.serverUrl=http://<SERVER_TAILSCALE_IP>:5174"
adb install -r app\build\outputs\apk\debug\app-debug.apk
```

On macOS/Linux use `./gradlew`. The APK is at `android/app/build/outputs/apk/debug/app-debug.apk`; its version comes from `android/app/build.gradle.kts` (`2.2.0`, versionCode 11 at the time of writing).

The debug build is signed with a different key from the release APK. If Android returns `INSTALL_FAILED_UPDATE_INCOMPATIBLE`, uninstall the old app first:

```powershell
adb uninstall com.agentsanywhere.app
adb install app\build\outputs\apk\debug\app-debug.apk
```

Uninstalling clears the app's local data. Sign in again afterward.

If you set `-PagentsAnywhere.serverUrl=...`, the debug build's "Agents Anywhere Cloud" entry points at that URL. The explicit **Connect to your own server instance** option is clearer and works the same way.

### Install Tailscale and connect

1. Install Tailscale from Google Play, sign in, and enable the VPN toggle.
2. Open Agents Anywhere → **Connect to your own server instance**.
3. Enter `http://<SERVER_TAILSCALE_IP>:5174`.
4. Sign in with the account created on the Server.

### QR login

The QR code contains the Server URL of the page that generated it:

- The **Web UI** at `http://<SERVER_TAILSCALE_IP>:5174` puts that Tailscale URL in the QR. This is the reliable choice when the phone is on Tailscale.
- The **Desktop** puts its own configured Server URL in the QR. If the Desktop is still configured with `http://<SERVER_LAN_IP>:5174`, the phone will fail even though the Desktop itself works.

To use the Desktop QR, first change the Desktop's Server URL to the Tailscale URL and sign in again. Otherwise generate the QR from the Web UI at the Tailscale URL.

### Verify

- The phone's browser can open `http://<SERVER_TAILSCALE_IP>:5174/api/v2/health` and receives JSON.
- The app's **Devices** screen shows the Windows device (for example `APRITA`) as online.
- Open a session, send a prompt, and see the reply.
- Open the same session on the Desktop and send a follow-up; both timelines should match.

### iOS

The App Store build may still be 2.1.0, and iOS blocks plain HTTP to `100.64/10`. Use the HTTPS/MagicDNS path from the Tailscale section, or use the Web UI over HTTPS.

## 6. Verification checklist

Run these in order:

| # | Check | Expected |
| --- | --- | --- |
| 1 | `curl http://<SERVER_TAILSCALE_IP>:5174/api/v2/health` | 200, `"status":"ok"`; `"version"` matches the checked-out line |
| 2 | `curl .../health/ready` | database, Redis, realtime all `ok`; schema matches the code |
| 3 | `compose ps` | `postgres-next`, `redis-next`, `server-next` up; `migrate-next` exited 0 |
| 4 | Windows Desktop | device online, no Connector conflict |
| 5 | Device runtimes | Codex / Claude / DSH / Command Code as expected |
| 6 | Command Code session | model, effort, permission mode, reply, tool call, approval |
| 7 | Restart Desktop | session shows recovery; "continue task" resumes without duplicate history |
| 8 | Android | signs in over Tailscale, sees the device, sends a prompt |
| 9 | Android ↔ Desktop | the same session updates in both directions |
| 10 | Phone off Wi-Fi | Tailscale URL still works; LAN URL does not |

## 7. Troubleshooting

| Symptom | Check / fix |
| --- | --- |
| Phone says "Could not reach the server" | It is using the LAN URL while off the LAN. Use `http://<SERVER_TAILSCALE_IP>:5174`. Verify the phone browser can open `/api/v2/health`. |
| QR sign-in still fails after logout | The QR was generated by the Desktop, which embeds its LAN URL. Generate it from the Web UI at the Tailscale URL, or change the Desktop Server URL first. |
| Phone can ping Tailscale but the app fails | Confirm Android Tailscale VPN is on. Android permits HTTP for self-hosted servers; iOS does not permit raw `100.64/10` HTTP. |
| Device list is empty | The Windows Desktop/Connector is not running. Start it from the tray; wait for online. |
| "Another Connector is running" | Quit the other Desktop/CLI Connector. A stale `~/.agents-anywhere/connector-runtime.json` may name a dead PID; the lease check normally reclaims it. Stop all app processes and retry before editing that file. |
| Codex/Claude missing | They are discovered from `PATH`. Install the CLI, sign in once, restart the Desktop. |
| `commandcode_cli_missing` | Install `command-code`, or set `cliPath` to the bundled `dist/index.mjs` or `%APPDATA%\npm\cmdc.cmd`. |
| `commandcode_authentication_required` | Run `cmdc login` (Command Code CLI; see its docs), or set `COMMAND_CODE_API_KEY` in the Connector environment. |
| `commandcode_acp_disabled` | Remove `acp` from `mods.disabled` in Command Code settings, then restart the runtime. |
| Command Code tile missing | Restart the Desktop after the new build; check the Connector log at `%USERPROFILE%\.agents-anywhere\logs`. The runtime appears only from a Connector that includes the provider. |
| 2.1.0 Android cannot create a session | 2.1.0 lacks the 2.2.0 workflow UI. Install the 2.2.0 APK or use the 2.2.0 Web UI. |
| Server says migration/schema mismatch | Run `compose logs migrate-next`; check `/health/ready`; do not start an older Server against the migrated database. |
| `yarn dist:win` fails with `EBUSY default_app.asar` | Build to `$env:TEMP\aa-dist` with `electron-builder --config.directories.output=...`, or close the file watcher/antivirus scan and retry. |
| Web UI attachment send fails over plain HTTP | `crypto.subtle` is unavailable in a non-secure context. Use Desktop/Android, or enable HTTPS (Tailscale serve). |
| OAuth redirect or share link uses the wrong host | `AGENT_SERVER_PUBLIC_ORIGIN` must match the URL clients use. Change it in `server.env`, restart `server-next`, and re-enter the URL if it changed. |

## 8. Security notes

- Keep `server.env` outside the repository, mode `600`, and never commit `AGENT_SERVER_SECRET`, connector tokens, or `COMMAND_CODE_API_KEY`.
- Do not publish port `5174` to the internet. Tailscale is the remote access path; no router port forwarding is required.
- Use Tailscale ACLs to limit which users/devices can reach the Server.
- Prefer HTTPS (`tailscale serve`) when clients are not on a trusted LAN, and for browser-based attachment uploads.
- The Server stores messages, tool output, and attachments. Treat the Mac's disk and backups as sensitive.
- Rotate the setup token by restarting the Server if it is exposed; setup tokens expire after 15 minutes.

## 9. Related documentation

- [Installation and first use](getting-started.md)
- [Docker deployment](../docker/README.md)
- [Upgrade guide](upgrading.md)
- [Desktop Workbench](../desktop-workbench/README.md)
- [Android build](../android/README.md)
- [Connector](../connector/README.md)
- [DSH Bridge Next](../dsh-bridge-next/README.md)
- [Command Code ACP docs](https://commandcode.ai/docs/acp)
- [Command Code headless docs](https://commandcode.ai/docs/headless)
