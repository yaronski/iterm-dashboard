# iTerm Dashboard

A macOS menu bar app that shows all open iTerm2 windows grouped by project workspace.

## Features

- **Project grouping** — sessions grouped by working directory with git status badges
- **Claude instance tracking** — memory, CPU, uptime, and busy/idle status for all running Claude Code instances
- **Busy/idle detection** — real-time status indicator showing whether Claude is actively working or waiting for input
- **Companion app tracking** — detects BBEdit, PyCharm, and Finder windows; projects open in these apps appear in the workspace grouping even without terminal sessions
- **Nexus terminal support** — also queries Nexus terminal windows alongside iTerm2
- **Selective window raising** — click a project header to raise only that project's windows; companion app clicks raise only the matching window via AXRaise, not all app windows
- **Color-coded metrics** — semantic colors: blue=navigation, purple=AI/Claude, cyan=remote, green/amber/red=health
- **SF Symbols** — native macOS icons for folders, terminals, CPU, insights
- **Resumable Claude sessions** — sidebar and menu bar show recent Claude Code sessions with click-to-resume
- **Click to focus** — click a session to raise that iTerm2 window, click a companion app to raise its project window
- **Tooltips** — hover for full path, profile, terminal size, and process list
- **Git status** — dedicated row per project showing branch, modified/new/staged counts, ahead/behind, stashes, and last commit age in human-readable format
- **Tree connectors** — pixel-drawn tree lines for visual hierarchy
- **System insights** — warnings for high memory, dirty files, unpushed commits, stale instances
- **Auto-refresh** — updates every 30 seconds, plus on menu open
- **Configurable sidebar strip** — hide or set transparency of the 2px accent strip via the status bar menu (persisted)

## Install

No dependencies beyond Python 3 and PyObjC.

```bash
open iTermDashboard.app
```

To launch on login, add `iTermDashboard.app` to System Settings > General > Login Items.

## Manual Setup (clone install on macOS)

If you cloned this repo (rather than downloading a pre-built `.app`), follow these steps once. Tested on macOS with Homebrew Python and a non-English system locale (de_DE).

### 1. Create a virtualenv and install PyObjC

```bash
cd /Applications/iterm-dashboard          # or wherever you cloned
python3 -m venv .venv
source .venv/bin/activate
pip install pyobjc-core pyobjc-framework-Cocoa pyobjc-framework-Quartz
```

### 2. Test it runs

```bash
.venv/bin/python iTermDashboard.app/Contents/MacOS/itermdashboard
```

A status bar icon (">_") should appear top-right. `Ctrl-C` does **not** quit — run `pkill -f itermdashboard` instead, or use `open` (next step).

### 3. Make `open` work (fix the shebang)

The bundled shebang is `#!/usr/bin/env python3`, which resolves to the system Python — and the system Python does **not** have PyObjC. Point it at your venv Python instead:

```bash
sed -i '' '1s|.*|#'"$(pwd)/.venv/bin/python3"'|' \
  iTermDashboard.app/Contents/MacOS/itermdashboard

# Verify
head -1 iTermDashboard.app/Contents/MacOS/itermdashboard
# → #!/Applications/iterm-dashboard/.venv/bin/python3
```

Now `open` works:

```bash
open iTermDashboard.app
```

> **Note:** `git pull` overwrites this change. Re-run the `sed` command after each pull, or add an alias:
> ```bash
> echo 'alias fix-shebang="sed -i '\'''\'' '\''1s|.*|#'"$(pwd)"'/.venv/bin/python3|'\'' iTermDashboard.app/Contents/MacOS/itermdashboard"' >> ~/.zshrc
> ```

### 4. Launch on login (autostart)

Add the `.app` to login items:

**System Settings → General → Login Items → +** → select `iTermDashboard.app`.

Alternatively, a LaunchAgent (auto-restarts after crashes, uses venv Python directly):

```bash
cat > ~/Library/LaunchAgents/com.deepai.itermdashboard.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.deepai.itermdashboard</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Applications/iterm-dashboard/.venv/bin/python</string>
        <string>/Applications/iterm-dashboard/iTermDashboard.app/Contents/MacOS/itermdashboard</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.deepai.itermdashboard.plist
```

Adjust the absolute paths if you cloned elsewhere.

## Useful Commands

```bash
# Launch the menu bar app
open iTermDashboard.app

# Quit from terminal
pkill -f itermdashboard

# Relaunch (quit + open)
pkill -f itermdashboard; sleep 1; open iTermDashboard.app

# View the main script
cat iTermDashboard.app/Contents/MacOS/itermdashboard

# Check if it's running
pgrep -f itermdashboard && echo "running" || echo "not running"

# See its resource usage
ps aux | grep itermdashboard | grep -v grep
```

## How It Works

1. Queries iTerm2 via AppleScript for all windows/tabs/sessions
2. Runs a single `ps` call to get process trees per TTY
3. Runs a single batched `lsof` call to resolve CWDs for all shell PIDs
4. Runs `git status --porcelain=v2 --branch` (one call per project) for git info
5. Caches everything in a background thread; menu renders instantly from cache

## Project Structure

```
iTermDashboard.app/
  Contents/
    Info.plist                  # App bundle config (LSUIElement for menu-bar-only)
    MacOS/itermdashboard        # Main Python executable
    Resources/AppIcon.icns      # Dock/Finder icon
    Resources/AppIcon.png       # Source icon image
```
