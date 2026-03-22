# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Claude Island is a macOS menu bar app that displays Dynamic Island-style notifications from Claude Code CLI sessions. It installs hooks into `~/.claude/hooks/` that communicate session state via a Unix domain socket. The app monitors multiple Claude Code sessions, shows live status in a notch overlay anchored to the MacBook notch, and lets users approve/deny tool permissions directly from the UI.

## Build Commands

```bash
# Build (ad-hoc signed, production)
./scripts/build.sh

# Build via xcodebuild directly
xcodebuild -scheme ClaudeIsland -configuration Release build

# Create release DMG (after build)
./scripts/create-release.sh --skip-notarization

# Lint
swiftlint lint --strict ClaudeIsland/
swiftformat --lint ClaudeIsland/

# Run all pre-commit checks
prek run --all-files

# Format Swift code
swiftformat ClaudeIsland/
```

There are no unit tests in this project. The Xcode project has no test targets.

## Pre-commit Hooks

Pre-commit runs: trailing whitespace, end-of-file fixer, YAML/JSON checks, merge conflict detection, private key detection, SwiftFormat, SwiftLint (`--strict`), shellcheck (on `scripts/*.sh`), ruff (on `ClaudeIsland/Resources/*.py`), and markdownlint. Direct commits to `main` are blocked by the `no-commit-to-branch` hook.

Install: `prek install --hook-type pre-commit --hook-type pre-push`

## Architecture

### Event-Driven State Machine

All session state flows through a unidirectional architecture:

1. **HookSocketServer** (`Services/Hooks/HookSocketServer.swift`) — Unix domain socket server that receives `HookEvent` structs from the Python hook script
2. **SessionEvent** (`Models/SessionEvent.swift`) — Enum defining all possible state mutations (hook received, permission approved/denied, file updated, tool completed, subagent events, etc.)
3. **SessionStore** (`Services/State/SessionStore.swift`) — Singleton `actor` that is the single source of truth. All state mutations flow through its `process(_ event:)` method. External consumers observe state via `sessionsStream()` (AsyncStream)
4. **SessionState** (`Models/SessionState.swift`) — Value type (`struct`) representing complete state for one Claude session, including phase, chat items, tool tracking, and subagent state
5. **SessionPhase** (`Models/SessionPhase.swift`) — Explicit state machine enum with validated transitions: `idle → processing → waitingForInput → waitingForApproval → compacting → ended`

### Hook System

- **HookInstaller** (`Services/Hooks/HookInstaller.swift`) — Auto-installs the Python hook script and updates `~/.claude/settings.json` on app launch
- **claude-island-state.py** (`Resources/claude-island-state.py`) — Python 3.14+ hook script bundled in the app. Sends session events to the app via Unix socket; for `PermissionRequest` events, blocks waiting for approve/deny response
- **PythonRuntimeDetector** — Detects available Python runtime at launch for hook command generation

### UI Layer

- **NotchWindow/NotchWindowController** (`UI/Window/`) — Custom `NSWindow` positioned at the screen notch
- **NotchViewModel** (`Core/NotchViewModel.swift`) — SwiftUI state for the notch UI (open/closed/popping status, content type)
- **NotchView** (`UI/Views/NotchView.swift`) — Main SwiftUI view with animated notch overlay
- **Module system** (`Core/Modules/`, `UI/Modules/`) — Pluggable visual modules (activity spinner, session dots, permission indicator, timer, token rings, etc.) with `ModuleRegistry` and `ModuleLayoutEngine`
- **MarkdownRenderer** — Uses `swift-markdown` for rendering chat messages

### Session Monitoring

- **ClaudeSessionMonitor** (`Services/Session/`) — Detects active Claude Code processes
- **ConversationParser** (`Services/Session/ConversationParser.swift`) — Parses JSONL conversation files for chat history, tool results, and conversation metadata
- **AgentFileWatcher** — Watches for subagent (Task tool) activity files
- **JSONLInterruptWatcher** — Detects user interrupts from JSONL data

### Key Services

- **TmuxController/TmuxTargetFinder** (`Services/Tmux/`) — Tmux integration for sending keystrokes to approve/deny permissions in terminal
- **ToolApprovalHandler** — Orchestrates permission approval flow (socket response + optional tmux keystroke)
- **WindowFinder/WindowFocuser** (`Services/Window/`) — Finds and focuses terminal windows via Accessibility API
- **ClaudeAPIService** (`Services/TokenTracking/`) — Optional token usage tracking via Claude API
- **Sparkle** integration (`Services/Update/`) — Auto-updates with custom `NotchUserDriver`

### Dependencies (Swift Package Manager via Xcode)

- **Sparkle** — Auto-update framework
- **OcclusionKit** — Terminal window visibility detection
- **swift-markdown** — Markdown parsing for chat rendering
- **swift-subprocess** — Process execution

## Code Style

- **Swift 6 strict concurrency** — The project uses `@Observable`, `Sendable`, `actor`, `Mutex<T>`, and structured concurrency throughout. `@MainActor` is the default isolation for the app target
- **SwiftFormat** config in `.swiftformat` — 4-space indent, 150 max line width, `--self insert`, sorted imports, `organizeDeclarations` enabled. `HookSocketServer.swift` is excluded from `organizeDeclarations` due to timeout
- **SwiftLint** config in `.swiftlint.yml` — `--strict` mode, 70+ opt-in rules, `force_unwrapping` as warning. Custom rule: use `os.Logger` instead of `print()`
- **Logging** — Use `os.Logger` (subsystem `com.engels74.ClaudeIsland`), never `print()`
- Targets macOS 15.6+, Swift 6.2
