# om-custom-font

Install font **files** on [Omarchy](https://omarchy.org/) 4 and wire them into the
system properly.

## Why this exists

Omarchy handles fonts well, as long as the font is packaged. It does not handle
font files:

| You have | Omarchy's answer |
|---|---|
| A font already installed | `omarchy font set "Family"` — works, any family, not a fixed list |
| A font in the repos or AUR | `omarchy install font <display-name> <package> <family>` |
| One of six curated Nerd Fonts | Menu → Install → Style → Font |
| **A `.ttf`/`.otf` on disk** | **nothing** |

That last row is the gap. A font you bought, downloaded from a foundry, built
yourself, or that simply isn't packaged has no path in. There's nothing wrong
with Omarchy's design here — it just assumes a package manager, and some fonts
don't come that way.

To be clear about what is *not* a gap: `omarchy font list` is
`fc-list :spacing=100`, so it's fully dynamic — once a font is installed by any
means it shows up and is selectable. And `omarchy install font` is a general
command, not a fixed menu; the curated list of six is only the GUI shortcut, and
you can add your own rows to `~/.config/omarchy/extensions/omarchy-menu.jsonc`.

## Is this tool worth it?

The manual equivalent is genuinely short:

```bash
cp Font.ttf ~/.local/share/fonts/ && fc-cache -f && omarchy font set "Family Name"
```

If that's all you need, do that. `omarchy-font-set` accepts any installed family,
so it works fine.

What the tool adds is the parts that are fiddly rather than long:

- **Family-name detection** via `fc-query` — the name is rarely the filename
  (`CascadiaCode.ttf` → `CaskaydiaMono Nerd Font`), and `omarchy font set` fails
  on a wrong guess.
- **Variable vs. traditional detection** — warns before you install one file of a
  nine-file family and get synthetic bold for the next six months.
- **Whole-directory installs** with family disambiguation when an archive
  contains several.
- **The GTK side** — `monospace-font-name` is a separate setting from
  fontconfig's `monospace` alias, and Omarchy 4 doesn't set it.
- **A reset path** back to stock, including the stale-config check below.
- **A stale Omarchy 3 fontconfig override detector** — see below; this one is a
  real silent failure.

## Targets Omarchy 4

Omarchy 4 ("Quattro") replaced Waybar, Walker, Mako, SwayOSD, hyprlock, hypridle,
swaybg, and polkit-gnome with a single Quickshell-based *omarchy shell*, and made
fontconfig the single source of truth for font resolution. Earlier releases of
this tool targeted Omarchy 3 and no longer apply.

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

# Also update GTK apps (file dialogs, GTK-based apps)
om-custom-font set ~/Downloads/Font.ttf --gnome
om-custom-font set ~/Downloads/Font.ttf --gnome-mono   # monospace only
om-custom-font set ~/Downloads/Font.ttf --gnome-font   # UI font only

# Install without setting as default (adds to the Omarchy font menu)
om-custom-font add ~/Downloads/Font.ttf

# Show current font settings across all systems
om-custom-font current

# Reset everything to Omarchy defaults
om-custom-font reset-all

# Restore previous GNOME settings from backup
om-custom-font revert-gnome
```

## Font size is not this tool's job

This tool sets font **family**. Omarchy 4 owns font **size** through one knob
that moves the shell, GTK, and terminals in lockstep:

```bash
omarchy display text size          # show current
omarchy display text size 16       # set (9-20 px)
omarchy display text size reset    # back to defaults (12px / 1.0 / 9pt)
```

It is also exposed in the shell's **Display** panel as the *Text size* slider.

The knob is anchored at 12px:

| `text size` | omarchy shell `[font] base-size` | GTK `text-scaling-factor` | Terminal pt |
|---|---|---|---|
| 12 (default) | 12 | 1.0 | 9 |
| 14 | 14 | ~1.18 | 11 |
| 16 | 16 | ~1.36 | 12 |

`om-custom-font` never writes a size. The `--gnome*` flags swap the *family* and
carry the existing point size over untouched, because `omarchy-display-text-size`
computes `text-scaling-factor` by quantizing against the GTK `font-name` point
size — writing that point size directly desyncs the knob.

`--gnome-size=N` existed in earlier versions and now errors out with a pointer to
`omarchy display text size`. `revert-gnome` likewise restores only the two family
settings; it deliberately does **not** restore `text-scaling-factor`, since
putting back one third of a three-way knob is what causes the desync described
below.

### Known: text size looks unsynchronised with terminals

Reports of "it resizes system text but not terminal text" are usually one of:

- **Coarse quantization.** Terminal pt is `round(px * 9/12)`, so 14px and 15px
  both land on 11pt — a slider step that visibly moves the shell may not move
  the terminal at all.
- **No live reload.** Kitty gets `SIGUSR1` and Ghostty `SIGUSR2`, but **Foot has
  no config-reload signal** — running windows keep their startup size until
  relaunched. Omarchy sends a notification when it detects this.
- **A stale config.** `omarchy-display-text-size` rewrites terminal sizes with
  anchored `sed` patterns (`^font-size = `, `^size =`, `^font_size `, `:size=`).
  A hand-edited config that doesn't match those patterns is skipped silently.

`om-custom-font current` prints the shell/GTK/terminal values side by side so you
can see which one is out of step.

## How It Works

### Font Installation

When you run `set` or `add`, the tool:

1. Copies font file(s) to `~/.local/share/fonts/`
2. Updates the font cache with `fc-cache`
3. Detects the font family name automatically
4. For `set`, hands the family to `omarchy-font-set`

For directories, it installs all `.ttf`/`.otf` files and prompts you to choose a
family if multiple are found.

### Variable vs Traditional Fonts

The tool detects whether a font is:

- **Variable font**: single file containing all weights/styles (e.g. Adwaita Sans). Works fully from one file.
- **Traditional font**: separate files per weight/style (e.g. JetBrainsMono). Warns if you install only one file and suggests installing the directory.

### What Gets Updated

`omarchy-font-set` does the system-wide work. In Omarchy 4 that is:

| Component | Mechanism |
|-----------|-----------|
| **fontconfig (source of truth)** | rewrites `~/.config/fontconfig/fonts.conf` |
| Omarchy shell (bar, menus, notifications, OSD, lock) | `Style.fontFamily` defaults to the `monospace` alias; shell restarted |
| Ghostty | `~/.config/ghostty/config` + `SIGUSR2` |
| Kitty | `~/.config/kitty/kitty.conf` + `SIGUSR1` |
| Alacritty | `~/.config/alacritty/alacritty.toml` |
| Foot | `~/.config/foot/foot.ini` (needs relaunch) |
| Qt apps | resolve `monospace` via fontconfig |
| Hooks | fires the `font-set` hook |
| GNOME/GTK (optional, this tool) | gsettings `org.gnome.desktop.interface` |

Waybar, SwayOSD, and hyprlock no longer exist in Omarchy 4 — they were absorbed
into the omarchy shell, which follows fontconfig.

> `omarchy-font-set` **overwrites** `~/.config/fontconfig/fonts.conf` wholesale.
> Any hand-written fontconfig rules there will be lost. Put custom rules in
> `~/.config/fontconfig/conf.d/` instead.

### Stale Omarchy 3 override

Omarchy 3 wrote a full `~/.config/fontconfig/fonts.conf` using
`mode="assign"`. Omarchy 4's package-owned `/etc/fonts/conf.d/50-omarchy.conf`
already assigns `monospace`, so a leftover assign-style user rule **never matches
and is silently ignored**: terminals and GTK show your custom font while
`fc-match monospace` still resolves to `JetBrainsMono Nerd Font`. Omarchy 4 uses
`mode="prepend_first"`, which wins.

`om-custom-font current` flags this as `[STALE - Omarchy 3 format]`. Fix it by
re-applying the font:

```bash
omarchy font set "Your Font Family"
```

### Monospace-only font menu

`omarchy font list` is `fc-list :spacing=100`, so **only monospace families
appear** in the Omarchy font menu. A proportional family can still be set
explicitly and works fine for GTK UI text, but won't be listed. The tool warns
when the family you installed is not monospace.

### GNOME/GTK Integration

Omarchy runs Hyprland, not GNOME, but GTK apps still read fonts from gsettings:

- File dialogs (open/save)
- GTK-based applications

GTK's `monospace-font-name` is a *separate* setting from fontconfig's `monospace`
alias — without these flags your GTK text views stay on `Adwaita Mono` while
everything else follows the font you set.

Use `--gnome`, `--gnome-font`, or `--gnome-mono` to update them. Each creates a
timestamped backup of the two family settings that you can restore with
`revert-gnome`. Omarchy 4 does not set these keys itself, so `reset-all` uses
`gsettings reset` to return them to their schema defaults (`Adwaita Sans 11` /
`Adwaita Mono 11`).

### Related Omarchy 4 commands

```bash
omarchy font list                  # available monospace fonts
omarchy font current               # fc-match monospace
omarchy font set <name>            # apply a family
omarchy install font <display-name> <package> <family>   # install any font *package*
omarchy display text size [n]      # the size knob
```

If your font is packaged, use `omarchy install font` — it's the blessed path and
it keeps the font under the package manager, which this tool cannot do.

## Requirements

- [Omarchy 4](https://omarchy.org/) (provides `omarchy-font-set`, `omarchy-font-current`, `omarchy-display-text-size`)
- fontconfig (`fc-cache`, `fc-query`, `fc-list`, `fc-scan`, `fc-match`)
- gsettings (for the `--gnome*` options)

## License

MIT
