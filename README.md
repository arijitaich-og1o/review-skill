# reviewer-pro

A **god-mode Python code reviewer** custom agent for GitHub Copilot in VS Code. Goes far beyond syntax and linting — it traces every function end-to-end through its real execution pipeline to find the bugs that only show up in production.

## What it does

Seven review dimensions, every time:

| Dimension | What it catches |
|---|---|
| **Runtime / pipeline failures** | Unhandled exceptions, `None` propagation, `KeyError`/`AttributeError` on real inputs, off-by-one, mutable defaults |
| **Security & data integrity** | SQL/command injection, path traversal, `eval`/`pickle`/`yaml.load`, secrets in logs, TOCTOU races |
| **Concurrency & race conditions** | Shared state without locks, check-then-act races, un-awaited `asyncio` tasks, deadlock ordering |
| **Performance & scale** | O(n²) loops, N+1 queries, unbounded memory growth, missing pagination |
| **Blocking calls** | `requests` inside `async def`, `time.sleep` in async code, blocking I/O on hot paths |
| **Dead code** | Unreachable branches, always-true/false conditions, unused imports and parameters |
| **Duplication** | Copy-pasted blocks that have drifted — where the copies have already diverged is often a real bug |

Output is always a **single severity-ranked report** (🔴 Critical → 🟠 High → 🟡 Medium → 🔵 Low → ⚪ Nit) with pipeline context, concrete trigger conditions, and minimal fixes.

## Installation

### Option 1 — Workspace (team-shared, per-project)

Copy `.github/agents/reviewer-pro.agent.md` into your project's `.github/agents/` folder:

```bash
# From your project root
mkdir -p .github/agents
curl -o .github/agents/reviewer-pro.agent.md \
  https://raw.githubusercontent.com/arijitaich-og1o/review-skill/main/.github/agents/reviewer-pro.agent.md
```

Commit and push — everyone on the team gets the agent automatically.

### Option 2 — User profile (personal, all workspaces)

Copy the file to your VS Code user prompts folder so it's available everywhere:

| OS | Path |
|---|---|
| Windows | `%APPDATA%\Code\User\prompts\reviewer-pro.agent.md` |
| macOS | `~/Library/Application Support/Code/User/prompts/reviewer-pro.agent.md` |
| Linux | `~/.config/Code/User/prompts/reviewer-pro.agent.md` |

```bash
# macOS / Linux
curl -o "$HOME/Library/Application Support/Code/User/prompts/reviewer-pro.agent.md" \
  https://raw.githubusercontent.com/arijitaich-og1o/review-skill/main/.github/agents/reviewer-pro.agent.md
```

```powershell
# Windows (PowerShell)
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/arijitaich-og1o/review-skill/main/.github/agents/reviewer-pro.agent.md" `
  -OutFile "$env:APPDATA\Code\User\prompts\reviewer-pro.agent.md"
```

### Option 3 — Clone and symlink

```bash
git clone https://github.com/arijitaich-og1o/review-skill.git
# Then copy or symlink .github/agents/reviewer-pro.agent.md wherever needed
```

## Requirements

- VS Code **1.99+**
- **GitHub Copilot** extension (any plan)
- Agent mode enabled (the agent file is auto-discovered — no configuration needed)

## Usage

1. Open the Copilot Chat panel (`Ctrl+Alt+I` / `Cmd+Alt+I`)
2. Click the agent picker (the `@` icon or agent dropdown) and select **reviewer-pro**, or type `@reviewer-pro` in chat
3. Paste or attach your code and ask for a review:

```
@reviewer-pro review this
```

```
@reviewer-pro what could break in production?
```

```
@reviewer-pro deep review of auth.py
```

The agent activates automatically on phrases like "review this", "deep review", "god review", "what could fail", and "how do I harden this".

## Example output

```
# GOD Code Review: auth.py

**Verdict:** Not safe to ship — the session token is logged at DEBUG level and the
password reset flow has a TOCTOU race that allows account takeover.

**Pipeline summary:** Request enters `login()`, credentials are validated against
the DB, a session token is minted and returned. The token is also passed to the
logger before being returned to the caller, leaking it into log aggregators.
The reset flow checks token validity and then invalidates it in two separate
DB calls with no transaction, creating a replay window.

---

## 🔴 Critical
### [C1] Session token leaked to logs — `login()`, line ~42
**Pipeline:** Token is created in `_mint_token()`, passed to `logger.debug()`
before being returned — any log aggregator (Datadog, Splunk, CloudWatch)
stores it in plaintext.
**Problem:** `logger.debug("token=%s", token)` logs the live session secret.
**Trigger:** Any DEBUG-level logging in staging or production.
**Fix:** Remove the log line, or log only the first 8 chars: `token[:8] + "…"`.
...
```

## Project structure

```
.
├── .github/
│   └── agents/
│       └── reviewer-pro.agent.md   # The custom agent definition
└── README.md
```

## License

MIT
