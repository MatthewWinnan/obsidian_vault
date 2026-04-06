# Omarchy Research

> **Source**: [omarchy.org](https://omarchy.org) | [GitHub](https://github.com/basecamp/omarchy) | [Manual](https://learn.omacom.io/2/the-omarchy-manual)
> **Description**: "Beautiful, Modern & Opinionated Linux by DHH" - An Arch-based distro from 37signals/Basecamp

---

## Core Stack Comparison

| Component             | Omarchy                             | My Config                               | Status      |
| --------------------- | ----------------------------------- | --------------------------------------- | ----------- |
| Window Manager        | Hyprland                            | Hyprland                                | ✅ Same      |
| Terminal              | Alacritty (default), Ghostty, Kitty | I want to add improved ghostty configs. | ✅ Same      |
| Shell Prompt          | Starship                            | -                                       | ✅ Same      |
| Multiplexer           | Tmux                                | Zellij                                  | ✅ Same      |
| Editor                | Neovim (omarchy-nvim)               | Nixvim                                  | ✅ Same base |
| Launcher              | Walker                              | Rofi/Wofi                               | ✅ Samej     |
| Bar                   | Waybar                              | Waybar                                  | ✅ Same      |
| Notifications         | Mako                                | -                                       | ✅ Same      |
| Lock Screen           | Hyprlock                            | -                                       | ✅ Same      |
| Idle Daemon           | Hypridle                            | -                                       | ✅ Same      |
| File Manager          | Nautilus                            | Nautilus                                | ✅ Same      |
| PDF Viewer            | Evince                              | Zathura, Sioyek                         | ✅ Same      |
| Image Viewer          | imv                                 | imv                                     | ✅ Same      |
| Video Player          | mpv                                 | mpv, vlc                                | ✅ Same      |
| Screenshot Annotation | satty                               | swappy                                  | ✅ Same      |

---

## CLI Tools Comparison

| Tool         | Purpose                 | Omarchy | My Config            |
| ------------ | ----------------------- | ------- | -------------------- |
| `eza`        | ls replacement          | ✅       | ✅ Have               |
| `zoxide`     | Smart cd                | ✅       | ✅ Have               |
| `fzf`        | Fuzzy finder<br>        | ✅       | ⚠️ Learn some combos |
| `ripgrep`    | Fast grep               | ✅       | ✅ Have               |
| `fd`         | Fast find               | ✅       | ✅ Have               |
| `bat`        | cat with highlighting   | ✅       | ✅ Have               |
| `btop`       | System monitor          | ✅       | ✅ Have               |
| `lazygit`    | Git TUI                 | ✅       | ✅ Have               |
| `lazydocker` | Docker TUI<br>          | ✅       | ✅ Have               |
| `dust`       | Disk usage analyzer     | ✅       | ✅ Have               |
| `fastfetch`  | System info             | ✅       | ✅ Have               |
| `tldr`       | Simplified man pages    | ✅       | ✅ Have (tealdeer)    |
| `jq`         | JSON processor          | ✅       | ✅ Have               |
| `gum`        | Shell script UI toolkit | ✅       | ❌ Skipped            |
| `impala`     | WiFi TUI (iwd)          | ✅       | ✅ Just added         |
| `bluetui`    | Bluetooth TUI           | ✅       | ✅ Have (bluetuith)   |
| `mise`       | Dev env manager         | ✅       | ❌ Skipped (Nix handles this) |

---

## Desktop Apps Comparison

| App           | Purpose                   | Omarchy | My Config              |
| ------------- | ------------------------- | ------- | ---------------------- |
| Chromium      | Browser                   | ✅       | ✅ Have (+ qutebrowser) |
| 1Password     | Password manager          | ✅       | ❌ Different solution?  |
| Obsidian      | Note-taking               | ✅       | ✅ Have                 |
| Typora        | Markdown editor           | ✅       | ✅ Same                 |
| LibreOffice   | Office suite              | ✅       | ✅ Have                 |
| Spotify       | Music                     | ✅       | ✅ Have                 |
| ~~Signal~~    | ~~Messaging~~             | ~~✅~~   | ~~❌ Different~~        |
| OBS Studio    | Recording/streaming       | ✅       | ✅ Have                 |
| ~~Kdenlive~~  | ~~Video editing~~         | ~~✅~~   | ~~❌ Consider adding~~  |
| Pinta         | Image editing             | ✅       | ✅ Have                 |
| ~~Xournal++~~ | ~~PDF annotation~~        | ~~✅~~   | ~~❌ Consider adding~~  |
| LocalSend     | Cross-device file sharing | ✅       | ⚠️ View config         |
| Steam         | Gaming                    | ✅       | ✅ Have                 |
| Tailscale     | VPN                       | ✅       | ⚠️, we need to get     |

---

## AI Integration

| Tool         | Alias   | Purpose                 | My Config                    |
| ------------ | ------- | ----------------------- | ---------------------------- |
| Claude Code  | `cx`    | AI coding assistant     | ✅ Have                       |
| ~~OpenCode~~ | ~~`c`~~ | ~~Alternative AI tool~~ | ~~❌ Check out~~              |
| Voxtype      | -       | Voice dictation         | ✅ Have (voxtype-vulkan)      |

---

## Shell Aliases (Ideas to Adopt)

```bash
# File navigation - ADOPT THESE
alias ls='eza -lh --group-directories-first --icons=auto'
alias lsa='ls -a'
alias lt='eza --tree --level=2 --long --icons --git'
alias ff="fzf --preview 'bat --style=numbers --color=always {}'"
alias eff='$EDITOR "$(ff)"'  # Edit file from fzf

# Smart cd with zoxide - ADOPT THIS PATTERN
alias cd="zd"
zd() {
  if (( $# == 0 )); then
    builtin cd ~ || return
  elif [[ -d $1 ]]; then
    builtin cd "$1" || return
  else
    z "$@"
  fi
}

# Quick open - USEFUL
open() { xdg-open "$@" >/dev/null 2>&1 & }

# Directory shortcuts
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'

# Tool shortcuts
alias d='docker'
alias t='tmux attach || tmux new -s Work'
alias n='nvim .'  # or n() for function version

# Git shortcuts
alias g='git'
alias gcm='git commit -m'
alias gcam='git commit -a -m'
alias gcad='git commit -a --amend'
```

---

## Hyprland Keybindings (Notable Differences)

| Binding | Omarchy Action | Consider Adopting? |
|---------|----------------|-------------------|
| `Super+Space` | App launcher | ⚠️ Check my binding |
| `Super+W` | Close window | ⚠️ Check my binding |
| `Super+T` | Toggle float/tile | ⚠️ Check my binding |
| `Super+L` | Toggle layout (dwindle/scroll) | ✅ Interesting |
| `Super+O` | Pop window (float + pin) | ✅ Useful for reference windows |
| `Super+S` | Scratchpad toggle | ✅ Adopt this |
| `Super+G` | Toggle window grouping | ✅ Tabbed windows |
| `Super+J` | Toggle split | ⚠️ Check |
| `Super+K` | Show keybindings | ✅ Great for discoverability |
| `Super+Ctrl+Space` | Background selector | ✅ Nice workflow |
| `Super+Shift+Ctrl+Space` | Theme selector | ✅ Nice workflow |
| `Super+Comma` | Dismiss notification | ✅ Useful |
| `Super+Ctrl+L` | Lock screen | ⚠️ Check my binding |
| `Ctrl+Alt+Delete` | Close ALL windows | ⚠️ Dangerous but useful |

---

## Tmux Config Highlights

Their tmux config has some nice patterns:

```bash
# Prefix - they use C-Space (I use Zellij)
set -g prefix C-Space

# Pane splits - intuitive
bind h split-window -v  # horizontal
bind v split-window -h  # vertical

# Window navigation with Alt+number
bind -n M-1 select-window -t 1
bind -n M-2 select-window -t 2
# ... etc

# Session navigation
bind -n M-Up switch-client -p
bind -n M-Down switch-client -n

# Nice defaults
set -g mouse on
set -g base-index 1           # 1-indexed
set -g status-position top    # Top status bar
set -g detach-on-destroy off  # Don't detach when session closes
set -g history-limit 50000
```

---

## Themes Available

19 built-in themes (I use Stylix for unified theming):
- catppuccin, catppuccin-latte
- tokyo-night
- nord
- gruvbox
- kanagawa
- rose-pine
- everforest
- matte-black, vantablack
- hackerman
- lumon
- miasma
- osaka-jade
- retro-82
- ristretto
- ethereal
- flexoki-light
- white

---

## Packages to Consider Adding

### High Priority
- [x] ~~`gum`~~ - Skipped (not useful for my workflow)
- [x] `dust` - Better disk usage visualization ✅ 2026-04-05
- [x] `LocalSend` - AirDrop alternative for file sharing ✅ 2026-04-05
- [ ] `Xournal++` - PDF annotation/signing

### Medium Priority
- [x] `Pinta` - Simple image editor (alternative to GIMP) ✅ 2026-04-05
- [ ] `Kdenlive` - Video editing
- [x] `satty` - Screenshot annotation ✅ 2026-04-05 (home/programs/satty)
- [x] `Walker` - Hyprland-native launcher ✅ 2026-04-05 (home/tools/walker.nix)

### Spotify/Music
- [x] `spicetify` - Spotify theming (Catppuccin Mocha) ✅ 2026-04-05
- [x] `spotatui` - TUI Spotify client (via pkgs-unstable) ✅ 2026-04-05

### Voice
- [x] `voxtype-vulkan` - Voice-to-text for Wayland ✅ 2026-04-05

### Shell Aliases
- [x] File navigation aliases (ls, lsa, lt) ✅ 2026-04-05
- [x] fzf aliases (ff, eff) ✅ 2026-04-05
- [x] Smart cd with zoxide (zd function) ✅ 2026-04-05
- [x] open function (xdg-open wrapper) ✅ 2026-04-05
- [x] Tool shortcuts (d, t, n, g, gcm, gcam, gcad) ✅ 2026-04-05

### Workflow Ideas
- [ ] Scratchpad workflow (`Super+S` toggle pattern)
- [ ] Window grouping for tabbed windows
- [ ] Central menu system like `omarchy-menu`
- [ ] Keybindings help overlay (`Super+K`)

---

## Notable Scripts/Utilities

Omarchy has ~180 custom scripts in `/bin/` for:
- Battery monitoring
- Theme management
- Screenshot/screen recording
- WiFi/Bluetooth control panels
- Window management helpers
- System updates with snapshots

Could create similar NixOS modules for some of these.

---

## Summary

Omarchy is well-designed for a keyboard-driven workflow. Main takeaways:
1. **Already aligned**: Core tools (eza, zoxide, fzf, ripgrep, lazygit, etc.)
2. **Different choices**: Zellij vs Tmux, Rofi vs Walker, Zathura vs Evince
3. **Worth adopting**: Scratchpad workflow, window grouping, gum, LocalSend
4. **Nice patterns**: Shell aliases, keybinding help menu, theme selectors

---

## Progress Log

| Date       | Items Added                                                                            |
| ---------- | -------------------------------------------------------------------------------------- |
| 2026-04-05 | dust, LocalSend, Pinta, satty, Walker, spicetify, spotatui, voxtype-vulkan, shell aliases |
