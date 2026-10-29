---
date: 2026-01-31T12:00:00+0000
author: Claude
git_commit: 8825d57
branch: fix-multi-tab-sessions
repository: agent-deck
feature: "Redesign tmux status bar for clarity"
status: implemented
last_updated: 2026-01-31
---

# Implementation Plan: Status Bar Redesign

**Goal**: Redesign the tmux status bar to clearly separate waiting sessions from tmux windows, eliminate key binding conflicts, and show session hierarchy.

## Problem Summary

The current status bar confuses users because:
1. **No visual separation** between notification bar (waiting sessions) and tmux window list
2. **Key binding conflict**: Ctrl+b 1-6 is hijacked for waiting sessions, blocking window switching
3. **Status-right format** is unclear when session name matches folder name

## Proposed Solution

**New layout:**
```
Waiting: [!] ◐ sess1  [@] ◐ sess2    [1] zsh* [2] claude    group │ session │ ctrl+q detach
[---------- status-left ----------]  [-- window-list --]    [---------- status-right ----------]
```

Key changes:
- Use shift-key symbols `[!][@]#$%^]` for waiting sessions (no conflict with windows)
- Add "Waiting:" label when waiting sessions exist
- Add yellow `◐` indicator matching TUI waiting status
- Change window format from `1:zsh*` to `[1] zsh*` for visual consistency
- Show `group │ session │ ctrl+q detach` on right side

---

## Phase 1: Update Key Bindings to Use Shift Keys

**Objective**: Change waiting session shortcuts from `Ctrl+b 1-6` to `Ctrl+b !@#$%^` to eliminate conflict with tmux window switching.

### Changes

#### 1.1 Add shift key mapping constant

**File**: `internal/tmux/tmux.go`
**Location**: Near line 2570 (before BindSwitchKeyWithAck)

Add:
```go
// shiftKeyMap maps numbers 1-6 to their Shift equivalents (US keyboard)
// Used for waiting session shortcuts to avoid conflict with window selection
var shiftKeyMap = map[string]string{
	"1": "!", "2": "@", "3": "#",
	"4": "$", "5": "%", "6": "^",
}
```

#### 1.2 Update BindSwitchKeyWithAck()

**File**: `internal/tmux/tmux.go:2579-2597`

Change the key binding to use shift symbols:
```go
func BindSwitchKeyWithAck(key, targetSession, sessionID string) error {
	// Convert number to shift key (e.g., "1" -> "!")
	shiftKey := key
	if mapped, ok := shiftKeyMap[key]; ok {
		shiftKey = mapped
	}

	signalFile, err := GetAckSignalPath()
	if err != nil {
		// Fall back to simple binding
		cmd := exec.Command("tmux", "bind-key", shiftKey, "switch-client", "-t", targetSession)
		return cmd.Run()
	}

	script := fmt.Sprintf("echo '%s' > '%s' && tmux switch-client -t '%s'",
		sessionID, signalFile, targetSession)
	cmd := exec.Command("tmux", "bind-key", shiftKey, "run-shell", script)
	return cmd.Run()
}
```

#### 1.3 Update UnbindKey()

**File**: `internal/tmux/tmux.go:2627-2639`

Remove the default restore logic (shift keys have no default binding):
```go
func UnbindKey(key string) error {
	// Convert number to shift key
	shiftKey := key
	if mapped, ok := shiftKeyMap[key]; ok {
		shiftKey = mapped
	}

	// Just unbind - shift keys have no default binding to restore
	return exec.Command("tmux", "unbind-key", shiftKey).Run()
}
```

---

## Phase 2: Update Notification Bar Format

**Objective**: Add "Waiting:" label, use shift-key display symbols, and add yellow `◐` indicator.

### Changes

#### 2.1 Add shift key display mapping

**File**: `internal/session/notifications.go`
**Location**: Near line 20 (after imports)

Add:
```go
// shiftKeyDisplay maps assigned keys (1-6) to their shift symbol equivalents for display
// Matches the actual tmux key binding (Ctrl+b ! for session 1, etc.)
var shiftKeyDisplay = map[string]string{
	"1": "!", "2": "@", "3": "#",
	"4": "$", "5": "%", "6": "^",
}
```

#### 2.2 Update FormatBar()

**File**: `internal/session/notifications.go:149-165`

```go
// FormatBar returns the formatted status bar text
func (nm *NotificationManager) FormatBar() string {
	nm.mu.RLock()
	defer nm.mu.RUnlock()

	if len(nm.entries) == 0 {
		return ""  // Empty - let native window list show without prefix
	}

	var parts []string
	for _, e := range nm.entries {
		// Convert "1" -> "!" for display (matches actual Ctrl+b shortcut)
		displayKey := e.AssignedKey
		if mapped, ok := shiftKeyDisplay[e.AssignedKey]; ok {
			displayKey = mapped
		}
		// [!] + yellow ◐ (matches TUI waiting indicator) + session name
		// #[fg=#e0af68] = tmux color format for yellow (ColorYellow from styles.go)
		// #[default] resets to status bar default color
		parts = append(parts, fmt.Sprintf("[%s] #[fg=#e0af68]◐#[default] %s", displayKey, e.Title))
	}

	// "Waiting: [!] ◐ sess1  [@] ◐ sess2  " - double space between entries, trailing spaces for separation
	return "Waiting: " + strings.Join(parts, "  ") + "  "
}
```

**Note**: Return empty string when no waiting sessions. This lets the native tmux window list display cleanly without a prefix.

---

## Phase 3: Update Status Bar Configuration

**Objective**: Add GroupName to Session struct, update status-right format, and customize window display format.

### Changes

#### 3.1 Add GroupName field to Session struct

**File**: `internal/tmux/tmux.go:350-356`

Add `GroupName` field:
```go
type Session struct {
	Name        string
	DisplayName string
	GroupName   string    // Group name for status bar display (e.g., "devops" from "projects/devops")
	WorkDir     string
	Command     string
	Created     time.Time
	InstanceID  string
	// ... rest of fields unchanged
}
```

#### 3.2 Update ConfigureStatusBar()

**File**: `internal/tmux/tmux.go:780-806`

```go
func (s *Session) ConfigureStatusBar() {
	// Right side: group | session | detach hint
	var rightStatus string
	if s.GroupName != "" {
		// Full format with group: "group │ session │ ctrl+q detach"
		rightStatus = fmt.Sprintf("%s │ %s │ #[fg=#565f89]ctrl+q detach#[default] ", s.GroupName, s.DisplayName)
	} else {
		// No group: "session │ ctrl+q detach"
		rightStatus = fmt.Sprintf("%s │ #[fg=#565f89]ctrl+q detach#[default] ", s.DisplayName)
	}

	// Window format: [1] zsh* instead of default 1:zsh*
	// Provides visual consistency with notification bar format
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

#### 3.3 Pass GroupName when syncing display name

**File**: `internal/session/instance.go:2533-2539`

Update `SyncTmuxDisplayName()` to also sync group name:
```go
// SyncTmuxDisplayName updates the tmux status bar to reflect the current title.
func (i *Instance) SyncTmuxDisplayName() {
	if tmuxSess := i.GetTmuxSession(); tmuxSess != nil && tmuxSess.Exists() {
		tmuxSess.DisplayName = i.Title
		tmuxSess.GroupName = extractGroupName(i.GroupPath)
		tmuxSess.ConfigureStatusBar()
	}
}
```

**Note**: `extractGroupName()` already exists at `internal/session/groups.go:273-281`. It extracts the leaf name from a group path (e.g., "projects/devops" -> "devops").

#### 3.4 Export extractGroupName or inline it

**File**: `internal/session/groups.go:271-281`

The function is currently unexported (lowercase). Two options:

**Option A** (preferred): Export it by renaming to `ExtractGroupName`:
```go
// ExtractGroupName extracts the display name from a group path
// e.g., "parent/child" -> "child", "root" -> "root"
func ExtractGroupName(path string) string {
	if path == "" {
		return ""
	}
	if idx := strings.LastIndex(path, "/"); idx != -1 {
		return path[idx+1:]
	}
	return path
}
```

Then update all internal callers to use `ExtractGroupName`.

**Option B**: Since `SyncTmuxDisplayName` is in the same package, no change needed - it can call the unexported `extractGroupName` directly.

Going with **Option B** - no change needed to groups.go.

---

## Phase 4: Initialize GroupName on Session Creation

**Objective**: Ensure GroupName is set when tmux sessions are created, not just when synced.

### Changes

#### 4.1 Update NewInstance()

**File**: `internal/session/instance.go:164-179`

After creating tmux session, set GroupName:
```go
func NewInstance(title, projectPath string) *Instance {
	id := generateID()
	tmuxSess := tmux.NewSession(title, projectPath)
	tmuxSess.InstanceID = id

	groupPath := extractGroupPath(projectPath)

	return &Instance{
		ID:          id,
		Title:       title,
		ProjectPath: projectPath,
		GroupPath:   groupPath,
		Tool:        "shell",
		Status:      StatusIdle,
		CreatedAt:   time.Now(),
		tmuxSession: tmuxSess,
	}
}
```

The tmuxSession.GroupName will be set on first `SyncTmuxDisplayName()` call (which happens when the session is started/attached).

**Alternative**: Set it immediately:
```go
func NewInstance(title, projectPath string) *Instance {
	id := generateID()
	groupPath := extractGroupPath(projectPath)

	tmuxSess := tmux.NewSession(title, projectPath)
	tmuxSess.InstanceID = id
	tmuxSess.GroupName = extractGroupName(groupPath)  // Set immediately
	// ...
}
```

Going with the first approach - rely on `SyncTmuxDisplayName()` since that's the pattern already used for DisplayName.

---

## Files to Modify

| File | Changes |
|------|---------|
| `internal/tmux/tmux.go:350-356` | Add `GroupName` field to Session struct |
| `internal/tmux/tmux.go:780-806` | Update ConfigureStatusBar() - new status-right format, window-status-format |
| `internal/tmux/tmux.go:2570` | Add shiftKeyMap constant |
| `internal/tmux/tmux.go:2579-2597` | Update BindSwitchKeyWithAck() to use shift keys |
| `internal/tmux/tmux.go:2627-2639` | Update UnbindKey() to unbind shift keys |
| `internal/session/notifications.go:20` | Add shiftKeyDisplay constant |
| `internal/session/notifications.go:149-165` | Update FormatBar() - new format with labels and indicators |
| `internal/session/instance.go:2533-2539` | Update SyncTmuxDisplayName() to set GroupName |

---

## Visual Comparison

**Before:**
```
⚡ [1] sess1 [2] sess2 [3] sess3 1:zsh* 2:claude  ctrl+q detach │ 📁 task-server | task-server
```

**After (with waiting sessions):**
```
Waiting: [!] ◐ sess1  [@] ◐ sess2    [1] zsh* [2] claude    My Group │ task-server │ ctrl+q detach
```

**After (no waiting sessions):**
```
[1] zsh* [2] claude                                          My Group │ task-server │ ctrl+q detach
```

---

## Success Criteria

### Automated Verification

- [x] `make build` - Code compiles without errors
- [x] `make test` - All existing tests pass (note: TestSession_SendCtrlC is a pre-existing flaky test)
- [x] `make lint` - No linting errors

### Manual Verification

- [x] **Key binding test**: Create a waiting session, verify Ctrl+b ! switches to it (not Ctrl+b 1)
- [x] **Window switching test**: With waiting sessions, verify Ctrl+b 1 still switches to window 1
- [x] **Visual test - with waiting**: Status bar shows `Waiting: [!] ◐ name` format
- [x] **Visual test - without waiting**: Status bar shows clean window list without "Waiting:" prefix
- [x] **Group display test**: Session in a group shows `groupname │ sessionname │ ctrl+q detach`
- [x] **No group test**: Session without group shows `sessionname │ ctrl+q detach`
- [x] **Color test**: Yellow `◐` indicator matches TUI waiting status color

---

## Implementation Notes

1. **US Keyboard assumption**: Shift+1-6 produces `!@#$%^` on US keyboards. International keyboards may differ. This could be made configurable in future if needed.

2. **Empty case handling**: When no waiting sessions exist, `FormatBar()` returns empty string. This lets the native tmux window list display without any prefix, which looks cleaner.

3. **GroupName propagation**: The GroupName flows from `Instance.GroupPath` -> `extractGroupName()` -> `tmuxSession.GroupName` -> `ConfigureStatusBar()`. This happens on every `SyncTmuxDisplayName()` call.

4. **Window format**: Changed from `1:zsh*` to `[1] zsh*` for visual consistency. The brackets make it clear these are window indices, matching the notification bar style.
