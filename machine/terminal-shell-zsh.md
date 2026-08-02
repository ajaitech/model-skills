# Terminal / zsh Shell

Model: interactive + scripted zsh on Darwin. Config load order: `/etc/zshenv` → `~/.zshenv` → (login) `/etc/zprofile` → `~/.zprofile` → (interactive) `/etc/zshrc` → `~/.zshrc` → (login) `/etc/zlogin` → `~/.zlogin`.

## Functions vs aliases
- Alias: static text substitution, no arg logic — `alias gs='git status'`.
- Function: use whenever args, conditionals, or multiple commands are needed:
```
myfunc() { local x="$1"; echo "$x"; }
```
Prefer functions over aliases for anything beyond a literal command swap — aliases can't reorder or validate arguments.

## PATH hygiene
| Task | Command |
|---|---|
| Print PATH one entry per line | `echo $PATH \| tr ':' '\n'` |
| Detect DUPLICATE PATH entries | `echo $PATH \| tr ':' '\n' \| sort \| uniq -d` |
| Auto-dedupe live (put before PATH exports in `~/.zshrc`) | `typeset -U path` |
| Every binary that resolves for a name, in PATH order | `which -a <cmd>` |
| What zsh will actually run | `type <cmd>` |
Duplicate entries slow every lookup and cause "wrong version ran" bugs (two `python3`, two `node`). Fix at the source: find which rc file appended the dupe (`grep -n PATH ~/.zshrc ~/.zprofile ~/.zshenv`) rather than patching downstream.

## Quoting & command-injection safety
- Always double-quote variable expansions: `"$var"`, `"${arr[@]}"`. Unquoted expansion is the #1 source of word-splitting and injection bugs.
- NEVER interpolate untrusted input directly into a string passed to `sh -c`, `eval`, or backticks. Untrusted = anything from a network response, a file the user didn't author, or another process's output.
- Prefer arrays + `"$@"` passthrough over building a command string and re-splitting it.
- Avoid `eval`; if unavoidable, quote every substitution inside it.
- For filenames that may contain spaces/newlines, use `-print0`/`-0` (see find/xargs below), not raw `$(...)` word-splitting.

## Process control
| Task | Command |
|---|---|
| Find PIDs by name/pattern | `pgrep -fl <pattern>` |
| Kill by pattern | `pkill -f <pattern>` |
| List background jobs in this shell | `jobs -l` |
| Detach a process, survive logout | `nohup <cmd> > out.log 2>&1 &` |
| Foreground/background a job | `fg %1` / `bg %1` |
| Check a specific PID is alive | `kill -0 <pid> 2>/dev/null && echo alive` |
Always `pgrep` before `pkill` to confirm the pattern only matches intended processes — a loose pattern kills unrelated jobs.

## Safe find/xargs
| Task | Command |
|---|---|
| Find + null-safe xargs | `find . -name '*.log' -print0 \| xargs -0 rm` |
| Find + exec per file (safest, no re-splitting) | `find . -name '*.tmp' -exec rm {} \;` |
| Bounded search root | `find . -regex '...'` — never `find /` |
- With `find -regex` alternation, put the longest alternative first: `.*\.\(tsx\|ts\)` not `.*\.\(ts\|tsx\)` — the shorter pattern matches first and silently truncates results.
