# macOS System — Darwin

Model: macOS on the Darwin kernel (this machine: Darwin 25.6, zsh default shell). For OS-level inspection/repair — services, security store, code signing, package management, disk state. Verify flags via `man <cmd>` or `<cmd> --help` before relying on an unfamiliar one from memory.

## launchd / launchctl
| Task | Command |
|---|---|
| List all loaded services | `launchctl list` |
| Filter by label substring | `launchctl list \| grep <substr>` |
| Load a LaunchAgent/Daemon | `launchctl load ~/Library/LaunchAgents/<plist>` |
| Unload | `launchctl unload ~/Library/LaunchAgents/<plist>` |
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
| Sign | `codesign --force --deep --sign "<Developer ID>" <path>` |
| Verify | `codesign --verify --deep --strict --verbose=2 <path>` |
| Gatekeeper assessment | `spctl --assess --type execute -vv <path>` |
| Submit for notarization | `xcrun notarytool submit <file.zip> --keychain-profile "<profile>" --wait` |
| Staple ticket | `xcrun stapler staple <path>` |
| Notarization history | `xcrun notarytool history --keychain-profile "<profile>"` |
Verify current flags via `xcrun notarytool --help` — surface has shifted across Xcode versions.

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
