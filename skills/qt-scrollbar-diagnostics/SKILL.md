---
name: qt-scrollbar-diagnostics
description: >
  Diagnose scrollbars that are present but invisible in Qt 6 / Qt Widgets desktop
  apps on Linux Wayland — establishing whether a scrollbar actually exists,
  measuring its contrast against the surrounding background, and finding what can
  and cannot change it. Triggers: app has no scrollbar, scrollbar invisible,
  scrollbar not showing, cannot see the scrollbar, scrollbar too faint, scrollbar
  white on white, scrollbar hidden, no scroll bar on the side of the window,
  okular no scrollbar, kate no scrollbar, libreoffice no scrollbar, pdf viewer no
  scrollbar, low contrast scrollbar, qt scrollbar theme, QT_STYLE_SHEET scrollbar
  ignored, QT_QPA_PLATFORMTHEME gtk3 scrollbar, kde app scrollbar invisible,
  diagnose missing scrollbar. Excludes scrolling-layout configuration and
  compositor-side overlay scrollbars for X11 clients.
---

# Qt Scrollbar Diagnostics

Field-tested on Okular 26.08 (Qt 6.11, Qt Widgets, Wayland) on Omarchy
(Arch/Hyprland). The measurement technique is general to any XDG desktop on Linux.

**Confidence markers used throughout:**

- `[PROVEN]` — directly observed and measured
- `[INFERRED]` — conclusion reached from evidence, mechanism not confirmed

---

## Rule 0 — measure, do not look

Do not judge a scrollbar by eye and do not accept the user's description of it.
Eye judgement is exactly what fails here: the Okular scrollbar was a 12/255
contrast difference, present in every screenshot, invisible to a person.

Render the region and read the pixel values. On Wayland:

```bash
# 1. find the window geometry
hyprctl activewindow -j | python3 -c \
  "import json,sys; d=json.load(sys.stdin); print(d['at'][0], d['at'][1], d['size'][0], d['size'][1])"

# 2. capture it (x, y, width, height)
grim -g "$x,$y ${w}x${h}" shot.png

# 3. read a vertical column through the scrollbar, as a run-length profile
magick shot.png -crop 20x$((h-100))+$((w-20))+50 +repage \
  -colorspace gray txt:- | python3 -c "
import sys, re
from collections import Counter
col = {}
for line in sys.stdin:
    m = re.match(r'(\d+),(\d+): \((\d+)', line)
    if m: col.setdefault(int(m.group(1)), []).append(int(m.group(3)))
c = col[10]
runs, st = [], 0
for i in range(1, len(c)+1):
    if i == len(c) or c[i] != c[st]:
        runs.append((c[st], st, i-1, i-st)); st = i
for v, a, b, n in runs:
    if n >= 4: print(f'  level {v:3d}  y {a}-{b}  ({n}px)')
"
```

A run-length profile of one column tells you the whole story: groove level,
handle level, handle extent, handle position, and whether the arrows exist at all.

`[PROVEN]` Cross-instance pixel histograms are **not** comparable. The handle
sits at a different height in each capture depending on scroll position, so
"dominant colours" differed between two runs of the *same* configuration. Compare
fixed regions or read the run profile instead.

---

## Rule 1 — prove whether a scrollbar exists

Three independent tests, cheapest first.

### 1a. Does it change when the content scrolls?

An auto-hiding or overlay scrollbar appears only during interaction. An
always-present one is in the frame at all times. Scroll and capture immediately:

```bash
hyprctl eval "hl.dispatch(hl.dsp.send_shortcut({ mods = '', key = 'Page_Down' }))"
grim -g "$x,$y ${w}x${h}" after.png    # capture with no intervening sleep
```

**If the handle moved, the scrollbar is present and tracking.** That single fact
eliminates the entire auto-hide / overlay family of explanations.

`[PROVEN]` On Okular the handle profile shifted monotonically across three
captures (rows 295-320, then 301-326, then 313-338) while the groove stayed at
242. The scrollbar was fully functional and always visible.

### 1b. Is the app's own setting on?

Check before hunting themes. Settings are usually in the app, not the desktop.

`[PROVEN]` Okular keeps this under **Settings → General → Appearance →
"Show scrollbars"**, alongside "Use smooth scrolling". It was already ticked.
Note the config file is `~/.config/okularrc`, not `~/.config/okularpart/okularrc`,
and some keys live in a sibling `~/.config/okularpartrc` — check both.

### 1c. Does another scrollbar in the same window render correctly?

This is the highest-value test and it is cheap. Okular's sidebar tree had a
perfectly visible scrollbar while the document view had none, in the same process,
same style, same theme. That immediately localises the fault to one widget rather
than the desktop.

---

## Rule 2 — know what is actually painting the widget

Do this before trying any fix, because it determines which fixes can possibly work.

```bash
# which Qt style / platform theme is loaded
tr '\0' '\n' < /proc/<pid>/maps | grep -oE '[^/]*\.so[^ ]*' \
  | sort -u | grep -iE 'style|theme|gtk|breeze|fusion'

# available alternatives to try
ls /usr/lib/qt*/plugins/platformthemes/ /usr/lib/qt*/plugins/styles/ 2>/dev/null
```

`[PROVEN]` Okular had `libqgtk3.so` mapped. Omarchy sets this globally at
`/usr/share/omarchy/default/hypr/envs.lua` via `hl.env("QT_QPA_PLATFORMTHEME",
"gtk3")` so that Qt apps inherit the desktop GTK theme's palette.

**Critical consequence:** the GTK3 platform theme paints scrollbars with real GTK
widgets, which means **Qt stylesheets do not reach them.** See Rule 3.

---

## Rule 3 — test every theming knob on a throwaway instance

Never conclude "I can't restyle it" from a failed attempt on the user's live
window. Run a controlled instance and measure it.

```bash
env "$@" okular --unique "$PDF" &      # --unique forces a separate process
sleep 3
PID=$(pgrep -n -f 'okular --unique')
```

**Always verify the variable reached the process.** A surprising number of
"failures" are typos that never got delivered:

```bash
tr '\0' '\n' < /proc/$PID/environ | grep -o 'QT_STYLE_SHEET=.\{0,40\}'
```

**Always include a positive control.** `QT_STYLE_SHEET='* { background: #FF0000; }'`
then count saturated pixels in the capture:

```bash
magick shot.png -colorspace sRGB \
  -fx "(r>0.9&&g<0.1&&b<0.1)||(g>0.9&&r<0.1)||(b>0.9&&r<0.1)?1:0" \
  -format "%[fx:mean*w*h]" info:
```

Zero red pixels with the variable confirmed in `/proc` is a real result.

`[PROVEN]` Results on Okular — every one measured, none changed a single pixel of
the scrollbar:

| Override | Result |
|---|---|
| `QT_STYLE_SHEET='* { background: #FF0000; }'` | 0 red pixels (control proves it reached the process) |
| `QT_STYLE_SHEET` recolouring `QScrollBar::handle` | no change |
| `QT_STYLE_OVERRIDE=Fusion` | no change |
| `QT_QPA_PLATFORMTHEME` unset | no change — Qt auto-detects gtk3 when unset |
| `QT_QPA_PLATFORMTHEME=xdgdesktopportal` | no change |
| `QT_QPA_PLATFORMTHEME=minimal` | no change |
| `GTK_THEME=HighContrast` | no change |
| `~/.config/gtk-3.0/gtk.css` scrollbar rules | no change |

Note the unset case: **unsetting `QT_QPA_PLATFORMTHEME` does not disable it.**
Qt probes for an available platform theme and finds gtk3. You must name a
different one explicitly.

`[INFERRED]` Okular's page view draws its own scrollbar with baked-in colours,
which is why nothing external perturbs it.

---

## Rule 4 — changing the page background will not help

Tempting and wrong. "The scrollbar is invisible against the white page, so let me
make the page grey."

The scrollbar is a **separate widget** docked at the window edge, roughly 13px
wide, with its **own opaque** groove. It does not sit on top of the canvas, so
canvas colour never shows through it.

`[PROVEN]` Tested on Okular via **Settings → General → Appearance → "Use custom
background color"**, set to `#8B8B8B`. Scrollbar pixels came back byte-identical:

```
before   trough #F2F2F2   handle #FEFEFE
after    trough #F2F2F2   handle #FEFEFE
```

The only visible effect was a grey strip down the left margin, which looked worse.

The setting key is `UseCustomBackgroundColor` under `[PageView]` in
`~/.config/okularpartrc`.

---

## Rule 5 — find a contrast-inverting workaround

Once the scrollbar's own colours are fixed by the app, the only lever left is the
page background behind it, or the app's own colour scheme.

For Okular: **Settings → Color Scheme → Breeze Dark**. It offers Default,
Breeze Dark, and Breeze Light.

`[PROVEN]` Breeze Dark repaints the scrollbar:

```
trough  #232323   (35)
handle  #3C3C3C   (60)
```

A dark track against the still-white page is unmistakable.

**Check what the scheme actually inverts before recommending it.** Breeze Dark
recolours the application chrome only — toolbar, sidebar, menus. **The PDF page
stays white with dark text.** A user who says "I don't like dark pages" may still
be fine with this. Ask, do not assume; and do not describe it as a dark-page mode
if it is not one.

---

## Rule 6 — long-running processes hold stale settings

A config file is read at **startup**. Editing it changes nothing for a process
that is already running.

`[PROVEN]` Two Okular windows on two workspaces were **separate processes**
(pids 32276 and 25626). `ColorScheme=BreezeDark` was written to `~/.config/okularrc`
and applied to one window. The other stayed light indefinitely because it had been
running since before the change.

When a fix appears to work "sometimes", check whether the windows are separate
processes:

```bash
hyprctl clients -j | python3 -c "
import json,sys
for c in json.load(sys.stdin):
    if 'okular' in c.get('class','').lower():
        print('ws', c['workspace']['name'], 'pid', c['pid'], c['title'][:45])
"
```

Either apply the change in each running instance, or restart them.

---

## Appendix A — Hyprland 0.56 Lua API

Hyprland 0.55+ uses Lua config. `hyprctl dispatch <old syntax>` **silently
fails** — it wraps your arguments as `hl.dispatch(<args>)` and evaluates them as
Lua, so `hyprctl dispatch workspace 3` becomes `hl.dispatch(workspace 3)` and
errors on an undefined global. Use `hyprctl eval` with real Lua:

```bash
hl.dispatch(hl.dsp.focus({ workspace = '3' }))
hl.dispatch(hl.dsp.focus({ window = 'org.kde.okular' }))
hl.dispatch(hl.dsp.send_shortcut({ mods = '', key = 'Page_Down' }))
hl.dispatch(hl.dsp.send_shortcut({ mods = 'ALT', key = 'S' }))
```

Gotchas found the hard way:

- `send_shortcut` requires `mods`, even as an empty string
- A window selector like `window = 'org.kde.okular'` may not match; omit `window`
  to target the focused window
- `hl.dsp.workspace` only has `move`, `rename`, `swap_monitors`, `toggle_special`.
  Switching workspaces is `hl.dsp.focus({ workspace = 'N' })`
- Always verify the resulting state with `hyprctl activeworkspace` /
  `hyprctl activewindow`; a command returning `ok` does not mean it did anything
- Introspect with `for k in pairs(t)`, but note lazily-populated tables return
  nothing to `pairs`

---

## Appendix B — driving a GUI without a mouse

On Wayland there is no `xdotool` and no synthetic click dispatcher. Work with
keyboard mnemonics and focus traversal.

- `Alt+<mnemonic>` opens a menu — verify by screenshot before navigating
- `Home` then `Down` N times is more reliable than counting from a remembered
  position, because menus reopen on the **last used** item
- **Verify the highlight with a screenshot before each `Right`/`Return`.** A
  miscount lands you in the wrong menu and the next keypress does something
  unexpected
- Dialog buttons respond to their mnemonic: `Alt+O` for OK worked when `Return`
  did not
- When a checkbox is the target, `Shift+Tab` back to it — `Space` on the adjacent
  colour button opens a colour picker instead

**Always screenshot to confirm state between steps.** The cost is a few seconds;
the cost of skipping it is a settings dialog in an unknown state.

---

## Appendix C — cleanup discipline

After a diagnostic session involving live application windows:

- Revert every experimental setting change and verify the config file afterwards
- Kill throwaway instances (`pgrep -f 'app --unique'`)
- Restore the user's original workspace
- Confirm no test files were left in their config directory

Be explicit with the user about what was changed and what was reverted.