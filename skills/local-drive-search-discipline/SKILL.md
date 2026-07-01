---
name: local-drive-search-discipline
description: Safe local-drive search/CLI on the WSL2 host. Load BEFORE any rg/grep/ack/find/fd. Hard rule — never search from / or broadly across /mnt/c (the Windows drive over a slow 9p mount); scope to the workspace. Prevents multi-minute search hangs that stall the run.
---

# local-drive-search-discipline

Stop runaway filesystem searches that hang the agent. On this host a content
search rooted at `/` walks `/mnt/c` (the entire Windows drive over a slow 9p
mount) and never returns — it stalls the whole run. Real incident: a `grep` with
an empty path searched from `/`, blocked ~70 min on Chrome cache files under
`/mnt/c`, and the Paperclip run was killed as stalled.

## Hard rules (worst offenders — never break)

1. **Root at the workspace.** Search tools (`rg`, `grep`, `ack`, `find`, `fd`)
   root at `$PAPERCLIP_WORKSPACE_CWD` (the repo) by default. Pass that path
   explicitly if the tool would otherwise default to cwd and cwd is unknown.
2. **Never search from `/`, `~`, or an empty path.** An empty/omitted path arg
   that resolves to `/` is the bug. Always give a real, bounded path.
3. **Never traverse `/mnt/c` broadly.** `/mnt/c` is the Windows drive on a slow
   9p mount — hundreds of thousands of files (AppData, Chrome caches, venvs).
   A broad search there never returns. Reading ONE explicit
   `/mnt/c/<full/path/to/file>` is fine; walking `/mnt/c` or any parent of it is not.
4. **Evidence in a task/issue is INLINE, not on disk.** If a prompt gives you a
   bundle, read the prompt — do NOT search the filesystem for the issue id.

## Mandatory hygiene (layer 2 — always, even when scoped)

- Exclude heavy dirs: `--glob='!node_modules'  --glob='!.git'  --glob='!*cache*'`
  (rg/fd) or `--exclude-dir={node_modules,.git}` (grep).
- Depth-cap broad walks: `find <path> -maxdepth 4 ...`, `fd --max-depth 4`.
- Prefer `rg` over `grep -r` (respects .gitignore, faster, parallel).
- These help but DO NOT replace rule 3 — exclusions alone did NOT stop the real
  hang, because `/mnt/c` itself is the slow tree, not node_modules.

## Known paths — go directly, don't hunt

The #1 cause of a goose-chase is landing in one repo and searching for a file
that lives in another. Your `$PAPERCLIP_WORKSPACE_CWD` is only ONE project — do
NOT assume everything is under it. If a task names an absolute path, **run/read
that exact path directly** — do not `rg`/`find`/`ls` around to "locate" it first.

Canonical host locations (WSL2, user `kronik`):

| What | Path |
|---|---|
| Main project repos (TSP, Modular-Alpha/Mooncatcher, etc.) | `/home/kronik/scratch/<repo>` |
| Trading-Signal-Platform runtime repo | `/home/kronik/scratch/Trading-Signal-Platform` |
| Paperclip agent Python tools (fire scripts, self-improve, audit CLI) | `/home/kronik/scratch/paperclip-agent-tools` |
| — its self-improve + audit code | `/home/kronik/scratch/paperclip-agent-tools/python/agent-workflow-audit` |
| An agent's own instructions / AGENTS.md | `~/.paperclip/instances/default/companies/<companyId>/agents/<agentId>/instructions/` |
| Second-Brain Obsidian vaults (TSP, Mooncatcher planning) | `/mnt/c/Users/david/Documents/projects/<project>/` (alias: `/mnt/c/Users/david/My Documents/projects/`) |
| shared-brain vault (credentials, connections, runbooks) | `/mnt/c/Users/david/Documents/shared-brain/` |

Rules for using this map:
- If you were handed an **absolute path** (e.g. `bash /home/kronik/scratch/paperclip-agent-tools/.../run_self_improve.sh`), **run it as given.** It is correct even if your `cwd` is a different repo. Do NOT search for it.
- A tool in `paperclip-agent-tools` is NOT under a project's runtime repo — the routine may spawn you in a project workspace (e.g. `/home/kronik/scratch/Trading-Signal-Platform`) while the tool lives in `paperclip-agent-tools`. That is expected; use the absolute path, don't hunt.
- Vault paths under `/mnt/c/...` are for reading **one explicit file** (the allowed escape hatch, rule 3) — never `rg`/`find` across `/mnt/c` (slow 9p, hangs the run).

## Quick reference

```bash
# GOOD — scoped to the repo, heavy dirs excluded
rg --glob='!node_modules' --glob='!.git' 'PATTERN' "$PAPERCLIP_WORKSPACE_CWD"

# GOOD — read one explicit Windows file (allowed escape hatch)
cat /mnt/c/Users/david/somefile.txt

# BAD — roots at / , walks /mnt/c, hangs the run
rg 'PATTERN' /
grep -r 'PATTERN'            # empty path -> cwd or / depending on tool
rg --hidden 'PATTERN' .      # if cwd is / or above /mnt, fatal
```

Per-tool best-practices (rg/grep/ack/fd/find/ls/du/jq/awk/sed), the `/home` vs
`/mnt/c` boundary, and 9p perf detail: read `references/cli-cheatsheet.md`.
