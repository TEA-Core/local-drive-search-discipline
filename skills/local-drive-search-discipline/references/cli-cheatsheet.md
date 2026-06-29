# Local-drive CLI cheatsheet (WSL2 host)

Token-aware best-practices for search/traversal/data CLI tools on this host.
Read this when you need per-tool detail; the hard rules live in SKILL.md.

## The filesystem boundary (why this matters)

| Path | What it is | Search cost |
|---|---|---|
| `$PAPERCLIP_WORKSPACE_CWD` | the agent's repo workspace | cheap — search here |
| `/home/kronik/...` | native WSL ext4 | fast |
| `/mnt/c/...` | **Windows C: drive over 9p** | **pathologically slow** — every stat is an RPC |
| `/` | includes `/mnt/c` | fatal for any broad search |

A broad search rooted at `/` is forced to walk `/mnt/c` — AppData, Chrome
`CacheStorage`, ComfyUI venvs, hundreds of thousands of files. It does not
return in any usable time. Always root searches in the workspace.

## Token awareness (don't blow up your own context)

- Cap output: `rg -m 50`, `... | head -n 100`. A search that prints 10k lines
  costs you context AND money.
- Prefer filenames first (`rg -l`, `fd`) then read the few that matter.
- `rg --json` only when you parse it; the human-readable form is cheaper to scan.
- Never `cat` a whole node_modules file / minified bundle / lockfile into context.

## Search tools

### rg (ripgrep) — preferred
```bash
rg 'PATTERN' "$PAPERCLIP_WORKSPACE_CWD"        # scoped
rg -l 'PATTERN' "$PAPERCLIP_WORKSPACE_CWD"     # filenames only (cheap)
rg --glob='!node_modules' --glob='!.git' --glob='!*cache*' 'PATTERN' <path>
rg -m 50 'PATTERN' <path>                       # cap matches
```
- Respects `.gitignore` by default (good). `--hidden` disables that — use with care, it pulls in `.git`.
- NEVER `rg 'PATTERN' /` or `rg --hidden 'PATTERN' .` when cwd is `/` or above `/mnt`.

### grep
```bash
grep -rn --exclude-dir={node_modules,.git,.cache} 'PATTERN' <path>
```
- `grep -r` with NO path searches cwd — confirm cwd is the repo first (`pwd`).
- Slower than rg; use rg unless grep-specific behavior is needed.

### ack
```bash
ack 'PATTERN' "$PAPERCLIP_WORKSPACE_CWD"
```
- Skips VCS dirs by default; still pass an explicit path — never bare `ack PATTERN` from `/`.

## Traversal tools

### fd (preferred over find)
```bash
fd --max-depth 4 'name' "$PAPERCLIP_WORKSPACE_CWD"
fd -e ts --max-depth 6 . "$PAPERCLIP_WORKSPACE_CWD"
```

### find
```bash
find "$PAPERCLIP_WORKSPACE_CWD" -maxdepth 4 -name '*.ts'    # ALWAYS -maxdepth
```
- A bare `find /` or `find .` (cwd unknown) walks `/mnt/c` — same hang. Always
  give an explicit path AND `-maxdepth`.

### ls / du
```bash
ls -la <path>                       # fine, single dir
du -sh <path>                       # OK on workspace; NEVER `du -sh /` (walks /mnt/c)
timeout 8 du -sh <path>             # cap if unsure of size
```

## Data tools (operate on FILES/STDIN, not the filesystem)

These don't traverse — but mind their input size.
```bash
jq '.field' file.json                          # not on a 500MB file blindly
jq -c '.[]' big.json | head                     # stream + cap
awk 'NR<=100' file                              # cap rows
sed -n '1,50p' file                             # windowed read, not whole file
```
- Piping a huge file through jq/awk/sed is a token + memory risk; slice first.

## Golden rule

If a search/traversal command does not name an explicit path under the
workspace (or `/home/kronik`), stop and add one. The default is never `/`.
