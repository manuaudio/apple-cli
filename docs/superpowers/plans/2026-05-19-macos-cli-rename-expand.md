# macOS CLI — Rename + Feature Expansion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename `apple-cli` → `macOS CLI` everywhere (binary, repo, GitHub, chief platform, memory) and add 9 new agentic control commands (Bluetooth, Trash, Spotlight, File, LoginItems, Dock, VPN, Notification Center read/clear, Wallpaper get/set).

**Architecture:** Two sequential phases — Phase A renames the entire project (Swift package, binary, GitHub repo, all calling code in chief); Phase B adds new command files under the renamed structure. Both phases end with a build + smoke test before committing.

**Tech Stack:** Swift 5.9, ArgumentParser, IOBluetooth, FileManager, NSWorkspace, UserNotifications, osascript subprocess, `mdfind` subprocess, `defaults` subprocess, `scutil` subprocess. All new commands follow the existing `Process.capture(args:timeout:)` pattern.

---

## File Map

**apple-cli repo — modified:**
- `Package.swift` — rename package name, target name, source path; add IOBluetooth framework
- `Sources/apple-cli/` → rename dir to `Sources/macos-cli/`
- `Sources/macos-cli/AppleCLI.swift` → renamed to `MacOSCLI.swift`; struct renamed; commandName → `"macos"`; register new subcommands; bump version to `0.6.0`
- `Sources/macos-cli/Commands/SystemCommand.swift` — add `VPNCommand` as new subcommand
- `Sources/macos-cli/Commands/NotifyCommand.swift` — add `ListCmd` and `ClearCmd` subcommands
- `Sources/macos-cli/Commands/DisplayCommand.swift` — add `WallpaperCommand` subcommand
- `install.sh` — update binary name, built binary search pattern, all "apple-cli" strings

**apple-cli repo — new files:**
- `Sources/macos-cli/Commands/BluetoothCommand.swift`
- `Sources/macos-cli/Commands/TrashCommand.swift`
- `Sources/macos-cli/Commands/SpotlightCommand.swift`
- `Sources/macos-cli/Commands/FileCommand.swift`
- `Sources/macos-cli/Commands/LoginItemsCommand.swift`
- `Sources/macos-cli/Commands/DockCommand.swift`

**chief repo — modified:**
- `scripts/apple_mail_sync.py:167` — binary path
- `ingest/whatsapp/from_desktop_db.py` — docstring command ref
- `scripts/_dump_familia_to_json.py` — error string
- `scripts/familia_calendar_writer.py` — error strings (3 lines)
- `aura/skills/reply-action-request/SKILL.md` — command refs + "apple-cli" label
- `aura/skills/reply-task-capture/SKILL.md` — command refs
- `aura/skills/chat-reply/SKILL.md` — "apple-cli" label

**memory — modified:**
- `~/.claude/projects/-Users-aura-Developer-chief/memory/project_apple_cli.md` — full update
- `~/.claude/projects/-Users-aura-Developer-chief/memory/MEMORY.md` — pointer line

---

## Phase A: Rename

---

### Task 1: Rename Package.swift

**Files:**
- Modify: `~/Developer/apple-cli/Package.swift`

- [ ] **Step 1: Replace package name, target name, and source path in Package.swift**

Open `~/Developer/apple-cli/Package.swift`. Replace the entire contents with:

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "macos-cli",
    platforms: [.macOS(.v13)],
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.3.0"),
    ],
    targets: [
        .executableTarget(
            name: "macos-cli",
            dependencies: [
                .product(name: "ArgumentParser", package: "swift-argument-parser"),
            ],
            path: "Sources/macos-cli",
            swiftSettings: [
                .unsafeFlags([
                    "-framework", "EventKit",
                    "-framework", "Contacts",
                    "-framework", "CoreGraphics",
                    "-framework", "AppKit",
                    "-framework", "ApplicationServices",
                    "-framework", "Vision",
                    "-framework", "PDFKit",
                    "-framework", "CoreLocation",
                    "-framework", "IOBluetooth",
                ])
            ]
        ),
    ]
)
```

- [ ] **Step 2: Rename source directory**

```bash
mv ~/Developer/apple-cli/Sources/apple-cli ~/Developer/apple-cli/Sources/macos-cli
```

- [ ] **Step 3: Verify directory rename**

```bash
ls ~/Developer/apple-cli/Sources/
```

Expected: `macos-cli` (no `apple-cli` entry)

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/apple-cli
git add Package.swift
git add Sources/
git commit -m "refactor: rename package + source dir apple-cli → macos-cli"
```

---

### Task 2: Rename main struct + commandName

**Files:**
- Modify then rename: `~/Developer/apple-cli/Sources/macos-cli/AppleCLI.swift` → `MacOSCLI.swift`

- [ ] **Step 1: Update AppleCLI.swift struct — rename and change commandName**

Replace the entire contents of `~/Developer/apple-cli/Sources/macos-cli/AppleCLI.swift` with:

```swift
import ArgumentParser
import Foundation

@main
struct MacOSCLI: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "macos",
        abstract: "macOS CLI — full agentic control of macOS via the terminal",
        version: "0.6.0",
        subcommands: [
            // Personal data (EventKit + Contacts.framework + Notes via SQLite)
            RemindersCommand.self,
            CalendarCommand.self,
            ContactsCommand.self,
            NotesCommand.self,
            // System controls
            SystemCommand.self,
            AppsCommand.self,
            ScreenCommand.self,
            StorageCommand.self,
            NotifyCommand.self,
            SpeechCommand.self,
            InfoCommand.self,
            // UI automation (Accessibility + Screen Recording required)
            MouseCommand.self,
            KeyboardCommand.self,
            AxCommand.self,
            ScreenshotCommand.self,
            OcrCommand.self,
            WindowCommand.self,
            // App integrations (Automation permission required)
            SafariCommand.self,
            MailCommand.self,
            MessagesCommand.self,
            PhotosCommand.self,
            MusicCommand.self,
            FinderCommand.self,
            SetupCommand.self,
            // 0.6 additions
            ShortcutsCommand.self,
            PdfCommand.self,
            FocusCommand.self,
            ProcessCommand.self,
            DiskCommand.self,
            LocationCommand.self,
            // 0.5.5
            VoiceMemosCommand.self,
            // 0.6.0 — new agentic commands
            BluetoothCommand.self,
            TrashCommand.self,
            SpotlightCommand.self,
            FileCommand.self,
            LoginItemsCommand.self,
            DockCommand.self,
        ]
    )
}
```

- [ ] **Step 2: Rename the file**

```bash
mv ~/Developer/apple-cli/Sources/macos-cli/AppleCLI.swift \
   ~/Developer/apple-cli/Sources/macos-cli/MacOSCLI.swift
```

- [ ] **Step 3: Commit**

```bash
cd ~/Developer/apple-cli
git add -A Sources/macos-cli/
git commit -m "refactor: rename AppleCLI → MacOSCLI, commandName apple → macos, v0.6.0"
```

---

### Task 3: Update install.sh

**Files:**
- Modify: `~/Developer/apple-cli/install.sh`

- [ ] **Step 1: Replace install.sh with updated content**

```bash
cat > ~/Developer/apple-cli/install.sh << 'INSTALLEOF'
#!/bin/bash
# macOS CLI installer
# Usage: curl -sSL https://raw.githubusercontent.com/manuaudio/macos-cli/main/install.sh | bash
# Or:    git clone https://github.com/manuaudio/macos-cli && cd macos-cli && ./install.sh

set -e

REPO_URL="https://github.com/manuaudio/macos-cli.git"
INSTALL_DIR="/usr/local/bin"
BINARY_NAME="macos"
CLONE_DIR="/tmp/macos-cli-install"

echo ""
echo "macOS CLI installer"
echo "==================="
echo ""

# ── Check for Swift ──────────────────────────────────────────────────────────
if ! command -v swift &>/dev/null; then
    echo "❌  Swift not found."
    echo "    Install Xcode Command Line Tools first:"
    echo "    xcode-select --install"
    echo "    Then re-run this script."
    exit 1
fi
echo "✅  Swift $(swift --version 2>&1 | head -1 | awk '{print $3}')"

# ── Check install dir ────────────────────────────────────────────────────────
if [ ! -d "$INSTALL_DIR" ]; then
    echo "Creating $INSTALL_DIR..."
    sudo mkdir -p "$INSTALL_DIR"
fi

# ── Clone or use existing repo ───────────────────────────────────────────────
if [ -f "Package.swift" ] && [ -d "Sources" ]; then
    REPO_DIR="$(pwd)"
    echo "✅  Using local repo: $REPO_DIR"
else
    echo "📦  Cloning macos-cli..."
    rm -rf "$CLONE_DIR"
    git clone --depth 1 "$REPO_URL" "$CLONE_DIR" 2>&1 | tail -1
    REPO_DIR="$CLONE_DIR"
fi

# ── Build ────────────────────────────────────────────────────────────────────
echo "🔨  Building (this takes ~30s)..."
cd "$REPO_DIR"
swift build -c release --quiet 2>/dev/null || swift build -c release 2>&1 | grep -E "error:|Build complete"

BUILT_BINARY=$(find .build -name "macos-cli" -type f ! -name "*.d" 2>/dev/null | grep release | head -1)
if [ -z "$BUILT_BINARY" ]; then
    echo "❌  Build failed — binary not found"
    exit 1
fi

# ── Install ──────────────────────────────────────────────────────────────────
echo "📋  Installing to $INSTALL_DIR/$BINARY_NAME..."
if cp "$BUILT_BINARY" "$INSTALL_DIR/$BINARY_NAME" 2>/dev/null; then
    chmod +x "$INSTALL_DIR/$BINARY_NAME"
else
    sudo cp "$BUILT_BINARY" "$INSTALL_DIR/$BINARY_NAME"
    sudo chmod +x "$INSTALL_DIR/$BINARY_NAME"
fi

echo "✅  Installed: $($INSTALL_DIR/$BINARY_NAME --version)"
echo ""

# ── Setup ────────────────────────────────────────────────────────────────────
echo "Running permission check..."
echo ""
"$INSTALL_DIR/$BINARY_NAME" setup
INSTALLEOF
chmod +x ~/Developer/apple-cli/install.sh
```

- [ ] **Step 2: Commit**

```bash
cd ~/Developer/apple-cli
git add install.sh
git commit -m "refactor: update install.sh for macos-cli rename"
```

---

### Task 4: Update README.md

**Files:**
- Modify: `~/Developer/apple-cli/README.md`

- [ ] **Step 1: Bulk replace all `apple-cli` references in README**

```bash
cd ~/Developer/apple-cli
sed -i '' 's|apple-cli|macos-cli|g' README.md
sed -i '' 's|manuaudio/apple-cli|manuaudio/macos-cli|g' README.md
sed -i '' 's|# apple-cli|# macOS CLI|g' README.md
sed -i '' 's|/usr/local/bin/apple|/usr/local/bin/macos|g' README.md
```

- [ ] **Step 2: Replace `apple` command invocations in README (binary calls only)**

This replaces `` `apple `` (backtick-quoted CLI calls) with `` `macos ``:

```bash
cd ~/Developer/apple-cli
sed -i '' 's/`apple /`macos /g' README.md
sed -i '' "s/^apple /macos /g" README.md
```

- [ ] **Step 3: Update version badge in README**

```bash
cd ~/Developer/apple-cli
sed -i '' 's/version-0\.[0-9]\.[0-9]-blue/version-0.6.0-blue/g' README.md
```

- [ ] **Step 4: Verify README looks right**

```bash
grep -n "apple-cli\|/apple\b\| apple " ~/Developer/apple-cli/README.md | grep -v "com.apple\|apple.swift\|swift-argument-parser\|AppleLanguages" | head -20
```

Expected: zero lines (all `apple-cli` and `` `apple `` refs replaced). Only `com.apple` or similar system identifiers should remain.

- [ ] **Step 5: Prepend v0.6.0 entry to CHANGELOG.md**

Create a temp file with the new entry, then prepend it:

```bash
cd ~/Developer/apple-cli
cat > /tmp/changelog_entry.md << 'EOF'
## [0.6.0] — 2026-05-19

### Renamed

- Project renamed from `apple-cli` to `macOS CLI`. Binary: `apple` → `macos`. Repo: manuaudio/apple-cli → manuaudio/macos-cli.

### Added — 6 new commands + 3 extensions

- `bluetooth` — list paired devices, connect, disconnect (IOBluetooth.framework)
- `trash` — move to trash, empty, list contents
- `spotlight` — search files via mdfind
- `file` — headless file ops: list, copy, move, delete, stat, read
- `login-items` — list/add/remove startup items via System Events
- `dock` — list/add/remove/restart Dock pins
- `system vpn` — VPN status/connect/disconnect via scutil
- `notify list/clear` — read and clear Notification Center
- `display wallpaper` — get/set desktop wallpaper

---

EOF
cat /tmp/changelog_entry.md CHANGELOG.md > /tmp/changelog_merged.md
mv /tmp/changelog_merged.md CHANGELOG.md
head -5 CHANGELOG.md
```

Expected: first line of CHANGELOG.md is `## [0.6.0] — 2026-05-19`.

- [ ] **Step 6: Commit**

```bash
cd ~/Developer/apple-cli
git add README.md CHANGELOG.md
git commit -m "docs: update README + CHANGELOG for macOS CLI rename and v0.6.0"
```

---

### Task 5: Build + verify renamed binary

**Files:**
- Read: `~/Developer/apple-cli/Sources/macos-cli/MacOSCLI.swift` (to confirm struct is correct before building)

- [ ] **Step 1: Build release binary**

```bash
cd ~/Developer/apple-cli
swift build -c release 2>&1 | tail -5
```

Expected: `Build complete!` with 0 errors. Warnings are OK.

- [ ] **Step 2: Verify binary exists with new name**

```bash
ls -la ~/Developer/apple-cli/.build/release/macos-cli
```

Expected: file exists, non-zero size.

- [ ] **Step 3: Test commandName changed**

```bash
~/Developer/apple-cli/.build/release/macos-cli --help | head -3
```

Expected: first line contains `macos` not `apple`.

- [ ] **Step 4: Test version**

```bash
~/Developer/apple-cli/.build/release/macos-cli --version
```

Expected: `0.6.0`

- [ ] **Step 5: Install new binary**

```bash
cp ~/Developer/apple-cli/.build/release/macos-cli ~/.local/bin/macos
chmod +x ~/.local/bin/macos
```

- [ ] **Step 6: Remove old binary**

```bash
rm -f ~/.local/bin/apple
ls ~/.local/bin/ | grep -E "^apple$|^macos$"
```

Expected: only `macos` listed, no `apple`.

- [ ] **Step 7: Smoke test installed binary**

```bash
macos --version
macos calendar events --json --limit 1 2>&1 | head -5
macos contacts search "Manu" --json 2>&1 | head -5
```

Expected: version `0.6.0`, JSON arrays from calendar and contacts.

---

### Task 6: Rename GitHub repo + update remote

- [ ] **Step 1: Rename GitHub repo via gh CLI**

```bash
gh repo rename macos-cli --repo manuaudio/apple-cli --yes
```

Expected: `✓ Renamed repository manuaudio/apple-cli to manuaudio/macos-cli`

- [ ] **Step 2: Update git remote URL**

```bash
cd ~/Developer/apple-cli
git remote set-url origin https://github.com/manuaudio/macos-cli.git
git remote -v
```

Expected: both fetch and push show `manuaudio/macos-cli.git`.

- [ ] **Step 3: Push all commits so far**

```bash
cd ~/Developer/apple-cli
git push origin main
```

- [ ] **Step 4: Rename local directory**

```bash
mv ~/Developer/apple-cli ~/Developer/macos-cli
```

- [ ] **Step 5: Verify git still works from new location**

```bash
cd ~/Developer/macos-cli
git status
git log --oneline -3
```

Expected: clean status (or only .build dirty), remote still pointing to `manuaudio/macos-cli`.

---

### Task 7: Update chief platform — scripts and skills

Working directory: `~/Developer/chief`

**Files:**
- Modify: `scripts/apple_mail_sync.py`
- Modify: `ingest/whatsapp/from_desktop_db.py`
- Modify: `scripts/_dump_familia_to_json.py`
- Modify: `scripts/familia_calendar_writer.py`
- Modify: `aura/skills/reply-action-request/SKILL.md`
- Modify: `aura/skills/reply-task-capture/SKILL.md`
- Modify: `aura/skills/chat-reply/SKILL.md`

- [ ] **Step 1: Fix binary path in apple_mail_sync.py**

In `~/Developer/chief/scripts/apple_mail_sync.py` line 167, change:
```python
        _sub.run(["/Users/aura/.local/bin/apple", "apps", "launch", "Mail"],
```
to:
```python
        _sub.run(["/Users/aura/.local/bin/macos", "apps", "launch", "Mail"],
```

```bash
sed -i '' 's|/Users/aura/.local/bin/apple|/Users/aura/.local/bin/macos|g' \
    ~/Developer/chief/scripts/apple_mail_sync.py
```

- [ ] **Step 2: Fix docstring in from_desktop_db.py**

```bash
sed -i '' 's/`apple ocr/`macos ocr/g' \
    ~/Developer/chief/ingest/whatsapp/from_desktop_db.py
```

- [ ] **Step 3: Fix error strings in _dump_familia_to_json.py**

```bash
sed -i '' 's/"  WARN: apple calendar/"  WARN: macos calendar/g' \
    ~/Developer/chief/scripts/_dump_familia_to_json.py
```

- [ ] **Step 4: Fix error strings in familia_calendar_writer.py**

```bash
sed -i '' 's/"apple calendar/"macos calendar/g' \
    ~/Developer/chief/scripts/familia_calendar_writer.py
```

- [ ] **Step 5: Update reply-action-request/SKILL.md**

In `~/Developer/chief/aura/skills/reply-action-request/SKILL.md`:

Replace:
```
**Calendar event (Familia / household)** — use apple-cli via Bash:

apple calendar create --title "<title>" --start "YYYY-MM-DD HH:MM" \
```
with:
```
**Calendar event (Familia / household)** — use macOS CLI via Bash:

macos calendar create --title "<title>" --start "YYYY-MM-DD HH:MM" \
```

Replace:
```
**iMessage / SMS** — use apple-cli via Bash (apple-mcp retired 2026-05-18):

apple messages send --to "<phone-or-email>" --text "<message body>"
```
with:
```
**iMessage / SMS** — use macOS CLI via Bash (apple-mcp retired 2026-05-18):

macos messages send --to "<phone-or-email>" --text "<message body>"
```

```bash
sed -i '' 's/use apple-cli via Bash/use macOS CLI via Bash/g' \
    ~/Developer/chief/aura/skills/reply-action-request/SKILL.md
sed -i '' 's/^apple calendar/macos calendar/g' \
    ~/Developer/chief/aura/skills/reply-action-request/SKILL.md
sed -i '' 's/^apple messages/macos messages/g' \
    ~/Developer/chief/aura/skills/reply-action-request/SKILL.md
```

- [ ] **Step 6: Update reply-task-capture/SKILL.md**

```bash
sed -i '' 's|$(go env GOPATH)/bin/apple reminders|/Users/aura/.local/bin/macos reminders|g' \
    ~/Developer/chief/aura/skills/reply-task-capture/SKILL.md
```

- [ ] **Step 7: Update chat-reply/SKILL.md**

```bash
sed -i '' 's/apple-cli calendar enumeration/macOS CLI calendar enumeration/g' \
    ~/Developer/chief/aura/skills/chat-reply/SKILL.md
```

- [ ] **Step 8: Verify all chief refs are clean**

```bash
grep -rn '\.local/bin/apple\b\|"apple calendar\|"apple messages\|"apple reminders\|apple-cli via Bash\|apple-cli calendar' \
    ~/Developer/chief/scripts/ \
    ~/Developer/chief/ingest/ \
    ~/Developer/chief/aura/skills/ 2>/dev/null
```

Expected: zero output.

- [ ] **Step 9: Commit chief changes**

```bash
cd ~/Developer/chief
git add scripts/apple_mail_sync.py \
        ingest/whatsapp/from_desktop_db.py \
        scripts/_dump_familia_to_json.py \
        scripts/familia_calendar_writer.py \
        aura/skills/reply-action-request/SKILL.md \
        aura/skills/reply-task-capture/SKILL.md \
        aura/skills/chat-reply/SKILL.md
git commit -m "refactor: update apple binary refs to macos (apple-cli → macOS CLI rename)"
```

---

### Task 8: Update memory files

**Files:**
- Modify: `~/.claude/projects/-Users-aura-Developer-chief/memory/project_apple_cli.md`
- Modify: `~/.claude/projects/-Users-aura-Developer-chief/memory/MEMORY.md`

- [ ] **Step 1: Rewrite project_apple_cli.md**

Replace the entire contents of `~/.claude/projects/-Users-aura-Developer-chief/memory/project_apple_cli.md` with:

```markdown
---
name: project-macos-cli
description: "macOS CLI — Aura's Swift CLI tool for full agentic macOS control (Calendar, Contacts, Bluetooth, Trash, Spotlight, and 30+ other commands)"
metadata:
  type: project
---

`macOS CLI` is a Swift command-line tool at `/Users/aura/Developer/macos-cli`, built and maintained by Aura + Manu as the primary macOS integration layer for the chief platform.

**Binary:** `~/.local/bin/macos` (invoke as `macos <command>`)

**Current version:** 0.6.0

**Repository:** local at `/Users/aura/Developer/macos-cli`. Published to GitHub at manuaudio/macos-cli.

**Why:** Replaces osascript one-liners and avoids MCP round-trip latency for Apple operations. Per the CLI-first rule in aura/CLAUDE.md, macOS CLI is the preferred path for any Apple framework operation before reaching for browser automation or MCP.

**Full command surface (v0.6.0):** reminders, calendar, contacts, notes, system (battery/audio/wifi/clipboard/display/vpn), apps, screen, storage, notify (send/list/clear), speech, info, mouse, keyboard, ax, screenshot, ocr, window, safari, mail, messages, photos, music, finder, setup, shortcuts, pdf, focus, process, disk, location, voice-memos, bluetooth, trash, spotlight, file, login-items, dock

**How to apply:** When writing or reviewing scripts that touch Apple Calendar, Contacts, Reminders, OCR, Finder, Bluetooth, or file ops — check if macOS CLI has a subcommand before writing osascript or using MCP. Run `macos --help` or read `/Users/aura/Developer/macos-cli/README.md` for the full subcommand list.
```

- [ ] **Step 2: Rename the file to match the new name slug**

```bash
mv ~/.claude/projects/-Users-aura-Developer-chief/memory/project_apple_cli.md \
   ~/.claude/projects/-Users-aura-Developer-chief/memory/project_macos_cli.md
```

- [ ] **Step 3: Update MEMORY.md pointer**

In `~/.claude/projects/-Users-aura-Developer-chief/memory/MEMORY.md`, replace:
```
- [apple-cli — Swift CLI for Apple integrations, v0.5.1](project_apple_cli.md) — at ~/Developer/apple-cli; prefer over osascript/MCP for Calendar, Contacts, Reminders, OCR, Finder ops
```
with:
```
- [macOS CLI — full agentic macOS control, v0.6.0](project_macos_cli.md) — at ~/Developer/macos-cli; binary: `macos`; 35+ commands including bluetooth, trash, spotlight, file, dock
```

```bash
sed -i '' 's|apple-cli — Swift CLI for Apple integrations, v0.5.1\](project_apple_cli.md) — at ~/Developer/apple-cli; prefer over osascript/MCP for Calendar, Contacts, Reminders, OCR, Finder ops|macOS CLI — full agentic macOS control, v0.6.0](project_macos_cli.md) — at ~/Developer/macos-cli; binary: `macos`; 35+ commands including bluetooth, trash, spotlight, file, dock|g' \
    ~/.claude/projects/-Users-aura-Developer-chief/memory/MEMORY.md
```

- [ ] **Step 4: Triple audit — search for any remaining apple-cli / old binary refs across the whole system**

```bash
echo "=== chief repo ==="
grep -rn "apple-cli\|\.local/bin/apple\b" ~/Developer/chief/ \
    --include="*.py" --include="*.sh" --include="*.md" --include="*.ts" \
    --exclude-dir=".git" 2>/dev/null | grep -v "apple_contacts\|apple_mail\|apple_imessage\|apple_cal\|apple_rem\|#.*apple-cli" | head -20

echo "=== memory files ==="
grep -rn "apple-cli\|\.local/bin/apple" \
    ~/.aura/memory/ \
    ~/.claude/projects/-Users-aura-Developer-chief/memory/ 2>/dev/null | head -20

echo "=== macos-cli repo ==="
grep -rn '"apple"\|apple-cli' ~/Developer/macos-cli/Sources/ 2>/dev/null | head -20
```

Expected: zero hits in all three sections. Investigate and fix any that appear before continuing.

---

## Phase B: New Commands

All new Swift files go in `~/Developer/macos-cli/Sources/macos-cli/Commands/`.

---

### Task 9: Add BluetoothCommand

**Files:**
- Create: `~/Developer/macos-cli/Sources/macos-cli/Commands/BluetoothCommand.swift`

- [ ] **Step 1: Write BluetoothCommand.swift**

```swift
import ArgumentParser
import Foundation
import IOBluetooth

struct BluetoothCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "bluetooth",
        abstract: "Bluetooth device management — list paired devices, connect, disconnect",
        subcommands: [ListCmd.self, ConnectCmd.self, DisconnectCmd.self]
    )

    struct ListCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "list", abstract: "List paired Bluetooth devices")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let rawDevices = IOBluetoothDevice.pairedDevices() ?? []
            let devices = rawDevices.compactMap { $0 as? IOBluetoothDevice }
            let items: [[String: Any]] = devices.map { device in
                [
                    "name": device.name ?? "Unknown",
                    "address": device.addressString ?? "",
                    "connected": device.isConnected(),
                    "rssi": device.rawRSSI() as Any,
                ]
            }
            if json {
                let data = try JSONSerialization.data(withJSONObject: items, options: [.prettyPrinted, .sortedKeys])
                print(String(data: data, encoding: .utf8)!)
            } else {
                if items.isEmpty { print("No paired Bluetooth devices."); return }
                print(String(format: "%-30s %-20s %s", "NAME", "ADDRESS", "CONNECTED"))
                print(String(repeating: "-", count: 60))
                for item in items {
                    let connected = (item["connected"] as? Bool == true) ? "yes" : "no"
                    print(String(format: "%-30s %-20s %s",
                        (item["name"] as? String ?? "").prefix(28),
                        (item["address"] as? String ?? "").prefix(18),
                        connected))
                }
            }
        }
    }

    struct ConnectCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "connect", abstract: "Connect a paired Bluetooth device")
        @Argument(help: "Device name or Bluetooth address (e.g. 00-11-22-33-44-55)") var nameOrAddress: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            guard let device = bluetoothFindDevice(nameOrAddress) else {
                throw ValidationError("Device not found: \(nameOrAddress). Use 'bluetooth list' to see paired devices.")
            }
            let result = device.openConnection()
            guard result == kIOReturnSuccess else {
                throw ValidationError("Failed to connect '\(device.name ?? nameOrAddress)': IOReturn 0x\(String(result, radix: 16))")
            }
            if json {
                print("{\"connected\": true, \"name\": \"\(device.name ?? nameOrAddress)\"}")
            } else {
                print("Connected: \(device.name ?? nameOrAddress)")
            }
        }
    }

    struct DisconnectCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "disconnect", abstract: "Disconnect a Bluetooth device")
        @Argument(help: "Device name or Bluetooth address") var nameOrAddress: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            guard let device = bluetoothFindDevice(nameOrAddress) else {
                throw ValidationError("Device not found: \(nameOrAddress). Use 'bluetooth list' to see paired devices.")
            }
            let result = device.closeConnection()
            guard result == kIOReturnSuccess else {
                throw ValidationError("Failed to disconnect '\(device.name ?? nameOrAddress)': IOReturn 0x\(String(result, radix: 16))")
            }
            if json {
                print("{\"disconnected\": true, \"name\": \"\(device.name ?? nameOrAddress)\"}")
            } else {
                print("Disconnected: \(device.name ?? nameOrAddress)")
            }
        }
    }
}

private func bluetoothFindDevice(_ nameOrAddress: String) -> IOBluetoothDevice? {
    let rawDevices = IOBluetoothDevice.pairedDevices() ?? []
    let devices = rawDevices.compactMap { $0 as? IOBluetoothDevice }
    let query = nameOrAddress.lowercased()
    return devices.first { device in
        (device.name?.lowercased() == query) ||
        (device.addressString?.lowercased() == query)
    }
}
```

- [ ] **Step 2: Build to verify it compiles**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|warning:|Build complete"
```

Expected: `Build complete!` with 0 errors.

- [ ] **Step 3: Smoke test**

```bash
macos bluetooth list --json 2>&1 | head -10
```

Expected: valid JSON array (may be empty if no paired devices, but no crash).

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/BluetoothCommand.swift
git commit -m "feat: add bluetooth command (list, connect, disconnect)"
```

---

### Task 10: Add TrashCommand

**Files:**
- Create: `~/Developer/macos-cli/Sources/macos-cli/Commands/TrashCommand.swift`

- [ ] **Step 1: Write TrashCommand.swift**

```swift
import ArgumentParser
import Foundation
import AppKit

struct TrashCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "trash",
        abstract: "Manage the Trash — move files to trash, empty, list contents",
        subcommands: [AddCmd.self, EmptyCmd.self, ListCmd.self]
    )

    struct AddCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "add", abstract: "Move a file or folder to the Trash")
        @Argument(help: "Path of file or directory to trash") var path: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let url = URL(fileURLWithPath: (path as NSString).expandingTildeInPath)
            guard FileManager.default.fileExists(atPath: url.path) else {
                throw ValidationError("Path not found: \(path)")
            }
            var resultURL: NSURL?
            try FileManager.default.trashItem(at: url, resultingItemURL: &resultURL)
            let trashPath = resultURL?.path ?? "~/.Trash/"
            if json {
                print("{\"trashed\": true, \"original\": \"\(url.path)\", \"trash_path\": \"\(trashPath)\"}")
            } else {
                print("Moved to Trash: \(url.path)")
            }
        }
    }

    struct EmptyCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "empty", abstract: "Empty the Trash")
        @Flag(name: .long, help: "Skip Finder confirmation") var force = false
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let script = force
                ? "tell application \"Finder\" to empty trash"
                : "tell application \"Finder\" to empty trash with security"
            let result = Process.capture(args: ["/usr/bin/osascript", "-e", script], timeout: 30)
            if result == nil {
                throw ValidationError("Empty trash timed out after 30s. Trash may be large.")
            }
            if json {
                print("{\"emptied\": true}")
            } else {
                print("Trash emptied.")
            }
        }
    }

    struct ListCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "list", abstract: "List Trash contents")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let trashURL = URL(fileURLWithPath: NSHomeDirectory()).appendingPathComponent(".Trash")
            guard FileManager.default.fileExists(atPath: trashURL.path) else {
                if json { print("[]") } else { print("Trash is empty.") }
                return
            }
            let keys: [URLResourceKey] = [.nameKey, .fileSizeKey, .isDirectoryKey, .contentModificationDateKey]
            let items = try FileManager.default.contentsOfDirectory(
                at: trashURL, includingPropertiesForKeys: keys, options: [.skipsHiddenFiles])
            if json {
                let result: [[String: Any]] = items.compactMap { url in
                    guard let resources = try? url.resourceValues(forKeys: Set(keys)) else { return nil }
                    return [
                        "name": resources.name ?? url.lastPathComponent,
                        "path": url.path,
                        "size": resources.fileSize ?? 0,
                        "is_directory": resources.isDirectory ?? false,
                        "modified": (resources.contentModificationDate?.timeIntervalSince1970 ?? 0),
                    ]
                }
                let data = try JSONSerialization.data(withJSONObject: result, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                if items.isEmpty { print("Trash is empty."); return }
                print(String(format: "%-40s %10s", "NAME", "SIZE"))
                print(String(repeating: "-", count: 52))
                for url in items.sorted(by: { $0.lastPathComponent < $1.lastPathComponent }) {
                    let resources = try? url.resourceValues(forKeys: Set(keys))
                    let size = resources?.fileSize.map { formatBytes($0) } ?? "-"
                    print(String(format: "%-40s %10s", url.lastPathComponent.prefix(38), size))
                }
            }
        }
    }
}

private func formatBytes(_ bytes: Int) -> String {
    let kb = Double(bytes) / 1024
    if kb < 1 { return "\(bytes)B" }
    let mb = kb / 1024
    if mb < 1 { return String(format: "%.1fK", kb) }
    let gb = mb / 1024
    if gb < 1 { return String(format: "%.1fM", mb) }
    return String(format: "%.1fG", gb)
}
```

- [ ] **Step 2: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 3: Smoke test**

```bash
macos trash list --json 2>&1 | head -5
```

Expected: valid JSON array.

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/TrashCommand.swift
git commit -m "feat: add trash command (add, empty, list)"
```

---

### Task 11: Add SpotlightCommand

**Files:**
- Create: `~/Developer/macos-cli/Sources/macos-cli/Commands/SpotlightCommand.swift`

- [ ] **Step 1: Write SpotlightCommand.swift**

```swift
import ArgumentParser
import Foundation

struct SpotlightCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "spotlight",
        abstract: "Search the filesystem via Spotlight (mdfind)",
        subcommands: [SearchCmd.self]
    )

    struct SearchCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "search", abstract: "Search files and folders via Spotlight")
        @Argument(help: "Search query") var query: String
        @Flag(name: .long, help: "Match filename only (faster)") var nameOnly = false
        @Option(name: .long, help: "Filter by kind: app, image, pdf, audio, video, document, folder") var kind: String?
        @Option(name: .long, help: "Limit results (default 50)") var limit: Int = 50
        @Option(name: .long, help: "Search within this directory") var inDir: String?
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            var args = ["/usr/bin/mdfind"]

            if nameOnly {
                args += ["-name", query]
            } else {
                args.append(query)
            }

            if let dir = inDir {
                let expanded = (dir as NSString).expandingTildeInPath
                args += ["-onlyin", expanded]
            }

            if let kindFilter = kind {
                let kindMap = [
                    "app": "kMDItemContentTypeTree == 'com.apple.application'",
                    "image": "kMDItemContentTypeTree == 'public.image'",
                    "pdf": "kMDItemContentType == 'com.adobe.pdf'",
                    "audio": "kMDItemContentTypeTree == 'public.audio'",
                    "video": "kMDItemContentTypeTree == 'public.movie'",
                    "document": "kMDItemContentTypeTree == 'public.text'",
                    "folder": "kMDItemContentType == 'public.folder'",
                ]
                if let predicate = kindMap[kindFilter] {
                    // Combine kind filter with query
                    args = ["/usr/bin/mdfind", "\(predicate) && \(nameOnly ? "kMDItemFSName == '*\(query)*'cd" : "kMDItemTextContent == '*\(query)*'cd")"]
                }
            }

            guard let output = Process.capture(args: args, timeout: 15) else {
                throw ValidationError("Spotlight search timed out after 15s.")
            }

            let allPaths = output
                .components(separatedBy: "\n")
                .map { $0.trimmingCharacters(in: .whitespaces) }
                .filter { !$0.isEmpty }

            let paths = Array(allPaths.prefix(limit))

            if json {
                let fm = FileManager.default
                let items: [[String: Any]] = paths.map { path in
                    let url = URL(fileURLWithPath: path)
                    let isDir = (try? url.resourceValues(forKeys: [.isDirectoryKey]))?.isDirectory ?? false
                    let size = (try? url.resourceValues(forKeys: [.fileSizeKey]))?.fileSize ?? 0
                    let modified = (try? url.resourceValues(forKeys: [.contentModificationDateKey]))?.contentModificationDate
                    return [
                        "path": path,
                        "name": url.lastPathComponent,
                        "is_directory": isDir,
                        "size": size,
                        "modified": modified?.timeIntervalSince1970 ?? 0,
                    ]
                }
                let wrapper: [String: Any] = ["count": paths.count, "total_matches": allPaths.count, "results": items]
                let data = try JSONSerialization.data(withJSONObject: wrapper, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                if paths.isEmpty { print("No results for: \(query)"); return }
                print("Found \(allPaths.count) matches (showing \(paths.count)):")
                paths.forEach { print($0) }
            }
        }
    }
}
```

- [ ] **Step 2: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 3: Smoke test**

```bash
macos spotlight search "README" --name-only --limit 3 --json 2>&1 | head -20
```

Expected: JSON with `results` array containing file paths.

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/SpotlightCommand.swift
git commit -m "feat: add spotlight command (mdfind-backed file search)"
```

---

### Task 12: Add FileCommand

**Files:**
- Create: `~/Developer/macos-cli/Sources/macos-cli/Commands/FileCommand.swift`

- [ ] **Step 1: Write FileCommand.swift**

```swift
import ArgumentParser
import Foundation

struct FileCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "file",
        abstract: "Headless file operations — list, copy, move, delete, stat, read",
        subcommands: [ListCmd.self, CopyCmd.self, MoveCmd.self, DeleteCmd.self, StatCmd.self, ReadCmd.self]
    )

    struct ListCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "list", abstract: "List directory contents")
        @Argument(help: "Directory path (default: current directory)") var path: String = "."
        @Flag(name: .long, help: "Include hidden files") var all = false
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let expanded = (path as NSString).expandingTildeInPath
            let url = URL(fileURLWithPath: expanded)
            var isDir: ObjCBool = false
            guard FileManager.default.fileExists(atPath: expanded, isDirectory: &isDir), isDir.boolValue else {
                throw ValidationError("Not a directory: \(path)")
            }
            let keys: [URLResourceKey] = [.nameKey, .fileSizeKey, .isDirectoryKey, .contentModificationDateKey, .isHiddenKey]
            var options: FileManager.DirectoryEnumerationOptions = [.skipsSubdirectoryDescendants]
            if !all { options.insert(.skipsHiddenFiles) }
            let items = try FileManager.default.contentsOfDirectory(at: url, includingPropertiesForKeys: keys, options: options)
                .sorted { $0.lastPathComponent < $1.lastPathComponent }

            if json {
                let result: [[String: Any]] = items.compactMap { itemURL in
                    guard let resources = try? itemURL.resourceValues(forKeys: Set(keys)) else { return nil }
                    return [
                        "name": resources.name ?? itemURL.lastPathComponent,
                        "path": itemURL.path,
                        "size": resources.fileSize ?? 0,
                        "is_directory": resources.isDirectory ?? false,
                        "modified": resources.contentModificationDate?.timeIntervalSince1970 ?? 0,
                    ]
                }
                let data = try JSONSerialization.data(withJSONObject: result, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                print(String(format: "%-8s %-12s %s", "TYPE", "SIZE", "NAME"))
                print(String(repeating: "-", count: 60))
                for itemURL in items {
                    let resources = try? itemURL.resourceValues(forKeys: Set(keys))
                    let isDirectory = resources?.isDirectory ?? false
                    let typeLabel = isDirectory ? "DIR" : "FILE"
                    let size = isDirectory ? "-" : (resources?.fileSize.map { formatFileBytes($0) } ?? "-")
                    print(String(format: "%-8s %-12s %s", typeLabel, size, itemURL.lastPathComponent))
                }
            }
        }
    }

    struct CopyCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "copy", abstract: "Copy a file or directory")
        @Argument(help: "Source path") var src: String
        @Argument(help: "Destination path") var dst: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let srcURL = URL(fileURLWithPath: (src as NSString).expandingTildeInPath)
            let dstURL = URL(fileURLWithPath: (dst as NSString).expandingTildeInPath)
            guard FileManager.default.fileExists(atPath: srcURL.path) else {
                throw ValidationError("Source not found: \(src)")
            }
            // If dst is an existing directory, copy into it
            var isDir: ObjCBool = false
            let finalDst: URL
            if FileManager.default.fileExists(atPath: dstURL.path, isDirectory: &isDir), isDir.boolValue {
                finalDst = dstURL.appendingPathComponent(srcURL.lastPathComponent)
            } else {
                finalDst = dstURL
            }
            try FileManager.default.copyItem(at: srcURL, to: finalDst)
            if json {
                print("{\"copied\": true, \"src\": \"\(srcURL.path)\", \"dst\": \"\(finalDst.path)\"}")
            } else {
                print("Copied: \(srcURL.path) → \(finalDst.path)")
            }
        }
    }

    struct MoveCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "move", abstract: "Move or rename a file or directory")
        @Argument(help: "Source path") var src: String
        @Argument(help: "Destination path") var dst: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let srcURL = URL(fileURLWithPath: (src as NSString).expandingTildeInPath)
            let dstURL = URL(fileURLWithPath: (dst as NSString).expandingTildeInPath)
            guard FileManager.default.fileExists(atPath: srcURL.path) else {
                throw ValidationError("Source not found: \(src)")
            }
            var isDir: ObjCBool = false
            let finalDst: URL
            if FileManager.default.fileExists(atPath: dstURL.path, isDirectory: &isDir), isDir.boolValue {
                finalDst = dstURL.appendingPathComponent(srcURL.lastPathComponent)
            } else {
                finalDst = dstURL
            }
            try FileManager.default.moveItem(at: srcURL, to: finalDst)
            if json {
                print("{\"moved\": true, \"src\": \"\(srcURL.path)\", \"dst\": \"\(finalDst.path)\"}")
            } else {
                print("Moved: \(srcURL.path) → \(finalDst.path)")
            }
        }
    }

    struct DeleteCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "delete", abstract: "Permanently delete a file or directory (use 'trash add' for recoverable delete)")
        @Argument(help: "Path to delete") var path: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let url = URL(fileURLWithPath: (path as NSString).expandingTildeInPath)
            guard FileManager.default.fileExists(atPath: url.path) else {
                throw ValidationError("Path not found: \(path)")
            }
            try FileManager.default.removeItem(at: url)
            if json {
                print("{\"deleted\": true, \"path\": \"\(url.path)\"}")
            } else {
                print("Deleted: \(url.path)")
            }
        }
    }

    struct StatCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "stat", abstract: "Show file metadata")
        @Argument(help: "File or directory path") var path: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let expanded = (path as NSString).expandingTildeInPath
            let url = URL(fileURLWithPath: expanded)
            guard FileManager.default.fileExists(atPath: expanded) else {
                throw ValidationError("Path not found: \(path)")
            }
            let attrs = try FileManager.default.attributesOfItem(atPath: expanded)
            let isDir = (attrs[.type] as? FileAttributeType) == .typeDirectory
            let size = attrs[.size] as? Int ?? 0
            let created = (attrs[.creationDate] as? Date)?.timeIntervalSince1970 ?? 0
            let modified = (attrs[.modificationDate] as? Date)?.timeIntervalSince1970 ?? 0
            let permissions = attrs[.posixPermissions] as? Int ?? 0
            let owner = attrs[.ownerAccountName] as? String ?? ""

            if json {
                let result: [String: Any] = [
                    "path": url.path,
                    "name": url.lastPathComponent,
                    "is_directory": isDir,
                    "size": size,
                    "permissions": String(format: "%o", permissions),
                    "owner": owner,
                    "created": created,
                    "modified": modified,
                ]
                let data = try JSONSerialization.data(withJSONObject: result, options: [.prettyPrinted, .sortedKeys])
                print(String(data: data, encoding: .utf8)!)
            } else {
                print("Path:        \(url.path)")
                print("Type:        \(isDir ? "directory" : "file")")
                print("Size:        \(formatFileBytes(size))")
                print("Permissions: \(String(format: "%o", permissions))")
                print("Owner:       \(owner)")
                print("Created:     \(Date(timeIntervalSince1970: created))")
                print("Modified:    \(Date(timeIntervalSince1970: modified))")
            }
        }
    }

    struct ReadCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "read", abstract: "Print text file contents")
        @Argument(help: "File path") var path: String
        @Option(name: .long, help: "Max bytes to read (default: 102400 = 100KB)") var maxBytes: Int = 102_400

        func run() throws {
            let url = URL(fileURLWithPath: (path as NSString).expandingTildeInPath)
            guard FileManager.default.fileExists(atPath: url.path) else {
                throw ValidationError("File not found: \(path)")
            }
            let data = try Data(contentsOf: url)
            guard let text = String(data: data.prefix(maxBytes), encoding: .utf8) else {
                throw ValidationError("File is not valid UTF-8 text: \(path)")
            }
            print(text)
            if data.count > maxBytes {
                fputs("[truncated: \(data.count - maxBytes) bytes remaining — use --max-bytes to read more]\n", stderr)
            }
        }
    }
}

private func formatFileBytes(_ bytes: Int) -> String {
    let kb = Double(bytes) / 1024
    if kb < 1 { return "\(bytes)B" }
    let mb = kb / 1024
    if mb < 1 { return String(format: "%.1fK", kb) }
    let gb = mb / 1024
    if gb < 1 { return String(format: "%.1fM", mb) }
    return String(format: "%.1fG", gb)
}
```

- [ ] **Step 2: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 3: Smoke test**

```bash
macos file list ~/Developer/macos-cli --json 2>&1 | head -20
macos file stat ~/Developer/macos-cli/Package.swift --json 2>&1
```

Expected: JSON arrays/objects with path/size/name fields.

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/FileCommand.swift
git commit -m "feat: add file command (list, copy, move, delete, stat, read)"
```

---

### Task 13: Add LoginItemsCommand

**Files:**
- Create: `~/Developer/macos-cli/Sources/macos-cli/Commands/LoginItemsCommand.swift`

- [ ] **Step 1: Write LoginItemsCommand.swift**

```swift
import ArgumentParser
import Foundation

struct LoginItemsCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "login-items",
        abstract: "Manage login items (apps that launch at startup)",
        subcommands: [ListCmd.self, AddCmd.self, RemoveCmd.self]
    )

    struct ListCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "list", abstract: "List current login items")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let script = """
            tell application "System Events"
                set itemList to {}
                repeat with li in login items
                    set end of itemList to (name of li) & "|" & (path of li) & "|" & (hidden of li as string)
                end repeat
                return itemList
            end tell
            """
            guard let output = Process.capture(args: ["/usr/bin/osascript", "-e", script], timeout: 10) else {
                throw ValidationError("Login items query timed out — Automation permission for System Events may be missing.")
            }
            let trimmed = output.trimmingCharacters(in: .whitespacesAndNewlines)
            let items: [[String: Any]] = trimmed.isEmpty ? [] : trimmed
                .components(separatedBy: ", ")
                .compactMap { entry -> [String: Any]? in
                    let parts = entry.components(separatedBy: "|")
                    guard parts.count >= 3 else { return nil }
                    return [
                        "name": parts[0].trimmingCharacters(in: .whitespaces),
                        "path": parts[1].trimmingCharacters(in: .whitespaces),
                        "hidden": parts[2].trimmingCharacters(in: .whitespaces) == "true",
                    ]
                }
            if json {
                let data = try JSONSerialization.data(withJSONObject: items, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                if items.isEmpty { print("No login items."); return }
                print(String(format: "%-30s %s", "NAME", "PATH"))
                print(String(repeating: "-", count: 70))
                for item in items {
                    print(String(format: "%-30s %s",
                        (item["name"] as? String ?? "").prefix(28),
                        item["path"] as? String ?? ""))
                }
            }
        }
    }

    struct AddCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "add", abstract: "Add an app to login items")
        @Argument(help: "Path to the .app bundle") var path: String
        @Flag(name: .long, help: "Launch hidden (no window on startup)") var hidden = false
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let expanded = (path as NSString).expandingTildeInPath
            guard FileManager.default.fileExists(atPath: expanded) else {
                throw ValidationError("Path not found: \(path)")
            }
            let hiddenStr = hidden ? "true" : "false"
            let script = """
            tell application "System Events"
                make new login item at end with properties {hidden:\(hiddenStr), path:"\(expanded)"}
            end tell
            """
            guard Process.capture(args: ["/usr/bin/osascript", "-e", script], timeout: 10) != nil else {
                throw ValidationError("Add login item timed out.")
            }
            let name = URL(fileURLWithPath: expanded).deletingPathExtension().lastPathComponent
            if json {
                print("{\"added\": true, \"name\": \"\(name)\", \"path\": \"\(expanded)\"}")
            } else {
                print("Added login item: \(name)")
            }
        }
    }

    struct RemoveCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "remove", abstract: "Remove a login item by name")
        @Argument(help: "App name (as shown in 'login-items list')") var name: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let script = """
            tell application "System Events"
                delete (first login item whose name is "\(name)")
            end tell
            """
            guard let result = Process.capture(args: ["/usr/bin/osascript", "-e", script], timeout: 10) else {
                throw ValidationError("Remove login item timed out.")
            }
            if result.contains("error") {
                throw ValidationError("Login item not found: \(name)")
            }
            if json {
                print("{\"removed\": true, \"name\": \"\(name)\"}")
            } else {
                print("Removed login item: \(name)")
            }
        }
    }
}
```

- [ ] **Step 2: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 3: Smoke test**

```bash
macos login-items list --json 2>&1 | head -10
```

Expected: valid JSON array.

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/LoginItemsCommand.swift
git commit -m "feat: add login-items command (list, add, remove)"
```

---

### Task 14: Add DockCommand

**Files:**
- Create: `~/Developer/macos-cli/Sources/macos-cli/Commands/DockCommand.swift`

- [ ] **Step 1: Write DockCommand.swift**

```swift
import ArgumentParser
import Foundation

struct DockCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "dock",
        abstract: "Manage Dock pinned apps — list, add, remove, restart",
        subcommands: [ListCmd.self, AddCmd.self, RemoveCmd.self, RestartCmd.self]
    )

    struct ListCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "list", abstract: "List pinned Dock apps and folders")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            // Read persistent-apps and persistent-others from Dock plist
            guard let output = Process.capture(
                args: ["/usr/bin/defaults", "read", "com.apple.dock", "persistent-apps"],
                timeout: 5, fallback: ""
            ).nilIfEmpty() else {
                if json { print("[]") } else { print("No pinned apps.") }
                return
            }

            // Parse app paths from the plist text output
            let paths = output.components(separatedBy: "\n")
                .filter { $0.contains("_CFURLString = ") }
                .compactMap { line -> String? in
                    let parts = line.components(separatedBy: "= ")
                    guard parts.count >= 2 else { return nil }
                    return parts[1].trimmingCharacters(in: .init(charactersIn: "\" ;"))
                }

            if json {
                let items: [[String: Any]] = paths.map { path in
                    let url = URL(fileURLWithPath: path)
                    return [
                        "name": url.deletingPathExtension().lastPathComponent,
                        "path": path,
                    ]
                }
                let data = try JSONSerialization.data(withJSONObject: items, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                if paths.isEmpty { print("No pinned apps."); return }
                for path in paths {
                    let name = URL(fileURLWithPath: path).deletingPathExtension().lastPathComponent
                    print("\(name) — \(path)")
                }
            }
        }
    }

    struct AddCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "add", abstract: "Pin an app to the Dock")
        @Argument(help: "Path to .app bundle") var path: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let expanded = (path as NSString).expandingTildeInPath
            guard FileManager.default.fileExists(atPath: expanded) else {
                throw ValidationError("App not found: \(path)")
            }
            let name = URL(fileURLWithPath: expanded).deletingPathExtension().lastPathComponent
            let entry = "<dict><key>tile-data</key><dict><key>file-data</key><dict><key>_CFURLString</key><string>\(expanded)</string><key>_CFURLStringType</key><integer>0</integer></dict></dict></dict>"
            let result = Process.run(args: [
                "/usr/bin/defaults", "write", "com.apple.dock",
                "persistent-apps", "-array-add", entry
            ])
            guard result == 0 else {
                throw ValidationError("Failed to add app to Dock (defaults write returned \(result))")
            }
            Process.run(args: ["/usr/bin/killall", "Dock"])
            if json {
                print("{\"added\": true, \"name\": \"\(name)\", \"path\": \"\(expanded)\"}")
            } else {
                print("Added to Dock: \(name) (Dock restarted)")
            }
        }
    }

    struct RemoveCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "remove", abstract: "Remove an app from the Dock by name")
        @Argument(help: "App name (as shown in 'dock list')") var name: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            // Use PlistBuddy to remove the entry with matching app name
            let plistPath = NSHomeDirectory() + "/Library/Preferences/com.apple.dock.plist"
            guard let appsOutput = Process.capture(
                args: ["/usr/bin/defaults", "read", "com.apple.dock", "persistent-apps"],
                timeout: 5) else {
                throw ValidationError("Could not read Dock preferences.")
            }

            // Find the index of the entry matching `name`
            let lines = appsOutput.components(separatedBy: "\n")
            var currentIndex = -1
            var matchIndex = -1
            for line in lines {
                if line.contains("{") { currentIndex += 1 }
                if line.contains("_CFURLString = ") {
                    let path = line.components(separatedBy: "= ").last?.trimmingCharacters(in: .init(charactersIn: "\" ;")) ?? ""
                    let appName = URL(fileURLWithPath: path).deletingPathExtension().lastPathComponent
                    if appName.lowercased() == name.lowercased() {
                        matchIndex = currentIndex
                        break
                    }
                }
            }

            guard matchIndex >= 0 else {
                throw ValidationError("App not found in Dock: \(name). Use 'dock list' to see pinned apps.")
            }

            let result = Process.run(args: [
                "/usr/libexec/PlistBuddy", "-c",
                "Delete :persistent-apps:\(matchIndex)", plistPath
            ])
            guard result == 0 else {
                throw ValidationError("Failed to remove from Dock (PlistBuddy returned \(result))")
            }
            // Convert binary plist to XML so defaults can read it
            Process.run(args: ["/usr/bin/plutil", "-convert", "xml1", plistPath])
            Process.run(args: ["/usr/bin/killall", "Dock"])
            if json {
                print("{\"removed\": true, \"name\": \"\(name)\"}")
            } else {
                print("Removed from Dock: \(name) (Dock restarted)")
            }
        }
    }

    struct RestartCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "restart", abstract: "Restart the Dock (applies pending changes)")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            Process.run(args: ["/usr/bin/killall", "Dock"])
            if json {
                print("{\"restarted\": true}")
            } else {
                print("Dock restarted.")
            }
        }
    }
}

private extension String {
    func nilIfEmpty() -> String? {
        return self.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty ? nil : self
    }
}
```

- [ ] **Step 2: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 3: Smoke test**

```bash
macos dock list --json 2>&1 | head -20
```

Expected: JSON array of pinned Dock apps.

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/DockCommand.swift
git commit -m "feat: add dock command (list, add, remove, restart)"
```

---

### Task 15: Extend SystemCommand — add VPN subcommand

**Files:**
- Modify: `~/Developer/macos-cli/Sources/macos-cli/Commands/SystemCommand.swift`

- [ ] **Step 1: Add VPNCommand struct to SystemCommand.swift**

Find the `struct SystemCommand` definition and update the `subcommands` list to add `VPNCommand.self`:

```swift
// In SystemCommand.swift, update the configuration:
static let configuration = CommandConfiguration(
    commandName: "system",
    abstract: "macOS system controls — battery, audio, Wi-Fi, display, clipboard, VPN",
    subcommands: [
        BatteryCommand.self,
        AudioCommand.self,
        WifiCommand.self,
        ClipboardCommand.self,
        DisplayCommand.self,
        VPNCommand.self,     // ADD THIS LINE
    ]
)
```

Then add this struct **at the end of SystemCommand.swift**, before the final closing brace or after the `DisplayCommand` struct:

```swift
// MARK: - VPN

struct VPNCommand: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "vpn",
        abstract: "VPN connection management — list, connect, disconnect",
        subcommands: [StatusCmd.self, ConnectCmd.self, DisconnectCmd.self]
    )

    struct StatusCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "status", abstract: "List VPN configurations and their state")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            guard let output = Process.capture(args: ["/usr/sbin/scutil", "--nc", "list"], timeout: 10) else {
                throw ValidationError("VPN status query timed out.")
            }
            let lines = output.components(separatedBy: "\n").filter { !$0.trimmingCharacters(in: .whitespaces).isEmpty }
            // Each line: * (Connected) <UUID> [<Protocol>] "<Name>"
            let items: [[String: Any]] = lines.compactMap { line -> [String: Any]? in
                let connected = line.contains("(Connected)")
                // Extract UUID
                let uuidPattern = try? NSRegularExpression(pattern: "[0-9A-F]{8}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{4}-[0-9A-F]{12}")
                let uuidRange = uuidPattern?.firstMatch(in: line, range: NSRange(line.startIndex..., in: line)).flatMap { Range($0.range, in: line) }
                let uuid = uuidRange.map { String(line[$0]) } ?? ""
                // Extract name between last pair of quotes
                let nameMatch = line.range(of: "\"[^\"]+\"$", options: .regularExpression)
                let name = nameMatch.map { String(line[$0]).trimmingCharacters(in: .init(charactersIn: "\"")) } ?? line
                // Extract protocol
                let protoMatch = line.range(of: "\\[[^\\]]+\\]", options: .regularExpression)
                let proto = protoMatch.map { String(line[$0]).trimmingCharacters(in: .init(charactersIn: "[]")) } ?? ""
                return ["name": name, "uuid": uuid, "protocol": proto, "connected": connected]
            }
            if json {
                let data = try JSONSerialization.data(withJSONObject: items, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                if items.isEmpty { print("No VPN configurations found."); return }
                for item in items {
                    let status = (item["connected"] as? Bool == true) ? "Connected" : "Disconnected"
                    print("[\(status)] \(item["name"] ?? "") (\(item["protocol"] ?? ""))")
                }
            }
        }
    }

    struct ConnectCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "connect", abstract: "Connect a VPN by name")
        @Argument(help: "VPN name (as shown in 'system vpn status')") var name: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let result = Process.run(args: ["/usr/sbin/scutil", "--nc", "start", name])
            guard result == 0 else {
                throw ValidationError("Failed to start VPN '\(name)'. Check VPN name with 'system vpn status'.")
            }
            if json {
                print("{\"connecting\": true, \"name\": \"\(name)\"}")
            } else {
                print("Connecting to VPN: \(name)")
            }
        }
    }

    struct DisconnectCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "disconnect", abstract: "Disconnect a VPN by name")
        @Argument(help: "VPN name") var name: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let result = Process.run(args: ["/usr/sbin/scutil", "--nc", "stop", name])
            guard result == 0 else {
                throw ValidationError("Failed to stop VPN '\(name)'.")
            }
            if json {
                print("{\"disconnected\": true, \"name\": \"\(name)\"}")
            } else {
                print("Disconnected VPN: \(name)")
            }
        }
    }
}
```

- [ ] **Step 2: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 3: Smoke test**

```bash
macos system vpn status --json 2>&1 | head -10
```

Expected: valid JSON array (may be empty if no VPN configs).

- [ ] **Step 4: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/SystemCommand.swift
git commit -m "feat: extend system command with vpn subcommand (list, connect, disconnect)"
```

---

### Task 16: Extend NotifyCommand — add list + clear

**Files:**
- Modify: `~/Developer/macos-cli/Sources/macos-cli/Commands/NotifyCommand.swift`

- [ ] **Step 1: Read current NotifyCommand.swift to find subcommands list**

```bash
head -30 ~/Developer/macos-cli/Sources/macos-cli/Commands/NotifyCommand.swift
```

- [ ] **Step 2: Add ListCmd and ClearCmd to subcommands array**

Update the `CommandConfiguration` in `NotifyCommand` to include the new subcommands:

```swift
static let configuration = CommandConfiguration(
    commandName: "notify",
    abstract: "Send and manage macOS notifications",
    subcommands: [
        SendCmd.self,   // existing
        ListCmd.self,   // new
        ClearCmd.self,  // new
    ]
)
```

Then add these structs to `NotifyCommand.swift`:

```swift
struct ListCmd: ParsableCommand {
    static let configuration = CommandConfiguration(commandName: "list", abstract: "List delivered notifications in Notification Center")
    @Flag(name: .long, help: "Output JSON") var json = false

    func run() throws {
        let center = UNUserNotificationCenter.current()
        let sema = DispatchSemaphore(value: 0)
        var notifications: [UNNotification] = []
        center.getDeliveredNotifications { notes in
            notifications = notes
            sema.signal()
        }
        guard sema.wait(timeout: .now() + 10) == .success else {
            throw ValidationError("Notification Center query timed out.")
        }
        let items: [[String: Any]] = notifications.map { note in
            [
                "id": note.request.identifier,
                "title": note.request.content.title,
                "body": note.request.content.body,
                "delivered": note.date.timeIntervalSince1970,
                "bundle_id": note.request.content.userInfo["bundle_id"] as? String ?? "",
            ]
        }
        if json {
            let data = try JSONSerialization.data(withJSONObject: items, options: [.prettyPrinted])
            print(String(data: data, encoding: .utf8)!)
        } else {
            if items.isEmpty { print("No notifications in Notification Center."); return }
            for item in items {
                let title = item["title"] as? String ?? ""
                let body = item["body"] as? String ?? ""
                print("• \(title): \(body)")
            }
        }
    }
}

struct ClearCmd: ParsableCommand {
    static let configuration = CommandConfiguration(commandName: "clear", abstract: "Clear notifications from Notification Center")
    @Option(name: .long, help: "Clear specific notification by ID (clears all if omitted)") var id: String?
    @Flag(name: .long, help: "Output JSON") var json = false

    func run() throws {
        let center = UNUserNotificationCenter.current()
        let sema = DispatchSemaphore(value: 0)
        if let notifId = id {
            center.removeDeliveredNotifications(withIdentifiers: [notifId])
            sema.signal()
        } else {
            center.removeAllDeliveredNotifications()
            sema.signal()
        }
        sema.wait()
        if json {
            print("{\"cleared\": true, \"id\": \(id.map { "\"\($0)\"" } ?? "null")}")
        } else {
            print("Cleared \(id ?? "all") notifications.")
        }
    }
}
```

- [ ] **Step 3: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 4: Smoke test**

```bash
macos notify list --json 2>&1 | head -10
```

Expected: valid JSON array.

- [ ] **Step 5: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/NotifyCommand.swift
git commit -m "feat: extend notify command with list and clear subcommands"
```

---

### Task 17: Extend DisplayCommand — add wallpaper

**Files:**
- Modify: `~/Developer/macos-cli/Sources/macos-cli/Commands/DisplayCommand.swift`

- [ ] **Step 1: Read current DisplayCommand.swift subcommands list**

```bash
head -20 ~/Developer/macos-cli/Sources/macos-cli/Commands/DisplayCommand.swift
```

- [ ] **Step 2: Add WallpaperCommand to subcommands**

Update the `CommandConfiguration` in `DisplayCommand`:

```swift
static let configuration = CommandConfiguration(
    commandName: "display",
    abstract: "Display brightness, dark mode, and wallpaper control",
    subcommands: [
        BrightnessCmd.self,    // existing
        DarkModeCmd.self,      // existing
        WallpaperCmd.self,     // new
    ]
)
```

Then add this struct to `DisplayCommand.swift`:

```swift
struct WallpaperCmd: ParsableCommand {
    static let configuration = CommandConfiguration(
        commandName: "wallpaper",
        abstract: "Get or set the desktop wallpaper",
        subcommands: [GetCmd.self, SetCmd.self]
    )

    struct GetCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "get", abstract: "Get current wallpaper path for each display")
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let screens = NSScreen.screens
            let items: [[String: Any]] = screens.enumerated().compactMap { index, screen in
                guard let url = NSWorkspace.shared.desktopImageURL(for: screen) else { return nil }
                return [
                    "display": index,
                    "name": screen.localizedName,
                    "path": url.path,
                ]
            }
            if json {
                let data = try JSONSerialization.data(withJSONObject: items, options: [.prettyPrinted])
                print(String(data: data, encoding: .utf8)!)
            } else {
                for item in items {
                    print("Display \(item["display"] ?? 0) (\(item["name"] ?? "")): \(item["path"] ?? "")")
                }
            }
        }
    }

    struct SetCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "set", abstract: "Set the desktop wallpaper (all displays)")
        @Argument(help: "Path to image file") var path: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            let expanded = (path as NSString).expandingTildeInPath
            guard FileManager.default.fileExists(atPath: expanded) else {
                throw ValidationError("Image file not found: \(path)")
            }
            let url = URL(fileURLWithPath: expanded)
            let screens = NSScreen.screens
            guard !screens.isEmpty else {
                throw ValidationError("No displays found.")
            }
            for screen in screens {
                try NSWorkspace.shared.setDesktopImageURL(url, for: screen, options: [:])
            }
            if json {
                print("{\"set\": true, \"path\": \"\(expanded)\", \"displays\": \(screens.count)}")
            } else {
                print("Wallpaper set to: \(expanded) (\(screens.count) display\(screens.count == 1 ? "" : "s"))")
            }
        }
    }
}
```

- [ ] **Step 3: Build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | grep -E "error:|Build complete"
```

- [ ] **Step 4: Smoke test**

```bash
macos display wallpaper get --json 2>&1
```

Expected: JSON with path to current wallpaper image.

- [ ] **Step 5: Commit**

```bash
cd ~/Developer/macos-cli
git add Sources/macos-cli/Commands/DisplayCommand.swift
git commit -m "feat: extend display command with wallpaper get/set"
```

---

### Task 18: Final build + comprehensive smoke test

- [ ] **Step 1: Full release build**

```bash
cd ~/Developer/macos-cli
swift build -c release 2>&1 | tail -3
```

Expected: `Build complete!`

- [ ] **Step 2: Install final binary**

```bash
cp ~/Developer/macos-cli/.build/release/macos-cli ~/.local/bin/macos
macos --version
```

Expected: `0.6.0`

- [ ] **Step 3: Smoke test all 9 new commands**

```bash
macos bluetooth list --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('bluetooth ✓')"
macos trash list --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('trash ✓')"
macos spotlight search "Package.swift" --name-only --limit 2 --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('spotlight ✓')"
macos file list ~/Developer/macos-cli --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('file ✓')"
macos login-items list --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('login-items ✓')"
macos dock list --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('dock ✓')"
macos system vpn status --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('vpn ✓')"
macos notify list --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('notify list ✓')"
macos display wallpaper get --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('wallpaper ✓')"
```

Expected: all 9 lines print `✓`.

- [ ] **Step 4: Smoke test 5 pre-existing commands (regression check)**

```bash
macos system battery --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('battery ✓')"
macos calendar events --json --limit 1 2>&1 | head -3
macos contacts search "Manu" --json 2>&1 | head -3
macos system clipboard get 2>&1 | head -1
macos system wifi status --json 2>&1 | python3 -c "import sys,json; json.load(sys.stdin); print('wifi ✓')"
```

Expected: valid output from all 5.

---

### Task 19: Triple audit + push + tag

- [ ] **Step 1: Triple audit — no remaining old refs**

```bash
echo "=== apple-cli in macos-cli repo ==="
grep -rn "apple-cli\|commandName.*apple\b\|\"apple\"" ~/Developer/macos-cli/Sources/ 2>/dev/null \
    | grep -v "swift-argument-parser\|com.apple" | head -10

echo "=== apple binary refs in chief ==="
grep -rn '\.local/bin/apple\b\|"apple-cli\|use apple-cli' \
    ~/Developer/chief/scripts/ \
    ~/Developer/chief/ingest/ \
    ~/Developer/chief/aura/skills/ 2>/dev/null | grep -v ".pyc" | head -10

echo "=== memory files ==="
grep -rn "apple-cli\|project_apple_cli\|\.local/bin/apple" \
    ~/.aura/memory/ \
    ~/.claude/projects/-Users-aura-Developer-chief/memory/ 2>/dev/null | head -10

echo "=== installed binary ==="
ls ~/.local/bin/ | grep -E "^apple$|^macos$"
```

Expected: zero lines in first three sections. Last section: only `macos`, no `apple`.

- [ ] **Step 2: Fix any hits from audit (before continuing)**

If any references appear, fix them inline — do not proceed to Step 3 until the audit is clean.

- [ ] **Step 3: Tag v0.6.0**

```bash
cd ~/Developer/macos-cli
git tag v0.6.0 -m "v0.6.0: rename apple-cli → macOS CLI + 9 new agentic commands"
```

- [ ] **Step 4: Push everything to GitHub**

```bash
cd ~/Developer/macos-cli
git push origin main --tags
```

Expected: push succeeds to `https://github.com/manuaudio/macos-cli`.

- [ ] **Step 5: Push chief changes**

```bash
cd ~/Developer/chief
git push origin main
```

- [ ] **Step 6: Notify Manu via Telegram**

```bash
python3 ~/Developer/chief/scripts/send_to_telegram.py \
  --chat-id 8531085191 \
  --text "macOS CLI v0.6.0 shipped.

Rename complete:
• Binary: apple → macos
• Repo: manuaudio/apple-cli → manuaudio/macos-cli
• All chief scripts + skills updated
• Memory files updated

9 new commands:
• macos bluetooth list/connect/disconnect
• macos trash add/empty/list
• macos spotlight search
• macos file list/copy/move/delete/stat/read
• macos login-items list/add/remove
• macos dock list/add/remove/restart
• macos system vpn status/connect/disconnect
• macos notify list/clear
• macos display wallpaper get/set

Triple audit passed — no remaining apple-cli references."
```
