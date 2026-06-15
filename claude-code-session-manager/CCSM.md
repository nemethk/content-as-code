# Claude Code Session Manager: Search and Resume Past Conversations Fast

> *You had that brilliant debugging session last Tuesday. Claude walked you through the whole thing. Now you want to pick it up again — but you can't find it.*

I use Claude Code every day. The sessions accumulate fast: debugging a production issue, drafting a refactor, exploring a new API, writing tests. Each conversation is saved locally. Each one is potentially resumable.

The problem is that "locally" means a directory full of UUID-named files with no index, no search, and no way to know what's inside without opening them one by one.

That friction motivated me to build `ccsm` — **Claude Code Session Manager**.

---

## The Problem Is Structural

When Claude Code saves a session, it writes a `.jsonl` file into a deeply nested directory:

```
~/.claude/projects/
  -home-user-Dev-myapp/
    cc928331-e2fa-4542-992e-f1fded2deb08.jsonl
    2803b936-7cb5-499f-804a-3804351f4b94.jsonl
  -home-user-Dev-api-service/
    a1b2c3d4-...jsonl
```

The folder name is a slug derived from your project path. The filename is a UUID. There are no human-readable labels anywhere.

Claude Code has a `--resume` flag that can pick up any session right where you left off:

```bash
claude --resume 2803b936-7cb5-499f-804a-3804351f4b94
```

It's a powerful feature. But to use it, you have to already know the UUID. And that's exactly what you don't have when you're trying to find an old conversation.

---

## What's Actually Inside a Session File

Each `.jsonl` file is a structured log of the conversation — one JSON object per line. Every user message, every assistant response, every tool call. The data is all there.

```json
{"type":"user","timestamp":"2026-06-14T09:23:11Z","cwd":"/home/user/Dev/myapp","message":{"role":"user","content":"fix the auth bug in the login flow"}}
{"type":"assistant","timestamp":"2026-06-14T09:23:14Z","message":{"role":"assistant","content":[...]}}
```

The first user message is effectively the session title. The `cwd` field identifies which project it was for. The timestamps record when it happened.

All the metadata needed to build a useful index is sitting right there — it just needs to be read.

---

## Building the Index

`ccsm` is a Go CLI that walks `~/.claude/projects/`, parses each `.jsonl` file, and extracts the useful metadata: UUID, project path, timestamps, first message, referenced files, and message count.

The result is a clean, sortable list:

```bash
ccsm list

DATE        TIME   UUID                                  PROJECT              SUMMARY
2026-06-14  09:23  2803b936-7cb5-499f-804a-3804351f4b94  ~/Dev/myapp          fix the auth bug in the login flow
2026-06-13  14:51  cc928331-e2fa-4542-992e-f1fded2deb08  ~/Dev/myapp          refactor the payment service
2026-06-12  11:30  a1b2c3d4-0f8e-4a12-bb23-9c8d7e6f5a4b  ~/Dev/api-service    add rate limiting middleware
```

From chaos to context in one command.

`ccsm` earns its keep once sessions accumulate across projects. If you're still finding everything by scrolling, you don't need it yet. Once you've lost a session you wanted to resume, or started a conversation that duplicated one you'd already had, you do.

---

## Five Commands

### `ccsm list` — What Have I Been Working On?

```bash
ccsm list                          # all sessions, newest first
ccsm list --project myapp          # filter by project path substring
ccsm list --since 2026-06-01       # only sessions from this date forward
ccsm list --min-turns 2            # hide aborted starts and claude -p calls
ccsm list --ai                     # AI-generated one-liner per session (cached)
ccsm list --json                   # machine-readable output for scripting
```

The `--project` filter is a substring match, so `ccsm list --project api` catches anything with "api" in the path.

The SUMMARY column starts as a cleaned excerpt of your first message with key file references. You can upgrade it to an AI-generated one-liner:

```bash
ccsm list --ai
```

Running it shows progress as each session is summarized, then displays the results. Summaries are stored in `~/.cache/ccsm/` — run `--ai` once, and every future `ccsm list` (no flag needed) shows them instantly at no cost.

The `--json` flag outputs the full session record for each entry — UUID, project path, timestamps, first message, turn count, and a `files` array of filesystem paths extracted from the conversation. That last field is useful for filtering by file:

```bash
ccsm list --json | jq '[.[] | select(.files[]? | contains("migrations"))]'  # sessions that touched migration files
```

### `ccsm search` — Find That Specific Conversation

```bash
ccsm search "auth bug"
ccsm search postgres
ccsm search kubernetes --json
```

Search scans **every user message** across all turns, not just the first one. It returns the matching snippet and turn number so you can immediately see why each session matched:

```
DATE        TIME   UUID      PROJECT       TURN  MATCH
2026-06-03  19:41  74437e33  ~/Dev/config  #97   create a LICENSE file
2026-06-14  17:12  2803b936  ~/Dev/config  #11   ...href="LICENSE"><img src=...
```

The MATCH column makes false positives obvious at a glance — you can see instantly whether "LICENSE" appeared in an actual message or inside a pasted README badge URL.

### `ccsm show` — Preview Before Resuming

```bash
ccsm show 2803b936                 # UUID prefix is enough (like git)
ccsm show 2803b936 --turns 10      # show more turns (default: 5)
```

This prints the first few typed messages from that session so you can confirm it's the right one before resuming. UUID prefix matching means you only need to type the first 8 characters — the same pattern git uses for commit hashes.

### `ccsm summarize` — Understand What Was Actually Accomplished

```bash
ccsm summarize 2803b936
```

This generates a detailed, structured summary of the session using Claude:

```
Session:  2803b936-7cb5-499f-804a-3804351f4b94
Project:  ~/Dev/myapp
Date:     2026-06-14 09:23
Turns:    43

**What was worked on:** Fixed authentication bug in the login flow and added
session token refresh logic.

**Key steps:**
- Identified missing token expiry check in auth middleware
- Added refresh endpoint with sliding window logic
- Updated tests to cover the new token refresh path

**Outcome:** Auth bug fixed and merged; token refresh deployed to staging.
```

`ccsm show` gives you the raw conversation. `ccsm summarize` gives you the structured outcome. Both use UUID prefix matching — you only need to type enough characters to be unique.

### `ccsm upgrade` — Stay Current

```bash
sudo ccsm upgrade
```

Downloads and installs the latest release from GitHub. Skips the download if you're already on the latest version.

---

## Resuming a Session

Once you have a UUID, resuming is one command:

```bash
claude --resume 2803b936-7cb5-499f-804a-3804351f4b94
```

Claude picks up exactly where you left off — full conversation history, previous context intact.

There's also a cost argument. Starting fresh on an existing problem means re-explaining context: the codebase structure, constraints, what you already tried. That costs tokens. A resumed session has all of that established already — and because Claude Code uses prompt caching, those historical turns are served from cache at a fraction of regular input token cost. Fewer tokens, faster responses, same continuity.

**With fzf**, you can make this a single interactive workflow:

```bash
ccsm list | fzf | awk '{print $3}' | xargs claude --resume
```

Fuzzy-search your sessions, select one, and resume — all in one pipeline. I added this as a shell alias:

```bash
alias cr='ccsm list | fzf | awk '"'"'{print $3}'"'"' | xargs claude --resume'
```

Now `cr` is all I type to browse and resume any session.

---

## Shell Completion

`ccsm` supports tab completion for all commands and flags. Add one line to your shell config:

```bash
# bash — add to ~/.bashrc
source <(ccsm completion bash)

# zsh — add to ~/.zshrc
source <(ccsm completion zsh)
```

After reloading your shell, `ccsm <Tab>` suggests commands and flags.

---

## The `/sessions` Skill: Claude Reasoning on Top of the Binary

`ccsm` ships with a Claude Code skill that adds natural language reasoning on top of the raw data.

Install it once — from a cloned repo:

```bash
cp skill/sessions.md ~/.claude/skills/sessions.md
```

Or download it directly:

```bash
curl -fsSL https://raw.githubusercontent.com/nemethk/claude-code-session-manager/main/skill/sessions.md \
  -o ~/.claude/skills/sessions.md
```

Then use it inside any Claude Code conversation:

```
/sessions                              → numbered list of all sessions
/sessions find postgres migration      → Claude filters by relevance, explains why
/sessions show 2803b936                → inspect the turns before resuming
/sessions resume 2803b936              → prints the claude --resume command
```

The skill calls `ccsm list --json` via the Bash tool and uses Claude to reason over the structured output. This means semantic search without indexing — you can describe a session in natural language and Claude will match it against what you've been working on.

The architecture is a clean split: the binary does the fast, reliable data work; the skill does the reasoning.

---

## A Few Technical Notes

**Why Go?** Fast startup, single binary, no runtime dependencies. The tool runs on every `ccsm list` invocation — startup time matters.

**JSONL parsing edge case:** Some tool output lines in session files are very large (tens of KB of JSON). The default Go scanner buffer (64KB) overflows them. `ccsm` uses a 4MB buffer to handle this without silently dropping lines.

**Content extraction:** User messages come in two shapes in the JSONL — plain strings and typed block arrays. The parser handles both and skips `tool_result` blocks, which are responses from tools rather than things the user typed. Only extractable messages count toward the turn count.

**UUID prefix matching:** Like `git log --abbrev-commit`, `ccsm` lets you reference sessions by any unique prefix of their UUID. The full 36-character UUID is only needed for `claude --resume` itself.

**AI summaries without noise:** `ccsm list --ai` and `ccsm summarize` call `claude --no-session-persistence` under the hood. Without this flag, each AI call would create a new session in your history — polluting the very list you're trying to read.

**Summary cache:** One-line summaries are stored in `~/.cache/ccsm/<uuid>.txt`. They're generated once and reused on every subsequent `ccsm list`. Detailed summaries from `ccsm summarize` are printed on demand and not cached — they're meant for deep dives, not the overview.

**Custom sessions directory:** If your Claude sessions live somewhere other than `~/.claude/projects`, set `CCSM_SESSIONS_DIR` to override the default.

---

## Installation

**macOS / Linux:**

```bash
curl -fsSL https://raw.githubusercontent.com/nemethk/claude-code-session-manager/main/scripts/install.sh | bash
```

**Homebrew:**

```bash
brew install nemethk/tap/ccsm
```

**Go:**

```bash
GOBIN=/usr/local/bin go install github.com/nemethk/claude-code-session-manager@latest
```

---

## The Bigger Idea

Claude Code stores a lot of useful data locally. Session files, project contexts, conversation histories — it's all on disk, structured, and readable.

The tool ecosystem around that data is still young. `ccsm` is one small piece: a read-only index over your session files. It doesn't modify anything, doesn't require API access, and doesn't depend on Anthropic infrastructure. It just reads what's already there and makes it navigable.

If you use Claude Code regularly, your session history is a personal knowledge base of every problem you've solved with Claude's help. The only question is whether you can find it when you need it.

`ccsm` makes that possible.

---

*Source and installation: [github.com/nemethk/claude-code-session-manager](https://github.com/nemethk/claude-code-session-manager)*
