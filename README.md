# Qt Scrollbar Diagnostics for AI Coding Agents

One [agent skill](https://github.com/anthropics/skills) for diagnosing scrollbars
that render but are effectively invisible — the "there is no scrollbar on the side
of the window" class of bug — in Qt 6 desktop apps on Wayland.

It came out of a real session: Okular 26.08 on Omarchy (Arch/Hyprland) showed no
document scrollbar at all. It turned out the scrollbar was present, functional,
and drawn in near-white on a near-white page.

## Skill

### `qt-scrollbar-diagnostics`

Finding and fixing low-contrast or unthemed scrollbars in Qt/Qt 6 widgets apps.

- Pixel-level screenshot measurement to establish whether a scrollbar exists
- Ruling out auto-hiding / overlay scrollbars by tracking the handle across scrolls
- Separating app settings from OS theme configuration
- Testing `QT_STYLE_SHEET`, `QT_STYLE_OVERRIDE`, `QT_QPA_PLATFORMTHEME`,
  `GTK_THEME`, and `gtk.css` with throwaway instances
- The `QT_QPA_PLATFORMTHEME=gtk3` behaviour on Omarchy
- Why "change the page background" usually cannot work
- Driving app UI by keyboard on Wayland, where no synthetic mouse click is available

## Why this exists

Almost every instinct in this class of bug is wrong. Specifically:

| Instinct | What actually happens |
|---|---|
| "It's an auto-hiding scrollbar" | Usually not — prove it by scrolling and watching the handle |
| "The app has scrollbars turned off" | Usually already on; check before hunting themes |
| "The desktop theme is at fault" | Frequently not — Qt 6 may not use the desktop theme for this widget at all |
| "A Qt stylesheet will fix it" | Platform-theme styles often ignore stylesheets for scrollbars entirely |
| "Changing the document background will help" | The trough is opaque and belongs to the scrollbar widget, not the canvas |
| "The config file changed, so every window changed" | Long-running processes hold stale settings until restarted |

Each of those was tested here and each was wrong. The skill encodes the
measurements that settled them.

## The worked example

Okular 26.08, `libQt6Widgets` based, running under Omarchy:

```
trough  #F2F2F2      (242)
handle  #FEFEFE      (254)
page    #FEFEFE      (254)
```

A 12/255 contrast difference between handle and trough — invisible at 1:1, but
blinding the moment the levels are stretched.

The fix is **Settings → Color Scheme → Breeze Dark**, which recolours Okular's
chrome and repaints the scrollbar track dark against the still-white page. The
document itself is *not* inverted.

Note that Breeze Dark changes the application shell only. If a user objects to
dark documents, check that before recommending it — in this case they did, and it
turned out not to apply.

## AI disclosure

This skill was written by an AI agent (`opencode`, running under Hyprland on the
affected machine) from an interactive troubleshooting session.

What the agent did directly:

- Captured and pixel-analysed screenshots with `grim` and ImageMagick
- Drove Okular's menus and dialogs by synthesised keystrokes over `hyprctl`
- Ran controlled A/B tests using throwaway `okular --unique` instances
- Read process environments from `/proc/<pid>/environ` to verify that test
  variables actually reached the process under test
- Made and reverted several configuration changes on the live system

What the agent could **not** do:

- Establish the internal mechanism. It could only prove *that* the widget ignores
  every theming input, not *why*. The conclusion that Okular paints the scrollbar
  itself is marked `[INFERRED]`.
- Produce a fix inside Okular. The workaround is a workaround, not an upstream
  patch.
- Verify the fix on anything but one application version on one machine.

Every claim in the skill is tagged `[PROVEN]` or `[INFERRED]` accordingly. Treat
the `[PROVEN]` parts as measured and the `[INFERRED]` parts as hypotheses.

## Install

```bash
git clone https://github.com/ssearles/qt-scrollbar-diagnostics
cp -r qt-scrollbar-diagnostics/skills/qt-scrollbar-diagnostics ~/.agents/skills/
```

## License

MIT