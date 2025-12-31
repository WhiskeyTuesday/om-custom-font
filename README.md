# om-custom-font

Custom font installer for [Omarchy](https://omarchy.org/) with full system integration.

## Features

- Install fonts from single file or directory (full family)
- Automatic detection of variable fonts vs traditional font families
- Sets font across all Omarchy components (waybar, terminal, hyprlock, swayosd, fontconfig)
- Optional GNOME/GTK integration (Walker, file dialogs)
- Timestamped backups for GNOME settings with restore menu
- Reset to Omarchy defaults with one command

## Installation

```bash
cp om-custom-font ~/.local/bin/
chmod +x ~/.local/bin/om-custom-font
```

## Usage

```bash
# Install and set as system default
om-custom-font set ~/Downloads/MyFont.ttf
om-custom-font set ~/FontFamily/                    # directory with full family

# Also update GTK apps (Walker, file dialogs)
om-custom-font set ~/Downloads/Font.ttf --gnome
om-custom-font set ~/Downloads/Font.ttf --gnome-mono   # monospace only
om-custom-font set ~/Downloads/Font.ttf --gnome-font   # UI font only
om-custom-font set ~/Downloads/Font.ttf --gnome --gnome-size=12  # custom size (experimental)

# Install without setting as default (adds to Omarchy font menu)
om-custom-font add ~/Downloads/Font.ttf

# Show current font settings across all systems
om-custom-font current

# Reset everything to Omarchy defaults
om-custom-font reset-all

# Restore previous GNOME settings from backup
om-custom-font revert-gnome
```

## How It Works

### Font Installation

When you run `set` or `add`, the tool:

1. Copies font file(s) to `~/.local/share/fonts/`
2. Updates the font cache with `fc-cache`
3. Detects the font family name automatically

For directories, it installs all `.ttf`/`.otf` files and prompts you to choose a family if multiple are found.

### Variable vs Traditional Fonts

The tool detects whether a font is:

- **Variable font**: Single file containing all weights/styles (e.g., Adwaita Sans). Works fully with single file.
- **Traditional font**: Separate files per weight/style (e.g., JetBrainsMono). Warns if you only install one file, suggests installing the directory.

### What Gets Updated

| Component | Config Location |
|-----------|-----------------|
| Waybar | `~/.config/waybar/style.css` |
| Terminal (Ghostty/Kitty/Alacritty) | respective config files |
| Hyprlock | `~/.config/hypr/hyprlock.conf` |
| SwayOSD | `~/.config/swayosd/style.css` |
| Fontconfig | `~/.config/fontconfig/fonts.conf` |
| GNOME/GTK (optional) | gsettings `org.gnome.desktop.interface` |

### GNOME/GTK Integration

Omarchy uses Hyprland, not GNOME, but many apps use GTK and read font settings from gsettings:

- **Walker** (app launcher)
- **File dialogs** (open/save)
- **Any GTK application**

Use `--gnome`, `--gnome-font`, or `--gnome-mono` to update these. Each creates a timestamped backup you can restore with `revert-gnome`.

## Requirements

- [Omarchy](https://omarchy.org/) (provides `omarchy-font-set`, `omarchy-font-list`, etc.)
- fontconfig (`fc-cache`, `fc-query`, `fc-list`, `fc-scan`)
- xmlstarlet
- gsettings (for `--gnome` options)

## License

MIT
