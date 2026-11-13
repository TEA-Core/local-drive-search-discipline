# local-drive-search-discipline

<p align="center">
  <img alt="local-drive-search-discipline: a robot with a magnifying glass searching a workspace folder map, search the workspace, never walk the whole drive" src="docs/assets/banner.png" width="100%" />
</p>

local-drive-search-discipline is a Paperclip company skill that prevents runaway local-drive searches from hanging agent runs on a Windows Subsystem for Linux 2 (WSL2) host.

## What it is

On a WSL2 host, a content search rooted at `/` walks `/mnt/c` — the entire Windows drive over a slow 9p mount, including application data, browser caches, and virtual environments. It never returns in usable time and stalls the whole run.

The incident behind this skill, on 2026-06-29: a `grep` with an empty path searched from `/`, blocked for about 70 minutes on browser cache files, and the run was killed as stalled.

The skill is loaded before any search or traversal tool runs, so the guard is in place before the first command.

- **Root at the workspace.** Search tools root at `$PAPERCLIP_WORKSPACE_CWD` by default; pass that path explicitly when the working directory is unknown.
- **Never search from `/`, `~`, or an empty path.** An empty or omitted path that resolves to `/` is the bug this skill exists to stop.
- **Never traverse `/mnt/c` broadly.** Reading one explicit `/mnt/c/<full/path/to/file>` is fine; walking `/mnt/c` or any parent of it is not.
- **Go directly to known paths.** When a task names an absolute path, run or read that exact path instead of searching for it in the current workspace.
- **Keep searches cheap.** Exclude heavy directories, cap traversal depth, and cap output so a search does not flood the context.

## Quick start

Load the skill before running any `rg`, `grep`, `ack`, `find`, or `fd` command:

- [Skill](skills/local-drive-search-discipline/SKILL.md) — the hard guards, always loaded.
- [CLI cheatsheet](skills/local-drive-search-discipline/references/cli-cheatsheet.md) — per-tool detail, read on demand.

A correctly scoped search looks like this:

```bash
rg --glob='!node_modules' --glob='!.git' 'PATTERN' "$PAPERCLIP_WORKSPACE_CWD"
```

Reading one explicit file on the Windows drive is also allowed:

```bash
cat /mnt/c/full/path/to/somefile.txt
```

Never do this:

```bash
rg 'PATTERN' /
grep -r 'PATTERN'
```

The second command has no path, so it searches the current directory — fatal when that directory is `/` or above `/mnt`.

## Registration

The skill is registered in the company skill registry. It is bootstrapped through a one-line pointer in each baseline agent's `AGENTS.md`, together with `paperclipSkillSync.desiredSkills`.

## Components

| Component | What it provides |
| --- | --- |
| [`SKILL.md`](skills/local-drive-search-discipline/SKILL.md) | Always-loaded hard guards: workspace rooting, forbidden search roots, broad `/mnt/c` traversal ban, mandatory exclusions and depth caps, a known-paths navigation map, and a quick reference. |
| [`references/cli-cheatsheet.md`](skills/local-drive-search-discipline/references/cli-cheatsheet.md) | Token-aware per-tool best practices for `rg`, `grep`, `ack`, `fd`, `find`, `ls`, `du`, `jq`, `awk`, and `sed`, plus the `/home` versus `/mnt/c` cost boundary and 9p performance detail. |

## Hard rules

1. Root searches at `$PAPERCLIP_WORKSPACE_CWD`. Give an explicit, bounded path when a tool would otherwise guess.
2. Never search from `/`, `~`, or an empty path.
3. Never traverse `/mnt/c` broadly. One explicit file read is the only exception.
4. Treat evidence given inline in a task as inline. Do not search the filesystem for it.
5. Always exclude heavy directories (`node_modules`, `.git`, caches) and depth-cap broad walks (`find -maxdepth 4`, `fd --max-depth 4`). These help, but they do not replace rule 3: `/mnt/c` itself is the slow tree.
6. Prefer `rg` over `grep -r`. It respects `.gitignore`, is faster, and runs in parallel.

The full rules, the known-paths map, and worked good/bad examples are in [`SKILL.md`](skills/local-drive-search-discipline/SKILL.md).

## Documentation

- [Skill](skills/local-drive-search-discipline/SKILL.md)
- [CLI cheatsheet](skills/local-drive-search-discipline/references/cli-cheatsheet.md)
- [Banner prompt](docs/assets/banner-prompt.txt)

## Roadmap

Shipped:

- [x] Initial skill: hard guards that scope searches to the workspace and forbid `/` and broad `/mnt/c` traversal, with mandatory heavy-directory exclusions and depth caps
- [x] Token-aware CLI cheatsheet for search, traversal, and data tools
- [x] Known-paths navigation map, so a handed absolute path is run directly instead of searched for in the wrong workspace

Planned:

- [ ] No planned items are currently recorded in this repository. Add new work here when it is planned, with evidence from an issue, plan, or merged proposal.

## Contributing

Keep changes to the skill and its cheatsheet factual and scoped: every rule should trace to observed host behaviour or a reproduced hang.

When a pull request changes project status, tick or add the matching [roadmap](#roadmap) item in that same pull request.
