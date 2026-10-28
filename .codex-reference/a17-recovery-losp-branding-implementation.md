# Android 17 LOSP recovery branding implementation

Implemented the LOSP recovery palette and restored the Android 16 LOSP artwork. No unrelated recovery behavior, strings, device logic, layout, input handling, animation, or progress assets were changed. No build or flash was performed.

## Files changed

- `recovery_ui/screen_ui.cpp`
- `res-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/images/{fastbootd,logo_image,logo_image_switch,ic_back,ic_back_sel}.png` (25 PNGs)

## Palette

| UI element | Recovery | Fastbootd |
|---|---|---|
| `INFO`, `HEADER` | `#10C8C8` | `#FFD835` |
| `MENU` | `#D8D8D8` | `#D8D8D8` |
| `MENU_SEL_BG`, `SCROLLBAR` | `#007A7A` | `#E65100` |
| `MENU_SEL_BG_ACTIVE` | `#10C8C8` | `#FFD835` |
| `MENU_SEL_FG` | black | black |
| `BATTERY_LOW` | `#FD3535` | `#FD3535` |

`LOG` remains `#C4C4C4`; `TEXT_FILL` remains black at alpha 160.

## Android 17 menu surface and corners

`MENU_BG` remains in the enum and `SetColor()` because `TextMenu::DrawItems()` and `GraphicMenu::DrawItems()` call it. Its fill is now opaque black, matching the existing framebuffer clear and avoiding a visible ordinary-row panel. Android 17's current `TextMenu` drawing loop still calls `DrawHighlightBar()` for every row; with a short press, ordinary rows therefore paint black over the black canvas. The existing long-press ternary uses `MENU_SEL_BG_ACTIVE` before checking whether a row is selected, so it can apply the active accent to all rows during the existing long-press state. This selection behavior was preserved as requested.

A search of `minui` and `recovery_ui` found no native rounded-rectangle or corner-radius primitive. Selected bars remain rectangular using the existing `gr_fill()` path and unchanged A17 insets, dimensions, margins, and clipping. No custom raster helper or bitmap background was added.

## Artwork reuse

All 25 PNGs were restored byte-for-byte from Android 16 LOSP commit `f0188ad1e63b946500a7e00bbc489cbdb5d663f0`. All five paths are present in every requested density directory. No other recovery assets were changed.

## Validation

- `git diff --check HEAD^ HEAD` and `git diff --check HEAD~2 HEAD`: passed.
- Confirmed all 25 expected asset paths exist and byte-match the Android 16 LOSP reference commit.
- Confirmed `MENU_BG` remains declared, has a `SetColor()` case, and both menu implementations retain their callers.
- Confirmed source retains the A17 row bar dimensions and margin-aware x/width calculations.
- Confirmed recovery/fastbootd color choices remain mode-aware in `SetColor()`.
- Rounded-rectangle API search returned no suitable primitive.
- No build or device flash was run.

## Commits

1. `49cb7767` — `recovery: apply LOSP palette to Android 17 UI`
2. `6c72ff66` — `recovery: restore LOSP recovery and fastboot artwork`

Both commits are signed off. The two source/artwork commits contain only their intended files. The supplied reference patch and audit remain untracked input files in `.codex-reference` and were not modified.
