# Karmik — Developer Tools

## Overview

This spec covers the tools and agents that make Karmik useful for software development
workflows: git operations, GitHub/GitLab integration, shell command execution, dev
environment inspection, a documentation browser, and a personal code snippet library.

Developer tools are primarily available on desktop platforms (macOS, Windows, Linux) and
on Android with a terminal emulator (Termux). iOS and Web have reduced availability due
to sandboxing.

All tools follow the `Tool` interface from `specs/07-tool-registry.md`.

---

## Git Tools

Git tools call the system `git` binary via `shell.run` internally. They require `git` to
be installed and the working directory to be inside a git repository.

**Platform availability**: macOS, Windows, Linux, Android (with Termux). Not available on
iOS (no shell) or Web.

```
git.status
  → Working tree status
  Input: { "cwd": string }
  Output: { "branch": string, "ahead": int, "behind": int,
            "staged": [{ "path", "status" }],
            "unstaged": [{ "path", "status" }],
            "untracked": string[] }

git.log
  → Commit history
  Input: { "cwd": string, "limit"?: int, "branch"?: string, "since"?: ISO8601 }
  Output: [{ "hash", "shortHash", "author", "date": ISO8601, "subject", "body"?: string }]

git.diff
  → Show diff between working tree, staging, or commits
  Input: { "cwd": string, "staged"?: bool, "from"?: string, "to"?: string,
           "path"?: string, "contextLines"?: int }
  Output: { "diff": string, "filesChanged": int, "insertions": int, "deletions": int }

git.blame
  → Show who last modified each line of a file
  Input: { "cwd": string, "path": string, "lines"?: { "start": int, "end": int } }
  Output: [{ "line": int, "content": string, "hash", "author", "date": ISO8601 }]

git.branch
  → List branches
  Input: { "cwd": string, "all"?: bool }
  Output: { "current": string, "local": string[], "remote"?: string[] }

git.stash_list
  → List stashes
  Input: { "cwd": string }
  Output: [{ "index": int, "message": string, "branch": string, "date": ISO8601 }]

git.commit_message
  → Generate a commit message from the current diff (AI-generated, not a git command)
  Input: { "cwd": string, "style"?: "conventional|short|detailed" }
  Output: { "subject": string, "body"?: string, "type"?: string }
  Note: calls git.diff internally, then runs inference to generate the message

git.commit
  → Stage all changes and create a commit (Co-pilot required)
  Input: { "cwd": string, "message": string, "stageAll"?: bool }
  Output: { "hash": string, "committed": bool }
  Co-pilot: always — shows staged diff + message before committing

git.push
  → Push to remote (Co-pilot required)
  Input: { "cwd": string, "remote"?: string, "branch"?: string, "force"?: bool }
  Output: { "pushed": bool, "remote": string, "branch": string }
  Co-pilot: always; `force: true` additionally shows a stern warning
```

**Permission**: `tools.git.*` off by default. Read tools (`git.status`, `git.log`,
`git.diff`, `git.blame`, `git.branch`, `git.stash_list`) can be granted as a group.
Write tools (`git.commit`, `git.push`) require separate explicit grant.

**Privacy Mode**: `git.*` tools are purely local (call local `git` binary) — all work
in Privacy Mode. Exception: `git.push` calls a remote (blocked if remote is not LAN IP).

---

## GitHub / GitLab Tools

REST API calls to forge services. Credentials (personal access token) stored in
platform keystore. The forge host is auto-detected from `git remote get-url origin`.

```
github.list_issues
  → List issues for a repository
  Input: { "repo"?: string, "state"?: "open|closed|all", "label"?: string, "limit"?: int }
    // "repo": "owner/repo" — auto-detected from git remote if omitted
  Output: [{ "number", "title", "state", "labels": [], "author", "createdAt": ISO8601,
             "url": string }]

github.get_issue
  → Get a single issue with its comments
  Input: { "repo"?: string, "number": int }
  Output: { "number", "title", "body", "state", "labels": [], "comments": [] }

github.create_issue
  → Create a new issue
  Input: { "repo"?: string, "title": string, "body"?: string, "labels"?: string[] }
  Output: { "number": int, "url": string }

github.list_prs
  → List pull requests
  Input: { "repo"?: string, "state"?: "open|closed|all", "limit"?: int }
  Output: [{ "number", "title", "state", "author", "base", "head", "url", "draft": bool }]

github.get_pr
  → Get a PR with its diff and review comments
  Input: { "repo"?: string, "number": int }
  Output: { "number", "title", "body", "state", "diff": string,
            "comments": [], "reviews": [] }

github.comment
  → Post a comment on an issue or PR (Co-pilot required)
  Input: { "repo"?: string, "number": int, "body": string }
  Output: { "id": int, "url": string }
  Co-pilot: always

github.create_pr
  → Create a pull request (Co-pilot required)
  Input: { "repo"?: string, "title": string, "body"?: string,
           "head": string, "base"?: string, "draft"?: bool }
  Output: { "number": int, "url": string }
  Co-pilot: always
```

**GitLab variants**: identical tool shapes under `gitlab.*` namespace. Uses GitLab REST
API v4 (`https://gitlab.com/api/v4` or self-hosted GitLab URL from git remote).

**Bitbucket**: basic read support (`bitbucket.list_prs`, `bitbucket.list_issues`) using
Bitbucket Cloud REST API 2.0.

**Auto-detection**: when `repo` is omitted, Karmik runs `git remote get-url origin` in
the current `cwd` to determine the repo. Supports:
- `https://github.com/owner/repo.git` → github
- `git@github.com:owner/repo.git` → github
- `https://gitlab.com/owner/repo.git` → gitlab

**Privacy Mode**: GitHub/GitLab APIs are public internet — blocked in Privacy Mode.
Self-hosted GitLab on LAN is allowed in LAN-only mode (detected by IP range).

---

## Shell / Terminal Tool

Executes shell commands in a sandboxed subprocess. This is the escape hatch for anything
not covered by a dedicated tool — package managers, build systems, custom scripts, etc.

```
shell.run
  → Execute a shell command
  Input: {
    "command": string,
    "cwd"?: string,          // defaults to user's home directory
    "timeoutSeconds"?: int,  // default 30, max 300
    "env"?: {}               // additional environment variables
  }
  Output: { "stdout": string, "stderr": string, "exitCode": int }

  Platform: macOS (zsh), Linux (bash), Windows (PowerShell), Android/Termux (bash).
            NOT available on iOS or Web.
```

**Sandbox rules** (enforced by command pattern matching before execution):

| Rule | Blocked commands |
|---|---|
| No filesystem wipe | `rm -rf /`, `rm -rf ~`, `rm -rf *` from root or home |
| No privilege escalation | `sudo`, `su`, `doas` |
| No disk operations | `dd`, `mkfs`, `fdisk`, `parted`, `shred` |
| No power/system | `shutdown`, `reboot`, `halt`, `poweroff`, `init 0` |
| No self-modification | `chmod 777 /`, `chown root` |

Blocked patterns are checked via regex before the command is passed to the OS.
Any match → Co-pilot confirmation required + audit log entry with `"blocked_pattern"` field.

**Network access**: not blocked (the shell can run `curl`, `wget`, `npm install`, etc.).
This is intentional — developers need network in their shell. In Privacy Mode, `shell.run`
is restricted to commands that don't produce network calls (enforced by honour-system warning,
not technical block, since arbitrary shell network blocking is impractical).

**Working directory scope**: if `cwd` is not a subdirectory of the user's home directory
or a user-configured project root list, the command is blocked with an error:
"Working directory outside allowed scope."

**Permission**: `tools.shell.run` off by default. Requires explicit grant + a warning:
"Shell access lets agents run arbitrary commands on your device. Only grant this to agents
you fully trust."

---

## Dev Environment Inspector

Inspects what's running on the local machine. Implemented via `shell.run` internally.

```
devenv.ports
  → List open listening ports with process info
  Input: { "filter"?: string }  // filter by process name
  Output: [{ "port": int, "protocol": "tcp|udp", "process": string, "pid": int }]
  Implementation: macOS/Linux: `lsof -iTCP -iUDP -n -P`; Windows: `netstat -ano`

devenv.processes
  → List running processes
  Input: { "name"?: string, "limit"?: int }
  Output: [{ "pid": int, "name": string, "cpu": float, "memoryMb": float }]
  Implementation: macOS/Linux: `ps aux`; Windows: `tasklist`

devenv.docker
  → List running Docker containers
  Input: {}
  Output: [{ "id", "name", "image", "status", "ports": [{ "host", "container" }] }]
  Implementation: `docker ps --format json`; requires Docker CLI installed

devenv.env
  → Read specific environment variables (allowlist-based)
  Input: { "keys": string[] }  // specific variable names to read
  Output: { [key: string]: string }
  Note: cannot read PATH-adjacent secrets (GITHUB_TOKEN, AWS_SECRET_KEY, etc. are blocked
        by a denylist regardless of what "keys" specifies)

devenv.ping
  → Check if a host/port is reachable
  Input: { "host": string, "port"?: int, "timeoutMs"?: int }
  Output: { "reachable": bool, "latencyMs"?: float }
```

**Platform**: macOS, Linux, Windows, Android. Not available on iOS or Web.
**Privacy Mode**: all `devenv.*` tools are local-only — allowed in Privacy Mode.

---

## Documentation Browser

Fetches, chunks, embeds, and stores documentation for semantic search. Agents can retrieve
relevant docs sections as context rather than sending the entire doc to the model.

```
docs.fetch
  → Download and index documentation from a URL or package name
  Input: { "url"?: string, "package"?: string, "version"?: string, "force"?: bool }
    // "package" shortcut: e.g. { "package": "flutter", "version": "3.22" } resolves to
    // the known Flutter docs URL. Known packages: flutter, react, nextjs, fastapi, etc.
  Output: { "id": string, "title": string, "chunksIndexed": int, "sizeKb": int }

docs.search
  → Semantic search across all cached documentation
  Input: { "query": string, "source"?: string, "limit"?: int }
  Output: [{ "id", "source", "title", "chunk": string, "score": float, "url": string }]

docs.list
  → List all cached documentation sources
  Input: {}
  Output: [{ "id", "title", "url", "version"?: string, "chunksIndexed": int,
             "fetchedAt": ISO8601, "sizeKb": int }]

docs.delete
  → Remove a cached documentation source
  Input: { "id": string }
  Output: { "deleted": bool }
```

**Storage**: separate `karmik_docs.db` database (not mixed with conversation or memory data).
Documents are chunked at ~500 tokens with 50-token overlap. Each chunk is embedded with
all-MiniLM-L6-v2 (same embedding model as the memory system). sqlite-vec handles similarity
search.

**Fetch pipeline**:
1. `browser.fetch` retrieves the page (or recursively crawls a docs site up to 500 pages)
2. HTML is stripped to text; navigation/header/footer noise is removed via heuristics
3. Markdown is chunked by section (headings as boundaries)
4. Each chunk is embedded and stored with its source URL and section title
5. Doc source is marked `fetchedAt` for freshness tracking (stale after 30 days)

**Privacy Mode**: `docs.fetch` calls `browser.fetch` (internet) — blocked in Privacy Mode.
`docs.search` operates on already-cached local data — allowed in Privacy Mode.

---

## Code Snippet Library

A personal, semantic-search-enabled library of reusable code snippets.

```
snippets.store
  → Save a code snippet
  Input: { "code": string, "language"?: string, "description": string, "tags"?: string[] }
  Output: { "id": string }

snippets.search
  → Semantic and full-text search over saved snippets
  Input: { "query": string, "language"?: string, "limit"?: int }
  Output: [{ "id", "description", "language", "tags": [], "code": string,
             "score": float, "createdAt": ISO8601 }]

snippets.get
  → Get a snippet by ID
  Input: { "id": string }
  Output: { "id", "description", "language", "tags": [], "code": string }

snippets.update
  → Update a snippet's metadata or code
  Input: { "id": string, "code"?: string, "description"?: string, "tags"?: string[] }
  Output: { "updated": bool }

snippets.delete
  → Delete a snippet
  Input: { "id": string }
  Output: { "deleted": bool }
```

**Storage**: `snippets` table in `karmik.db`. Embeddings generated from `description + "\n\n" + code`
combined text for best semantic retrieval. Also FTS5-indexed for keyword search.

**Privacy**: entirely local. Works in Privacy Mode.

**IDE integration** (see IDE integration section below): snippets can be inserted directly
into VS Code or JetBrains from the IDE extension, backed by this same store.

---

## IDE Integration

### VS Code Extension (`karmik-vscode`)

A VS Code extension that connects to Karmik's local MCP server (`localhost:5173`) and
exposes Karmik capabilities directly inside VS Code.

**Features**:
- **Ask Karmik** (command palette): open a chat with the active agent without leaving VS Code
- **Snippet insert**: `Ctrl+Shift+K S` → search `snippets.*`, insert at cursor
- **Explain selection**: right-click → "Ask Karmik to explain" → sends selection to agent
- **Commit message**: `Ctrl+Shift+K C` → calls `git.commit_message` → pastes result into SCM commit box
- **Memory search**: `Ctrl+Shift+K M` → semantic search over Karmik memories → insert as comment

The extension connects to Karmik via the MCP server when Karmik desktop is running.
Falls back to a "Karmik not running" notice if the MCP server is not available.

### JetBrains Plugin (`karmik-jetbrains`)

Same capabilities as the VS Code extension, implemented as a JetBrains IDE plugin (IntelliJ
platform: Android Studio, IntelliJ IDEA, PyCharm, WebStorm, etc.).

**Features**: identical to VS Code extension. IDE action names:
- **Karmik: Ask** — opens a tool window chat panel
- **Karmik: Insert Snippet** — snippet search + insert
- **Karmik: Generate Commit Message** — calls `git.commit_message`, inserts into commit dialog

Both extensions are open-source and ship in Stage 2. In Stage 1, the clipboard-based
workflow covers the same use cases: user copies code, asks Karmik in the overlay.

---

## New Built-in Agents (Developer)

| Agent | Mode | Default Tools | Purpose |
|---|---|---|---|
| **Git Assistant** | ReAct | `git.*`, `github.*`, `memory.store` | Commit messages, PR descriptions, change summaries, issue triage |
| **Code Reviewer** | Plan-Execute | `git.diff`, `github.get_pr`, `code.run`, `snippets.search` | Multi-file PR review with findings |
| **Doc Helper** | ReAct | `docs.fetch`, `docs.search`, `snippets.store` | Answer questions from cached local docs |
| **Shell Assistant** | ReAct + Co-pilot | `shell.run`, `devenv.*`, `files.*` | System diagnostics, process management, safe shell task execution |
| **API Tester** | ReAct | `http.get`, `http.post`, `memory.store`, `snippets.store` | Natural language → REST API call + schema inference + save as snippet |

---

## Example Developer Flows

### Generate commit message

```
User: "Write a commit message for my staged changes"
        │
Git Assistant (ReAct):
  → git.status { cwd: "/home/user/project" }
    ✓ Branch: feature/smart-home  ·  3 staged files
  → git.diff { cwd: "...", staged: true }
    ✓ +247 -12 lines across 3 files
  → [inference: summarize diff → generate conventional commit message]
  Response: "feat(smart-home): add Shelly adapter and mDNS auto-discovery

  - Implement ShellyAdapter with local REST API client
  - Add mDNS scanner for _shelly._tcp.local discovery
  - Register adapter in SmartHomeAdapterRegistry"

  [Co-pilot card]: Commit with this message? [Approve] [Edit] [Cancel]
```

### Debug a running service

```
User: "My API server isn't responding, what's happening?"
        │
Shell Assistant (ReAct + Co-pilot):
  → devenv.ports {}
    ✓ port 8080: process "node" (pid 4521)
  → devenv.processes { name: "node" }
    ✓ node (pid 4521): 94% CPU, 512MB RAM
  → shell.run { command: "tail -n 50 /tmp/api.log", cwd: "~" }
    ✓ [log output with error trace]
  Response: "Your API server (node, pid 4521) is running but stuck at 94% CPU.
             The log shows it's in an infinite retry loop on the database connection.
             Likely cause: database is down or wrong credentials. Want me to check
             if port 5432 is open?"
```
