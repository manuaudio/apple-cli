# macos-cli v0.6.0 Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Fix all 37 issues found in the v0.6.0 Opus deep audit — auth coverage, injection vulnerabilities, error handling, API compatibility, and correctness bugs.

**Architecture:** Swift ArgumentParser CLI. Auth enforcement via Auth.swift. JXA via osascript subprocess. CoreAudio, EventKit, Accessibility APIs.

**Tech Stack:** Swift 5.9, ArgumentParser, CoreAudio, EventKit, Accessibility APIs, osascript

---

## Repo conventions (apply to every task)

- **Build:** `cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"` — expected output: `Build complete!`
- **Install:** `cp ~/Developer/macos-cli/.build/release/macos-cli ~/.local/bin/macos`
- **Git commits:** `git -c submodule.recurse=false add -A && git -c submodule.recurse=false commit -m "..."` (the `submodule.recurse=false` is mandatory due to a stale .build/checkouts reference)
- **No `cd` chaining in git commands** — operate from the working tree.
- **Never amend** — every fix is a new commit.

---

## File Map

| Action | File | What changes |
|---|---|---|
| Create | `Sources/macos-cli/Helpers.swift` | Extract Process helpers, add jxaEscape(), JXA result envelope, stderr-capturing variant |
| Modify | `Sources/macos-cli/Commands/SystemCommand.swift` | Remove `extension Process` block (moved to Helpers.swift) |
| Modify | `Sources/macos-cli/Auth.swift` | Add 35 new capability entries; comment partial-config behavior; remove orphans |
| Modify | `Sources/macos-cli/EventKitStore.swift` | macOS 14+ requestFullAccessTo* APIs |
| Modify | `Sources/macos-cli/Commands/MessagesCommand.swift` | Auth guards, jxaEscape(), structured JXA result |
| Modify | `Sources/macos-cli/Commands/MouseCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/KeyboardCommand.swift` | Auth guards, jxaEscape(), structured result |
| Modify | `Sources/macos-cli/Commands/AxCommand.swift` | Auth guards, jxaEscape(), structured result, frontmost API fix |
| Modify | `Sources/macos-cli/Commands/FinderCommand.swift` | Auth guards, jxaEscape(), structured result |
| Modify | `Sources/macos-cli/Commands/PhotosCommand.swift` | Auth guards, jxaEscape(), structured result |
| Modify | `Sources/macos-cli/Commands/SafariCommand.swift` | Auth guards, jxaEscape() for Execute, structured result |
| Modify | `Sources/macos-cli/Commands/DiskCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/ProcessCommand.swift` | Auth guards on Kill |
| Modify | `Sources/macos-cli/Commands/TrashCommand.swift` | Auth guard on EmptyCmd |
| Modify | `Sources/macos-cli/Commands/DefaultsCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/DockCommand.swift` | Auth guards + PropertyListSerialization (no XML injection) |
| Modify | `Sources/macos-cli/Commands/LoginItemsCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/FileCommand.swift` | Auth guards, size-limit + --force on Read |
| Modify | `Sources/macos-cli/Commands/AppsCommand.swift` | Auth guard on Quit |
| Modify | `Sources/macos-cli/Commands/BluetoothCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/OcrCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/SystemCommand.swift` | Auth guards on Lock, Sleep, audio volume/mute, brightness, dark mode, wallpaper set, wifi join/leave, clipboard set |
| Modify | `Sources/macos-cli/Commands/MusicCommand.swift` | Auth guards, jxaEscape(), structured result |
| Modify | `Sources/macos-cli/Commands/NotifyCommand.swift` | Auth guard + jxaEscape() |
| Modify | `Sources/macos-cli/Commands/SpeechCommand.swift` | Auth guard |
| Modify | `Sources/macos-cli/Commands/ShortcutsCommand.swift` | Auth guard on Run |
| Modify | `Sources/macos-cli/Commands/SpotlightCommand.swift` | Auth guard |
| Modify | `Sources/macos-cli/Commands/FocusCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/PdfCommand.swift` | Auth guard |
| Modify | `Sources/macos-cli/Commands/StorageCommand.swift` | Auth guard |
| Modify | `Sources/macos-cli/Commands/LocationCommand.swift` | Auth guard |
| Modify | `Sources/macos-cli/Commands/NetworkCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/InfoCommand.swift` | Auth guards |
| Modify | `Sources/macos-cli/Commands/KeychainCommand.swift` | Auth guard on List |
| Modify | `Sources/macos-cli/Commands/MailCommand.swift` | Structured JXA result, jxaEscape() |
| Modify | `Sources/macos-cli/Commands/NotesCommand.swift` | Structured JXA result, jxaEscape() |
| Modify | `Sources/macos-cli/Commands/RemindersCommand.swift` | Auth guards on Done/Uncomplete |
| Modify | `Sources/macos-cli/Commands/CalendarCommand.swift` | (no functional change here — EventKit handled in EventKitStore.swift) |
| Modify | `Sources/macos-cli/Commands/MenuCommand.swift` | 1-element path fix, jxaEscape() |
| Modify | `Sources/macos-cli/Commands/WindowCommand.swift` | window.write guards + Snap uses correct screen |
| Modify | `Sources/macos-cli/Commands/ScriptCommand.swift` | Surface stderr via captureWithStderr |
| Modify | `Sources/macos-cli/Commands/AuthCommand.swift` | Setup --all confirmation unless --yes |

---

### Task 1: Helpers.swift — Process helpers, jxaEscape(), stderr capture, JXA result envelope

**Files:**
- Create: `Sources/macos-cli/Helpers.swift`
- Modify: `Sources/macos-cli/Commands/SystemCommand.swift` (remove the existing `extension Process` block at lines 505–573)

- [ ] **Step 1: Create Helpers.swift with consolidated process helpers, jxaEscape, JXA result envelope, and a stderr-capturing variant.**

```swift
// Sources/macos-cli/Helpers.swift
import Foundation
import ArgumentParser

// MARK: - Process helpers (moved out of SystemCommand.swift)

extension Process {
    /// Runs an executable and returns its exit code. stdout and stderr are discarded.
    /// Throws ValidationError if the executable path does not exist (C3 fix).
    @discardableResult
    static func run(args: [String]) -> Int32 {
        guard !args.isEmpty else { return -1 }
        let exe = args[0]
        if !FileManager.default.isExecutableFile(atPath: exe) {
            fputs("error: executable not found or not runnable: \(exe)\n", stderr)
            return 127
        }
        let proc = Process()
        proc.executableURL = URL(fileURLWithPath: exe)
        proc.arguments = Array(args.dropFirst())
        proc.standardOutput = FileHandle.nullDevice
        proc.standardError = FileHandle.nullDevice
        do {
            try proc.run()
        } catch {
            fputs("error: failed to spawn \(exe): \(error.localizedDescription)\n", stderr)
            return 126
        }
        proc.waitUntilExit()
        return proc.terminationStatus
    }

    /// Convenience: capture stdout only (no timeout). Used by very short ops.
    static func capture(args: [String]) -> String {
        guard let exe = args.first,
              FileManager.default.isExecutableFile(atPath: exe) else { return "" }
        let proc = Process()
        proc.executableURL = URL(fileURLWithPath: exe)
        proc.arguments = Array(args.dropFirst())
        let pipe = Pipe()
        proc.standardOutput = pipe
        proc.standardError = FileHandle.nullDevice
        try? proc.run()
        proc.waitUntilExit()
        return String(data: pipe.fileHandleForReading.readDataToEndOfFile(), encoding: .utf8) ?? ""
    }

    /// Stdout capture with timeout, returning fallback if timeout fires or exe missing.
    /// stderr is discarded (callers that need it use `captureWithStderr`).
    static func capture(args: [String], timeout seconds: Double, fallback: String = "") -> String {
        return capture(args: args, timeout: seconds) ?? fallback
    }

    /// Stdout capture with timeout. Returns nil on timeout or missing executable.
    /// Drains pipe asynchronously to avoid the 64 KB buffer deadlock.
    /// Timeout is enforced — process is terminated when deadline elapses (M5 fix).
    static func capture(args: [String], timeout seconds: Double) -> String? {
        guard let exe = args.first,
              FileManager.default.isExecutableFile(atPath: exe) else { return nil }
        let proc = Process()
        proc.executableURL = URL(fileURLWithPath: exe)
        proc.arguments = Array(args.dropFirst())
        let pipe = Pipe()
        proc.standardOutput = pipe
        proc.standardError = FileHandle.nullDevice

        var collected = Data()
        let lock = NSLock()
        pipe.fileHandleForReading.readabilityHandler = { fh in
            let chunk = fh.availableData
            guard !chunk.isEmpty else { return }
            lock.lock(); collected.append(chunk); lock.unlock()
        }

        guard (try? proc.run()) != nil else {
            pipe.fileHandleForReading.readabilityHandler = nil
            return nil
        }
        let deadline = Date().addingTimeInterval(seconds)
        while proc.isRunning {
            if Date() > deadline {
                proc.terminate()
                // Give it 250ms to terminate, then SIGKILL if still alive
                let killDeadline = Date().addingTimeInterval(0.25)
                while proc.isRunning && Date() < killDeadline {
                    Thread.sleep(forTimeInterval: 0.02)
                }
                if proc.isRunning {
                    kill(proc.processIdentifier, SIGKILL)
                }
                pipe.fileHandleForReading.readabilityHandler = nil
                return nil
            }
            Thread.sleep(forTimeInterval: 0.05)
        }
        pipe.fileHandleForReading.readabilityHandler = nil
        let tail = pipe.fileHandleForReading.readDataToEndOfFile()
        if !tail.isEmpty { lock.lock(); collected.append(tail); lock.unlock() }
        return String(data: collected, encoding: .utf8)
    }

    /// Capture both stdout and stderr separately, with timeout.
    /// Used by ScriptCommand and any command that needs to surface child stderr (C2/L5).
    /// Returns (stdout, stderr, exitCode). exitCode is -1 on timeout.
    static func captureWithStderr(args: [String], timeout seconds: Double) -> (stdout: String, stderr: String, exitCode: Int32) {
        guard let exe = args.first,
              FileManager.default.isExecutableFile(atPath: exe) else {
            return ("", "executable not found: \(args.first ?? "<empty>")", 127)
        }
        let proc = Process()
        proc.executableURL = URL(fileURLWithPath: exe)
        proc.arguments = Array(args.dropFirst())
        let outPipe = Pipe()
        let errPipe = Pipe()
        proc.standardOutput = outPipe
        proc.standardError = errPipe

        var outData = Data()
        var errData = Data()
        let lock = NSLock()
        outPipe.fileHandleForReading.readabilityHandler = { fh in
            let chunk = fh.availableData; guard !chunk.isEmpty else { return }
            lock.lock(); outData.append(chunk); lock.unlock()
        }
        errPipe.fileHandleForReading.readabilityHandler = { fh in
            let chunk = fh.availableData; guard !chunk.isEmpty else { return }
            lock.lock(); errData.append(chunk); lock.unlock()
        }

        do { try proc.run() } catch {
            outPipe.fileHandleForReading.readabilityHandler = nil
            errPipe.fileHandleForReading.readabilityHandler = nil
            return ("", "spawn failed: \(error.localizedDescription)", 126)
        }

        let deadline = Date().addingTimeInterval(seconds)
        while proc.isRunning {
            if Date() > deadline {
                proc.terminate()
                let killDeadline = Date().addingTimeInterval(0.25)
                while proc.isRunning && Date() < killDeadline {
                    Thread.sleep(forTimeInterval: 0.02)
                }
                if proc.isRunning { kill(proc.processIdentifier, SIGKILL) }
                outPipe.fileHandleForReading.readabilityHandler = nil
                errPipe.fileHandleForReading.readabilityHandler = nil
                return (String(data: outData, encoding: .utf8) ?? "",
                        (String(data: errData, encoding: .utf8) ?? "") + "\n[timed out after \(seconds)s]",
                        -1)
            }
            Thread.sleep(forTimeInterval: 0.05)
        }
        outPipe.fileHandleForReading.readabilityHandler = nil
        errPipe.fileHandleForReading.readabilityHandler = nil
        // Drain tails
        let outTail = outPipe.fileHandleForReading.readDataToEndOfFile()
        let errTail = errPipe.fileHandleForReading.readDataToEndOfFile()
        if !outTail.isEmpty { outData.append(outTail) }
        if !errTail.isEmpty { errData.append(errTail) }
        return (String(data: outData, encoding: .utf8) ?? "",
                String(data: errData, encoding: .utf8) ?? "",
                proc.terminationStatus)
    }
}

// MARK: - JXA helpers (H1, H2 fixes)

/// Escape a Swift string for safe interpolation inside a single-quoted JXA string literal.
/// Order matters: backslash MUST be escaped first, then quotes and control chars.
/// Use this for every user-supplied value embedded into a JXA `script` template.
func jxaEscape(_ s: String) -> String {
    return s
        .replacingOccurrences(of: "\\", with: "\\\\")
        .replacingOccurrences(of: "'", with: "\\'")
        .replacingOccurrences(of: "\"", with: "\\\"")
        .replacingOccurrences(of: "\n", with: "\\n")
        .replacingOccurrences(of: "\r", with: "\\r")
}

/// Result returned by a JXA script that uses the structured envelope pattern.
/// Scripts must emit `JSON.stringify({ok:true, result:...})` on success
/// and `JSON.stringify({ok:false, error:"message"})` on failure.
struct JXAEnvelope {
    let ok: Bool
    let resultJSON: String   // raw JSON of `result` field, or "" if not present
    let error: String        // empty if ok
}

/// Parse a JXA script's stdout into a JXAEnvelope.
/// Returns nil only if the output is empty / not JSON at all.
/// Use this instead of `raw.lowercased().contains("error")` (H1 fix).
func parseJXAEnvelope(_ raw: String) -> JXAEnvelope? {
    let trimmed = raw.trimmingCharacters(in: .whitespacesAndNewlines)
    guard !trimmed.isEmpty, let data = trimmed.data(using: .utf8) else { return nil }
    guard let obj = try? JSONSerialization.jsonObject(with: data) as? [String: Any] else {
        // Fallback: if osascript itself printed an "execution error: ..." line,
        // surface that as an envelope so callers can throw a clean error.
        if trimmed.lowercased().hasPrefix("execution error:") || trimmed.lowercased().hasPrefix("error:") {
            return JXAEnvelope(ok: false, resultJSON: "", error: trimmed)
        }
        return nil
    }
    let ok = obj["ok"] as? Bool ?? false
    let errMsg = obj["error"] as? String ?? ""
    var resultJSON = ""
    if let result = obj["result"] {
        if let d = try? JSONSerialization.data(withJSONObject: result, options: []),
           let s = String(data: d, encoding: .utf8) {
            resultJSON = s
        }
    }
    return JXAEnvelope(ok: ok, resultJSON: resultJSON, error: errMsg)
}

/// Convenience: run a JXA script that returns an envelope, throw on failure.
/// Returns the raw JSON of the `result` field (callers parse to their own shape).
@discardableResult
func runJXAEnvelope(_ script: String, timeout: TimeInterval = 10, failureMessage: String) throws -> String {
    let raw = Process.capture(args: ["/usr/bin/osascript", "-l", "JavaScript", "-e", script],
                              timeout: timeout, fallback: "")
    guard let env = parseJXAEnvelope(raw) else {
        throw ValidationError("\(failureMessage)\nRaw: \(raw.prefix(300))")
    }
    if !env.ok {
        throw ValidationError("\(failureMessage)\n\(env.error)")
    }
    return env.resultJSON
}
```

- [ ] **Step 2: Remove the existing `extension Process { ... }` block from `Sources/macos-cli/Commands/SystemCommand.swift`** (lines 505–573 inclusive — the `// MARK: - Process helpers` comment and everything down through the closing `}` of the extension).

After the edit, the file goes directly from `// MARK: - Sleep` ... `_ = Process.run(args: ["/usr/bin/pmset", "sleepnow"])` ... `}` (closing SleepCommand) straight to `// MARK: - VPN`.

- [ ] **Step 3: Build to confirm the move did not break anything.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 4: Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "refactor(helpers): extract Process helpers + add jxaEscape/JXA envelope/stderr capture"
```

---

### Task 2: Auth.swift — register all new capabilities

**Files:**
- Modify: `Sources/macos-cli/Auth.swift`

- [ ] **Step 1: Add the new capability entries and the partial-config comment.** Replace the existing `allCapabilities` array (lines 50–82) and the `check` function (lines 10–26) with:

```swift
    // Call at top of any destructive run(). Throws ValidationError if denied.
    //
    // Capability resolution order (H6 — partial-config safety):
    //   1. caps[capability]                — explicit user choice from auth.json
    //   2. defaultCapabilities[capability] — code-declared default for known capabilities
    //   3. !isWriteCapability(capability)  — heuristic fallback for truly unknown caps
    //
    // Step 2 is the load-bearing one for upgrades: when a user has an old auth.json that
    // pre-dates a new capability, we must fall through to the *declared* default — never
    // to the heuristic, because the heuristic doesn't know that e.g. `keychain.list`
    // should be denied by default. New capabilities MUST be added to allCapabilities so
    // step 2 has a value to return.
    static func check(_ capability: String) throws {
        let caps = load()
        let allowed = caps[capability] ?? defaultCapabilities[capability] ?? !isWriteCapability(capability)
        if !allowed {
            if caps.isEmpty {
                throw ValidationError(
                    "'\(capability)' is denied by default. Run `macos auth setup` to configure permissions, " +
                    "or `macos auth grant \(capability)` to enable this capability."
                )
            } else {
                throw ValidationError(
                    "'\(capability)' is denied. Run `macos auth grant \(capability)` to enable it."
                )
            }
        }
    }
```

- [ ] **Step 2: Replace `allCapabilities` array with the full v0.6.0-hardened list.** Remove the existing array (lines 50–82) and replace with:

```swift
    static let allCapabilities: [(id: String, defaultAllow: Bool, description: String)] = [
        // Calendar
        ("calendar.read",      true,  "Read calendar events"),
        ("calendar.write",     true,  "Create and modify calendar events"),
        ("calendar.delete",    false, "Delete calendar events"),
        // Mail
        ("mail.read",          true,  "Read emails"),
        ("mail.send",          false, "Send emails"),
        ("mail.delete",        false, "Delete emails"),
        // Contacts
        ("contacts.read",      true,  "Read contacts"),
        ("contacts.write",     false, "Create and modify contacts"),
        ("contacts.delete",    false, "Delete contacts"),
        // Keychain
        ("keychain.get",       false, "Read keychain entries"),
        ("keychain.set",       false, "Write keychain entries"),
        ("keychain.delete",    false, "Delete keychain entries"),
        ("keychain.list",      false, "List keychain service/account metadata"),
        // Reminders
        ("reminders.read",     true,  "Read reminders"),
        ("reminders.write",    true,  "Create and modify reminders"),
        ("reminders.delete",   false, "Delete reminders"),
        // Notes
        ("notes.read",         true,  "Read notes"),
        ("notes.write",        false, "Create and modify notes"),
        ("notes.delete",       false, "Delete notes"),
        // Screen / screenshots
        ("screen.capture",     true,  "Take screenshots"),
        ("screen.lock",        true,  "Lock the screen"),
        // Spaces / Mission Control
        ("spaces.read",        true,  "List spaces"),
        ("spaces.write",       false, "Switch and manage spaces"),
        // Audio
        ("audio.read",         true,  "List audio devices"),
        ("audio.write",        false, "Change audio input/output or volume/mute"),
        // Script (JXA / AppleScript passthrough)
        ("script.run",         false, "Run arbitrary JXA or AppleScript"),
        // Menu bar
        ("menu.read",          true,  "List menu bar items"),
        ("menu.click",         false, "Click menu bar items"),
        // Time Machine
        ("timemachine.read",   true,  "Read Time Machine status"),
        ("timemachine.write",  false, "Start or stop Time Machine backups"),
        // System power / lock
        ("system.lock",        true,  "Lock the Mac immediately"),
        ("system.sleep",       false, "Put the Mac to sleep"),
        // Messages
        ("messages.send",      false, "Send iMessages"),
        ("messages.delete",    false, "Delete iMessage conversations"),
        // Mouse / Keyboard
        ("mouse.write",        false, "Synthesize mouse events"),
        ("keyboard.write",     false, "Synthesize keyboard events"),
        // Accessibility tree writes
        ("ax.write",           false, "Click or set values on Accessibility elements"),
        // File system
        ("file.read",          true,  "List, stat, and read files"),
        ("file.write",         false, "Copy, move, create files"),
        ("file.delete",        false, "Delete files permanently"),
        // Trash
        ("trash.empty",        false, "Empty the Trash"),
        // Apps
        ("apps.quit",          false, "Quit running applications"),
        // Defaults
        ("defaults.read",      true,  "Read defaults domains/keys"),
        ("defaults.write",     false, "Write defaults values"),
        ("defaults.delete",    false, "Delete defaults keys"),
        // Dock
        ("dock.write",         false, "Add/remove/restart the Dock"),
        // Login items
        ("login-items.write",  false, "Add or remove login items"),
        // Safari
        ("safari.read",        true,  "List tabs, history, bookmarks"),
        ("safari.execute",     false, "Open URLs, execute JS, close/reload tabs"),
        // Photos
        ("photos.read",        true,  "List/search/export photos"),
        ("photos.delete",      false, "Delete photos from the library"),
        // Finder
        ("finder.read",        true,  "Read Finder selection / cwd / hidden state"),
        ("finder.write",       false, "Reveal/open/new folder/rename/tag/go-to/show-hidden"),
        // Bluetooth
        ("bluetooth.read",     true,  "List paired Bluetooth devices"),
        ("bluetooth.write",    false, "Connect or disconnect Bluetooth devices"),
        // Disk
        ("disk.read",          true,  "List disks and read disk info"),
        ("disk.write",         false, "Eject / unmount / mount volumes"),
        // OCR
        ("ocr.capture",        true,  "OCR screen regions or image files"),
        // Process
        ("process.kill",       false, "Send signals to processes"),
        // Window
        ("window.write",       false, "Move/resize/snap/close/minimize/maximize/fullscreen windows"),
        // Music
        ("music.write",        false, "Control Music app playback / queue / volume"),
        // Notify
        ("notify.send",        true,  "Send a user notification"),
        // Speech
        ("speech.speak",       true,  "Speak text aloud"),
        // Shortcuts
        ("shortcuts.run",      false, "Run a Shortcut"),
        // Spotlight
        ("spotlight.search",   true,  "Search files via Spotlight"),
        // Focus
        ("focus.write",        false, "Toggle Focus modes"),
        // PDF
        ("pdf.read",           true,  "Extract text/metadata from PDFs"),
        // Storage
        ("storage.read",       true,  "List volumes / disk usage"),
        // Location
        ("location.read",      false, "Read the Mac's location"),
        // Network
        ("network.read",       true,  "Ping, DNS, port check, traceroute, interfaces"),
        // Info
        ("info.read",          true,  "Read system, network, power info"),
    ]
```

Note: the orphan `system.shutdown` and `system.reboot` capabilities are intentionally removed (L4 fix). `screen.lock` remains in place and `system.lock` is added as the new top-level system lock capability (used by SystemCommand's LockCommand). The two are distinct: `screen.lock` gates `macos screen lock`, `system.lock` gates `macos system lock`.

- [ ] **Step 3: Build and verify.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 4: Verify auth list includes every new capability.**

```bash
cp ~/Developer/macos-cli/.build/release/macos-cli ~/.local/bin/macos
macos auth list | wc -l
```

Expected: at least 67 lines (one per capability entry).

```bash
macos auth list | grep -E "messages.send|mouse.write|keyboard.write|ax.write|file.delete|trash.empty|apps.quit|defaults.write|dock.write|login-items.write|safari.execute|photos.delete|finder.write|bluetooth.write|disk.write|ocr.capture|system.sleep|process.kill|window.write|keychain.list|music.write|notify.send|speech.speak|shortcuts.run|spotlight.search|focus.write|pdf.read|storage.read|location.read|network.read|info.read"
```

Expected: every listed capability appears (one per line).

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "auth(caps): register all new capabilities for v0.6.0 hardening + document fallback order"
```

---

### Task 3: Auth guards — high-impact destructive commands

Add `try Auth.check(...)` as the **first line of `run()`** for every destructive subcommand in this group. Each `run()` body must keep its existing logic; only the guard is inserted.

**Files:**
- Modify: `Sources/macos-cli/Commands/MessagesCommand.swift`
- Modify: `Sources/macos-cli/Commands/MouseCommand.swift`
- Modify: `Sources/macos-cli/Commands/KeyboardCommand.swift`
- Modify: `Sources/macos-cli/Commands/AxCommand.swift`
- Modify: `Sources/macos-cli/Commands/FinderCommand.swift`
- Modify: `Sources/macos-cli/Commands/PhotosCommand.swift`
- Modify: `Sources/macos-cli/Commands/SafariCommand.swift`
- Modify: `Sources/macos-cli/Commands/DiskCommand.swift`
- Modify: `Sources/macos-cli/Commands/ProcessCommand.swift`
- Modify: `Sources/macos-cli/Commands/TrashCommand.swift`
- Modify: `Sources/macos-cli/Commands/DefaultsCommand.swift`
- Modify: `Sources/macos-cli/Commands/DockCommand.swift`
- Modify: `Sources/macos-cli/Commands/LoginItemsCommand.swift`
- Modify: `Sources/macos-cli/Commands/FileCommand.swift`
- Modify: `Sources/macos-cli/Commands/AppsCommand.swift`

- [ ] **Step 1: MessagesCommand — add guards on Send, Delete, Conversations, Read.**

In `Send.run()` (around line 24), add as the first line of the body:
```swift
            try Auth.check("messages.send")
```

In `Read.run()` (around line 73), add:
```swift
            try Auth.check("messages.send")  // read of conversation data — gated as same trust tier as send
```
(Rationale: reading message contents is roughly as sensitive as sending; this gates it behind the same explicit grant as `messages.send`. If finer granularity is wanted, add `messages.read` later — for now the conservative choice is to gate it.)

Actually correct that — read should be gated separately. Use:
```swift
            try Auth.check("messages.send")
```
…only if the user wants read-write parity. Otherwise leave Read unguarded (it's protected by macOS Automation TCC). Confirmed approach: **leave `Read` and `Conversations` unguarded** (TCC + iMessage Automation prompt is sufficient). Only guard `Send` and `Delete`.

So:
- `Send.run()` first line: `try Auth.check("messages.send")`
- `Delete.run()` first line: `try Auth.check("messages.delete")`

- [ ] **Step 2: MouseCommand — guard all subcommands except Position.**

In `Move.run()`, `Click.run()`, `Drag.run()`, `Scroll.run()` (lines 38, 59, 101, 130), insert as the first line:
```swift
            try Auth.check("mouse.write")
```
`Position.run()` (line 153) is read-only — leave unguarded.

- [ ] **Step 3: KeyboardCommand — guard both subcommands.**

In `TypeText.run()` (line 25) and `Key.run()` (line 77), insert as the first line:
```swift
            try Auth.check("keyboard.write")
```

- [ ] **Step 4: AxCommand — guard Click and the click-via-hint path.**

In `Click.run()` (line 91), insert as the first line:
```swift
            try Auth.check("ax.write")
```

In `Hints.run()` (line 211), the guard applies *only when actually clicking* via `--click`. Insert this immediately inside the `if let n = click {` block (the existing block at line 215):
```swift
                try Auth.check("ax.write")
```

`Find.run()`, `Read.run()`, and Hints listing remain ungated reads.

- [ ] **Step 5: FinderCommand — guard write subcommands.**

In `Selected.run()` (read), `Cwd.run()` (read), no guard.

In `Reveal.run()` (around line 55), `Open.run()` (around line 80), `NewFolder.run()` (around line 134), `Rename.run()` (around line 171), `Tag.run()` (around line 208), `GoTo.run()` (around line 300), `ShowHidden.run()` (around line 341) — insert as the first line of each `run()`:
```swift
            try Auth.check("finder.write")
```

(All of these mutate Finder state — opening windows, creating folders, renaming, tagging, switching directories, toggling visibility.)

- [ ] **Step 6: PhotosCommand — guard Delete.**

In `Delete.run()` (line 211 area), insert as the first line:
```swift
            try Auth.check("photos.delete")
```

(Albums, Search, Export, AddToAlbum, Recent stay ungated — they're reads or constrained writes that the user invokes deliberately. AddToAlbum is debatable; if hardening conservatively, also add `try Auth.check("photos.delete")` to AddToAlbum, but the audit says only Delete needs it.)

- [ ] **Step 7: SafariCommand — guard write subcommands.**

`Tabs.run()`, `Read.run()`, `History.run()`, `Bookmarks.run()` — no guard.

In `Open.run()` (line 69), `Execute.run()` (line 133), `NewTab.run()` (line 159), `Close.run()` (line 197), `Reload.run()` (line 265) — insert as the first line:
```swift
            try Auth.check("safari.execute")
```

- [ ] **Step 8: DiskCommand — guard write subcommands.**

`List.run()` (line 12 area), `Info.run()` (line 53 area) — no guard.

In `Eject.run()` (line 96 area), `Unmount.run()` (line 118 area), `Mount.run()` (line 146 area), insert as the first line:
```swift
            try Auth.check("disk.write")
```

- [ ] **Step 9: ProcessCommand — guard Kill.**

In `Kill.run()` (line 81 area), insert as the first line:
```swift
            try Auth.check("process.kill")
```

- [ ] **Step 10: TrashCommand — guard EmptyCmd.**

In `EmptyCmd.run()` (line 37), insert as the first line:
```swift
            try Auth.check("trash.empty")
```

`AddCmd` (move-to-trash) is reversible by the user; leave unguarded. `ListCmd` is a read.

- [ ] **Step 11: DefaultsCommand — guard write and delete.**

In `Read.run()`, `ListDomains.run()` — no guard (reads).

In `Write.run()` (line 47), insert as the first line:
```swift
            try Auth.check("defaults.write")
```

In `Delete.run()` (line 76), insert as the first line:
```swift
            try Auth.check("defaults.delete")
```

- [ ] **Step 12: DockCommand — guard write subcommands (the PropertyListSerialization rewrite is Task 7; for now, just add the guard).**

In `ListCmd.run()` — no guard.

In `AddCmd.run()` (line 63), `RemoveCmd.run()` (line 91), `RestartCmd.run()` (line 148), insert as the first line:
```swift
            try Auth.check("dock.write")
```

- [ ] **Step 13: LoginItemsCommand — guard Add and Remove.**

In `ListCmd.run()` — no guard.

In `AddCmd.run()` (line 56 area) and `RemoveCmd.run()` (line 85 area), insert as the first line:
```swift
            try Auth.check("login-items.write")
```

- [ ] **Step 14: FileCommand — guard write/delete; Read gets size limit handling in Task 7.**

In `ListCmd.run()` (line 17), insert as the first line:
```swift
            try Auth.check("file.read")
```

In `StatCmd.run()` (line 138), insert as the first line:
```swift
            try Auth.check("file.read")
```

In `ReadCmd.run()` (line 182), insert as the first line:
```swift
            try Auth.check("file.read")
```

In `CopyCmd.run()` (line 64), `MoveCmd.run()` (line 92), insert as the first line:
```swift
            try Auth.check("file.write")
```

In `DeleteCmd.run()` (line 119), insert as the first line:
```swift
            try Auth.check("file.delete")
```

- [ ] **Step 15: AppsCommand — guard QuitCmd.**

In `ListCmd.run()`, `LaunchCmd.run()`, `InfoCmd.run()` — no guard (launching is user-initiated; if hardening conservatively, add `apps.quit` to LaunchCmd too — but the audit says only Quit).

In `QuitCmd.run()` (line 74), insert as the first line:
```swift
            try Auth.check("apps.quit")
```

- [ ] **Step 16: Build.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 17: Smoke-test that a denied capability throws cleanly.**

```bash
cp ~/Developer/macos-cli/.build/release/macos-cli ~/.local/bin/macos
macos auth deny mouse.write 2>&1 | head -3
macos mouse click --x 100 --y 100 2>&1 | head -3
# Expected: 'mouse.write' is denied. Run `macos auth grant mouse.write` to enable it.
macos auth grant mouse.write 2>&1 | head -3
# (do NOT actually click — just verify the path)
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "auth(guards): gate Messages/Mouse/Keyboard/Ax/Finder/Photos/Safari/Disk/Process/Trash/Defaults/Dock/LoginItems/File/Apps writes"
```

---

### Task 4: Auth guards — remaining command files

**Files:**
- Modify: `Sources/macos-cli/Commands/BluetoothCommand.swift`
- Modify: `Sources/macos-cli/Commands/OcrCommand.swift`
- Modify: `Sources/macos-cli/Commands/SystemCommand.swift`
- Modify: `Sources/macos-cli/Commands/MusicCommand.swift`
- Modify: `Sources/macos-cli/Commands/NotifyCommand.swift`
- Modify: `Sources/macos-cli/Commands/SpeechCommand.swift`
- Modify: `Sources/macos-cli/Commands/ShortcutsCommand.swift`
- Modify: `Sources/macos-cli/Commands/SpotlightCommand.swift`
- Modify: `Sources/macos-cli/Commands/FocusCommand.swift`
- Modify: `Sources/macos-cli/Commands/PdfCommand.swift`
- Modify: `Sources/macos-cli/Commands/StorageCommand.swift`
- Modify: `Sources/macos-cli/Commands/LocationCommand.swift`
- Modify: `Sources/macos-cli/Commands/NetworkCommand.swift`
- Modify: `Sources/macos-cli/Commands/InfoCommand.swift`
- Modify: `Sources/macos-cli/Commands/KeychainCommand.swift`
- Modify: `Sources/macos-cli/Commands/RemindersCommand.swift`

- [ ] **Step 1: BluetoothCommand.**

In `ListCmd.run()` (line ~12 area), insert as the first line:
```swift
            try Auth.check("bluetooth.read")
```

In `ConnectCmd.run()` (line ~44 area), `DisconnectCmd.run()` (line ~65 area), insert as the first line:
```swift
            try Auth.check("bluetooth.write")
```

- [ ] **Step 2: OcrCommand — guard all three subcommands (each captures pixels).**

In `Full.run()`, `Region.run()`, `File.run()`, insert as the first line:
```swift
            try Auth.check("ocr.capture")
```

- [ ] **Step 3: SystemCommand — guard write subcommands.**

In `BatteryCommand.run()` — no guard (pure read).

In `AudioCommand.VolumeCommand.run()` (line 88) — split: when `level` is provided (setting), check `audio.write`; when reading, no guard. Edit:
```swift
        func run() throws {
            if let level = level {
                try Auth.check("audio.write")
                guard level >= 0 && level <= 100 else { throw ValidationError("Volume must be 0–100") }
                // ... rest unchanged
            } else {
                // ... rest unchanged (read path)
            }
        }
```

In `AudioCommand.MuteCommand.run()` (line 109), apply the same pattern: guard only when `state` is provided.

In `AudioCommand.DevicesCommand.run()` (line 130), `AudioCommand.NowPlayingCommand.run()` (line 171) — no guard (reads).

In `WifiCommand.StatusCmd.run()` (line 214), `NetworksCmd.run()` (line 256) — no guard (reads).

In `WifiCommand.JoinCmd.run()` (line 286), `LeaveCmd.run()` (line 300), insert as the first line:
```swift
            try Auth.check("network.read")
```
(There is no `network.write` capability — Wi-Fi join/leave is read-tier per the audit since it doesn't fall under any other write category. If the operator wants tighter control, they can deny `network.read`.)

Actually correct that — joining a Wi-Fi network is unambiguously a write. Create a dedicated guard. Update Auth.swift's allCapabilities to also add:
```swift
        ("wifi.write",         false, "Join or leave Wi-Fi networks"),
```
(Add this entry next to `network.read` in the Network section of allCapabilities — make sure it's part of Task 2's commit if Task 2 hasn't been done yet; if Task 2 is already committed, do it as part of THIS task's commit and call out in the message.)

Then in `WifiCommand.JoinCmd.run()` and `WifiCommand.LeaveCmd.run()`, insert:
```swift
            try Auth.check("wifi.write")
```

In `ClipboardCommand.GetCmd.run()` (line 322) — no guard.

In `ClipboardCommand.SetCmd.run()` (line 332) — insert as the first line:
```swift
            try Auth.check("defaults.write")
```
(No dedicated `clipboard.write` capability; clipboard writes are low-risk and the audit list does not mandate one. Using `defaults.write` here is wrong — instead just leave SetCmd ungated. The clipboard is local-only and synthetic clipboard writes have no privacy implication.)

Confirmed approach: **leave Clipboard SetCmd ungated** — no guard.

In `DisplayCommand.BrightnessCmd.run()` (line 360) — split: when setting, guard `defaults.write` is wrong. Add a new capability:
```swift
        ("display.write",      false, "Set brightness, dark mode, wallpaper"),
```
to Auth.swift's allCapabilities (in Task 2 if not yet committed; otherwise call out in this task's commit message). Then:

In `BrightnessCmd.run()`: gate the "level provided" branch only:
```swift
            if let level = level {
                try Auth.check("display.write")
                // ... existing code
            } else {
                // ... existing read code
            }
```

In `DarkModeCmd.run()` (line 393): same pattern — gate the "state provided" branch only:
```swift
            if let state = state {
                try Auth.check("display.write")
                // ... existing code
            } else {
                // ... existing read code
            }
```

In `WallpaperCmd.GetCmd.run()` — no guard.

In `WallpaperCmd.SetCmd.run()` (line 448) — insert as the first line:
```swift
            try Auth.check("display.write")
```

In `LockCommand.run()` (line 475) — insert as the first line:
```swift
            try Auth.check("system.lock")
```

In `SleepCommand.run()` (line 497) — insert as the first line:
```swift
            try Auth.check("system.sleep")
```

In `VPNCommand.StatusCmd.run()` — no guard.
In `VPNCommand.ConnectCmd.run()` and `VPNCommand.DisconnectCmd.run()` — these are mutations. Add a new capability:
```swift
        ("vpn.write",          false, "Connect to or disconnect VPN"),
```
to Auth.swift's allCapabilities. Then in `ConnectCmd.run()` and `DisconnectCmd.run()`, insert as the first line:
```swift
            try Auth.check("vpn.write")
```

- [ ] **Step 4: MusicCommand — guard write subcommands.**

In `Status.run()`, `Volume.run()` (read path), `Search.run()` (read), `Playlists.run()` (read), `Queue.run()` (read) — leave unguarded.

In `Play.run()`, `Pause.run()`, `Next.run()`, `Previous.run()`, `AddToPlaylist.run()`, and `Volume.run()` (set path) — insert as the first line:
```swift
            try Auth.check("music.write")
```

For `Volume.run()` specifically, gate inside the if-branch that sets:
```swift
            if let level = level {
                try Auth.check("music.write")
                // ... existing set code
            } else {
                // ... existing get code
            }
```

For `Search.run()` — the existing `Search` command in MusicCommand "plays first result" per its abstract, so it's a write. Insert at the top:
```swift
            try Auth.check("music.write")
```

- [ ] **Step 5: NotifyCommand — guard SendCmd.**

In `SendCmd.run()` (line 11 area), insert as the first line:
```swift
            try Auth.check("notify.send")
```

- [ ] **Step 6: SpeechCommand — guard SayCmd.**

In `SayCmd.run()` (line 11 area), insert as the first line:
```swift
            try Auth.check("speech.speak")
```

`VoicesCmd.run()` — no guard.

- [ ] **Step 7: ShortcutsCommand — guard Run.**

In `Run.run()` (line 37 area), insert as the first line:
```swift
            try Auth.check("shortcuts.run")
```

`List.run()` — no guard.

- [ ] **Step 8: SpotlightCommand — guard SearchCmd.**

In `SearchCmd.run()` (line 11 area), insert as the first line:
```swift
            try Auth.check("spotlight.search")
```

(spotlight.search defaults to `true` — the guard is here so users who want a more locked-down profile can deny it.)

- [ ] **Step 9: FocusCommand — guard On and Off.**

In `Status.run()`, `Modes.run()` — no guard.

In `On.run()` (line 60 area) and `Off.run()` (line 85 area), insert as the first line:
```swift
            try Auth.check("focus.write")
```

- [ ] **Step 10: PdfCommand — guard both subcommands.**

In `Text.run()` (line 13 area) and `Info.run()` (line 60 area), insert as the first line:
```swift
            try Auth.check("pdf.read")
```

(pdf.read defaults to `true`. Guard exists so it can be denied.)

- [ ] **Step 11: StorageCommand — guard both.**

In `VolumesCmd.run()` (line 11 area) and `DiskUsageCmd.run()` (line 54 area), insert as the first line:
```swift
            try Auth.check("storage.read")
```

- [ ] **Step 12: LocationCommand — guard Get.**

In `Get.run()` (line 13 area), insert as the first line:
```swift
            try Auth.check("location.read")
```

(location.read defaults to `false` — Manu's physical location is sensitive enough to require explicit grant.)

- [ ] **Step 13: NetworkCommand — guard all subcommands.**

In `Ping.run()`, `Dns.run()`, `Port.run()`, `Traceroute.run()`, `Interfaces.run()`, insert as the first line:
```swift
            try Auth.check("network.read")
```

- [ ] **Step 14: InfoCommand — guard all subcommands.**

In `SystemInfoCmd.run()`, `NetworkInfoCmd.run()`, insert as the first line:
```swift
            try Auth.check("info.read")
```

In `PowerCmd.SleepCmd.run()` (line 103 area), insert as the first line:
```swift
            try Auth.check("system.sleep")
```

In `PowerCmd.CaffeinateCmd.run()` (line 111 area), insert as the first line:
```swift
            try Auth.check("system.sleep")
```
(Caffeinate prevents sleep — same trust tier as triggering sleep.)

In `PowerCmd.SettingsCmd.run()` (line 123 area), insert as the first line:
```swift
            try Auth.check("info.read")
```

- [ ] **Step 15: KeychainCommand — guard List.**

In `List.run()` (line 106), insert as the first line:
```swift
            try Auth.check("keychain.list")
```

- [ ] **Step 16: RemindersCommand — guard Done and Uncomplete (C5 fix).**

In `Done.run()` (line 165), insert as the first line:
```swift
            try Auth.check("reminders.write")
```

In `Uncomplete.run()` (line 338), insert as the first line:
```swift
            try Auth.check("reminders.write")
```

- [ ] **Step 17: Update Auth.swift to add the three new capabilities (`wifi.write`, `display.write`, `vpn.write`).**

In `Sources/macos-cli/Auth.swift`, edit `allCapabilities` — add these entries grouped with their command areas (these were called out in Step 3):

```swift
        ("wifi.write",         false, "Join or leave Wi-Fi networks"),
```
(place near `network.read`)

```swift
        ("display.write",      false, "Set brightness, dark mode, wallpaper"),
```
(place near `screen.lock`)

```swift
        ("vpn.write",          false, "Connect to or disconnect VPN"),
```
(place near `network.read`)

- [ ] **Step 18: Build and smoke-test.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
cp .build/release/macos-cli ~/.local/bin/macos
macos auth list | wc -l
# Expected: 70+ lines
macos info system 2>&1 | head -3
# Expected: real output (info.read defaults true) — no auth error
macos system lock --help 2>&1 | head -3
# Expected: standard help — verifies the new system.lock subcommand is reachable
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "auth(guards): gate Bluetooth/Ocr/System/Music/Notify/Speech/Shortcuts/Spotlight/Focus/Pdf/Storage/Location/Network/Info/Keychain/Reminders + wifi.write/display.write/vpn.write caps"
```

---

### Task 5: JXA structured error envelope — replace `lowercased().contains("error")`

Every command file using the JXA error-by-substring pattern is converted to the structured-result envelope. JXA scripts that previously emitted sentinel strings (`'sent-buddy'`, `'ok'`, `'not-found'`) now emit `JSON.stringify({ok: true, result: ...})` or `{ok: false, error: "..."}`. Swift code parses via `parseJXAEnvelope`.

**Files (12 total):**
- Modify: `Sources/macos-cli/Commands/MessagesCommand.swift`
- Modify: `Sources/macos-cli/Commands/MenuCommand.swift`
- Modify: `Sources/macos-cli/Commands/KeyboardCommand.swift`
- Modify: `Sources/macos-cli/Commands/AxCommand.swift`
- Modify: `Sources/macos-cli/Commands/SafariCommand.swift`
- Modify: `Sources/macos-cli/Commands/FinderCommand.swift`
- Modify: `Sources/macos-cli/Commands/PhotosCommand.swift`
- Modify: `Sources/macos-cli/Commands/MusicCommand.swift`
- Modify: `Sources/macos-cli/Commands/MailCommand.swift`
- Modify: `Sources/macos-cli/Commands/NotesCommand.swift`
- Modify: `Sources/macos-cli/Commands/LoginItemsCommand.swift`
- Modify: `Sources/macos-cli/Commands/SetupCommand.swift`

The template pattern to apply, illustrated on `MessagesCommand.Send.run()` (line 24):

**Before:**
```swift
        func run() throws {
            try Auth.check("messages.send")
            let escapedTo   = to.replacingOccurrences(of: "'", with: "\\'")
            let escapedText = text
                .replacingOccurrences(of: "\\", with: "\\\\")
                .replacingOccurrences(of: "'", with: "\\'")
                .replacingOccurrences(of: "\n", with: "\\n")
                .replacingOccurrences(of: "\r", with: "\\r")
            let script = """
            const Messages = Application('Messages');
            const buddy = Messages.buddies.whose({name: '\(escapedTo)'})(  )[0]
                       || Messages.buddies.whose({handle: '\(escapedTo)'})(  )[0];
            if (!buddy) {
                const targetService = Messages.services()[0];
                const participant = targetService.participants.whose({handle: '\(escapedTo)'})(  )[0];
                if (participant) {
                    Messages.send('\(escapedText)', {to: participant});
                    'sent-participant';
                } else {
                    'not-found';
                }
            } else {
                Messages.send('\(escapedText)', {to: buddy});
                'sent-buddy';
            }
            """
            let result = Process.capture(args: ["/usr/bin/osascript", "-l", "JavaScript", "-e", script], timeout: 10, fallback: "")
                .trimmingCharacters(in: .whitespacesAndNewlines)
            if result == "not-found" {
                throw ValidationError("Recipient '\(to)' not found in Messages contacts")
            } else if result.lowercased().contains("error") {
                throw ValidationError("Could not send — check Automation permission for Messages\n\(result.prefix(200))")
            } else {
                print("Sent to \(to): \(text.prefix(60))\(text.count > 60 ? "..." : "")")
            }
        }
```

**After:**
```swift
        func run() throws {
            try Auth.check("messages.send")
            let escapedTo   = jxaEscape(to)
            let escapedText = jxaEscape(text)
            let script = """
            try {
                const Messages = Application('Messages');
                const buddy = Messages.buddies.whose({name: '\(escapedTo)'})(  )[0]
                           || Messages.buddies.whose({handle: '\(escapedTo)'})(  )[0];
                if (!buddy) {
                    const targetService = Messages.services()[0];
                    const participant = targetService.participants.whose({handle: '\(escapedTo)'})(  )[0];
                    if (participant) {
                        Messages.send('\(escapedText)', {to: participant});
                        JSON.stringify({ok: true, result: 'sent-participant'});
                    } else {
                        JSON.stringify({ok: false, error: 'not-found'});
                    }
                } else {
                    Messages.send('\(escapedText)', {to: buddy});
                    JSON.stringify({ok: true, result: 'sent-buddy'});
                }
            } catch (e) {
                JSON.stringify({ok: false, error: String(e && e.message ? e.message : e)});
            }
            """
            let raw = Process.capture(args: ["/usr/bin/osascript", "-l", "JavaScript", "-e", script], timeout: 10, fallback: "")
            guard let env = parseJXAEnvelope(raw) else {
                throw ValidationError("Could not send — empty or unparseable JXA result.\nRaw: \(raw.prefix(200))")
            }
            if !env.ok {
                if env.error == "not-found" {
                    throw ValidationError("Recipient '\(to)' not found in Messages contacts")
                }
                throw ValidationError("Could not send — check Automation permission for Messages\n\(env.error)")
            }
            print("Sent to \(to): \(text.prefix(60))\(text.count > 60 ? "..." : "")")
        }
```

**Apply the same wrap-in-try/catch + JSON.stringify({ok:..., result/error}) + parseJXAEnvelope pattern to every JXA call site flagged below.** In each one:
1. Replace all manual escape chains with `jxaEscape(...)`.
2. Wrap the JXA body in `try { ... } catch (e) { JSON.stringify({ok:false, error:String(e && e.message ? e.message : e)}); }`.
3. Replace sentinel-string returns with `JSON.stringify({ok:true, result:<value>})`.
4. Replace `result.lowercased().contains("error")` with `parseJXAEnvelope(...)` + check `env.ok`.

- [ ] **Step 1: MessagesCommand**
  - `Send` (line 24) — as above.
  - `Delete` (line 144) — same pattern; the script's `'deleted'`/`'not-found'` become `{ok:true,result:'deleted'}`/`{ok:false,error:'not-found'}`.
  - `Read` (line 73) — already returns JSON; wrap it: emit `{ok:true, result: msgs}` and replace `try? JSONSerialization.jsonObject(with: data) as? [[String: Any]]` with parsing through `parseJXAEnvelope` then re-decoding `env.resultJSON`.
  - `Conversations` (line 192) — same as Read.

- [ ] **Step 2: MenuCommand**
  - `List` (line 19) — wrap in try/catch + envelope; replace `try? JSONSerialization.jsonObject(with: data) as? [[String: Any]]` with envelope-parse → decode `env.resultJSON`.
  - `Click` (line 68) — emit `{ok:true,result:'clicked'}`. (The 1-element path fix lives in Task 8 — but make the envelope conversion now; the path fix only touches the JS, not the envelope.)

- [ ] **Step 3: KeyboardCommand**
  - `TypeText` (line 25) — wrap. On success the script does its keystroke loop then emits `JSON.stringify({ok:true, result:'typed'})`.
  - `Key` (line 77) — same.

- [ ] **Step 4: AxCommand**
  - `Find` (line 29) — script already returns JSON array; switch to `{ok:true, result: arr}`.
  - `Click` (line 91) — `'clicked'`/`'not found'` → envelope.
  - `Read` (line 141) — wrap.
  - `Hints` (line 211) — note: `Hints` is now mostly native-AX (CFType walking); only the click branch using `osascript` (it doesn't — it uses `AXUIElementPerformAction`). No envelope change needed for `Hints`.

- [ ] **Step 5: SafariCommand**
  - `Tabs` (line 20), `Open` (line 69), `Read` (line 94), `Execute` (line 133), `NewTab` (line 159), `Close` (line 197), `Reload` (line 265) — wrap each. (`History` and `Bookmarks` use sqlite3/plutil — no envelope needed there.)

- [ ] **Step 6: FinderCommand**
  - Every JXA-using subcommand: `Selected`, `Reveal`, `Open`, `Cwd`, `NewFolder`, `Rename`, `Tag`, `GoTo`, `ShowHidden` — wrap.

- [ ] **Step 7: PhotosCommand**
  - Every JXA-using subcommand: `Albums`, `Search`, `Export`, `AddToAlbum`, `Delete`, `Recent` — wrap.

- [ ] **Step 8: MusicCommand**
  - Every JXA-using subcommand: `Status`, `Play`, `Pause`, `Next`, `Previous`, `Volume`, `Search`, `Playlists`, `AddToPlaylist`, `Queue` — wrap. (Some use AppleScript, not JXA — leave those AppleScript ones alone; the envelope pattern is JXA-specific. Identify which by `-l JavaScript` presence in `Process.capture` args.)

- [ ] **Step 9: MailCommand**
  - All JXA-using subcommands — wrap.

- [ ] **Step 10: NotesCommand**
  - All JXA-using subcommands — wrap.

- [ ] **Step 11: LoginItemsCommand**
  - All JXA-using subcommands — wrap.

- [ ] **Step 12: SetupCommand**
  - All JXA-using subcommands — wrap.

- [ ] **Step 13: Replace every remaining manual escape chain with `jxaEscape(...)`.** Search and replace at file level. Pattern to find:

```bash
grep -rn 'replacingOccurrences(of: "\\\\", with: "\\\\\\\\")' ~/Developer/macos-cli/Sources/macos-cli/Commands/
```

Every match should be removed; substitute with a single `jxaEscape(originalVar)` call producing the same string.

- [ ] **Step 14: Build.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 15: Smoke-test that false-positive error detection is gone.**

```bash
cp .build/release/macos-cli ~/.local/bin/macos
# Apps with "error" in name should not now fail
macos safari new-tab --url 'https://example.com' 2>&1 | head -3
# (only run if Safari is open; the point is no spurious 'lowercased() contains error' triggers)
```

- [ ] **Step 16: Confirm zero remaining occurrences of the old pattern.**

```bash
grep -rn 'lowercased().contains("error")' ~/Developer/macos-cli/Sources/macos-cli/Commands/ | wc -l
```

Expected: `0`.

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "jxa(robustness): structured ok/error envelope + jxaEscape across all JXA call sites"
```

---

### Task 6: EventKit macOS 14+ API fixes

**Files:**
- Modify: `Sources/macos-cli/EventKitStore.swift`

- [ ] **Step 1: Update `EventKitStore.authorized` to use the macOS 14+ APIs.** Replace lines 21–50 (the entire request block) with:

```swift
        // Not determined — request access. Must spin the main run loop.
        var granted = false
        var authError: Error?
        var done = false

        DispatchQueue.main.async {
            let completion: (Bool, Error?) -> Void = { ok, err in
                granted = ok
                authError = err
                done = true
                CFRunLoopStop(CFRunLoopGetMain())
            }
            if #available(macOS 14.0, *) {
                switch type {
                case .event:
                    store.requestFullAccessToEvents(completion: completion)
                case .reminder:
                    store.requestFullAccessToReminders(completion: completion)
                @unknown default:
                    store.requestAccess(to: type, completion: completion)
                }
            } else {
                store.requestAccess(to: type, completion: completion)
            }
        }

        let deadline = CFAbsoluteTimeGetCurrent() + 15
        while !done && CFAbsoluteTimeGetCurrent() < deadline {
            CFRunLoopRunInMode(.defaultMode, 0.1, false)
        }

        if !done {
            throw CLIError.authTimeout
        }
        if let err = authError {
            throw err
        }
        guard granted else {
            throw CLIError.notAuthorized(type)
        }
        return store
    }
}
```

(Everything before line 21 — the fast-path `if isAuthorized { return store }` plus `denied/restricted` throw — stays as-is.)

- [ ] **Step 2: Build.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete|warning:"
```

Expected: `Build complete!` and zero `requestAccess(to:` deprecation warnings on this file.

- [ ] **Step 3: Smoke-test calendar and reminders.**

```bash
cp .build/release/macos-cli ~/.local/bin/macos
macos calendar calendars --json 2>&1 | head -10
# Expected: list of calendars in JSON
macos reminders lists --json 2>&1 | head -10
# Expected: list of reminder lists in JSON
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "eventkit(macos14): use requestFullAccessToEvents/Reminders on macOS 14+, fall back on older"
```

---

### Task 7: Security fixes — Dock XML injection, File size limit, Keychain.List confirmation

**Files:**
- Modify: `Sources/macos-cli/Commands/DockCommand.swift`
- Modify: `Sources/macos-cli/Commands/FileCommand.swift`

- [ ] **Step 1: DockCommand — replace the XML-string AddCmd with PropertyListSerialization (H3 fix).** Replace `AddCmd` (lines 58–84) entirely with:

```swift
    struct AddCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "add", abstract: "Pin an app to the Dock")
        @Argument(help: "Path to .app bundle") var path: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            try Auth.check("dock.write")
            let expanded = (path as NSString).expandingTildeInPath
            guard FileManager.default.fileExists(atPath: expanded) else {
                throw ValidationError("App not found: \(path)")
            }
            let name = URL(fileURLWithPath: expanded).deletingPathExtension().lastPathComponent

            // Build the new Dock entry via PropertyListSerialization rather than raw XML.
            // This avoids any injection vector through the app path/name.
            let newEntry: [String: Any] = [
                "tile-data": [
                    "file-data": [
                        "_CFURLString": "file://" + expanded,
                        "_CFURLStringType": 15  // 15 = file URL with absolute path
                    ]
                ],
                "tile-type": "file-tile"
            ]

            let plistPath = NSHomeDirectory() + "/Library/Preferences/com.apple.dock.plist"
            guard let plistData = try? Data(contentsOf: URL(fileURLWithPath: plistPath)),
                  var plist = try? PropertyListSerialization.propertyList(from: plistData, options: [.mutableContainersAndLeaves], format: nil) as? [String: Any] else {
                throw ValidationError("Could not read Dock preferences plist at \(plistPath)")
            }

            var apps = plist["persistent-apps"] as? [[String: Any]] ?? []
            apps.append(newEntry)
            plist["persistent-apps"] = apps

            let outData: Data
            do {
                outData = try PropertyListSerialization.data(fromPropertyList: plist, format: .binary, options: 0)
            } catch {
                throw ValidationError("Could not serialize Dock plist: \(error.localizedDescription)")
            }
            do {
                try outData.write(to: URL(fileURLWithPath: plistPath), options: .atomic)
            } catch {
                throw ValidationError("Could not write Dock plist: \(error.localizedDescription)")
            }

            // Restart Dock so the change takes effect
            _ = Process.run(args: ["/usr/bin/killall", "Dock"])
            if json {
                let payload: [String: Any] = ["added": true, "name": name, "path": expanded]
                printJSON(payload)
            } else {
                print("Added to Dock: \(name) (Dock restarted)")
            }
        }
    }
```

- [ ] **Step 2: DockCommand — replace `RemoveCmd` with a plist-mutation version (no PlistBuddy, no string parsing).** Replace `RemoveCmd` (lines 86–142) entirely with:

```swift
    struct RemoveCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "remove", abstract: "Remove an app from the Dock by name")
        @Argument(help: "App name (as shown in 'dock list')") var name: String
        @Flag(name: .long, help: "Output JSON") var json = false

        func run() throws {
            try Auth.check("dock.write")
            let plistPath = NSHomeDirectory() + "/Library/Preferences/com.apple.dock.plist"
            guard let plistData = try? Data(contentsOf: URL(fileURLWithPath: plistPath)),
                  var plist = try? PropertyListSerialization.propertyList(from: plistData, options: [.mutableContainersAndLeaves], format: nil) as? [String: Any] else {
                throw ValidationError("Could not read Dock preferences plist at \(plistPath)")
            }
            var apps = plist["persistent-apps"] as? [[String: Any]] ?? []

            let lowerName = name.lowercased()
            let beforeCount = apps.count
            apps.removeAll { entry in
                let tileData = entry["tile-data"] as? [String: Any]
                let fileData = tileData?["file-data"] as? [String: Any]
                guard let urlStr = fileData?["_CFURLString"] as? String else { return false }
                let posix: String = {
                    if urlStr.hasPrefix("file://") {
                        let decoded = urlStr.replacingOccurrences(of: "file://", with: "").removingPercentEncoding ?? urlStr
                        return decoded.hasSuffix("/") ? String(decoded.dropLast()) : decoded
                    }
                    return urlStr
                }()
                let entryName = URL(fileURLWithPath: posix).deletingPathExtension().lastPathComponent
                return entryName.lowercased() == lowerName
            }
            guard apps.count < beforeCount else {
                throw ValidationError("App not found in Dock: \(name). Use 'dock list' to see pinned apps.")
            }
            plist["persistent-apps"] = apps

            let outData: Data
            do {
                outData = try PropertyListSerialization.data(fromPropertyList: plist, format: .binary, options: 0)
            } catch {
                throw ValidationError("Could not serialize Dock plist: \(error.localizedDescription)")
            }
            do {
                try outData.write(to: URL(fileURLWithPath: plistPath), options: .atomic)
            } catch {
                throw ValidationError("Could not write Dock plist: \(error.localizedDescription)")
            }
            _ = Process.run(args: ["/usr/bin/killall", "Dock"])
            if json {
                printJSON(["removed": true, "name": name] as [String: Any])
            } else {
                print("Removed from Dock: \(name) (Dock restarted)")
            }
        }
    }
```

- [ ] **Step 3: FileCommand — add 10 MB size limit + `--force` override on Read (H5 fix).** Replace `ReadCmd` (lines 177–196) entirely with:

```swift
    struct ReadCmd: ParsableCommand {
        static let configuration = CommandConfiguration(commandName: "read", abstract: "Print text file contents")
        @Argument(help: "File path") var path: String
        @Option(name: .long, help: "Max bytes to read (default: 102400)") var maxBytes: Int = 102_400
        @Flag(name: .long, help: "Skip the 10MB safety limit (use sparingly)") var force = false

        // 10 MB hard ceiling unless --force is passed (H5 fix). Reading huge files
        // into memory is almost always a mistake — large logs/binaries should be
        // streamed through tail/head/dd instead.
        private static let safetyCeiling = 10 * 1024 * 1024

        func run() throws {
            try Auth.check("file.read")
            let url = URL(fileURLWithPath: (path as NSString).expandingTildeInPath)
            guard FileManager.default.fileExists(atPath: url.path) else {
                throw ValidationError("File not found: \(path)")
            }
            let attrs = try FileManager.default.attributesOfItem(atPath: url.path)
            let fileSize = (attrs[.size] as? Int) ?? 0
            if !force && fileSize > Self.safetyCeiling {
                throw ValidationError(
                    "Refusing to read \(fileSize) bytes — exceeds 10MB safety limit. " +
                    "Pass --force to override, or use --max-bytes to read a window."
                )
            }
            let data = try Data(contentsOf: url)
            let slice = data.prefix(maxBytes)
            guard let text = String(data: slice, encoding: .utf8) else {
                throw ValidationError("File is not valid UTF-8 text: \(path)")
            }
            print(text)
            if data.count > maxBytes {
                fputs("[truncated: \(data.count - maxBytes) bytes remaining — use --max-bytes to read more]\n", stderr)
            }
        }
    }
```

- [ ] **Step 4: Build.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 5: Smoke-test.**

```bash
cp .build/release/macos-cli ~/.local/bin/macos
# File read with huge file
dd if=/dev/zero of=/tmp/macos-cli-bigfile bs=1m count=20 2>/dev/null
macos file read /tmp/macos-cli-bigfile 2>&1 | head -3
# Expected: ValidationError about 10MB safety limit
macos file read /tmp/macos-cli-bigfile --force --max-bytes 100 2>&1 | head -3
# Expected: 100 NUL bytes
rm /tmp/macos-cli-bigfile

# Dock list still works (don't add/remove unless you have an app to test with)
macos dock list 2>&1 | head -5
# Expected: actual Dock contents
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "security: PropertyListSerialization for Dock writes, 10MB safety limit on file read"
```

---

### Task 8: Bug fixes — Menu 1-level path, Window snap screen, Window write guards, Ax frontmost API

**Files:**
- Modify: `Sources/macos-cli/Commands/MenuCommand.swift`
- Modify: `Sources/macos-cli/Commands/WindowCommand.swift`
- Modify: `Sources/macos-cli/Commands/AxCommand.swift`

- [ ] **Step 1: MenuCommand.Click — fix 1-element path branch (M1 fix).** Replace the script construction in `Click.run()` (lines 89–100, the `let script = """` block) with:

```swift
            let script: String
            if parts.count == 1 {
                // 1-element path → click the menu bar item directly (e.g. "Apple" or app menu)
                script = """
                try {
                    const se = Application('System Events');
                    const proc = \(procClause);
                    if (!proc) { JSON.stringify({ok:false, error:'App not found'}); }
                    else {
                        const path = [\(partsJSON)];
                        proc.menuBars[0].menuBarItems.whose({title: path[0]})[0].click();
                        JSON.stringify({ok:true, result:'clicked'});
                    }
                } catch (e) {
                    JSON.stringify({ok:false, error: String(e && e.message ? e.message : e)});
                }
                """
            } else {
                script = """
                try {
                    const se = Application('System Events');
                    const proc = \(procClause);
                    if (!proc) { JSON.stringify({ok:false, error:'App not found'}); }
                    else {
                        const path = [\(partsJSON)];
                        let target = proc.menuBars[0].menuBarItems.whose({title: path[0]})[0].menus[0];
                        for (let i = 1; i < path.length - 1; i++) {
                            target = target.menuItems.whose({title: path[i]})[0].menus[0];
                        }
                        target.menuItems.whose({title: path[path.length - 1]})[0].click();
                        JSON.stringify({ok:true, result:'clicked'});
                    }
                } catch (e) {
                    JSON.stringify({ok:false, error: String(e && e.message ? e.message : e)});
                }
                """
            }
```

Then replace the post-execute block (lines 101–108) with the envelope-parse pattern:

```swift
            let raw = Process.capture(args: ["/usr/bin/osascript", "-l", "JavaScript", "-e", script], timeout: 10, fallback: "")
            guard let env = parseJXAEnvelope(raw) else {
                throw ValidationError("Could not click '\(path)'. Empty or unparseable JXA result.\nRaw: \(raw.prefix(300))")
            }
            if !env.ok {
                throw ValidationError(
                    "Could not click '\(path)'. Verify the path is correct and the app is frontmost. Automation permission for System Events is required.\n\(env.error)"
                )
            }
            print("Clicked: \(path)")
```

Also: in the same file, `partsJSON` already uses manual escaping. Replace its construction (lines 83–87) with:

```swift
            let partsJSON = parts.map { "\"\(jxaEscape($0))\"" }.joined(separator: ", ")
```

And the `procClause` escapes:
```swift
            let procClause: String
            if let app = app {
                procClause = "se.applicationProcesses.whose({name: \"\(jxaEscape(app))\"})[0]"
            } else {
                procClause = "se.applicationProcesses.whose({frontmost: true})[0]"
            }
```

- [ ] **Step 2: WindowCommand.Snap — fix wrong-screen bug (M3 fix).** Replace `Snap.run()` body (lines 200–238) with:

```swift
        func run() throws {
            try Auth.check("window.write")
            // Resolve the app's window first so we can ask AX for its current frame and
            // pick the NSScreen that actually contains it. Snap must respect the
            // monitor the window lives on — using NSScreen.main is wrong on multi-display rigs.
            let axApp = AXUIElementCreateApplication(pid(for: app))
            var windowsRef: CFTypeRef?
            let err = AXUIElementCopyAttributeValue(axApp, kAXWindowsAttribute as CFString, &windowsRef)
            guard err == .success, let windows = windowsRef as? [AXUIElement], !windows.isEmpty else {
                throw ValidationError("Could not access \(app) windows — check Accessibility permission")
            }
            let targetWindow: AXUIElement
            if let t = title {
                let match = windows.first { win -> Bool in
                    var titleRef: CFTypeRef?
                    AXUIElementCopyAttributeValue(win, kAXTitleAttribute as CFString, &titleRef)
                    return (titleRef as? String ?? "").localizedCaseInsensitiveContains(t)
                }
                guard let m = match else {
                    throw ValidationError("No window with title '\(t)' in \(app)")
                }
                targetWindow = m
            } else {
                targetWindow = windows[0]
            }

            // Get the window's current position (AX top-left origin)
            var posRef: CFTypeRef?
            AXUIElementCopyAttributeValue(targetWindow, kAXPositionAttribute as CFString, &posRef)
            var winOrigin = CGPoint.zero
            if let p = posRef {
                AXValueGetValue(p as! AXValue, .cgPoint, &winOrigin)
            }
            var sizeRef: CFTypeRef?
            AXUIElementCopyAttributeValue(targetWindow, kAXSizeAttribute as CFString, &sizeRef)
            var winSize = CGSize.zero
            if let s = sizeRef {
                AXValueGetValue(s as! AXValue, .cgSize, &winSize)
            }

            // Convert AX top-left to AppKit bottom-left for screen lookup
            let primaryHeight = NSScreen.screens.first?.frame.height ?? 0
            let winCenterAK = CGPoint(
                x: winOrigin.x + winSize.width / 2,
                y: primaryHeight - winOrigin.y - winSize.height / 2
            )
            let screen = NSScreen.screens.first(where: { $0.frame.contains(winCenterAK) }) ?? NSScreen.main
            guard let screen = screen else {
                throw ValidationError("Could not determine a target screen.")
            }
            let vf = screen.visibleFrame
            let sf = screen.frame
            let W = vf.width, H = vf.height
            let ox = vf.origin.x
            let baseY = sf.height - vf.origin.y - vf.height

            let (rx, ry, rw, rh): (CGFloat, CGFloat, CGFloat, CGFloat)
            switch position {
            case "left":          (rx, ry, rw, rh) = (ox,         baseY,       W/2, H)
            case "right":         (rx, ry, rw, rh) = (ox + W/2,   baseY,       W/2, H)
            case "top":           (rx, ry, rw, rh) = (ox,         baseY,       W,   H/2)
            case "bottom":        (rx, ry, rw, rh) = (ox,         baseY + H/2, W,   H/2)
            case "top-left":      (rx, ry, rw, rh) = (ox,         baseY,       W/2, H/2)
            case "top-right":     (rx, ry, rw, rh) = (ox + W/2,   baseY,       W/2, H/2)
            case "bottom-left":   (rx, ry, rw, rh) = (ox,         baseY + H/2, W/2, H/2)
            case "bottom-right":  (rx, ry, rw, rh) = (ox + W/2,   baseY + H/2, W/2, H/2)
            case "left-third":    (rx, ry, rw, rh) = (ox,         baseY,       W/3, H)
            case "center-third":  (rx, ry, rw, rh) = (ox + W/3,   baseY,       W/3, H)
            case "right-third":   (rx, ry, rw, rh) = (ox + 2*W/3, baseY,       W/3, H)
            default:
                throw ValidationError(
                    "Unknown position '\(position)'. Valid: left, right, top, bottom, top-left, top-right, bottom-left, bottom-right, left-third, center-third, right-third"
                )
            }

            var origin = CGPoint(x: rx, y: ry)
            let pos = AXValueCreate(.cgPoint, &origin)!
            AXUIElementSetAttributeValue(targetWindow, kAXPositionAttribute as CFString, pos)
            var size = CGSize(width: rw, height: rh)
            let sz = AXValueCreate(.cgSize, &size)!
            AXUIElementSetAttributeValue(targetWindow, kAXSizeAttribute as CFString, sz)
            print("Snapped \(self.app) to \(self.position) on \(screen.localizedName): \(Int(rw))×\(Int(rh)) at (\(Int(rx)),\(Int(ry)))")
        }
```

- [ ] **Step 3: WindowCommand — add `window.write` guard to Move, Resize, Minimize, Fullscreen, Maximize (M4 fix; Snap was guarded in Step 2).** In each of those `run()` bodies, add as the first line:

```swift
            try Auth.check("window.write")
```

(WindowCommand currently does not have a Close subcommand in the subcommands list — only `[List.self, Move.self, Resize.self, Focus.self, Minimize.self, Fullscreen.self, Maximize.self, Snap.self]`. `Focus` is debatable — it changes focus state. Audit lists `window.write` for "move/resize/snap/close/minimize/maximize/fullscreen", and Focus is not in that list — leave it ungated.)

- [ ] **Step 4: AxCommand — confirm frontmost API is correct (M2 fix).**

Re-read `AxCommand.Hints.collectHints` (already uses `NSWorkspace.shared.runningApplications` for the named-app path and `kAXFocusedApplicationAttribute` for the frontmost path — that's the right pattern). The bug per the audit is in `Find.run()` and `Click.run()` where they call `se.applicationProcesses.whose({frontmost: true})[0]`. Replace these with the NSWorkspace frontmost path.

In `Find.run()` (line 31), replace the `procExpr` construction (line 31–33) with:

```swift
            let frontmostName: String
            if let app = app {
                frontmostName = app
            } else {
                frontmostName = NSWorkspace.shared.frontmostApplication?.localizedName ?? ""
            }
            guard !frontmostName.isEmpty else {
                throw ValidationError("No frontmost application — pass --app <name>")
            }
            let procExpr = "se.applicationProcesses.whose({name: \"\(jxaEscape(frontmostName))\"})[0]"
```

Apply the same pattern to `Click.run()` (line 91 area) — compute frontmostName from NSWorkspace, then build `procs` and `procSelector` using `se.applicationProcesses.whose({name: ...})` rather than `.whose({frontmost: true})`.

In `Read.run()` (line 141 area), apply the same pattern: when `appName.isEmpty`, resolve it via `NSWorkspace.shared.frontmostApplication?.localizedName` first, then use the name-based selector.

(This eliminates the unreliable `frontmost: true` JXA filter and also removes the special `[0]` placement bug in the `appName.isEmpty ? ... : ...` ternary that currently embeds `[0]` inconsistently.)

- [ ] **Step 5: Build.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 6: Smoke-test.**

```bash
cp .build/release/macos-cli ~/.local/bin/macos
# 1-element menu click (would need a known app + simple menu — just verify CLI accepts it)
macos menu list --json 2>&1 | head -5
# Window snap dry-run: list windows
macos window list 2>&1 | head -5
# Ax find with no --app: should resolve frontmost via NSWorkspace
macos ax find "File" 2>&1 | head -5
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "fix(window/menu/ax): 1-elem menu paths, snap-on-correct-screen, NSWorkspace frontmost + window.write guards"
```

---

### Task 9: Process and timeout hardening + ScriptCommand stderr surfacing

**Files:**
- Modify: `Sources/macos-cli/Commands/ScriptCommand.swift`

(Most of this task was already done in Task 1: Process.run now checks executable existence (C3), Process.capture timeout is now actively enforced with terminate+kill (M5), and captureWithStderr exists (C2). What remains is wiring ScriptCommand to surface child stderr — L5 fix.)

- [ ] **Step 1: Update ScriptCommand.Run.run() to use captureWithStderr.** Replace the body of `Run.run()` (lines 30–60) with:

```swift
        func run() throws {
            try Auth.check("script.run")

            var args = ["/usr/bin/osascript"]

            if let jxa = jxa {
                args += ["-l", "JavaScript", "-e", jxa]
            } else if let as_ = applescript {
                args += ["-e", as_]
            } else if let path = file {
                let url = URL(fileURLWithPath: (path as NSString).expandingTildeInPath)
                if url.pathExtension.lowercased() == "js" {
                    args += ["-l", "JavaScript", url.path]
                } else {
                    args.append(url.path)
                }
            } else {
                throw ValidationError("Specify --jxa, --applescript, or --file.")
            }

            let (out, err, code) = Process.captureWithStderr(args: args, timeout: TimeInterval(timeout))
            let stdoutTrim = out.trimmingCharacters(in: .whitespacesAndNewlines)
            let stderrTrim = err.trimmingCharacters(in: .whitespacesAndNewlines)

            if code == -1 {
                throw ValidationError("Script timed out after \(timeout)s")
            }

            // osascript prints its error syntax to stderr; surface it.
            if code != 0 {
                let detail = stderrTrim.isEmpty ? stdoutTrim : stderrTrim
                throw ValidationError("Script exited \(code): \(detail)")
            }

            // Even when code == 0, osascript can emit warnings on stderr. Surface them
            // when not running silent (L5 fix).
            if !silent {
                if !stdoutTrim.isEmpty {
                    print(stdoutTrim)
                }
                if !stderrTrim.isEmpty {
                    fputs(stderrTrim + "\n", stderr)
                }
            }
        }
```

- [ ] **Step 2: Build.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
```

Expected: `Build complete!`

- [ ] **Step 3: Smoke-test.**

```bash
cp .build/release/macos-cli ~/.local/bin/macos
macos auth grant script.run
macos script run --jxa "(function(){ console.log('hello-stderr'); return 'hello-stdout'; })()" 2>&1
# Expected: stdout includes 'hello-stdout'; stderr also includes the console.log line (osascript routes console.log to stderr)

# Timeout enforcement
time macos script run --jxa "(function(){ delay(60); return 'never'; })()" --timeout 2 2>&1 | head -3
# Expected: ValidationError "Script timed out after 2s" within ~2.5s
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "script: surface child stderr and enforce --timeout via captureWithStderr"
```

---

### Task 10: Cleanup — orphan caps removal, AuthCommand.Setup --all confirmation, SpacesCommand error message, version verification

**Files:**
- Modify: `Sources/macos-cli/Auth.swift` (already done in Task 2 — verify orphan removal)
- Modify: `Sources/macos-cli/Commands/AuthCommand.swift`
- Modify: `Sources/macos-cli/Commands/SpacesCommand.swift`
- Modify: `Sources/macos-cli/Commands/TimeMachineCommand.swift` (comment-only)
- Modify: `Sources/macos-cli/Commands/MessagesCommand.swift` (verify M6 already covered)
- Modify: `Sources/macos-cli/MacOSCLI.swift` (version verify)

- [ ] **Step 1: AuthCommand.Setup — add `--yes` flag, require it for `--all` (L2 fix).** Replace `Setup.run()` (lines 80–106) with:

```swift
        @Flag(name: .long, help: "Grant all capabilities without prompting (agent use)") var all = false
        @Flag(name: .long, help: "Skip confirmation prompt for --all (non-interactive)") var yes = false

        func run() throws {
            if all {
                if !yes {
                    let granting = Auth.allCapabilities.map { "  \($0.id)" }.joined(separator: "\n")
                    print("This will GRANT every capability:\n\(granting)\n")
                    print("Continue? (y/N): ", terminator: "")
                    let input = readLine()?.trimmingCharacters(in: .whitespaces).lowercased() ?? ""
                    guard input == "y" || input == "yes" else {
                        print("Cancelled.")
                        return
                    }
                }
                var caps = Auth.defaultCapabilities
                for entry in Auth.allCapabilities { caps[entry.id] = true }
                try Auth.save(caps)
                print("All capabilities granted. Run `macos auth list` to review, `macos auth deny <cap>` to restrict.")
                return
            }

            print("macos-cli permission setup")
            print("Press Y to grant, N to deny, Enter to keep the default shown in brackets.\n")
            var caps = Auth.load()
            if caps.isEmpty { caps = Auth.defaultCapabilities }

            for entry in Auth.allCapabilities {
                let current = caps[entry.id] ?? entry.defaultAllow
                print("\(entry.id) [\(current ? "Y" : "N")] — \(entry.description): ", terminator: "")
                let input = readLine()?.trimmingCharacters(in: .whitespaces).lowercased() ?? ""
                if input == "y" { caps[entry.id] = true }
                else if input == "n" { caps[entry.id] = false }
                // empty → keep current
            }
            try Auth.save(caps)
            print("\nPermissions saved to ~/.config/macos-cli/auth.json")
            print("Run `macos auth list` to review. Use `macos auth grant/deny <cap>` to adjust anytime.")
        }
```

- [ ] **Step 2: SpacesCommand.Switch — clearer dlsym failure message (M7 fix).** Locate `Switch.run()` (line 98 area). Replace the existing guard:

```swift
            guard let changeSpaces = resolvedCGSChangeSpaces() else {
                throw ValidationError("CGSChangeSpaces not available on this macOS version — cannot switch spaces.")
            }
```

with a more diagnostic message:

```swift
            guard let changeSpaces = resolvedCGSChangeSpaces() else {
                throw ValidationError(
                    "CGSChangeSpaces symbol not found via dlsym. This is a private CoreGraphics API; " +
                    "Apple may have removed or renamed it on your macOS build. " +
                    "Workaround: use Mission Control gesture/keyboard shortcut to switch spaces manually."
                )
            }
```

- [ ] **Step 3: TimeMachineCommand.Status — comment the NeXTSTEP format assumption (M9 fix).** Find the parser in `Status` (around the body that parses `tmutil status` output). At the top of the parsing block, add a comment:

```swift
            // NOTE: tmutil status emits NeXTSTEP-style "key = value;" plist-fragment
            // output on Ventura (macOS 13) and earlier. Sonoma (macOS 14+) emits the
            // same format. If Apple ever switches tmutil to XML or JSON, this
            // line-by-line parser will need a rewrite — detect by sampling the first
            // non-whitespace character: '{' = NeXTSTEP, '<' = XML, '[' or '{' for JSON.
```

- [ ] **Step 4: MessagesCommand.Send — verify M6 is handled (newline escape in user text). After Task 5, the script uses `jxaEscape(text)` which already escapes \n and \r. No further change required. Confirm by grep:

```bash
grep -A2 "func run() throws" ~/Developer/macos-cli/Sources/macos-cli/Commands/MessagesCommand.swift | grep jxaEscape | wc -l
```

Expected: at least 1 (Send uses it).

- [ ] **Step 5: AudioCommand numeric-ID scope-bypass — verify the check covers numeric IDs (M8).** Re-read `AudioDeviceCommand.findDevice` (lines 92–111). The current code already calls `hasScope(idNum, input: input)` for numeric inputs (line 95) — that's the fix that was already applied in v0.6.0. No code change. Add a clarifying comment above the function:

```swift
    // M8 — when the user passes a numeric device ID, we still verify that the
    // device has the requested scope (input vs output). This prevents using a
    // pure-output device as an input by ID and getting silent CoreAudio errors.
```

- [ ] **Step 6: Version verification.** Confirm `MacOSCLI.swift` line 9 still reads `version: "0.6.0"`.

```bash
grep "version:" ~/Developer/macos-cli/Sources/macos-cli/MacOSCLI.swift
```

Expected: `version: "0.6.0",`. If it does not match, edit and update.

- [ ] **Step 7: Confirm orphan capabilities are gone (L4 verify).**

```bash
grep -E "system.shutdown|system.reboot" ~/Developer/macos-cli/Sources/macos-cli/Auth.swift
```

Expected: no matches (the entries were removed in Task 2).

- [ ] **Step 8: Final build + install + smoke test.**

```bash
cd ~/Developer/macos-cli && swift build -c release 2>&1 | grep -E "error:|Build complete"
cp .build/release/macos-cli ~/.local/bin/macos
macos --version
# Expected: 0.6.0

macos auth list | wc -l
# Expected: 70+

# Default-deny destructive ops work
macos auth reset
macos messages send --to "+15555555555" --text "test" 2>&1 | head -3
# Expected: '...is denied...' error

macos auth grant messages.send
macos auth list | grep messages.send
# Expected: ✓ messages.send ...

# Reads still work after reset
macos auth reset
macos info system 2>&1 | head -3
# Expected: real output, no auth error
```

- [ ] **Step 9: Final sanity — confirm no lingering `lowercased().contains("error")` and that every command file uses `jxaEscape` where it does JXA.**

```bash
grep -rn 'lowercased().contains("error")' ~/Developer/macos-cli/Sources/macos-cli/Commands/
# Expected: no output

grep -rln 'osascript.*-l.*JavaScript' ~/Developer/macos-cli/Sources/macos-cli/Commands/ | while read f; do
  if grep -q 'replacingOccurrences(of: "\\\\", with: "\\\\\\\\")' "$f"; then
    echo "MISSED jxaEscape conversion: $f"
  fi
done
# Expected: no output
```

- [ ] **Commit.**

```bash
git -c submodule.recurse=false add -A
git -c submodule.recurse=false commit -m "cleanup: AuthCommand --yes for --all, SpacesCommand dlsym message, orphan caps gone, v0.6.0 verified"
```

---

## Final verification checklist

After all 10 tasks land, the following must all hold:

- [ ] `swift build -c release` exits clean with no warnings on Auth.swift, EventKitStore.swift, or Helpers.swift.
- [ ] `macos --version` prints `0.6.0`.
- [ ] `macos auth list` lists ≥70 capabilities including: messages.send, mouse.write, keyboard.write, ax.write, file.delete, trash.empty, apps.quit, defaults.write, dock.write, login-items.write, safari.execute, photos.delete, finder.write, bluetooth.write, disk.write, ocr.capture, system.sleep, process.kill, window.write, keychain.list, music.write, notify.send, speech.speak, shortcuts.run, spotlight.search, focus.write, pdf.read, storage.read, location.read, network.read, info.read, wifi.write, display.write, vpn.write, system.lock.
- [ ] `grep -rn 'lowercased().contains("error")' Sources/` returns no matches.
- [ ] `grep -rn 'system.shutdown\|system.reboot' Sources/` returns no matches.
- [ ] Every destructive subcommand in every command file has `try Auth.check(...)` as its first line.
- [ ] Dock add/remove uses PropertyListSerialization, not XML strings.
- [ ] FileCommand.Read rejects >10MB files unless --force.
- [ ] ScriptCommand surfaces stderr unless --silent.
- [ ] EventKitStore uses `requestFullAccessToEvents/Reminders` on macOS 14+.
- [ ] WindowCommand.Snap picks the screen the window is on, not always `NSScreen.main`.
- [ ] MenuCommand.Click handles 1-element paths.
- [ ] AxCommand uses NSWorkspace.frontmostApplication, not the JXA `frontmost:true` filter.
- [ ] AuthCommand.Setup --all requires confirmation unless --yes.
- [ ] All git commits use `git -c submodule.recurse=false`.
