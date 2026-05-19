# macOS CLI — Rename + Feature Expansion Design

**Date:** 2026-05-19  
**Status:** Approved by Manu  
**Scope:** Rename `apple-cli` → `macOS CLI` everywhere, add 10 new agentic-control commands

---

## 1. Overview

`apple-cli` is being rebranded to **macOS CLI** — aligning with Apple's current OS naming and making clear it's a full macOS agent control layer, not just an Apple framework wrapper. The rename touches the Swift package, binary name, GitHub repo, chief platform scripts, skills, and memory files. After the rename, 10 new commands are added to close gaps in agentic Mac control.

---

## 2. Rename Scope

### 2a. apple-cli repo

| Item | Before | After |
|------|--------|-------|
| GitHub repo | `manuaudio/apple-cli` | `manuaudio/macos-cli` |
| Local dir | `~/Developer/apple-cli` | `~/Developer/macos-cli` |
| Package name (Package.swift) | `"apple-cli"` | `"macos-cli"` |
| Target name (Package.swift) | `"apple-cli"` | `"macos-cli"` |
| Source path | `Sources/apple-cli/` | `Sources/macos-cli/` |
| Main struct | `AppleCLI` | `MacOSCLI` |
| CLI commandName | `"apple"` | `"macos"` |
| Installed binary | `~/.local/bin/apple` | `~/.local/bin/macos` |
| install.sh | all "apple" refs | "macos" |
| README.md | all "apple-cli" / `apple` command refs | "macos-cli" / `macos` |
| CHANGELOG.md | header / refs | updated |
| abstract strings in Swift | mention "apple" | use "macOS" |

### 2b. chief platform

| File | What changes |
|------|-------------|
| `scripts/apple_mail_sync.py:167` | `/usr/local/bin/apple` → `/usr/local/bin/macos` |
| `ingest/whatsapp/from_desktop_db.py` | docstring `apple ocr` → `macos ocr` |
| `scripts/_dump_familia_to_json.py` | error string `apple calendar` → `macos calendar` |
| `scripts/familia_calendar_writer.py` | error strings `apple calendar` → `macos calendar` |
| `aura/skills/reply-action-request/SKILL.md` | `apple calendar`, `apple messages`, "use apple-cli" |
| `aura/skills/reply-task-capture/SKILL.md` | `apple reminders` |
| `aura/skills/chat-reply/SKILL.md` | "apple-cli calendar enumeration" |
| `CLAUDE.md` | apple-mcp-routing skill references, any apple-cli mentions |
| `README.md` | overview references |

### 2c. Memory files

| File | What changes |
|------|-------------|
| `~/.claude/projects/-Users-aura-Developer-chief/memory/project_apple_cli.md` | Full update: new name, repo, binary, path |
| `~/.claude/projects/-Users-aura-Developer-chief/memory/MEMORY.md` | Update pointer line |

### 2d. What does NOT change

- Script names like `apple_contacts_sync.py`, `apple_mail_sync.py` — these describe Apple's data source, not the CLI binary
- launchd daemon names `com.aura.apple-*` — bound to data source names, not the CLI tool
- DB table `apple_contacts` — canonical schema name, not related to the binary
- `com.apple.*` defaults/bundle ID strings in code — system identifiers
- `apple_to_iso` / `iso_to_apple` time functions in `ical.py` — Apple epoch helpers

---

## 3. New Commands

All 10 commands are new Swift files in `Sources/macos-cli/Commands/`. Each follows the existing pattern: `ParsableCommand`, `--json` flag, explicit timeout on any subprocess/JXA call, clean exit codes.

### 3.1 BluetoothCommand — `macos bluetooth`

- `list` — list paired/connected Bluetooth devices (name, address, connected status, type)
- `connect <address|name>` — connect a paired device
- `disconnect <address|name>` — disconnect a device

**Implementation:** `IOBluetooth.framework` (`IOBluetoothDevice.pairedDevices()`, `openConnection()`, `closeConnection()`). No JXA needed.

### 3.2 TrashCommand — `macos trash`

- `add <path>` — move file/directory to Trash (preserves name conflict handling)
- `empty` — empty the Trash (with `--force` to skip confirmation)
- `list` — list Trash contents with size and original path

**Implementation:** `FileManager.trashItem(at:resultingItemURL:)` for add. `NSWorkspace.shared.emptyTrash()` for empty. Enumerate `~/.Trash/` for list.

### 3.3 SpotlightCommand — `macos spotlight`

- `search <query>` — search via `mdfind` (name match + full-text), returns paths + metadata
- `search --name-only <query>` — `mdfind -name` for filename-only match
- `search --kind <app|image|pdf|...> <query>` — type-scoped search

**Implementation:** `Process` wrapping `mdfind`. Parses output into JSON array with `{path, name, kind, modified}`. 10s timeout.

### 3.4 FileCommand — `macos file`

- `list <path>` — list directory contents (name, size, modified, isDir, permissions)
- `copy <src> <dst>` — copy file or directory
- `move <src> <dst>` — move/rename file or directory
- `delete <path>` — delete file or directory (not trash — use `trash add` for recoverable delete)
- `stat <path>` — file metadata (size, permissions, owner, dates, type)
- `read <path>` — print file contents (text files; errors on binary)

**Implementation:** `FileManager` for all ops. `stat` via `attributesOfItem`. No subprocess.

### 3.5 LoginItemsCommand — `macos login-items`

- `list` — list current login items (name, path, enabled)
- `add <path>` — add app to login items
- `remove <path|name>` — remove from login items

**Implementation:** `ServiceManagement.framework` (`SMAppService`) on macOS 13+. Falls back to legacy `LSSharedFileList` API if needed.

### 3.6 DockCommand — `macos dock`

- `list` — list pinned Dock items (apps + folders) from `com.apple.dock` plist
- `add <app-path>` — pin app to Dock (`defaults write` + `killall Dock`)
- `remove <name>` — remove from Dock
- `restart` — `killall Dock` (applies deferred changes)

**Implementation:** Read/write `~/Library/Preferences/com.apple.dock.plist` via `Process`/`defaults`. Restart Dock after mutations.

### 3.7 VPN — extend SystemCommand

Add `macos system vpn` under the existing `SystemCommand`:

- `status` — list VPN configurations and connection state
- `connect <name>` — connect a VPN configuration
- `disconnect <name>` — disconnect

**Implementation:** `NetworkExtension.framework` (`NEVPNManager`, `NETunnelProviderManager`). Async → dispatch group with 15s timeout.

### 3.8 Notification Center — extend NotifyCommand

Add to existing `NotifyCommand`:

- `list` — query delivered notifications via `UNUserNotificationCenter.getDeliveredNotifications`
- `clear` — remove all delivered notifications (`removeAllDeliveredNotifications`)
- `clear --id <id>` — remove specific notification

**Implementation:** `UserNotifications.framework` — already used for sending. List/clear are additive subcommands.

### 3.9 Wallpaper — extend DisplayCommand

Add to existing `DisplayCommand`:

- `wallpaper get` — returns current wallpaper path for each display
- `wallpaper set <path>` — set wallpaper on all displays

**Implementation:** `NSWorkspace.shared.setDesktopImageURL(_:for:options:)`. Get via `NSWorkspace.shared.desktopImageURL(for:)`.

### 3.10 Scroll — already exists

`macos mouse scroll` is already implemented in `MouseCommand.Scroll`. No action needed.

---

## 4. Execution Order

1. **Phase 1 — Rename repo:** Package.swift, source dir, struct names, commandName, README, CHANGELOG, install.sh
2. **Phase 2 — Rename GitHub repo** via `gh repo rename macos-cli`
3. **Phase 3 — Build & verify** renamed binary compiles and all existing commands work
4. **Phase 4 — Install** new binary at `~/.local/bin/macos`, remove old `~/.local/bin/apple`
5. **Phase 5 — Update chief platform** scripts, skills, CLAUDE.md, memory files
6. **Phase 6 — Add new commands** one Swift file per command, register in MacOSCLI.swift
7. **Phase 7 — Build & verify new commands** compile and pass smoke tests
8. **Phase 8 — Triple audit** grep for any remaining `apple-cli`/`apple` binary refs
9. **Phase 9 — Push** to `manuaudio/macos-cli`, tag v0.6.0

---

## 5. Error Handling

All new commands follow existing patterns:
- Exit 0 + JSON on success
- Exit 1 + `{"error": "..."}` on failure
- Explicit timeout on any subprocess/JXA call (10–15s depending on operation)
- Permission errors surface as human-readable exit 64 messages

---

## 6. Testing

After each phase:
- `swift build -c release` must complete with 0 errors
- `macos --help` lists all commands
- Each new command: `macos <cmd> --help` shows expected subcommands
- Smoke test: `macos bluetooth list --json`, `macos trash list --json`, `macos spotlight search "test" --json`, etc.
- All pre-existing commands: spot-check 5 across categories
