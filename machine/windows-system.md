# Windows System (cross-platform delivery notes)

Model: this machine is macOS/Darwin + zsh. Use this file only when the TASK targets Windows — a build script, CI runner on `windows-latest`, deployment doc, or code shipped to Windows users. macOS-side commands live in `macos-system.md`.

## PowerShell equivalents (bash → PowerShell)
| bash | PowerShell |
|---|---|
| `ls -la` | `Get-ChildItem -Force` (`gci -Force`) |
| `cat file` | `Get-Content file` (`gc`) |
| `grep pattern file` | `Select-String -Pattern pattern -Path file` |
| `find . -name '*.log'` | `Get-ChildItem -Recurse -Filter *.log` |
| `rm -rf dir` | `Remove-Item -Recurse -Force dir` |
| `export VAR=val` | `$env:VAR = 'val'` (session-only) or `setx VAR val` (persistent, new shells only) |
| `which cmd` | `Get-Command cmd` |
| `ps aux \| grep name` | `Get-Process \| Where-Object {$_.Name -like '*name*'}` |
| `kill -9 pid` | `Stop-Process -Id pid -Force` |
| `curl url` | `Invoke-WebRequest url` (`iwr`), or native `curl.exe` on modern Windows |
| exit-code check | `$LASTEXITCODE` (external cmds) vs `$?` (bool, cmdlets) |
Verify cmdlet names/flags via `Get-Help <cmdlet> -Online` — surfaces shift between PowerShell 5.1 (Windows built-in) and PowerShell 7+ (`pwsh`, cross-platform); don't assume parity between the two.

## Path pitfalls
- Separator: `\` native, but most Win32 APIs and PowerShell accept `/` too. Never hardcode `\` in cross-platform scripts — use the language's path-join (`path.join`, `os.path.join`, `Join-Path`).
- Drive letters and UNC paths (`\\server\share`) have no POSIX equivalent — code assuming a single-rooted `/` filesystem breaks.
- Legacy 260-char `MAX_PATH` limit applies unless the target opts into long-path support (app manifest or `LongPathsEnabled` registry key) — deep `node_modules` trees are a common trigger.
- NTFS is case-insensitive-but-preserving by default (as is macOS's default volume) — code relying on case-sensitive filename collisions can behave differently across Windows, macOS, and case-sensitive Linux CI. Don't assume any two of the three match.

## Line endings
- Git on Windows defaults to CRLF checkout / LF commit, governed by `core.autocrlf`. Mixed-ending repos break diffs and shebang-sensitive scripts.
- Set explicit policy via `.gitattributes` (`* text=auto`, pin specific extensions `eol=lf`) rather than relying on each machine's local `core.autocrlf` — team configs otherwise diverge.
- `.sh` scripts run on Windows via WSL/Git-Bash MUST be LF — a CRLF shebang line fails with a cryptic "command not found".

## Cross-platform scripting notes
- Prefer Node, Python, or PowerShell-Core (`pwsh`, runs on macOS/Linux too) for scripts that must run identically everywhere — avoid bash-only syntax if `windows-latest` CI is in the matrix without WSL/Git-Bash.
- Env-var syntax differs even within Windows alone: `cmd.exe` `%VAR%` vs PowerShell `$env:VAR` vs `.env`-file loaders — state explicitly which shell a snippet targets.
