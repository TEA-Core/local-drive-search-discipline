# local-drive-search-discipline

A Paperclip company skill that prevents runaway local-drive searches from hanging agent runs on the WSL2 host.

## Why

On the sectoid WSL2 host, a content search rooted at `/` walks `/mnt/c` — the entire Windows drive over a slow 9p mount (AppData, Chrome caches, venvs). It never returns and stalls the Paperclip run. Real incident (2026-06-29): the Executive Reporter ran `grep` with an empty path, searched from `/`, blocked ~70 min on Chrome cache files, and the run was killed as stalled.

## What

- `skills/local-drive-search-discipline/SKILL.md` — caveman-ultra hard guards (always loaded): root searches at `$PAPERCLIP_WORKSPACE_CWD`; never search from `/`, `~`, empty path, or broadly across `/mnt/c`; mandatory heavy-dir exclusions + depth caps.
- `skills/local-drive-search-discipline/references/cli-cheatsheet.md` — token-aware per-tool best-practices (rg/grep/ack/fd/find/ls/du/jq/awk/sed), read on demand.

## Registration

Registered in the TSP company skill registry and bootstrapped via a one-line pointer in each baseline agent's AGENTS.md plus `paperclipSkillSync.desiredSkills`.

Authored from a root-cause investigation, 2026-06-29.
