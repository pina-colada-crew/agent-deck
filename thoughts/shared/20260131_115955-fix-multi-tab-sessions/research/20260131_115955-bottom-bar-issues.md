---
date: 2026-01-31T16:59:55+0000
researcher: Claude
git_commit: 8825d57
branch: fix-multi-tab-sessions
repository: agent-deck
topic: "Bottom bar issues when entering a session - shows wrong tabs and suboptimal layout"
tags: [research, codebase, tmux, status-bar, notifications, ui]
status: complete
last_updated: 2026-01-31
last_updated_by: Claude
last_updated_note: "Added follow-up with user's clarified design and implementation plan"
---

# Research: Bottom Bar Issues When Entering a Session

**Date**: 2026-01-31T16:59:55+0000
**Researcher**: Claude
**Git Commit**: 8825d57
**Branch**: fix-multi-tab-sessions
**Repository**: agent-deck

## Research Question

User observed when entering a session (task-server) that has only 2 tabs:

```
[1] sandboxing-research [2] fix the colors [3] coding-platform [4] env-20:server* 2:Tunnel-                                                                                    ctrl+q detach │ 📁 task-server | task-server
```

Issues identified:
1. **Wrong tabs displayed**: Shows `[1] sandboxing-research [2] fix the colors [3] coding-platform` - these are from OTHER sessions, not the current session's 2 tabs
2. **Layout suggestion**: Maybe `📁 task-server | task-server` should be on the left side for better clarity as `📂 session-name | 📁 folder-name`

## Summary

The bottom bar is **working as designed**, but the design may not match user expectations. The `[1] session-name` items are **NOT tmux windows/tabs** - they are agent-deck's **notification bar** showing sessions in "waiting" status across the entire system. The actual tmux window tabs (`env-20:server* 2:Tunnel-`) are from the native tmux window list.

**Key insight**: The notification bar shows waiting sessions globally (cross-session), not the windows within the current session. This is intentional for quick-switching between sessions requiring attention.

## Detailed Findings

### Understanding the Tmux Status Bar Structure

The tmux status bar has three sections:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ [status-left]          [window-list (center)]           [status-right]       │
└──────────────────────────────────────────────────────────────────────────────┘
```

In agent-deck's current implementation:
- **status-left**: Agent-deck's notification bar (`⚡ [1] session-name [2] session-name`)
- **window-list**: Native tmux windows for the CURRENT session (`1:zsh* 2:claude`)
- **status-right**: Session info (`ctrl+q detach │ 📁 task-server | task-server`)

### What User Sees vs What They Expected

| Section | What User Sees | What User Expected |
|---------|---------------|-------------------|
| Left | `[1] sandboxing-research [2] fix the colors [3] coding-platform` | Current session's 2 tab names |
| Center | `[4] env-20:server* 2:Tunnel-` (this is actually native tmux windows) | (implied) nothing extra |
| Right | `ctrl+q detach │ 📁 task-server \| task-server` | On the left side, clearer format |

### Component 1: Notification Bar (status-left)

**File**: `internal/session/notifications.go`

The notification bar tracks **waiting sessions** (status == StatusWaiting), not windows within the current session.

**Lines 149-165** - `FormatBar()`:
```go
func (nm *NotificationManager) FormatBar() string {
    if len(nm.entries) == 0 {
        return ""
    }
    var parts []string
    for _, e := range nm.entries {
        // Format: "[1] session-name [2] other-session"
        parts = append(parts, fmt.Sprintf("[%s] %s", e.AssignedKey, e.Title))
    }
    return "⚡ " + strings.Join(parts, " ")
}
```

**Lines 167-224** - `SyncFromInstances()` - The filtering logic:
```go
func (nm *NotificationManager) SyncFromInstances(instances []*Instance, currentSessionID string) (added, removed []string) {
    // Build set of currently waiting sessions (excluding current)
    waitingSet := make(map[string]*Instance)
    for _, inst := range instances {
        if inst.Status == StatusWaiting && inst.ID != currentSessionID {
            waitingSet[inst.ID] = inst  // All waiting sessions from ALL sessions
        }
    }
    // ... adds entries from waitingSet
}
```

**Key observation**: Line 176 filters by `StatusWaiting` but does NOT filter by session group or parent session. It shows **all** waiting sessions system-wide except the current one.

### Component 2: Status Right (session info)

**File**: `internal/tmux/tmux.go:780-806`

```go
func (s *Session) ConfigureStatusBar() {
    folderName := filepath.Base(s.WorkDir)
    // Format: "ctrl+q detach │ 📁 DisplayName | folderName"
    rightStatus := fmt.Sprintf("#[fg=#565f89]ctrl+q detach#[default] │ 📁 %s | %s ", s.DisplayName, folderName)

    cmd := exec.Command("tmux",
        "set-option", "-t", s.Name, "status", "on", ";",
        "set-option", "-t", s.Name, "status-style", "bg=#1a1b26,fg=#a9b1d6", ";",
        "set-option", "-t", s.Name, "status-left-length", "120", ";",
        "set-option", "-t", s.Name, "status-right", rightStatus, ";",
        "set-option", "-t", s.Name, "status-right-length", "80")
    _ = cmd.Run()
}
```

The format is: `📁 {DisplayName} | {folderName}` where both could be "task-server" if the session title matches the folder name.

### Component 3: Native Tmux Window List (center)

The part showing `env-20:server* 2:Tunnel-` is tmux's native window list. The `*` indicates the currently active window. This is NOT controlled by agent-deck - it's tmux's default behavior showing windows in the current session.

**Note**: `env-20:server* 2:Tunnel-` appears to be malformed/truncated. This suggests the status-left is overlapping into the window-list area, causing visual artifacts.

### Component 4: Global Status Bar Update

**File**: `internal/ui/home.go:934-959`

```go
func (h *Home) updateTmuxNotifications() {
    barText := h.notificationManager.FormatBar()

    if barText != h.lastBarText {
        if barText == "" {
            _ = tmux.ClearStatusLeftGlobal()
        } else {
            _ = tmux.SetStatusLeftGlobal(barText)  // Sets GLOBALLY for all sessions
        }
        _ = tmux.RefreshStatusBarImmediate()
    }
    h.updateKeyBindings()
}
```

**Critical**: `SetStatusLeftGlobal()` sets the notification bar **globally** (`tmux set-option -g status-left`), meaning ALL tmux sessions see the same notification bar content.

**File**: `internal/tmux/tmux.go:2505-2512`
```go
func SetStatusLeftGlobal(text string) error {
    escaped := strings.ReplaceAll(text, "'", "'\\''")
    cmd := exec.Command("tmux", "set-option", "-g", "status-left", escaped)
    return cmd.Run()
}
```

## Code References

- `internal/session/notifications.go:149-165` - FormatBar() formats the notification bar text
- `internal/session/notifications.go:167-224` - SyncFromInstances() builds the waiting sessions list (no group filtering)
- `internal/tmux/tmux.go:780-806` - ConfigureStatusBar() sets status-right with session info
- `internal/tmux/tmux.go:2505-2512` - SetStatusLeftGlobal() sets notification bar globally
- `internal/ui/home.go:934-959` - updateTmuxNotifications() syncs bar text to tmux

## Architecture Insights

### Current Design Intent

The notification bar was designed as a **cross-session quick-switch mechanism**:
- Shows sessions in "waiting" status (AI waiting for user input)
- Allows Ctrl+b 1-6 to quickly switch to any waiting session
- Uses global tmux options for performance (ONE call instead of per-session)

### Why This May Confuse Users

1. **Visual collision**: The numbered `[1] [2] [3]` looks like tmux window/tab numbers, but they're session quick-switch keys
2. **No context boundary**: Users expect to see only "their" session's tabs when inside a session
3. **Status-right layout**: `📁 session | folder` format is unclear when both are the same name

## Potential Fixes (For Future Implementation)

### Issue 1: Wrong tabs displayed

**Option A**: Keep global notification bar (current behavior)
- Pro: Fast cross-session switching
- Con: Can be confusing in multi-session scenarios
- Improvement: Use different visual format to distinguish from tab numbers (e.g., `⚡ sess1 ⚡ sess2` without numbers)

**Option B**: Hide notification bar when inside a session
- Set `status-left` to empty when attached to a session
- Show notifications only in the TUI home view
- Pro: Cleaner session view
- Con: Lose cross-session visibility

**Option C**: Filter by session group
- If sessions have groups, only show waiting sessions from the same group
- Requires sessions to be grouped (may not always apply)

### Issue 2: Layout improvement

Current: `ctrl+q detach │ 📁 task-server | task-server`

**Proposed**: Move session info to left side with clearer format:
- `📂 task-server │ 📁 /path/to/folder        ctrl+q detach`
- Or: `📂 session-name │ 📁 folder-name        ctrl+q detach`

This would require:
1. Moving the folder/session info from `status-right` to `status-left`
2. Adjusting the notification bar position or removing it when attached
3. Updating `ConfigureStatusBar()` in `internal/tmux/tmux.go`

## Open Questions

1. Should the notification bar be visible when inside a session, or only in the TUI?
2. If visible inside a session, should it be filtered to only related sessions (same group)?
3. What's the best visual distinction between notification entries and tmux window tabs?
4. Should the session/folder info use separate icons (📂 session vs 📁 folder)?
5. Should the detach hint move to the right side while session info moves to left?

---

## Follow-up Research: 2026-01-31T17:15:00+0000

### User's Clarified Design

The user clarified the core problem: **status-left and window-list are not visually separated** and users can't tell what each section represents.

**Proposed new layout:**

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Waiting: [!] ◐ sess1  [@] ◐ sess2   Windows: [1] zsh* [2] claude         group │ session │ ctrl+q detach            │
│ [---------- status-left ----------]          [---- window-list ----]     [---------- status-right ----------]       │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

**status-left** (notification bar):
- Add label: `Waiting:` prefix to identify this section
- Use shift-key symbols `[!] [@] [#]` etc. (Ctrl+b ! to switch, no conflict with windows)
- Add yellow circle `◐` after the key (matches "waiting" status indicator in TUI session list)
- Format: `Waiting: [!] ◐ sess1  [@] ◐ sess2`

**window-list** (native tmux):
- Add "Windows:" prefix via `window-status-format` customization
- Window numbers now match Ctrl+b shortcuts (1:zsh = Ctrl+b 1)

**status-right** (session info):
- New format: `group name | session name | ctrl+q detach`
- Shows hierarchy: group → session → action

### Implementation Plan

#### Change 1: Update FormatBar() in notifications.go

**File**: `internal/session/notifications.go:149-165`

**Current**:
```go
func (nm *NotificationManager) FormatBar() string {
    // ...
    for _, e := range nm.entries {
        parts = append(parts, fmt.Sprintf("[%s] %s", e.AssignedKey, e.Title))
    }
    return "⚡ " + strings.Join(parts, " ")
}
```

**New**:
```go
// shiftKeyDisplay maps numbers to shift symbols for display
var shiftKeyDisplay = map[string]string{
    "1": "!", "2": "@", "3": "#",
    "4": "$", "5": "%", "6": "^",
}

func (nm *NotificationManager) FormatBar() string {
    // ...
    for _, e := range nm.entries {
        // Convert "1" -> "!" for display (matches actual Ctrl+b shortcut)
        displayKey := shiftKeyDisplay[e.AssignedKey]
        if displayKey == "" {
            displayKey = e.AssignedKey
        }
        // [!] key binding + yellow ◐ (matches TUI waiting indicator) + session name
        // Use tmux color formatting: #[fg=#e0af68] for yellow (same as ColorYellow in styles.go)
        parts = append(parts, fmt.Sprintf("[%s] #[fg=#e0af68]◐#[default] %s", displayKey, e.Title))
    }
    return "Waiting: " + strings.Join(parts, "  ")  // Double space between entries
}
```

**Note**: `#[fg=#e0af68]` is tmux's color format for the yellow color (`ColorYellow` = `#e0af68` from `internal/ui/styles.go:36`). `#[default]` resets to the default color.

#### Change 2: Add GroupName to tmux Session struct

**File**: `internal/tmux/tmux.go:350-356`

**Current**:
```go
type Session struct {
    Name        string
    DisplayName string
    WorkDir     string
    Command     string
    Created     time.Time
    InstanceID  string
    // ...
}
```

**Add**:
```go
type Session struct {
    Name        string
    DisplayName string
    GroupName   string    // NEW: Group name for status bar display
    WorkDir     string
    Command     string
    Created     time.Time
    InstanceID  string
    // ...
}
```

#### Change 3: Update ConfigureStatusBar() in tmux.go

**File**: `internal/tmux/tmux.go:780-806`

**Current**:
```go
func (s *Session) ConfigureStatusBar() {
    folderName := filepath.Base(s.WorkDir)
    rightStatus := fmt.Sprintf("#[fg=#565f89]ctrl+q detach#[default] │ 📁 %s | %s ", s.DisplayName, folderName)
    // ...
}
```

**New**:
```go
func (s *Session) ConfigureStatusBar() {
    // Right side: group | session | detach hint
    groupPart := ""
    if s.GroupName != "" {
        groupPart = s.GroupName + " │ "
    }
    rightStatus := fmt.Sprintf("%s%s │ #[fg=#565f89]ctrl+q detach#[default] ", groupPart, s.DisplayName)

    // Window format: [1] zsh* instead of default 1:zsh*
    windowFormat := "[#{window_index}] #{window_name}#{window_flags}"

    cmd := exec.Command("tmux",
        "set-option", "-t", s.Name, "status", "on", ";",
        "set-option", "-t", s.Name, "status-style", "bg=#1a1b26,fg=#a9b1d6", ";",
        "set-option", "-t", s.Name, "status-left-length", "120", ";",
        "set-option", "-t", s.Name, "status-right", rightStatus, ";",
        "set-option", "-t", s.Name, "status-right-length", "80", ";",
        // Custom window format for consistent [N] style
        "set-option", "-t", s.Name, "window-status-format", windowFormat, ";",
        "set-option", "-t", s.Name, "window-status-current-format", windowFormat)
    _ = cmd.Run()
}
```

#### Change 4: Pass GroupName when creating tmux Session

Need to update all call sites that create `tmux.Session` to pass the group name from the `Instance.GroupPath`.

**Files to update**:
- `internal/tmux/tmux.go` - `NewSession()`, `NewSessionWithCommand()`, `Create()` functions
- `internal/session/instance.go` - where tmux sessions are created/configured

**Helper function** (in `internal/session/groups.go`):
```go
// extractGroupName already exists at line 271-281
func extractGroupName(path string) string {
    if path == "" {
        return ""
    }
    if idx := strings.LastIndex(path, "/"); idx != -1 {
        return path[idx+1:]
    }
    return path
}
```

Use this when setting `Session.GroupName` from `Instance.GroupPath`.

### Files to Modify

| File | Change |
|------|--------|
| `internal/session/notifications.go:149-165` | Update FormatBar() - add "Waiting:" label, shift keys `[!][@]`, ◐ indicator, "Windows:" suffix |
| `internal/tmux/tmux.go:350-356` | Add `GroupName` field to Session struct |
| `internal/tmux/tmux.go:780-806` | Update ConfigureStatusBar() - new right status format + window-status-format `[N] name` |
| `internal/tmux/tmux.go` | Update NewSession/NewSessionWithCommand to accept GroupName |
| `internal/session/instance.go` | Pass GroupPath → GroupName when creating tmux sessions |

### Visual Comparison

**Before**:
```
⚡ [1] sess1 [2] sess2 [3] sess3 1:zsh* 2:claude  ctrl+q detach │ 📁 task-server | task-server
```

**After**:
```
Waiting: [!] ◐ sess1  [@] ◐ sess2    Windows: [1] zsh* [2] claude    My Sessions │ task-server │ ctrl+q detach
```

Key changes:
- `[!] [@] [#]` = Shift+1, Shift+2, Shift+3 (Ctrl+b ! to switch)
- `[1] [2]` = Window numbers (Ctrl+b 1 to switch) - no conflict!
- `Waiting:` and `Windows:` labels make sections clear
- Consistent `[N]` format throughout, but different key types

### Key Binding Conflict Issue

**Problem discovered**: There's a conflict between waiting session shortcuts and tmux window shortcuts:

| Keys | When waiting sessions exist | When NO waiting sessions |
|------|---------------------------|-------------------------|
| Ctrl+b 1-6 | Switch to waiting session N | Select tmux window N (default) |
| Ctrl+b 7-9 | Select tmux window N | Select tmux window N |

**Current behavior** (from `internal/tmux/tmux.go:2570-2639`):
- Agent-deck binds Ctrl+b 1-6 to switch to waiting sessions
- When a waiting session is removed, `UnbindKey()` restores default window selection
- This means the SAME key does DIFFERENT things depending on context

**Example confusion**:
```
Waiting: [1] ◐ sess1  [2] ◐ sess2    1:zsh* 2:claude
```
- User sees window `1:zsh*` but Ctrl+b 1 switches to `sess1`, not the window!
- The `1:` in window name doesn't match the `[1]` shortcut

### Proposed Solutions

**Option A: Add "Windows:" label + show actual key bindings**
```
Waiting: [^b1] ◐ sess1  [^b2] ◐ sess2    Windows: 1:zsh* 2:claude
```
- Con: Still confusing because window 1 isn't accessible via Ctrl+b 1

**Option B: Use different keys for waiting sessions**
- Use Ctrl+b Shift+1-6 or Ctrl+b F1-F6 for waiting sessions
- Keep Ctrl+b 1-9 for windows (tmux default)
- Pro: No conflict, predictable behavior
- Con: Less convenient shortcut

**Option C: Show dynamic key hints**
```
Waiting: [^b1] ◐ sess1  [^b2] ◐ sess2    Windows: [^b3] 1:zsh* [^b4] 2:claude
```
- Window shortcuts start AFTER waiting sessions
- Pro: Clear what each key does
- Con: Complex to implement (need to customize tmux window-status-format)

**Option D: Disable window shortcuts when waiting sessions exist**
- When waiting sessions bound to 1-6, only show sessions in bar
- Hide or dim window numbers since they're not directly accessible
- Pro: Honest about what's accessible
- Con: Loses window switching capability

**CHOSEN: Option B with Shift modifier**:
- `Ctrl+b 1-9` = tmux windows (unchanged, predictable)
- `Ctrl+b Shift+1-6` = waiting sessions

On US keyboards, Shift+numbers produce symbols:
| Key | Shift+Key | Tmux binding |
|-----|-----------|--------------|
| 1 | ! | `!` |
| 2 | @ | `@` |
| 3 | # | `#` |
| 4 | $ | `$` |
| 5 | % | `%` |
| 6 | ^ | `^` |

So users press: `Ctrl+b !` for waiting session 1, `Ctrl+b @` for waiting session 2, etc.

### Updated Status Bar Format

```
Waiting: [!] ◐ sess1  [@] ◐ sess2    Windows: [1] zsh* [2] claude    group │ session │ ctrl+q detach
```

- `[!]` `[@]` etc. = Ctrl+b ! and Ctrl+b @ to switch to waiting sessions
- `[1]` `[2]` etc. = Ctrl+b 1 and Ctrl+b 2 to switch windows (no conflict!)
- Consistent `[N]` bracketed format makes everything scannable

### Key Benefits

1. **Clear separation**: "Waiting:" and "Windows:" labels identify sections
2. **Visual distinction**: ◐ circles match the TUI waiting indicator
3. **No key conflicts**: Shift+N for sessions, N for windows
4. **Predictable**: Ctrl+b 1 always goes to window 1, never hijacked
5. **Hierarchy on right**: `group │ session │ action` provides context

### Implementation Changes

#### Update BindSwitchKeyWithAck() in tmux.go

**File**: `internal/tmux/tmux.go:2579-2597`

**Current**:
```go
func BindSwitchKeyWithAck(key, targetSession, sessionID string) error {
    // ...
    cmd := exec.Command("tmux", "bind-key", key, "run-shell", script)
    return cmd.Run()
}
```

**New** - Map number to shift symbol:
```go
// shiftKeyMap maps numbers 1-6 to their Shift equivalents (US keyboard)
var shiftKeyMap = map[string]string{
    "1": "!", "2": "@", "3": "#",
    "4": "$", "5": "%", "6": "^",
}

func BindSwitchKeyWithAck(key, targetSession, sessionID string) error {
    // Convert number to shift key (e.g., "1" -> "!")
    shiftKey, ok := shiftKeyMap[key]
    if !ok {
        shiftKey = key // fallback
    }
    // ...
    cmd := exec.Command("tmux", "bind-key", shiftKey, "run-shell", script)
    return cmd.Run()
}
```

#### Update FormatBar() in notifications.go

**File**: `internal/session/notifications.go:149-165`

```go
// shiftKeyMap for display (same as tmux.go)
var shiftKeyDisplay = map[string]string{
    "1": "!", "2": "@", "3": "#",
    "4": "$", "5": "%", "6": "^",
}

func (nm *NotificationManager) FormatBar() string {
    // ...
    for _, e := range nm.entries {
        displayKey := shiftKeyDisplay[e.AssignedKey]
        if displayKey == "" {
            displayKey = e.AssignedKey
        }
        // [!] yellow ◐ + session name
        parts = append(parts, fmt.Sprintf("[%s] #[fg=#e0af68]◐#[default] %s", displayKey, e.Title))
    }
    return "Waiting: " + strings.Join(parts, "  ")
}
```

#### Update UnbindKey() in tmux.go

Need to unbind the shift keys instead of number keys:
```go
func UnbindKey(key string) error {
    shiftKey, ok := shiftKeyMap[key]
    if !ok {
        shiftKey = key
    }
    _ = exec.Command("tmux", "unbind-key", shiftKey).Run()
    // No need to restore default - shift keys don't have default bindings
    return nil
}
```

### Notes

- The group name display uses `extractGroupName()` which shows just the leaf name (e.g., "devops" not "projects/devops")
- Shift key symbols work on US keyboards; international keyboards may vary (could add config option later)
#### Add "Windows:" label to status-left

The "Windows:" label can be appended to `status-left` (after waiting sessions) rather than modifying `window-status-format` (which applies per-window).

**Update FormatBar() return**:
```go
func (nm *NotificationManager) FormatBar() string {
    // ... build parts ...

    if len(parts) == 0 {
        return "Windows: "  // Just show Windows label when no waiting sessions
    }
    // "Waiting: [!] ◐ sess1  [@] ◐ sess2    Windows: "
    return "Waiting: " + strings.Join(parts, "  ") + "    Windows: "
}
```

The native tmux window list will appear immediately after "Windows: ".

#### Customize window-status-format for consistent [N] style

**File**: `internal/tmux/tmux.go:780-806` - Add to `ConfigureStatusBar()`

Change window display from `1:zsh*` to `[1] zsh*` to match the waiting session format:

```go
func (s *Session) ConfigureStatusBar() {
    // ... existing setup ...

    // Window format: [1] zsh* instead of 1:zsh*
    windowFormat := "[#{window_index}] #{window_name}#{window_flags}"

    cmd := exec.Command("tmux",
        "set-option", "-t", s.Name, "status", "on", ";",
        "set-option", "-t", s.Name, "status-style", "bg=#1a1b26,fg=#a9b1d6", ";",
        "set-option", "-t", s.Name, "status-left-length", "120", ";",
        "set-option", "-t", s.Name, "status-right", rightStatus, ";",
        "set-option", "-t", s.Name, "status-right-length", "80", ";",
        // Custom window format for consistent [N] style
        "set-option", "-t", s.Name, "window-status-format", windowFormat, ";",
        "set-option", "-t", s.Name, "window-status-current-format", windowFormat)
    _ = cmd.Run()
}
```

This creates visual consistency:
- Waiting sessions: `[!] ◐ sess1` (Ctrl+b !)
- Windows: `[1] zsh*` (Ctrl+b 1)
