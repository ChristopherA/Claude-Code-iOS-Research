# Claude Code iOS App: Configuration, Persistence, and Isolation Research

**Date:** 2026-02-08
**Status:** Initial research — subject to change as Anthropic iterates

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Configuration Layers: Local CLI vs. Cloud/iOS](#configuration-layers)
3. [The iOS App Experience](#the-ios-app-experience)
4. [Memory and Learning Across Sessions](#memory-and-learning)
5. [Persistence Model](#persistence-model)
6. [Context Management](#context-management)
7. [iCloud Sync Considerations](#icloud-sync-considerations)
8. [Isolation: Pros and Cons](#isolation-pros-and-cons)
9. [Practical Recommendations](#practical-recommendations)

---

## 1. Architecture Overview <a name="architecture-overview"></a>

When you use Claude Code from the iOS app, you are **not running code on your phone or Mac**. The iOS app is a thin client that connects to an **ephemeral cloud VM** managed by Anthropic. The architecture is:

```
iOS App (thin client)
    └── claude.ai/code (web interface)
           └── Anthropic Cloud Infrastructure
                  └── Ephemeral Ubuntu 22.04 VM (per task)
                         ├── Cloned GitHub repository
                         ├── Language runtimes & package managers
                         ├── Claude Code agent
                         └── Git proxy (credential isolation)
```

Each coding task spawns a **fresh VM**. When the task completes, the VM is **destroyed** and its filesystem is wiped. There is no persistent disk between sessions.

---

## 2. Configuration Layers: Local CLI vs. Cloud/iOS <a name="configuration-layers"></a>

Claude Code has a layered configuration system. The critical question for iOS users is: **which layers survive into the cloud environment?**

### 2.1 User-Level Config (`~/.claude/`)

On your Mac (local CLI), `~/.claude/` contains:

```
~/.claude/
  settings.json          # Global user settings
  settings.local.json    # Local-only user settings
  CLAUDE.md              # Global instructions (all projects)
  rules/*.md             # User-level rules
  commands/              # Global custom slash commands
  projects/              # Session history + auto memory per project
    <project-hash>/
      memory/
        MEMORY.md        # Auto memory index (first 200 lines → system prompt)
        debugging.md     # Topic-specific memory files
```

**In cloud/iOS sessions: NONE of this is available.** The ephemeral VM has its own `~` directory (`/root/` or `/home/user/`) that starts empty. Your Mac's `~/.claude/` directory does not exist in the cloud environment.

### 2.2 Project-Level Config (`.claude/` in repo root)

```
.claude/
  settings.json          # Team-shared settings (committed to git)
  settings.local.json    # Personal project overrides (gitignored)
  CLAUDE.md              # Project instructions
  rules/*.md             # Modular project rules
  agents/                # Project-specific subagents
```

**In cloud/iOS sessions:**
- `.claude/settings.json` — **Available** (committed to repo, cloned into VM)
- `.claude/CLAUDE.md` — **Available** (committed to repo)
- `.claude/rules/*.md` — **Available** (committed to repo)
- `.claude/settings.local.json` — **NOT available** (gitignored, never committed)
- `.claude/agents/` — **Available if committed** to the repo

### 2.3 Project Root Files

- `CLAUDE.md` (in repo root) — **Available** (committed to repo)
- `CLAUDE.local.md` — **NOT available** (gitignored by convention)

### 2.4 Summary Table

| Config/File | Location | Local CLI | Cloud/iOS |
|---|---|---|---|
| `~/.claude/settings.json` | User home | Yes | **No** |
| `~/.claude/CLAUDE.md` | User home | Yes | **No** |
| `~/.claude/projects/*/memory/` | User home | Yes | **No** |
| `.claude/settings.json` | Repo root | Yes | **Yes** (via git) |
| `.claude/settings.local.json` | Repo root | Yes | **No** (gitignored) |
| `CLAUDE.md` | Repo root | Yes | **Yes** (via git) |
| `CLAUDE.local.md` | Repo root | Yes | **No** (gitignored) |
| `.claude/rules/*.md` | Repo root | Yes | **Yes** (via git) |
| Cloud env variables | Anthropic UI | N/A | **Yes** (configured per-environment) |

### 2.5 Settings Precedence

On local CLI (highest → lowest):
1. Managed settings (`/etc/claude-code/managed-settings.json`)
2. Command-line arguments
3. Local project settings (`.claude/settings.local.json`)
4. Shared project settings (`.claude/settings.json`)
5. User settings (`~/.claude/settings.json`)

On cloud/iOS, layers 2, 3, and 5 are absent. The effective precedence is:
1. Anthropic's cloud defaults
2. Shared project settings (`.claude/settings.json`)
3. Cloud environment variables (configured in the web UI)

---

## 3. The iOS App Experience <a name="the-ios-app-experience"></a>

### 3.1 Starting a Session

When you open Claude Code from the iOS app, you see three options:

1. **Default cloud environment** — Uses Anthropic's default VM configuration
2. **Choose a GitHub repository** — Clone a specific repo into the cloud environment
3. **Another option** (varies — may include recent repos or custom environments)

The "Default cloud environment" is a named configuration you create at `claude.ai/code` that specifies:
- **Environment name** (a label)
- **Network access level**: None, Limited (allowlist of ~100+ domains), or Full
- **Environment variables**: Key-value pairs in `.env` format

You can create multiple cloud environments (e.g., "work-backend", "personal-oss") and set one as default. When you start a task from iOS, it uses whichever environment is set as default unless you explicitly choose another.

### 3.2 GitHub Repository Selection

When you choose a repository:
1. The repo is **cloned fresh** into the ephemeral VM
2. The **default branch** (usually `main`) is checked out
3. If `.claude/settings.json` defines a `SessionStart` hook, it runs (installing deps, etc.)
4. The `CLAUDE.md` and `.claude/rules/*.md` files are loaded into Claude's context

You must have previously:
- Connected your GitHub account via OAuth on `claude.ai`
- Installed the Claude GitHub App on the target repositories

### 3.3 iOS "Projects" in the Claude App

The Claude iOS app has a general concept of "Projects" (not specific to Claude Code) that allows you to:
- Organize conversations by topic or project
- Attach project-level instructions (custom system prompts)
- Store files as project knowledge

**These Claude app "Projects" are separate from and unrelated to Claude Code's `.claude/` configuration.** A Claude app Project is a UI organizational concept for conversations. Claude Code's cloud environment is a VM-based code execution system. They do not share configuration, memory, or state.

If you create a Claude iOS Project called "My App" and also have a GitHub repo "my-app" with `.claude/settings.json`, these two are completely independent systems.

### 3.4 Session Lifecycle on iOS

```
1. Start task  →  Ephemeral VM provisioned
                  Repository cloned
                  SessionStart hooks run
                  CLAUDE.md loaded

2. Execute     →  Claude reads/writes code
                  Runs tests, linters, builds
                  User can steer via iOS interface
                  (Session continues even if phone sleeps/app closes)

3. Complete    →  Changes pushed to new branch on GitHub
                  User reviews diffs
                  User can create PR

4. Terminate   →  VM destroyed
                  Ephemeral storage wiped
                  NO persistent state remains
```

---

## 4. Memory and Learning Across Sessions <a name="memory-and-learning"></a>

### 4.1 Auto Memory (Local CLI Only)

Claude Code's auto memory feature stores learned patterns at:
```
~/.claude/projects/<project-hash>/memory/
  MEMORY.md          # First 200 lines loaded into system prompt
  topic-files.md     # Read on demand
```

Auto memory is:
- Written automatically by Claude as it works ("remember that we use pnpm")
- Persisted on your **local filesystem** across sessions
- Loaded at the start of each new local CLI session
- **NOT available in cloud/iOS sessions** (the `~/.claude/` directory doesn't exist in the VM)

### 4.2 What Persists Across Cloud/iOS Sessions

| Memory Type | Persists Across Cloud Sessions? | Mechanism |
|---|---|---|
| Auto memory (`~/.claude/projects/*/memory/`) | **No** | Local filesystem only |
| `CLAUDE.md` (project root) | **Yes** | Committed to git |
| `.claude/rules/*.md` | **Yes** | Committed to git |
| `~/.claude/CLAUDE.md` (user-level) | **No** | Local filesystem only |
| Conversation history | **No** (per-task only) | Visible in web UI, not carried forward |
| Cloud environment variables | **Yes** | Stored in Anthropic's cloud config |

### 4.3 The Learning Gap

This is the fundamental limitation: **Claude Code on iOS/cloud has no cross-session learning mechanism.**

On local CLI, auto memory creates a feedback loop:
```
Session 1: Claude learns "this project uses pnpm" → writes to MEMORY.md
Session 2: MEMORY.md loaded → Claude already knows to use pnpm
Session 3: MEMORY.md loaded → Claude already knows to use pnpm
```

On iOS/cloud, every session starts from scratch:
```
Session 1: Claude learns "this project uses pnpm" → writes to VM's MEMORY.md → VM destroyed
Session 2: Fresh VM → Claude must rediscover or be told again
Session 3: Fresh VM → Claude must rediscover or be told again
```

### 4.4 Workarounds for Cross-Session Memory on iOS

1. **Commit instructions to `CLAUDE.md`**: Put everything Claude needs to know in the project's `CLAUDE.md` file. This is the most reliable approach since it's committed to git and cloned every time.

2. **Use `.claude/rules/*.md`**: Break instructions into modular rule files. These are also committed to git.

3. **Cloud environment variables**: For secrets or environment-specific config, use the cloud environment's `.env` configuration.

4. **SessionStart hooks**: Define hooks in `.claude/settings.json` that run at the start of every cloud session. These can install dependencies, configure tools, etc.

5. **Feature request (not yet implemented)**: There is a [GitHub issue (#19244)](https://github.com/anthropics/claude-code/issues/19244) requesting a `CLAUDE_HOME` environment variable that would allow mapping `~/.claude` to persistent storage, but this does not exist yet.

---

## 5. Persistence Model <a name="persistence-model"></a>

### 5.1 What Persists Where

```
┌─────────────────────────────────────────────────────────┐
│                    PERSISTENT                            │
│                                                          │
│  GitHub Repository          Anthropic Cloud Config       │
│  ├── CLAUDE.md              ├── Cloud environment name   │
│  ├── .claude/settings.json  ├── Network access level     │
│  ├── .claude/rules/*.md     ├── Environment variables    │
│  ├── .claude/agents/        └── Default env selection    │
│  └── Source code                                         │
│                                                          │
├─────────────────────────────────────────────────────────┤
│                    EPHEMERAL (per task)                   │
│                                                          │
│  Cloud VM Filesystem                                     │
│  ├── ~/.claude/ (empty, not your Mac's)                  │
│  ├── Auto memory (written, then lost)                    │
│  ├── Session history                                     │
│  ├── Installed packages                                  │
│  ├── Build artifacts                                     │
│  └── Any non-committed file changes                      │
│                                                          │
├─────────────────────────────────────────────────────────┤
│                    LOCAL MAC ONLY                         │
│                                                          │
│  ~/.claude/                                              │
│  ├── settings.json (user prefs)                          │
│  ├── CLAUDE.md (user-level instructions)                 │
│  ├── projects/*/memory/ (auto memory)                    │
│  └── commands/ (custom slash commands)                   │
│                                                          │
│  CLAUDE.local.md (per-project, gitignored)               │
│  .claude/settings.local.json (gitignored)                │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Session Transfer

You can move between local and cloud contexts:

- **Cloud → Local ("Teleport")**: Use `/teleport` in your local CLI to pull a cloud session's branch and conversation into your terminal.
- **Local → Cloud**: Use `&` prefix or `claude --remote "task"` to spawn a cloud task from your terminal. This creates a **new** cloud session — it does not migrate your local session.

Both directions create new contexts rather than truly syncing state.

---

## 6. Context Management <a name="context-management"></a>

### 6.1 How Context is Built Per Session

In each cloud/iOS session, Claude's context is assembled from:

1. **System prompt** (Anthropic's base instructions)
2. **`CLAUDE.md`** from the repository root (loaded in full)
3. **`.claude/rules/*.md`** files (loaded in full)
4. **Parent directory `CLAUDE.md` files** (if the repo is in a subdirectory structure)
5. **Cloud environment variables** (available as env vars, not in the prompt)
6. **The user's task description** (what you type in the iOS app)
7. **Conversation history** (within the current session only)

What is **absent** from cloud context that would be present locally:
- `~/.claude/CLAUDE.md` (user-level instructions)
- Auto memory (`MEMORY.md` and topic files)
- `CLAUDE.local.md` (local project notes)
- Previous session history
- MCP server configurations

### 6.2 Context Window Considerations

Claude Code uses automatic summarization when the conversation exceeds the context window. On both local and cloud, long sessions will have earlier messages summarized. The difference is:

- **Local**: Auto memory preserves key learnings even after summarization compresses the conversation
- **Cloud/iOS**: Once summarized, early context is lossy, and there's no memory layer to compensate

---

## 7. iCloud Sync Considerations <a name="icloud-sync-considerations"></a>

### 7.1 Can iCloud Bridge the Gap?

The user's question raises an important point: the iOS app doesn't have access to Mac files, but iCloud syncs between devices. Could this help?

**Short answer: No, not with the current architecture.**

The cloud VM that runs Claude Code tasks does not mount iCloud Drive or any external filesystem. The VM only contains:
- The cloned GitHub repository
- Pre-installed language runtimes and tools
- Whatever `SessionStart` hooks install

Even if you stored your `~/.claude/CLAUDE.md` or `MEMORY.md` in iCloud Drive on your Mac, the cloud VM has no mechanism to access iCloud.

### 7.2 Theoretical iCloud-Based Workarounds

These are speculative and not officially supported:

1. **Symlink `~/.claude/` into an iCloud-synced Git repo**: You could commit your user-level `CLAUDE.md` and memory files to a private "dotfiles" repo. Then configure a `SessionStart` hook that clones that repo and symlinks the files into `~/.claude/` in the cloud VM. This is fragile and unsupported but theoretically possible.

2. **iCloud-synced CLAUDE.md editing**: If you keep your project's `CLAUDE.md` in iCloud Drive and edit it from your iPhone, you'd still need to commit and push it to GitHub for the cloud VM to pick it up. iCloud doesn't help directly — Git is the transport layer.

3. **Future Anthropic feature**: A persistent `~/.claude` volume that Anthropic manages and attaches to every cloud VM would solve this entirely. The `CLAUDE_HOME` feature request (#19244) is the closest existing ask.

### 7.3 What iCloud CAN Help With

- **Editing `CLAUDE.md` files on your phone**: If your repo is cloned locally and `CLAUDE.md` is accessible via iCloud Drive (e.g., your repo is in `~/Documents/` which syncs), you can edit it on your phone, then commit/push from your Mac. The next cloud session will pick up the changes.
- **Reviewing Claude's work**: If Claude pushes changes to a branch, you can review the diff on your phone via GitHub's iOS app or the web.

---

## 8. Isolation: Pros and Cons <a name="isolation-pros-and-cons"></a>

### 8.1 Pros of the Ephemeral VM Model

| Benefit | Details |
|---|---|
| **Security** | Your Git credentials never enter the sandbox. A compromised session cannot access your Mac, other repos, or Anthropic infrastructure. |
| **Reproducibility** | Every session starts from a known clean state (fresh clone + hooks). No accumulated cruft from previous sessions. |
| **Credential isolation** | Git operations go through a proxy that attaches real tokens externally. Even with full network access, the VM cannot exfiltrate credentials. |
| **Blast radius containment** | A destructive command (e.g., `rm -rf /`) only affects the ephemeral VM, not your Mac or other environments. |
| **Parallel safety** | Multiple tasks run in separate VMs with no shared state, eliminating race conditions or cross-contamination. |
| **No local resource consumption** | Your Mac's CPU, RAM, and disk are unaffected. You can run tasks from a phone with no development tools installed. |
| **Always-available environment** | Don't need your dev machine. Start tasks from anywhere — phone, tablet, borrowed computer. |

### 8.2 Cons of the Ephemeral VM Model

| Limitation | Details |
|---|---|
| **No cross-session memory** | Auto memory is lost when the VM is destroyed. Claude cannot learn from previous sessions. Every session starts "amnesiac" unless `CLAUDE.md` compensates. |
| **No user-level config** | `~/.claude/settings.json`, `~/.claude/CLAUDE.md`, and custom commands are absent. You must duplicate essential config into project-level files. |
| **No MCP servers** | Local MCP server configurations don't carry into the cloud environment. If you rely on MCP tools, they won't be available. |
| **No local file access** | Cannot reference files on your Mac, access local databases, run against local services, or test with local hardware. |
| **Dependency reinstallation** | Every session must reinstall dependencies (mitigated by `SessionStart` hooks and package manager caches, but still adds setup time). |
| **GitHub-only** | Only GitHub repositories are supported. GitLab, Bitbucket, self-hosted Git, and local repos cannot be used. |
| **Network restrictions** | Default "Limited" network mode may block access to private registries, internal APIs, or custom domains not on the allowlist. |
| **No iCloud/cloud storage access** | The VM cannot mount iCloud Drive, Google Drive, Dropbox, or any external filesystem. |
| **Conversation isolation** | Each task is a separate conversation. You cannot reference or build on previous task conversations. |
| **Secret management friction** | Secrets must be configured as cloud environment variables through the web UI. No integration with local keychains, 1Password, etc. |

### 8.3 Isolation Comparison Matrix

| Concern | Local CLI | Cloud/iOS | Notes |
|---|---|---|---|
| Can Claude damage my machine? | Possible (sandboxing optional) | **No** (ephemeral VM) | Cloud is safer for untrusted operations |
| Can Claude access other repos? | Yes (filesystem access) | **No** (only cloned repo) | Cloud is more restrictive |
| Can sessions share state? | Yes (via filesystem) | **No** (separate VMs) | Local is more flexible |
| Can Claude learn over time? | Yes (auto memory) | **No** (ephemeral) | Major iOS limitation |
| Can Claude access local services? | Yes (localhost) | **No** (separate network) | Can't test against local APIs |
| Credential exposure risk | Higher (local tokens) | **Lower** (proxy model) | Cloud is safer for credential handling |

---

## 9. Practical Recommendations <a name="practical-recommendations"></a>

### 9.1 Maximize What Travels With the Repo

Since `.claude/settings.json`, `CLAUDE.md`, and `.claude/rules/*.md` are the **only** configuration that persists across cloud sessions, invest heavily in these files:

```
# In your CLAUDE.md:
- Project architecture and conventions
- Key decisions and rationale
- Dependency management instructions (e.g., "use pnpm, not npm")
- Testing commands and expectations
- Code style preferences
- Common gotchas
```

```
# In .claude/rules/:
- coding-standards.md
- testing-conventions.md
- architecture-decisions.md
- deployment-notes.md
```

### 9.2 Use SessionStart Hooks

Define hooks in `.claude/settings.json` to automate cloud environment setup:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "command": "npm install",
        "description": "Install dependencies"
      },
      {
        "command": "cp .env.example .env",
        "description": "Set up environment file"
      }
    ]
  }
}
```

### 9.3 Treat CLAUDE.md as Your "Cloud Memory"

Since auto memory doesn't persist in cloud sessions, manually curate `CLAUDE.md` with the information auto memory would normally capture:

- After a productive local CLI session, review `~/.claude/projects/<project>/memory/MEMORY.md`
- Copy the most valuable entries into your project's `CLAUDE.md`
- Commit and push so cloud sessions benefit from local learnings

### 9.4 Organize Cloud Environments by Use Case

Create separate cloud environments for different needs:
- **"Restricted"**: No network access, for security-sensitive repos
- **"Standard"**: Limited network (allowlist), for most development
- **"Full Access"**: Unrestricted network, for repos that need private registries or unusual dependencies

### 9.5 Use /teleport for Continuity

When a cloud/iOS task produces work you want to continue locally:
1. Let the cloud task push its branch
2. On your Mac, run `/teleport` to pull the session into your local CLI
3. Continue working with full access to auto memory, MCP servers, and local config

---

## Open Questions for Further Research

1. **Will Anthropic add persistent `~/.claude` volumes for cloud sessions?** The `CLAUDE_HOME` feature request (#19244) suggests demand exists.
2. **Can `SessionStart` hooks be used to restore auto memory from a committed file?** (e.g., copy a committed `memory-backup.md` to `~/.claude/projects/*/memory/MEMORY.md`)
3. **How does Claude Cowork (desktop VM) handle this differently?** Cowork uses a persistent local VM rather than ephemeral cloud VMs — does it preserve `~/.claude` across sessions?
4. **Will MCP server support come to cloud environments?** Currently there's no mechanism to connect MCP servers to cloud VMs.
5. **What is the relationship between Claude app "Projects" (conversation organizer) and Claude Code cloud environments?** Could project-level knowledge from Claude Projects feed into Claude Code sessions?

---

## References

- [Claude Code on the web — Official Docs](https://code.claude.com/docs/en/claude-code-on-the-web)
- [Claude Code Settings — Official Docs](https://code.claude.com/docs/en/settings)
- [Manage Claude's Memory — Official Docs](https://code.claude.com/docs/en/memory)
- [Making Claude Code More Secure and Autonomous — Anthropic Engineering](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [Feature Request: CLAUDE_HOME — GitHub Issue #19244](https://github.com/anthropics/claude-code/issues/19244)
- [Using the GitHub Integration — Claude Help Center](https://support.claude.com/en/articles/10167454-using-the-github-integration)
