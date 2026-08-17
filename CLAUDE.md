# CLAUDE.md

## What this is

`om-custom-font` is a single Bash script (`./om-custom-font`, ~630 lines, no build
step, no tests) that installs font **files** from disk and wires the family into
Omarchy. It is a **thin wrapper around `omarchy-font-set`** plus the GTK/gsettings
side that Omarchy deliberately does not manage.

The repo is: `om-custom-font` (the script), `README.md`, `CLAUDE.md`, `LICENSE`.

### The scope boundary — don't blur it

Omarchy already covers packaged fonts (`omarchy install font <name> <pkg>
<family>`, a general command) and switching (`omarchy font list` is
`fc-list :spacing=100`, fully dynamic — *not* a fixed list; the six-font menu in
`omarchy-menu.jsonc` is only a GUI shortcut). The single uncovered case is a
`.ttf`/`.otf` sitting on disk that isn't packaged. That is this tool's entire
reason to exist.

So: do not add package installation, font downloading, or font-menu management.
If a change would be better served by `omarchy install font`, it doesn't belong
here. The README says this plainly and should keep saying it.

## Targets Omarchy 4

Check before changing anything:

```bash
cat /usr/share/omarchy/version     # 4.x
omarchy version
```

Omarchy 4 ("Quattro") collapsed Waybar, Walker, Mako, SwayOSD, hyprlock,
hypridle, swaybg, and polkit-gnome into one Quickshell process (the *omarchy
shell*). Anything in this repo that mentions those programs is an Omarchy 3
leftover and is a bug.

## The three rules

1. **Never write `~/.config/fontconfig/fonts.conf`.**
   `omarchy-font-set` owns that file and rewrites it wholesale. This tool calls
   `omarchy-font-set` and, on `reset-all`, deletes the override so the
   package-owned `/etc/fonts/conf.d/50-omarchy.conf` takes over again.

2. **Family is ours; size is Omarchy's. This tool writes no size, ever.**
   `omarchy-display-text-size` (a.k.a. `omarchy display text size`) is the single
   knob for shell `[font] base-size`, GTK `text-scaling-factor`, and terminal pt,
   and it quantizes the GTK factor against the `font-name` point size — so
   writing that point size here corrupts Omarchy's own math.

   Concretely: `--gnome-size` was removed (the flag now errors with a pointer),
   the `--gnome*` flags carry the existing pt over via `gnome_current_size()`,
   and `revert-gnome` restores only `font-name` / `monospace-font-name` — never
   `text-scaling-factor`, because restoring one third of a three-way knob is what
   produces the desync. Reading `text-scaling-factor` in `current` is fine;
   writing it is not. **Do not re-add a size flag.**

3. **Never edit `/usr/share/omarchy/`.** Reading it is the primary way to learn
   what changed; `omarchy update` overwrites any local edit.

## Where the truth lives

Read these directly rather than trusting docs (including this file):

| Question | Read |
|---|---|
| What does setting a font actually do? | `cat $(which omarchy-font-set)` |
| How is the current font resolved? | `cat $(which omarchy-font-current)` — it's `fc-match monospace` |
| Why is my font not in the menu? | `cat $(which omarchy-font-list)` — `fc-list :spacing=100`, monospace only |
| How does sizing work? | `cat $(which omarchy-display-text-size)` |
| What's the system default? | `/usr/share/fontconfig/conf.avail/50-omarchy.conf` |
| How does the shell pick a font? | `/usr/share/omarchy/shell/Commons/Style.qml` (`fontFamily` defaults to `"monospace"`) |

Note `[font]` in `~/.config/omarchy/shell.toml` only parses **integers**
(`parseInt` in `Style.qml`), so there is no `family` key there. The only
family overrides are fontconfig and the `OMARCHY_MENU_FONT` env var.

## Known Omarchy 4 gotchas this tool handles

- **Stale Omarchy 3 fontconfig override.** v3 wrote `mode="assign"`; on v4 the
  packaged `50-omarchy.conf` already assigns `monospace`, so the user rule never
  matches and is silently ignored. v4 uses `mode="prepend_first"`, which wins.
  Verified empirically:
  ```bash
  # prepend_first -> custom font; assign -> falls back to JetBrainsMono
  XDG_CONFIG_HOME=/tmp/testfc fc-match monospace
  ```
  `show_current` flags this as `[STALE - Omarchy 3 format]`.

- **Foot has no reload signal.** Family and size changes only apply to new
  windows. Ghostty gets `SIGUSR2`, Kitty `SIGUSR1`.

- **Terminal size is quantized.** `round(px * 9/12)` — 14px and 15px both give
  11pt, which reads as "the slider didn't move my terminal".

- **`omarchy-font-set` prints errors to stdout**, mixed with `pgrep` noise. Use
  `apply_omarchy_font()`, which captures output and only surfaces it on failure.
  Do not blanket-redirect to `/dev/null`.

## Conventions

- Bash with `set -e`; two-space indent; `local` everywhere inside functions.
- Prefer one `fc-scan`/`gsettings` call and parse it over repeated spawns —
  `get_font_info` and `show_current` both do this deliberately.
- Destructive paths (`reset-all`, single-file install with siblings) prompt or
  back up first. Keep it that way.
- Comments explain *why* Omarchy behaves a certain way, not what the Bash does.

## Verifying a change

There is no test suite. Do this:

```bash
bash -n om-custom-font          # syntax
./om-custom-font help
./om-custom-font current        # read-only, safe; exercises most parsing
```

For `set`/`reset-all`, prefer a throwaway `XDG_CONFIG_HOME` and a scratch
`FONTS_DIR` over mutating the live system. Do not run `reset-all` on the user's
machine to "check it works" — it deletes fonts from `~/.local/share/fonts`.
