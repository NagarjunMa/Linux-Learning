# Tmux: Complete Reference

## What Tmux Is

- Terminal multiplexer built on libevent; runs as a server/client daemon
- Written in C; runs on any POSIX system (Linux, macOS)
- A single tmux server (`tmux` process) manages multiple **sessions** → **windows** → **panes**
- All state lives in the server — closing the terminal doesn't kill anything

## Tmux vs iTerm2 — Why Tmux Wins for Coders

- **Persistence**: SSH disconnect / laptop lid close / terminal crash → session survives; `tmux attach` brings it back exactly as left
- **Remote-native**: runs on the *server*; iTerm is local-only — if your connection drops, work is lost
- **Keyboard-first**: zero mouse required; everything reachable with prefix + key
- **Scriptable**: `tmux new-session`, `send-keys`, `split-window` are shell commands → automate dev environments
- **Lightweight**: no GUI, no Electron overhead; works over SSH with 1Mbps connections
- **Portability**: same workflow on every server — no per-machine iTerm config

| Feature | iTerm2 | Tmux |
|---|---|---|
| Session persistence after disconnect | No | Yes |
| Works over SSH | No (it's local) | Yes (runs on server) |
| Scriptable layout setup | Limited | Full shell scripting |
| Mouse UX | Excellent | Optional |
| GPU rendering | Yes | No |
| Native macOS integration | Yes | No |
| Portability across servers | No | Yes |
| Resource overhead | High (GUI app) | Minimal |

## Architecture: Server / Client Model

- `tmux` first invocation starts a server (`/tmp/tmux-UID/default` socket)
- Every subsequent `tmux` command is a client talking to the server via Unix socket
- Server holds all session/window/pane state — terminal emulator is just a view
- `tmux kill-server` is the nuclear option — destroys everything

## The Three-Level Hierarchy

```
Session  ─── named context (e.g., "work", "ml-training")
  └── Window  ─── like a browser tab (numbered 0, 1, 2...)
        └── Pane  ─── split within a window (horizontal or vertical)
```

- Multiple sessions per server
- Multiple windows per session
- Multiple panes per window
- Each pane = an independent terminal (separate shell, separate process)

## Default Keybindings (Prefix = Ctrl-b)

**Sessions:**
```
prefix + $     rename session
prefix + s     list/switch sessions (interactive)
prefix + d     detach (leave session running)
prefix + (     previous session
prefix + )     next session
```

**Windows:**
```
prefix + c     new window
prefix + ,     rename window
prefix + n     next window
prefix + p     previous window
prefix + 0-9   jump to window by number
prefix + w     list all windows
prefix + &     kill window
```

**Panes:**
```
prefix + %     split vertically (side by side)
prefix + "     split horizontally (top/bottom)
prefix + arrows   move between panes
prefix + z     zoom pane (toggle fullscreen)
prefix + x     kill pane
prefix + q     show pane numbers
prefix + {/}   swap pane left/right
prefix + !     break pane into its own window
```

**Misc:**
```
prefix + [     enter copy mode (scroll, search, visual select)
prefix + ]     paste from tmux buffer
prefix + :     command prompt (tmux command line)
prefix + ?     list all keybindings
prefix + t     show clock
```

## Session Management Commands

```bash
tmux                          # new unnamed session
tmux new -s work              # new session named "work"
tmux attach                   # attach to last session
tmux attach -t work           # attach to "work"
tmux ls                       # list sessions
tmux kill-session -t work     # kill session
tmux kill-server              # kill everything
tmux rename-session -t old new
```

## Scripting Tmux — Automating Dev Environments

```bash
# Create a reproducible dev layout in one command
tmux new-session -d -s dev -x 220 -y 50
tmux rename-window -t dev:0 editor
tmux send-keys -t dev:editor 'nvim .' Enter

tmux new-window -t dev:1 -n server
tmux send-keys -t dev:server 'uvicorn app:app --reload' Enter

tmux new-window -t dev:2 -n logs
tmux split-window -h -t dev:logs
tmux send-keys -t dev:logs.left 'tail -f app.log' Enter
tmux send-keys -t dev:logs.right 'htop' Enter

tmux attach -t dev
```

Run this as a shell script → instant repeatable workspace.

## Configuration (~/.tmux.conf)

```bash
# Remap prefix to Ctrl-a (screen-style)
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# Vim-style pane navigation
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Sane split keys
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"

# Mouse support (optional)
set -g mouse on

# 256-color + true color
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",xterm-256color:Tc"

# Increase scrollback
set -g history-limit 50000

# Status bar
set -g status-style bg=black,fg=white
set -g window-status-current-style bg=blue,fg=white,bold
set -g status-right ' #{session_name} | %H:%M '

# Start windows at 1, not 0
set -g base-index 1
setw -g pane-base-index 1

# Reload config
bind r source-file ~/.tmux.conf \; display "Config reloaded"
```

## Copy Mode (Scrollback + Clipboard)

```
prefix + [        enter copy mode
q / Escape        exit copy mode
/                 search forward
?                 search backward
n / N             next/prev match
Space             start selection (vi mode)
Enter             copy selection → tmux buffer
prefix + ]        paste tmux buffer
```

Vi keys in copy mode (add to `.tmux.conf`):
```bash
setw -g mode-keys vi
bind -T copy-mode-vi v send -X begin-selection
bind -T copy-mode-vi y send -X copy-selection-and-cancel
```

## Tmux + SSH: The Real Power

```bash
# On remote server: start or reattach
ssh user@server
tmux new -s work        # start
# ... do work, disconnect

ssh user@server
tmux attach -t work     # everything still there

# Named sockets for multiple tmux servers
tmux -L project1 new -s main
tmux -L project1 attach
```

Workflow for AI/ML training runs:
- SSH into RunPod / remote GPU server
- `tmux new -s train`
- Start training, detach (`prefix + d`)
- Close laptop, come back hours later, `tmux attach -t train`
- Training still running, full log scrollback available

## Useful Plugins (TPM — Tmux Plugin Manager)

```bash
# Install TPM
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# Add to .tmux.conf
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'   # sane defaults
set -g @plugin 'tmux-plugins/tmux-resurrect'  # save/restore sessions across reboots
set -g @plugin 'tmux-plugins/tmux-continuum'  # auto-save sessions every 15min
set -g @plugin 'tmux-plugins/tmux-yank'       # sync copy mode to system clipboard

run '~/.tmux/plugins/tpm/tpm'
```

After editing `.tmux.conf`: `prefix + I` to install plugins.

- **tmux-resurrect**: `prefix + Ctrl-s` save, `prefix + Ctrl-r` restore — survives reboots
- **tmux-continuum**: auto-saves every 15 min; set `set -g @continuum-restore 'on'` for auto-restore on tmux start
- **tmux-yank**: `y` in copy mode copies to system clipboard (macOS pbcopy / Linux xclip)

## Common Coder Workflows

1. **Monorepo multi-service dev**: window per service (API, frontend, DB, logs)
2. **Remote ML training**: detach after launching, reattach to check loss curves
3. **Pair programming via tty**: two SSHers attach to same session (`tmux attach -t shared`) — both see same screen
4. **Quick scratchpad**: `tmux new-window`, run something, close — back to main window
5. **Monitoring dashboard**: one window with panes for `htop`, `nvidia-smi -l 1`, `tail -f train.log`, `df -h`
