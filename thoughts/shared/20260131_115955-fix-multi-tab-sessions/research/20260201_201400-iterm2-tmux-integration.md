---
date: 2026-02-01T20:14:00Z
researcher: Claude
git_commit: f40948870cb65f1cc2848912e62ac2c97dbcd4ec
branch: fix-multi-tab-sessions
repository: agent-deck
topic: "iTerm2 tmux Integration for Native Tab Support"
tags: [research, tmux, iterm2, tabs, terminal-integration]
status: complete
last_updated: 2026-02-01
last_updated_by: Claude
---

# Research: iTerm2 tmux Integration for Native Tab Support

**Date**: 2026-02-01T20:14:00Z
**Researcher**: Claude
**Git Commit**: f40948870cb65f1cc2848912e62ac2c97dbcd4ec
**Branch**: fix-multi-tab-sessions
**Repository**: agent-deck

## Research Question

How can agent-deck integrate with iTerm2's tmux integration mode to create native iTerm2 tabs when working with sessions, instead of managing sessions through the TUI?

## Summary

iTerm2 provides a special **tmux control mode** (`-CC` flag) that replaces tmux's text-based UI with native iTerm2 windows and tabs. This research explores how agent-deck could leverage this feature to provide a more native macOS experience where each agent-deck session appears as a native iTerm2 tab.

**Key Finding**: The integration is technically feasible but would require significant architectural changes. The `-CC` flag creates a fundamentally different interaction model where iTerm2 becomes the session manager instead of agent-deck's TUI.

## iTerm2 tmux Integration Overview

### How It Works

When tmux is invoked with the `-CC` flag, it enters **control mode**:

```bash
# Create new session with iTerm2 integration
tmux -CC

# Attach to existing session with iTerm2 integration
tmux -CC attach -t session-name
```

In control mode:
1. tmux outputs control commands instead of terminal UI
2. iTerm2 interprets these commands and creates native windows/tabs
3. Each tmux window becomes an iTerm2 tab
4. Each tmux pane becomes an iTerm2 split pane
5. tmux continues running on the server, providing session persistence

### Key Features

| Feature | Standard tmux | tmux -CC (Control Mode) |
|---------|--------------|-------------------------|
| Window management | tmux text UI | Native iTerm2 tabs |
| Pane splitting | tmux split-pane | Native iTerm2 split panes |
| Session persistence | Yes | Yes |
| Remote sessions (SSH) | Yes | Yes (remote tmux, local tabs) |
| Window dimensions | Shared across clients | Per-client (iTerm2 managed) |
| Copy/paste | tmux buffer | Native macOS clipboard |
| Scrollback | tmux capture-pane | Native iTerm2 scrollback |

### Limitations

1. **Tab Homogeneity**: A tab with tmux integration cannot mix tmux and non-tmux panes
2. **Shared Dimensions**: All tmux clients still see the same window dimensions (smallest wins)
3. **iTerm2 Only**: This feature is specific to iTerm2 on macOS

## Current Agent-Deck Architecture

### Session Management Model

Agent-deck currently uses a **TUI-centric model**:

```
┌─────────────────────────────────────────────────────┐
│  iTerm2 / Terminal                                  │
│  ┌───────────────────────────────────────────────┐  │
│  │  agent-deck TUI (Bubble Tea)                  │  │
│  │  ┌─────────────────┬───────────────────────┐  │  │
│  │  │ Session List    │ Preview/Analytics     │  │  │
│  │  │ - session-1     │                       │  │  │
│  │  │ - session-2     │                       │  │  │
│  │  │ - session-3     │                       │  │  │
│  │  └─────────────────┴───────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  When user attaches (Enter):                        │
│  ┌───────────────────────────────────────────────┐  │
│  │  tmux attach-session -t agentdeck_session-1   │  │
│  │  (Full-screen tmux session)                   │  │
│  │  Ctrl+Q to detach back to TUI                 │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Key Code Locations

**Session Creation** (`internal/tmux/tmux.go:660-678`):
```go
func (s *Session) Start(command string) error {
    // Creates detached session (NOT control mode)
    args := []string{"new-session", "-d", "-s", s.Name}
    if s.WorkDir != "" {
        args = append(args, "-c", s.WorkDir)
    }
    cmd := exec.Command("tmux", args...)
    // ...
}
```

**Session Attachment** (`internal/tmux/pty.go:23-33`):
```go
func (s *Session) Attach(ctx context.Context) error {
    // Standard attach (NOT control mode)
    cmd := exec.CommandContext(ctx, "tmux", "attach-session", "-t", s.Name)
    // ...
}
```

**TUI Integration** (`internal/ui/home.go:4207`):
```go
func (h *Home) attachSession(inst *session.Instance) tea.Cmd {
    // Uses tea.Exec to hand off terminal to tmux
    return tea.Exec(attachCmd{session: tmuxSess}, func(err error) tea.Msg {
        h.isAttaching.Store(false)
        return statusUpdateMsg{}
    })
}
```

## Integration Approaches

### Approach 1: Hybrid Mode (Recommended Starting Point)

Add an optional `-CC` mode for iTerm2 users while keeping the TUI as default.

**User Experience**:
```bash
# Normal mode (current behavior)
agent-deck

# iTerm2 native tab mode
agent-deck --iterm-tabs
# or
agent-deck -i
```

When enabled:
1. Agent-deck still shows TUI for session management
2. When user "attaches" to a session, instead of `tmux attach`, it opens a **new iTerm2 tab** with `tmux -CC attach`
3. User can have multiple sessions open as native tabs
4. Closing an iTerm2 tab detaches from (but doesn't kill) the session

**Implementation**:

```go
// internal/tmux/pty.go - New method
func (s *Session) AttachWithITermIntegration(ctx context.Context) error {
    // Use osascript to create new iTerm2 tab with tmux -CC
    script := fmt.Sprintf(`
        tell application "iTerm"
            tell current window
                create tab with default profile
                tell current session
                    write text "tmux -CC attach-session -t %s"
                end tell
            end tell
        end tell
    `, s.Name)

    cmd := exec.CommandContext(ctx, "osascript", "-e", script)
    return cmd.Run()
}
```

**Pros**:
- Minimal architecture change
- TUI remains primary interface
- Users get native tabs when they want them
- Works alongside existing features

**Cons**:
- Requires AppleScript (iTerm2-specific)
- Two different interaction models
- Status bar customization lost in native tabs

### Approach 2: Control Mode Daemon

Run agent-deck as a background daemon that manages a tmux server in control mode.

**Architecture**:
```
┌─────────────────────────────────────────────────────┐
│  iTerm2                                             │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐               │
│  │  Tab 1  │ │  Tab 2  │ │  Tab 3  │  Native tabs  │
│  │session-1│ │session-2│ │session-3│               │
│  └─────────┘ └─────────┘ └─────────┘               │
└───────────────────────────┬─────────────────────────┘
                            │
                   tmux -CC control protocol
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  tmux server                                        │
│  - session-1 (agentdeck_session-1)                  │
│  - session-2 (agentdeck_session-2)                  │
│  - session-3 (agentdeck_session-3)                  │
└───────────────────────────┬─────────────────────────┘
                            │
                   status updates / commands
                            │
                            ▼
┌─────────────────────────────────────────────────────┐
│  agent-deck daemon (background)                     │
│  - Session state management                         │
│  - MCP configuration                                │
│  - Status detection                                 │
│  - Notification system                              │
└─────────────────────────────────────────────────────┘
```

**User Experience**:
```bash
# Start daemon
agent-deck daemon start

# In iTerm2, connect to agent-deck managed tmux
tmux -CC attach -t agentdeck

# Or use agent-deck CLI to create sessions (opens in new tab)
agent-deck add /path/to/project -c claude
```

**Implementation Requirements**:
1. Background daemon process
2. Inter-process communication (Unix socket or HTTP)
3. CLI commands for daemon control
4. Integration with iTerm2's tmux command discovery

**Pros**:
- Fully native iTerm2 experience
- No TUI overhead for tab users
- Session management persists across terminal restarts

**Cons**:
- Major architectural change
- Need to manage daemon lifecycle
- Status detection more complex (no direct TUI updates)
- Feature parity with TUI challenging

### Approach 3: Protocol Handler

Register agent-deck as a protocol handler for custom URLs.

**User Experience**:
```bash
# From TUI, "open in tab" action generates URL
# Clicking opens: agentdeck://attach/session-1
# iTerm2 opens new tab via URL handler
```

**Pros**:
- Clean separation of concerns
- Works with other apps that support URL schemes

**Cons**:
- Requires macOS protocol handler registration
- More complex setup for users

## Technical Challenges

### 1. Session Naming Compatibility

Current agent-deck session names include special characters:
```
agentdeck_project-name_abc123
```

tmux control mode works with these, but iTerm2's display may truncate long names.

### 2. Status Bar in Control Mode

Agent-deck's custom tmux status bar (`ConfigureStatusBar()`) won't display in control mode since iTerm2 replaces the tmux UI entirely.

**Potential Solutions**:
- Use iTerm2's badge feature for status (shows in tab corner)
- Use iTerm2's status bar component API
- Rely on tab title updates for status indication

### 3. Notification Bar for Waiting Sessions

The current notification bar (`FormatBar()`) appears in tmux's status-left. In control mode, this would need alternative handling:

**Options**:
1. **macOS Notifications**: Send system notifications for waiting sessions
2. **iTerm2 Badge**: Show waiting count in tab badge
3. **Menu Bar App**: Separate status indicator in macOS menu bar
4. **iTerm2 Status Bar Component**: Custom component showing waiting sessions

### 4. Quick Session Switching

Current: Ctrl+B then shift+number (!, @, #) jumps to waiting session.

In control mode: Would need to use iTerm2's tab switching shortcuts or implement custom key handler.

### 5. Session Lifecycle Coordination

When user closes an iTerm2 tab:
- Should the tmux session be killed or just detached?
- How does agent-deck detect this?

**Solution**: Use tmux hooks:
```bash
# Detect when client detaches
tmux set-hook -g client-detached 'run-shell "agent-deck hook client-detached #{session_name}"'
```

## Implementation Roadmap

### Phase 1: Detection & Feature Flag

1. Add iTerm2 detection (already exists in `DetectTerminal()`)
2. Add configuration option: `iterm_native_tabs: bool`
3. Add CLI flag: `--iterm-tabs`

### Phase 2: Basic Tab Opening

1. Implement `AttachWithITermIntegration()` using AppleScript
2. Add "Open in Tab" action to TUI (new keybinding)
3. Test with existing session workflows

### Phase 3: Status Integration

1. Implement iTerm2 badge updates for session status
2. Add macOS notification support for waiting sessions
3. Create iTerm2 status bar component (optional)

### Phase 4: Full Integration (Optional)

1. Background daemon architecture
2. CLI-first workflow
3. Deep iTerm2 integration via Python API

## Code References

| File | Line | Description |
|------|------|-------------|
| `internal/tmux/tmux.go` | 152-216 | `DetectTerminal()` - Already detects iTerm2 |
| `internal/tmux/tmux.go` | 660-678 | `Start()` - Creates tmux sessions |
| `internal/tmux/pty.go` | 23-182 | `Attach()` - Current attachment mechanism |
| `internal/ui/home.go` | 4207 | `attachSession()` - TUI attach command |
| `internal/tmux/tmux.go` | 785 | `ConfigureStatusBar()` - Status bar config |
| `internal/session/notifications.go` | - | `FormatBar()` - Waiting session notifications |

## Related Research

- `thoughts/shared/20260131_115955-fix-multi-tab-sessions/research/20260131_115955-bottom-bar-issues.md` - Status bar design
- `thoughts/shared/20260131_115955-fix-multi-tab-sessions/research/20260201_135500-terminal-content-bleeding.md` - Terminal scrollback handling
- `thoughts/shared/20260131_115955-fix-multi-tab-sessions/plans/20260131_120000-status-bar-redesign.md` - Status bar implementation

## External References

- [iTerm2 tmux Integration Documentation](https://iterm2.com/documentation-tmux-integration.html)
- [tmux Control Mode Manual](https://man.openbsd.org/tmux#CONTROL_MODE)

## Open Questions

1. **User Preference Discovery**: How do users prefer to work? Should we survey existing users?

2. **Feature Parity**: Which TUI features are essential in native tab mode?
   - Session grouping?
   - Analytics panel?
   - MCP management?

3. **Cross-Terminal Support**: Should we investigate similar integrations for:
   - WezTerm (has multiplexing)
   - Kitty (has remote control)
   - Windows Terminal (tabs but no tmux integration)

4. **Daemon vs TUI**: For power users, is a daemon-based CLI workflow preferable?

5. **SSH Sessions**: How should remote tmux sessions (over SSH) interact with local iTerm2 tabs?

## Recommendations

**Short Term (Low Effort, High Value)**:
- Implement Approach 1 (Hybrid Mode) with "Open in Tab" action
- Add macOS notifications for waiting sessions when in native tab mode
- Keep TUI as primary interface

**Medium Term**:
- Add iTerm2 badge integration for status
- Investigate daemon architecture for users who prefer CLI-only workflow

**Long Term**:
- Consider full daemon mode as optional advanced feature
- Build iTerm2 status bar component
- Explore cross-terminal integration patterns
