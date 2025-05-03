# aki-mcp-sv (`@akinet/akimcp`)

Turn Claude on the web, ChatGPT, Grok, and Postman into secure operators for your local machine. AKIMCP v3 exposes a governed suite of 38 tools for files, shell, search, Git, SQLite, browser automation, DevTools, background tasks, localhost services, clipboard, notifications, ports, Postman control, and AIObox windows through one OAuth-gated MCP endpoint. *(Gemini support remains experimental.)*

One command opens a much larger operating surface: build and edit projects from the browser, inspect databases and local APIs, drive browser workflows, manage long-running jobs, debug through DevTools, and control Postman without giving every client unrestricted shell access.

<img width="1672" height="941" alt="AKIMCP v2 control panel" src="public/img/akimcp-v2.jpg" />

[![Version](https://img.shields.io/badge/version-3.0.1-blue.svg)](CHANGELOG.md) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) [![npm version](https://img.shields.io/npm/v/@akinet/akimcp.svg)](https://www.npmjs.com/package/@akinet/akimcp) [![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey.svg)](#install--run)

**Contents:** [Why this exists](#why-this-exists) · [When to use & Core Use-Cases](#when-to-use--core-use-cases) · [Install & Run](#install--run) · [Connecting from Claude web](#connecting-from-claude-web) · [Connecting from ChatGPT](#connecting-from-chatgpt) · [Connecting from Grok and Gemini](#connecting-from-grok-and-gemini) · [Connecting from Postman](#connecting-from-postman) · [Connecting local IDEs](#connecting-local-ides-cursor-claude-code-agy-codex) · [Autonomous Cloud Automation](#autonomous-cloud-automation-grok--local-mcp) · [Requirements](#requirements) · [Architecture](#architecture) · [Directory layout](#directory-layout) · [Configuration](#configuration) · [Exposing to the internet](#exposing-to-the-internet) · [Finding files](#finding-files) · [Security](#security)

## Why this exists

Claude.ai's web/Pro quota is far cheaper than paying per token via the API for equivalent usage. But most real work is project work: reading, editing, and running commands against files on your machine, not open-ended chat.

The Claude Desktop app already does local file access, but ties usage to a device ID you don't control, and running multiple accounts means repeated login/logout. With this web-based approach, you get true multi-account flexibility instead: just switch browser profiles to pick up a different account (e.g. several Claude Pro subscriptions), all pointed at the same local machine, no device lock-in.

**aki-mcp-sv** routes around both problems: run an MCP server on your machine, expose it over HTTPS through Tailscale Funnel, and connect it to claude.ai as a custom connector.

**The payoff:**
- **Local-first by default:** local tools (Cursor, Claude Code, AGY, Postman) connect straight to `127.0.0.1:9999` — zero latency and fully offline; a public ingress is only needed for remote/web AI.
- **Use your web quota** for local file and shell access, straight from the browser.
- **True multi-account flexibility:** switch browser profiles to instantly pick up a different account, all pointed at the same machine.
- **Safe by default:** a strict command whitelist, not a leaky blocklist — see [Security](#security).

### When to use & Core Use-Cases

- **At your desk:** a native Terminal/CLI (Claude Code, Antigravity CLI, Cursor) is still the fastest, most fluid option — use that.
- **Away from your desk (mobile / web / a machine that isn't yours):** use `aki-mcp-sv` via Claude Web, ChatGPT Mobile, or Grok to check on a running job, read logs, clean up temp files, or pull the latest code on your home/office machine.
- **On a schedule, with nobody watching:** pair Grok's scheduled prompts with `aki-mcp-sv` for cloud-triggered local execution — see [Autonomous Cloud Automation](#autonomous-cloud-automation-grok--local-mcp).

## Install & Run

> [!NOTE]
> **Is this safe to run?** Nothing is installed system-wide outside user directory, no background daemon is created, and no `sudo`/administrator privileges are required. Settings and tokens live safely at `~/.aki/mcpsv/`. The shell tool is whitelist-only (see [Security](#security)). Closing the terminal window stops the server.

### 1-Line Global Install (Recommended)

```bash
npm install -g @akinet/akimcp
akimcp
```

Or run instantly without installing:

```bash
npx @akinet/akimcp
```

### From Source (For Contributors)

```bash
git clone https://github.com/lacvietanh/aki-mcp-sv.git
cd aki-mcp-sv
npm install

# Run in isolated development mode (~/.aki/mcpsv-dev, ports 9997/9996)
npm run dev

# Or run in standard production mode (~/.aki/mcpsv, ports 9999/9998)
npm start
```

## Run

```bash
cp .env.example .env   # optional: only if you need PUBLIC_ORIGIN or another non-default var
npm start
```

Nothing needs preparing beforehand; `npm start` handles it:
- **OAuth and passphrase state** in `~/.aki/mcpsv/`: generated once and reused on later runs.
- **Funnel**: checks `tailscale funnel status`; if port `9999` isn't on yet, runs `tailscale funnel --bg 9999` (idempotent: never toggles an already-enabled port).
- Prints the **Remote MCP server URL** and **Passphrase** used on the confirmation page when a client connects.
- Opens the **control panel** at `http://127.0.0.1:9998/?t=<token>`. A step header maps the flow (0 Ingress · 1 Connectors · 2 Install rules · 3 Instructions · 4 Extension · 7 Security), then the sections follow it: 0 Ingress (optional; a 3-tab ingress picker: Tailscale + Funnel / Owned public origin / Hosted domain), 1 Connectors, 2 Install akidevrule, 3 Instructions prompt, 4 Browser utilities, 5 allowed Folders, 6 shell allowlist, 7 Security & connection limits (rate limit settings, blocked callers and Release, registered clients with Remove, callers active since start, the security log), 8 Tool providers (each provider's tools, whether its app is installed, an on/off switch, Detect again).

The default allowed root is your **home directory** (`$HOME`, or `%USERPROFILE%` on Windows): the one folder guaranteed to exist on any machine and to hold the projects you actually want Claude to reach. In plain terms, that means the whole home folder (Desktop, Documents, Downloads, Photos, everything under it), not just the projects you meant to share. Add/remove folders from **panel section 5**: click "+ Add folder…" and type an absolute path (`/Users/you/projects` or `C:\Users\you\projects`). Saving takes effect immediately for every tool — shell, find, search, and file read/write/edit alike — no restart. To change the root from the start: `MCP_DATA_DIR=/other/path npm start` (or `set MCP_DATA_DIR=D:\work` then `npm start` on Windows cmd).

Beyond `$MCP_DATA_DIR`, the filesystem tools are also granted `~/.aki` (where akidevrule deploys) and `~/.claude`, so claude.ai can read your **native** `CLAUDE.md` and skill router the same way Claude Code does, with no copying or staging.

`~/.claude` is granted at the folder level (the filesystem tools can't scope to individual files), so `.claude.json`/`auth-cache.json` (session tokens) and `history.jsonl` (chat history) inside it are also reachable through the connector. This row is locked in panel section 5, with no delete button by design so it can't be revoked by accident; the panel itself cannot remove it. If you don't want `~/.claude` granted at all, edit `~/.aki/mcpsv/setting.json` and remove the `~/.claude` entry from its `folders` list before connecting; claude.ai then loses access to your `CLAUDE.md` too.

`npm start` runs in the foreground: Ctrl+C to stop, restart manually when needed. **After editing code, Ctrl+C and `npm start` again** (Node doesn't hot-reload).

## Connecting from Claude web

1. Go to **claude.ai → Settings → Connectors → Add custom connector**
2. Enter any **Name**, then paste the **Remote MCP server URL** printed by `npm start`
3. Click **Connect** and enter the **Passphrase** when AKIMCP opens its confirmation page

Claude discovers OAuth automatically. No Client ID or Client Secret is needed.

Why not token-in-URL: `docs/ref/claude-connector.md`, `docs/research/claude-ai-oauth-connector.md`.

claude.ai connects and calls the in-house `aki__*` tool suite (38 tools). Tools of an app that is not installed (agy, Kiro CLI, Postman, Chrome, AIObox) are not served, and panel section 8 turns any optional provider on or off (agy and Kiro CLI start off); the client sees the change on its next tool list, usually a new chat. Every tool declares MCP annotations, so ChatGPT stops asking to confirm read-only tools:
- **Chromium Remote & Profiles**: `aki__chrome_profiles`, `aki__chrome_launch`, `aki__chrome_tabs`, `aki__chrome_interact`, `aki__chrome_stop` (opens the shared profile clones AIObox keeps in `~/.aki/cdp/profiles/` on a stealth port-0 launch, or attaches when AIObox already runs the profile; stop kills only a Chrome it launched; React/Vue synthetic typing, scroll-to-center click)
- **DevTools & CDP**: `aki__devtools_targets`, `aki__devtools_eval`, `aki__devtools_screenshot`
- **OS Native Integration**: `aki__notify_user` (desktop notification banner & chime sound), `aki__clipboard_read`, `aki__clipboard_write` (system clipboard read/write bridge)
- **Localhost & Intranet Fetch**: `aki__local_fetch` (SSRF-protected HTTP client for local backend APIs and LAN services)
- **Background Tasks**: `aki__task_start`, `aki__task_manage` (detached background execution, process-group teardown, zero-RAM direct log streaming & tailing)
- **Filesystem**: `aki__read_text_file`, `aki__write_file`, `aki__edit_file`, `aki__create_directory`, `aki__move_file`, `aki__get_file_info`, `aki__list_allowed_directories`
- **Search & Execution**: `aki__find_path`, `aki__search_content`, `aki__run_cmd`
- **Dev Servers & Ports**: `aki__port_status`, `aki__kill_port`
- **Git (read-only)**: `aki__git` with `op` = `status` | `diff` | `log` | `tags` (local, or a remote's via `ls-remote --tags`); compact output, cheaper than `run_cmd`. Writes go through `run_cmd`
- **SQLite Database**: `aki__sqlite_schema`, `aki__sqlite_query`
- **Agent & Context**: `aki__agy_run`, `aki__kiro_read`, `aki__akidevrule_context`
- **Postman Control**: `aki__postman_status`, `aki__postman_eval`, `aki__postman_rename_conversation`, `aki__postman_panel_fullwidth`
- **AIObox windows**: `aki__aiobox` (read) and `aki__aiobox_write` (act); a window is named by its handle such as `P7·W2`, its chat id or its targetId, and `op` is a free string: the ops are whatever AIObox publishes, so call `op=state` first, it returns AIObox's guide and op table

**Note on the connector icon:** claude.ai doesn't read the icon from the MCP server. It queries Google's favicon service with the tailnet's **apex domain**, not your host: `https://t2.gstatic.com/faviconV2?...&url=http://<tailnet>.ts.net&size=32`. `<tailnet>.ts.net` has no public DNS record, so Google returns 404 and claude.ai falls back to a default letter icon. This server serves `/favicon.ico` publicly, but no file placed here can change that result: your subdomain never appears in the query Google receives.

## Connecting from ChatGPT

Needs ChatGPT Plus/Pro (or Business/Enterprise/Edu) with **Developer mode** for custom connectors.

1. ChatGPT → Settings → Security and login → enable **Developer mode**
2. Create a custom connector / app → paste the same MCP URL (`https://your-machine.your-tailnet.ts.net/mcp`)
3. Enter the same **passphrase** on the confirmation page

ChatGPT self-registers via DCR (RFC 7591, PKCE, no secret) from `/.well-known/openid-configuration`. It uses the same folder allowlist and shell allowlist as Claude.

## Connecting from Grok and Gemini

Both ride the same MCP URL and passphrase flow — no separate transport or auth. They differ in *how* the client authenticates: the connector panel (section 1) lists the steps for each, and the Gemini Client ID/Secret are printed by `akimcp` at start.

**Grok — verified, production-ready:** **self-registers** via the `/register` DCR path like ChatGPT — paste only the MCP URL, no Client ID. Its real `redirect_uri` `https://grok.com/connectors-oauth-exchange-code/` was observed live 2026-08-09 and is allowlisted via `GROK_CALLBACK_PREFIX`. Verified working end to end (`authorize → token` 200). If a future Grok change moves that callback, a rejected registration logs `register REJECTED (redirect_uri not allowlisted): [...]` so the new value can be re-allowlisted.

**Gemini — experimental, connection works but tool use doesn't (yet)** (paid tiers only — Pro / Business / Enterprise; the free tier may not expose custom apps): pastes a **confidential client** (the pre-issued one in `oauth-client.json`) — set the custom app link to the MCP URL, then under Advanced Settings paste the Client ID / Client secret that `akimcp` prints at start. Gemini's redirect goes through Google's OAuth proxy `https://oauth-redirect.googleusercontent.com/r/...` (observed live 2026-08-09), allowlisted by `isAllowedRedirect` in `scripts/oauth.js`. **Caveat:** the OAuth handshake succeeds and Gemini accepts the instruction, but in repeated testing 2026-08-09 it did not reliably discover or drive the MCP tools — connection healthy, tool use unreliable. Claude and Grok are the dependable clients today.

## Connecting from Notion AI

Notion needs a public ingress and workspace permission to use **custom MCP servers**. The panel's Notion tab walks through the setup.

1. Enable **custom MCP servers** in your Notion workspace settings.
2. Open [Notion connections](https://www.notion.so/my-connections) and add an approved **Custom MCP server** with the **MCP URL** from the panel.
3. Enter the **Passphrase** when AKIMCP opens its confirmation page.
4. Add the connection inside an **agent**, then **republish the agent** so the published agent can use it.

Notion self-registers through OAuth, so there is no Client ID or Client secret to paste, and it refreshes on its own after Roll token. It uses the same tool suite and folder and shell allowlists as the other connectors. Protocol details and revocation: [docs/feat/security.md](docs/feat/security.md) § Remote auth.

## Connecting from Postman

Postman's AI Agent (Flows / Connected Accounts) has no OAuth redirect for third-party MCP servers and no persistent system-prompt field.

1. In the panel's Postman tab, click the filled JSON to copy. It targets the local loopback endpoint (`http://127.0.0.1:9999/mcp`) and carries a real minted access token — not the passphrase. Postman runs on this machine, so it connects with zero latency and needs no tunnel.
2. In Postman, add a new MCP server (Settings → Connected Accounts) and paste the JSON.
3. Send the instruction into each new chat, since Postman doesn't persist one across sessions: after **Launch**, the **Aki MCP for Postman** overlay's *Prompt instruction* box has **Send now**, or tick *Auto-inject into each new chat* (off by default). It is a separate, Postman-specific text (`scripts/postman/prompts/postman.md`), worded as a polite request because Postman's safety check flags directive-style instructions.

The Postman tab also has a **Launch** button that attaches control to the Postman desktop app itself — auto-clicking Approve/Continue/Run/Try again and toggling Thinking/Auto-run inside the Postman window, on top of opening it if it isn't already running. **Quit** stops that control daemon; **New window** asks it to open another Postman window. None of this runs at `npm start` boot — it starts only when Launch is clicked. The in-app overlay it injects is the **Aki MCP for Postman** panel (opened from a status-bar button): it shows the running version and an `akimcp.top` link under the title, keeps the **New Browser Tab** control in the **ANTI-BOT** section, and opens every external link — `akimcp.top`, the AkiDevRule **Repo** button, and each team's **View** — in your OS default browser through Postman's own link handler.

## Connecting local IDEs (Cursor, Claude Code, AGY, Codex)

Local tools run on the same machine as AKIMCP, so they connect **straight to the loopback engine** at `http://127.0.0.1:9999/mcp` — no tunnel, no internet, zero WAN round-trip, and they keep working fully offline. The Gatekeeper binds `127.0.0.1:9999` from the moment you run `akimcp`, whether or not a public ingress is configured. Bearer-token auth is still enforced (see [Security](#security)); grab the token from the panel at `http://127.0.0.1:9998` (the Postman tab shows the filled JSON; one access token serves every client, and panel section 1 can roll it if it leaks), or copy a config below and replace `YOUR_LOCAL_ACCESS_TOKEN`. (Ports are defaults; `--dev` mode uses `9997`/`9996` — the panel always shows the live values.)

**Cursor** — `~/.cursor/mcp.json` (or Settings → MCP Servers):

```json
{
  "mcpServers": {
    "aki-mcp": {
      "url": "http://127.0.0.1:9999/mcp",
      "headers": { "Authorization": "Bearer YOUR_LOCAL_ACCESS_TOKEN" }
    }
  }
}
```

**Claude Code CLI** — one line:

```bash
claude mcp add --transport http aki-mcp http://127.0.0.1:9999/mcp --header "Authorization: Bearer YOUR_LOCAL_ACCESS_TOKEN"
```

**Antigravity (AGY) CLI and IDE** — no token: they spawn `scripts/stdio.js` (the same tools, over stdio). Click **Apply to AGY CLI** in the panel's AGY tab, or merge this into `~/.gemini/config/mcp_config.json` (the file both read) and restart:

```json
{
  "mcpServers": {
    "akimcp": {
      "command": "node",
      "args": ["/absolute/path/to/aki-mcp-sv/scripts/stdio.js"]
    }
  }
}
```

The button also pre-allows the server for the CLI; the IDE asks for its own approval on the first tool call. Details: [`docs/ref/fact-agy-mcp-config.md`](docs/ref/fact-agy-mcp-config.md).

Any other client that spawns MCP servers over stdio can run the package the same way, with no gateway, tunnel or token: `npx -y @akinet/akimcp --stdio`.

**Codex CLI** — append to `~/.codex/config.toml` (don't overwrite; Codex reaches the local engine over streamable HTTP, token inlined so there's no env var to export):

```toml
[mcp_servers.aki-mcp]
url = "http://127.0.0.1:9999/mcp"
http_headers = { "Authorization" = "Bearer YOUR_LOCAL_ACCESS_TOKEN" }
```

**Postman Desktop** uses the same loopback URL — see [Connecting from Postman](#connecting-from-postman).

> Use the literal `127.0.0.1`, not `localhost`: on macOS `localhost` can resolve to IPv6 `::1` while the server listens on IPv4 only.

## Autonomous Cloud Automation (Grok + Local MCP)

Grok's scheduled prompts turn your machine into a headless "personal remote AI node": no browser tab, no desktop app, just `npm start` running in the background.

- **Cloud-triggered local execution:** set up a scheduled prompt in Grok (Automation) that fires at a fixed time.
- **Headless:** Grok's cloud service sends the request to `/mcp` over your Tailscale Funnel URL, and `aki-mcp-sv` runs the task — health check, log sweep, `git pull`, cleanup — with nothing open on your end.
- **Zero UI required:** as long as the process is running, no browser or app needs to be open for the scheduled task to execute.

## Requirements

- Node.js 22, on Windows, Linux, or macOS.
- **Windows only:** install [Git for Windows](https://git-scm.com/download/win) with its default options, nothing else to set up. The shell/search tools start Unix binaries (`ls cat grep head tail wc file stat ps df du whoami uname`) that ship in Git's `usr\bin`; akimcp finds that folder from `git` and adds it to its own `PATH` at start. `npm` and `npx` run through Node directly, since their `.cmd` shims cannot be started without a shell. Same category of prerequisite as Tailscale below, not a code dependency.
- Tailscale (one-time setup):
  1. [Install Tailscale](https://tailscale.com/download) and sign in (on macOS, the app or `brew install tailscale` both work as long as `tailscale` is on PATH)
  