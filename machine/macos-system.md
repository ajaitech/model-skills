# macOS System — Darwin

Model: macOS on the Darwin kernel (this machine: macOS 26.6, Darwin 25.6.0, arm64, zsh 5.9, Homebrew prefix `/opt/homebrew`). For OS-level inspection/repair — services, security store, code signing, package management, disk state. Verify flags via `man <cmd>` or `<cmd> --help` before relying on an unfamiliar one from memory.

## launchd / launchctl
| Task | Command |
|---|---|
| List all loaded services | `launchctl list` |
| Filter by label substring | `launchctl list \| grep <substr>` |
| Bootstrap (load) a LaunchAgent | `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/<plist>` |
| Bootout (unload) | `launchctl bootout gui/$(id -u)/<label>` |
| Legacy equivalents (still work, superseded) | `launchctl load` / `launchctl unload <plist>` |
| Enable a disabled service (persists across boots) | `launchctl enable gui/$(id -u)/<label>` |
| Start/stop a loaded job | `launchctl start <label>` / `launchctl stop <label>` |
| Print job status + last exit code | `launchctl print gui/$(id -u)/<label>` |
User agents: `~/Library/LaunchAgents`. System-wide: `/Library/Launch{Agents,Daemons}`. Apple-owned: `/System/Library/Launch{Agents,Daemons}` — never edit these.

## defaults (preferences / plist)
| Task | Command |
|---|---|
| Read a domain | `defaults read <domain>` |
| Read one key | `defaults read <domain> <key>` |
| Write | `defaults write <domain> <key> -<type> <value>` |
| Delete | `defaults delete <domain> [key]` |
Dock/Finder changes often need `killall Dock` / `killall Finder` to take visible effect.

## Keychain — /usr/bin/security
| Task | Command |
|---|---|
| Find a generic password entry (attrs only) | `security find-generic-password -s <service> -a <account>` |
| Print the secret value | `security find-generic-password -s <service> -a <account> -w` |
| Add | `security add-generic-password -s <service> -a <account> -w <value>` |
| Delete | `security delete-generic-password -s <service> -a <account>` |
| List keychains | `security list-keychains` |
| Unlock login keychain | `security unlock-keychain ~/Library/Keychains/login.keychain-db` |
NEVER print secret values into chat, logs, or commits — inspect existence/attributes only unless the user explicitly needs the raw local value.

## codesign / notarization
| Task | Command |
|---|---|
| Sign one target | `codesign --force --options runtime --timestamp --sign "<Developer ID Application: ...>" <path>` |
| Verify | `codesign --verify --deep --strict --verbose=2 <path>` |
| Gatekeeper assessment | `spctl --assess --type execute -vv <path>` |
| Submit for notarization | `xcrun notarytool submit <file.zip> --keychain-profile "<profile>" --wait` |
| Staple ticket | `xcrun stapler staple <path>` |
| Notarization history | `xcrun notarytool history --keychain-profile "<profile>"` |
`--deep` is DEPRECATED **for signing** as of macOS 13.0 (`man codesign`): it applies every signing option to every nested item, which is almost never intended. Sign nested code inside-out — frameworks/helpers/plug-ins first, outer bundle last — and keep `--deep` only on `--verify`. Hardened Runtime (`--options runtime`) and a secure timestamp (`--timestamp`) are prerequisites for notarization.
`notarytool` requires a stored credential: `xcrun notarytool store-credentials "<profile>" --apple-id <id> --team-id <TEAMID> --password <app-specific-password>` (run interactively; never commit the value). Verify current flags via `xcrun notarytool --help`.

## diskutil
| Task | Command |
|---|---|
| List disks/partitions | `diskutil list` |
| Info on a volume | `diskutil info <disk/volume>` |
| Mount / unmount | `diskutil mount <id>` / `diskutil unmount <id>` |
| Verify/repair APFS volume | `diskutil verifyVolume <id>` / `diskutil repairVolume <id>` |

## system_profiler
| Task | Command |
|---|---|
| Hardware overview | `system_profiler SPHardwareDataType` |
| Installed apps | `system_profiler SPApplicationsDataType` |
Full unfiltered `system_profiler` is slow — always scope to a `DataType`.

## Homebrew
| Task | Command |
|---|---|
| Update index | `brew update` |
| Upgrade all | `brew upgrade` |
| Health check | `brew doctor` |
| Find what owns a file | `brew list --verbose <formula>` |
| Prune old versions | `brew cleanup` |
Run `brew doctor` before installing on a machine flagged with PATH issues (see `terminal-shell-zsh.md`).
