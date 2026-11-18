# poe_mcp_suite — Guide for AI Agents

This file is the vendor-neutral entry point for any MCP-compatible AI agent
(Cursor, Windsurf, Cline, Codex, Zed, Gemini CLI, custom agents, …). It routes
you to the shared, plain-Markdown resources in this repo. If you are **Claude
Code**, read [`CLAUDE.md`](CLAUDE.md) instead/as well — it adds Claude-specific
mechanics (auto-triggering skills, a SessionStart hook, memory) layered on top of
everything below.

## What this is

A collection of MCP servers that give an agent deep, live integration with Path
of Exile — build theory-crafting and passive-tree simulation via Path of
Building, trade/economy lookups, wiki and crafting data, and loot filters. Each
server runs independently; the tools are agent-agnostic by construction (standard
MCP). See [`README.md`](README.md) for the server/tool overview and
[`INSTALL.md`](INSTALL.md) for setup (submodules, dependencies, `.mcp.json`).

The current league name is in `.mcp.json` as `POE_LEAGUE`; pass it explicitly on
tool calls rather than trusting auto-detection when leagues run concurrently.

## How to work in this repo

The suite uses a lightweight workflow framework. It is described once, in
[`playbooks/README.md`](playbooks/README.md) — the shared meta-framework
(context management, league pre-flight, trust hierarchy, narration norms). Read
it before a substantial task, then the specific playbook that matches. **The
playbooks are ordinary Markdown — any agent can read and follow them directly**,
even without skill/rule auto-loading. Key ideas:

- **Cursory-vs-detailed gate** — decide up front whether a request is a quick
  lookup (answer directly) or a multi-source/multi-stage analysis (engage the
  matching playbook and pause for approval before pulling data).
  [`playbooks/README.md`](playbooks/README.md) §1.
- **Freshness discipline** — the game changes every patch, and model training
  data lags. For any patch-specific fact (mods, node stats, unique stats,
  prices, league specifics), prefer fetching authoritative data via the tools
  over answering from memory. Model-aware staleness index:
  [`reference_data/freshness_index.md`](reference_data/freshness_index.md).
- **Cached game data** lives in [`reference_data/`](reference_data/) (gitignored
  but its `README.md` is committed — read it on a fresh clone). Check it before
  re-fetching; respect the `fetched:`/`patch:`/`league:` frontmatter.
- **Per-character analyses** live in [`character_data/`](character_data/) (see
  its `README.md`). Read a character's file before working on it; append
  findings as you go.

## Non-negotiables

- **Legal boundaries** — before extracting, modifying, or redistributing any
  Path of Exile game data, read
  [`legal_considerations.md`](legal_considerations.md). Stay inside the
  conservative redistribution boundary it documents; if unsure, ask the user.
- **GGG trade API** — the official trade API carries Terms-of-Service risk.
  Brief the user and get explicit consent before using trade tools (see
  `legal_considerations.md` → "GGG Terms of Service").
- **Secrets** — `.mcp.json` and `POE_SESSION_ID` are sensitive (session cookie /
  account access). Never log or commit them.
- **`403 Forbidden` on stash/character tools means a dead `POESESSID`, not a code
  bug.** Confirm with one non-API request before touching any code:
  `curl -s -o /dev/null -w "%{http_code}" https://www.pathofexile.com/my-account -H "Cookie: POESESSID=<id>"`
  → `401` = cookie dead (replace it), `200` = look elsewhere (account name,
  rate limit). The tell: poe.ninja pricing keeps working while GGG calls fail.
  See [`ISSUES.md`](ISSUES.md) → "Resolved — stash API access".
- **The live config is `~/.config/opencode/opencode.json`.** OpenCode spawns
  these MCP servers only from there; the repo's `.mcp.json` is for other agents
  and docs, and OpenCode ignores it. Editing only `.mcp.json` changes nothing
  for a running OpenCode session. Env changes (league, session id, account name)
  take effect only after the server process restarts.

## Working against the live PoB API (pob-mcp in TCP mode)

The PoB side of pob-mcp is Lua copied into the app directory and loaded **once at
startup**. Almost every mistake here therefore looks like "the tool hangs" or
"the build is wrong", never like a code error. Check these before debugging code:

- **Editing `PathOfBuilding/src/API/*.lua` does nothing by itself.** Copy it in
  (`pob-mcp\InstallTcpApi.ps1`, or just relaunch via `LaunchPoBWithAPI.bat`) **and
  restart PoB** — the Lua is read at launch only.
- **Launch PoB via `pob-mcp\LaunchPoBWithAPI.bat`**, which sets `POB_API_TCP=1`
  (port 59166) and re-applies the `Modules/Main.lua` patch. A double-clicked exe
  has no API server at all. A PoB update can wipe the patch; the batch self-heals.
- **"Nothing is listening on 59166" is usually a misdiagnosis.**
  `Get-NetTCPConnection -State Listen -LocalPort 59166` returns nothing even when
  the API is up, and `netstat` shows only `TIME_WAIT` after a restart. Probe with a
  raw `TcpClient` and read the banner (`{"ok":true,"ready":true,...}`).
- **Two PoB processes can run at once**, and only the one started with
  `POB_API_TCP=1` owns the port. Also `CloseMainWindow()` can free the port while
  the process stays alive — re-check, then `Stop-Process -Force` (PoB autosaves;
  a killed PoB loses nothing, but confirm the build file's mtime first).
- **`POB_RECONNECT_TIMEOUT_MS` is how long the client retries silently.** 300000
  makes a dead API look like a five-minute hang; it is 15000 in `opencode.json`.
- **Never call two pob tools concurrently.** The bridge answers
  `Concurrent request not supported`; awaited sequentially is fine, `Promise.all`
  is not.
- **New MCP tools appear only after the server process restarts.** Editing the
  `pob` block in `~/.config/opencode/opencode.json` makes OpenCode reload it —
  the tool count in the catalog is the signal that it happened.
- **Never hand-edit a build XML.** A UTF-8 BOM makes PoB load a blank level-1
  build (write with `UTF8Encoding($false)`); use `save_build` / `open_build_xml`
  instead, and pass `path` so the GUI autosaves to the right file.
- **Measure stats through a loadout, not per-set.** PoB derives loadouts from
  matching titles across tree spec / item set / skill set / config set; mixing
  families yields plausible but wrong numbers. See [`ISSUES.md`](ISSUES.md) →
  "Resolved — PoB loadouts".

## Contributing back

New or improved playbooks, reference data, and pitfalls are valuable to everyone
— see [`playbooks/README.md`](playbooks/README.md) for format conventions and
submission guidelines.

**Committing and pushing.** Submodules pin commit SHAs, and `pob-mcp`,
`poe-data-mcp` and `PathOfBuilding` are checked out in **detached HEAD**: after
committing inside one, move a branch (`git branch -f main <sha>`, `api-stdio` for
PathOfBuilding) or the commit is reachable only through the root gitlink. Push the
submodule **before** the root — a root commit pinning a SHA no remote has breaks
`git clone --recurse-submodules`. `charleslucas/*` rejects pushes from this
account (403); `.gitmodules` points at the `yungKillaGit` forks, each submodule
keeps `upstream` on charleslucas for update checks, and forks are synced with
`gh repo sync yungKillaGit/<repo> -b <branch>`.
