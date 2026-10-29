---
date: 2026-02-01T13:52:24Z
researcher: Claude
git_commit: d50a7036a07b9952cbc1e5b2143ae254a48be163
branch: fix-multi-tab-sessions
repository: agent-deck
topic: "Terminal content bleeding between sessions when entering/exiting"
tags: [research, codebase, terminal, session-switching, alternate-screen, bubbletea]
status: complete
last_updated: 2026-02-01
last_updated_by: Claude
---

# Research: Terminal Content Bleeding Between Sessions

**Date**: 2026-02-01T13:52:24Z
**Researcher**: Claude
**Git Commit**: d50a7036a07b9952cbc1e5b2143ae254a48be163
**Branch**: fix-multi-tab-sessions
**Repository**: agent-deck

## Research Question

When entering a session and then coming back out and going into another session, some content is cut off and scrolling up shows output from the previous session. The terminal doesn't load enough text content when clicking into a session - users expect to see all of it.

## Summary

The issue is caused by **alternate screen buffer handling conflicts** between:
1. BubbleTea TUI (uses alternate screen via `tea.WithAltScreen()`)
2. tmux sessions (have alternate screen disabled via `smcup@:rmcup@` in tmux.conf)
3. Session attachment/detachment (clears screen but doesn't properly reset scrollback)

When switching between sessions, content from the previous tmux session remains in the terminal's main scrollback buffer, creating the appearance of "bleeding" content. The core problem is that `clearScreen` (ESC[2J ESC[H) only clears the visible screen, not the scrollback buffer.

## Detailed Findings

### 1. Alternate Screen Buffer Configuration

**File**: `install.sh:429`
```bash
set -ag terminal-overrides ",xterm*:Tc:smcup@:rmcup@"
```

This **disables** the alternate screen buffer for tmux sessions by nullifying:
- `smcup` - "start mode cursor up" (enter alternate screen)
- `rmcup` - "reset mode cursor up" (exit alternate screen)

**Purpose**: Allows scrollback to persist in tmux, which is typically desirable for AI agent sessions where users want to scroll back through extensive output.

### 2. BubbleTea TUI Alternate Screen Usage

**File**: `cmd/agent-deck/main.go:312`
```go
p := tea.NewProgram(
    homeModel,
    tea.WithAltScreen(),  // TUI uses alternate screen
    tea.WithMouseCellMotion(),
)
```

BubbleTea runs in the alternate screen buffer, which:
- Saves the main screen content when entering
- Restores it when exiting

### 3. Session Attachment Flow

**File**: `internal/ui/home.go:4272-4280`
```go
return tea.Exec(attachCmd{session: tmuxSess}, func(err error) tea.Msg {
    h.isAttaching.Store(false)

    // Clear screen with synchronized output for atomic rendering
    fmt.Print(syncOutputBegin + clearScreen + syncOutputEnd)
    // ...
})
```

**File**: `internal/ui/home.go:43`
```go
clearScreen = "\033[2J\033[H"  // ESC[2J clears screen, ESC[H moves cursor home
```

**Problem**: `ESC[2J` only clears the **visible** screen area, not the scrollback buffer. When returning from a tmux session (which writes to the main buffer due to `smcup@:rmcup@`), that content remains in scrollback and can be seen when scrolling up.

### 4. PTY Attachment Process

**File**: `internal/tmux/pty.go:23-182`

When attaching to a session:
1. Terminal is set to raw mode (`term.MakeRaw`)
2. PTY is created for tmux attach
3. All output flows directly to stdout (main terminal buffer)
4. On detach via Ctrl+Q, terminal state is restored but scrollback is preserved

### 5. Content Capture and Preview

**File**: `internal/tmux/tmux.go:1026-1037`
```go
func (s *Session) CaptureFullHistory() (string, error) {
    // Limit to last 2000 lines
    cmd := exec.Command("tmux", "capture-pane", "-t", s.Name, "-p", "-J", "-S", "-2000")
    // ...
}
```

The preview pane in the TUI captures content correctly via tmux commands. This is **not** the issue - the issue is what the user sees when **attached** to a session.

### 6. Screen Clear Escape Sequences

| Sequence | Effect |
|----------|--------|
| `ESC[2J` | Clear visible screen only |
| `ESC[3J` | Clear scrollback buffer |
| `ESC[H` | Move cursor to home position |

**Missing**: `ESC[3J` to clear scrollback, or proper alternate screen buffer switching.

## Root Cause Analysis

The conflict arises from three design decisions:

1. **TUI uses alternate screen** - Good for preserving user's terminal state
2. **tmux has alternate screen disabled** - Good for scrollback in AI sessions
3. **Session detach only clears visible screen** - Leaves previous session content in scrollback

When the flow is:
```
TUI (alt screen) -> Attach Session A (main screen, output to main buffer)
                 -> Detach (clear visible, scrollback has Session A content)
                 -> Attach Session B (main screen, output to main buffer)
                 -> Detach (clear visible, scrollback now has Session A + B content mixed)
```

The user can scroll up and see content from Session A while in Session B, or see Session A content after returning to the TUI.

## Proposed Solutions

### Option 1: Clear Scrollback on Detach (Simple)

**File**: `internal/ui/home.go:43`
```go
// Change from:
clearScreen = "\033[2J\033[H"

// To:
clearScreen = "\033[2J\033[3J\033[H"  // Clear visible + scrollback + home
```

**Pros**: Simple one-line fix
**Cons**: User loses scrollback when detaching (may be unexpected)

### Option 2: Force Alternate Screen on Attach

Modify the attach sequence to explicitly enter alternate screen before attaching:

```go
// Before attach:
fmt.Print("\033[?1049h")  // Enter alternate screen

// After attach:
fmt.Print("\033[?1049l")  // Exit alternate screen (restores previous)
```

**Pros**: Proper separation of session content
**Cons**: May conflict with tmux's `smcup@:rmcup@` setting; loses scrollback in the session

### Option 3: Save/Restore Terminal State

Use more sophisticated terminal state management:
1. Save the TUI's alternate screen content before detaching
2. Clear scrollback when switching sessions
3. Restore when returning to TUI

**Pros**: Complete solution
**Cons**: More complex, may require terminal-specific handling

### Option 4: Remove smcup@:rmcup@ Override

Remove from `install.sh:429`:
```bash
# Remove this line:
set -ag terminal-overrides ",xterm*:Tc:smcup@:rmcup@"
```

**Pros**: Natural alternate screen behavior handles everything
**Cons**: Users lose scrollback when detaching from tmux sessions, which is valuable for AI agent output review

## Implemented Solution

**Clear scrollback on ATTACH (not detach)** - This preserves per-session scrollback:

**File**: `internal/ui/home.go` - `attachCmd.Run()`
```go
func (a attachCmd) Run() error {
    // Clear terminal scrollback BEFORE attaching to prevent content bleeding
    fmt.Print("\033[3J\033[2J\033[H")

    ctx := context.Background()
    return a.session.Attach(ctx)
}
```

**Why this works**:
1. Terminal emulator's scrollback is cleared when entering a session
2. tmux's internal scrollback (per-session, via `history-limit 10000`) is preserved
3. When you scroll up in a session, you see that session's history (from tmux)
4. Switching sessions doesn't cause content bleeding

**Key insight**: There are two scrollback buffers:
- **Terminal scrollback** (iTerm, Terminal.app, etc.) - shared across all output
- **tmux scrollback** (per-session) - preserved inside each tmux session

We clear the terminal's buffer on attach, but tmux maintains its own per-session history.

## Code References

- `cmd/agent-deck/main.go:312` - TUI alternate screen configuration
- `install.sh:429` - tmux smcup/rmcup override
- `internal/ui/home.go:43` - clearScreen constant
- `internal/ui/home.go:4279-4280` - Screen clear on detach
- `internal/tmux/pty.go:23-182` - PTY attach implementation
- `internal/tmux/tmux.go:1026-1037` - CaptureFullHistory for preview

## Architecture Insights

The terminal management in agent-deck has three layers:
1. **BubbleTea** - Manages the TUI in alternate screen mode
2. **PTY attachment** - Direct terminal access when in a session
3. **tmux** - Manages the actual AI agent sessions

Each layer has its own terminal state management, and the conflict arises at the boundaries when transitioning between them. The `tea.Exec` mechanism in BubbleTea is designed to handle this, but it relies on the executed command properly cleaning up after itself.

## Historical Context

The `smcup@:rmcup@` configuration was added intentionally in `install.sh` to allow users to scroll back through AI agent output after detaching from sessions. This is a common preference for terminal applications where users want to review extensive output.

## Open Questions

1. Would `ESC[3J` cause issues on any terminals? (Most modern terminals support it - iTerm2, Terminal.app, Alacritty, Kitty, WezTerm all support it)
2. Are there edge cases where the clear happens at the wrong time?
