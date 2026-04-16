# claude-leak-audit

See how much of your filesystem an AI agent can reconstruct from your Claude Code chat histories.

```
$ claude-leak-audit

=== Claude Leak Audit ===

[*] Found 1083 chat history files
[*] Total messages: 232122
[*] Extracting paths (10 parallel jt queries)...
[*] Unique paths extracted: 36038
[*] Scanning for secrets and PII...

  Secrets & PII
  ─────────────────────────────────────
  sk-* API keys              36
  sk-ant-* keys               0
  Bearer tokens              66
  GitHub tokens               0
  AWS access keys             0
  KEY=value                  12
  TOKEN=value                 0
  SECRET=value                0

  Email addresses            51 unique
  Public IPs                112 unique
  .env files               8022 refs

  Results
  ─────────────────────────────────────
  target:       /home/ok/inference
  depth:        3 levels
  sessions:     1083
  messages:     232122

  real paths:   467
  extracted:    1141
  matches:      380

  RECALL:       81.4%  (380 of 467 real paths recovered)
  PRECISION:    33.3%  (380 of 1141 extracted paths valid)
  missed:       87 paths not in histories
  stale:        761 paths no longer on disk
```

## What it does

1. Finds all `.claude` chat history JSONL files on your machine
2. Runs 10 parallel [jt](https://github.com/okaris/jsont) queries to extract every file path from tool calls, tool results, user messages, bash commands, subagent content, and compaction summaries
3. Scans for leaked secrets (API keys, Bearer tokens, etc.) and PII (emails, IPs)
4. Reconstructs a directory tree from the extracted paths
5. Compares it against your real filesystem and reports recall/precision

## Install

```bash
# 1. Install jt (required)
curl -fsSL i.jsont.sh | sh

# 2. Install claude-leak-audit
curl -fsSL https://raw.githubusercontent.com/okaris/claude-leak-audit/main/claude-leak-audit -o /usr/local/bin/claude-leak-audit
chmod +x /usr/local/bin/claude-leak-audit
```

## Usage

```bash
# Full scan — auto-detects all .claude projects, scans secrets, compares tree
claude-leak-audit

# Target a specific project
claude-leak-audit --claude-dir ~/.claude/projects/-home-ok-myproject

# Compare against a specific directory at depth 4
claude-leak-audit --target-dir /home/ok/myproject --level 4

# Show the reconstructed tree
claude-leak-audit --show-tree

# Show what exists on disk but wasn't found in histories
claude-leak-audit --show-missed

# Show ghost paths (deleted files still visible in history)
claude-leak-audit --show-extra

# Skip secrets scan
claude-leak-audit --no-secrets
```

## How it works

Claude Code stores conversation history as JSONL files in `~/.claude/projects/`. Every tool call, tool result, and user message is logged with full content.

The audit extracts paths from 10 data sources in parallel using [jt](https://github.com/okaris/jsont):

| Source | What it catches |
|--------|----------------|
| Read/Edit/Write inputs | Exact file paths the agent accessed |
| Glob/Grep inputs | Search paths |
| Bash commands | Paths in shell commands |
| Tool results | `ls` output, `grep` matches, file contents sent back to the API |
| User messages | Paths you typed or pasted |
| cwd field | Working directory on every message |
| Compact summaries | Full session history after context compaction |
| toolUseResult | Denormalized tool result content |
| Subagent tool results | Nested agent tool outputs |
| Subagent message content | Tool calls and results from spawned agents |

The secrets scan uses parallel grep for pattern matching (API keys, tokens, emails, IPs, .env references).

## What the numbers mean

- **Recall** — what % of your real directory tree (at the given depth) can be reconstructed from chat histories alone
- **Precision** — what % of extracted paths still exist on disk (low precision = many deleted/moved files visible in history)
- **Stale paths** — files that were deleted or moved but are still visible in the history. This is arguably worse than current paths — it's a temporal map of your filesystem

## Requirements

- [jt](https://github.com/okaris/jsont) (jsont) — `curl -fsSL i.jsont.sh | sh`
- bash, grep, awk, sort, comm (standard unix tools)

## License

MIT
