# Terminal Scrollback Bleeding Fix - Implementation Plan

## Overview

Fix the issue where terminal content from one session "bleeds" into another when switching between sessions. When a user enters session A, exits, then enters session B and scrolls up, they incorrectly see content from session A instead of only session B's history.

## Current State Analysis

### The Problem

Three systems interact with terminal buffers:

1. **BubbleTea TUI** (`cmd/agent-deck/main.go:312`)
   - Uses alternate screen via `tea.WithAltScreen()`
   - Preserves user's original terminal state

2. **tmux sessions** (`install.sh:429`)
   - Alternate screen is **disabled** via `smcup@:rmcup@`
   - This allows tmux scrollback to persist (desirable for AI agent output review)
   - Output goes to terminal's **main buffer**

3. **Session attachment** (`internal/ui/home.go`)
   - On detach, only cleared visible screen (`ESC[2J ESC[H`)
   - Did **not** clear terminal's scrollback buffer

### Root Cause

When BubbleTea exits alternate screen (via `tea.Exec`), the terminal restores its saved main buffer INCLUDING scrollback from before agent-deck started. This caused content from previous sessions to appear when attaching to a new session.

### Key Insight: Two Scrollback Buffers

| Buffer | Location | Scope | Persistence |
|--------|----------|-------|-------------|
| Terminal scrollback | iTerm2/Terminal.app/etc. | Shared across all output | Accumulates until cleared |
| tmux scrollback | Inside tmux server | Per-session | Preserved via `history-limit 10000` |

## Implemented Solution

**Sync terminal scrollback with tmux scrollback on attach.**

Instead of just clearing the terminal's scrollback (which would lose history), we:
1. Capture tmux's scrollback history WITH colors (`-e` flag)
2. Clear the terminal's scrollback
3. Print the tmux history to the terminal
4. Attach to tmux

This makes the terminal's scrollback match tmux's, so mouse scroll works naturally with full history and syntax highlighting.

### Code Changes

#### 1. New method: `CaptureFullHistoryWithColors`

**File**: `internal/tmux/tmux.go`

```go
// CaptureFullHistoryWithColors captures scrollback history with ANSI color codes preserved
func (s *Session) CaptureFullHistoryWithColors() (string, error) {
    // Same as CaptureFullHistory but with -e flag to preserve escape sequences (colors)
    // Used when we need to display the history with syntax highlighting intact
    cmd := exec.Command("tmux", "capture-pane", "-t", s.Name, "-p", "-e", "-J", "-S", "-2000")
    output, err := cmd.Output()
    if err != nil {
        return "", fmt.Errorf("failed to capture history with colors: %w", err)
    }
    return string(output), nil
}
```

#### 2. Modified `attachCmd.Run()`

**File**: `internal/ui/home.go`

```go
func (a attachCmd) Run() error {
    // Sync terminal scrollback with tmux's scrollback to prevent content bleeding
    // and enable seamless mouse scrolling through session history.
    //
    // The problem: When BubbleTea exits alternate screen, the terminal restores
    // its saved scrollback (from before agent-deck started), causing content from
    // other sessions to appear. We fix this by:
    // 1. Capturing the tmux session's scrollback history
    // 2. Clearing the terminal's scrollback
    // 3. Printing the tmux history to the terminal
    // 4. Then attaching (tmux takes over from current screen position)
    //
    // This makes the terminal's scrollback match tmux's, so mouse scroll works naturally.
    //
    // Terminal compatibility:
    // - ESC]1337;ClearScrollback BEL: iTerm2 proprietary (ignored by other terminals)
    // - ESC[3J: Standard xterm clear scrollback (works on most modern terminals:
    //   Terminal.app, Alacritty, Kitty, WezTerm, GNOME Terminal, Windows Terminal)
    // - ESC[2J ESC[H: Universal clear screen + cursor home
    if tty, err := os.OpenFile("/dev/tty", os.O_WRONLY, 0); err == nil {
        // Capture tmux's scrollback history with colors preserved (last 2000 lines)
        history, err := a.session.CaptureFullHistoryWithColors()
        if err == nil && history != "" {
            // Clear terminal scrollback (iTerm2 proprietary + standard xterm)
            tty.WriteString("\033]1337;ClearScrollback\007")
            tty.WriteString("\033[3J\033[2J\033[H")
            // Print tmux history to terminal's buffer
            tty.WriteString(history)
            tty.Sync()
        } else {
            // Fallback: just clear if we can't capture history
            tty.WriteString("\033]1337;ClearScrollback\007")
            tty.WriteString("\033[3J\033[2J\033[H")
            tty.Sync()
        }
        tty.Close()
    }

    ctx := context.Background()
    return a.session.Attach(ctx)
}
```

## Desired End State

When switching between sessions:
1. Enter session A → See session A content with full syntax highlighting
2. Scroll up with mouse → See session A's complete history (up to 2000 lines) with colors
3. Exit session A → Return to TUI
4. Enter session B → See only session B content (no session A contamination)
5. Scroll up with mouse → See session B's complete history with colors

### Verification

```bash
# Build and run
go build ./cmd/agent-deck && ./agent-deck

# Test flow:
# 1. Create/enter session A, generate some output with colors (e.g., Claude code blocks)
# 2. Scroll up with mouse - should see full history with syntax highlighting
# 3. Exit (Ctrl+Q)
# 4. Enter session B
# 5. Scroll up - should see ONLY session B content, no session A content
# 6. All content should have proper syntax highlighting
```

## Terminal Compatibility

| Terminal | Clear Method | Status |
|----------|--------------|--------|
| **iTerm2** | Proprietary `ClearScrollback` + standard | Fully supported |
| **Terminal.app** | Standard `ESC[3J` | Supported |
| **Alacritty** | Standard `ESC[3J` | Supported |
| **Kitty** | Standard `ESC[3J` | Supported |
| **WezTerm** | Standard `ESC[3J` | Supported |
| **GNOME Terminal** | Standard `ESC[3J` | Supported |
| **Windows Terminal** | Standard `ESC[3J` | Supported |
| **Older terminals** | May not clear | History still printed, some bleeding possible |

The iTerm2-specific escape sequence is an OSC command that other terminals will simply ignore.

## What We Learned

1. **Two scrollback buffers exist**: Terminal emulator's buffer and tmux's internal buffer are separate
2. **`smcup@:rmcup@` in tmux.conf** disables alternate screen, causing tmux output to go to terminal's main buffer
3. **BubbleTea's `tea.Exec`** exits alternate screen, restoring saved terminal state including scrollback
4. **Just clearing scrollback wasn't enough** - it made mouse scroll not work (empty buffer = nothing to scroll)
5. **The solution is synchronization** - copy tmux's scrollback to terminal's buffer before attaching
6. **The `-e` flag** on `tmux capture-pane` preserves ANSI escape sequences (colors, bold, etc.)
7. **Writing to `/dev/tty`** bypasses any stdout buffering for reliable escape sequence delivery

## References

- Research document: `thoughts/shared/20260131_115955-fix-multi-tab-sessions/research/20260201_135500-terminal-content-bleeding.md`
- tmux scrollback config: `install.sh:429` (`smcup@:rmcup@`)
- BubbleTea alternate screen: `cmd/agent-deck/main.go:312`
- Attach implementation: `internal/ui/home.go` (`attachCmd.Run()`)
- PTY attachment: `internal/tmux/pty.go:23-182`
- History capture: `internal/tmux/tmux.go` (`CaptureFullHistoryWithColors()`)
