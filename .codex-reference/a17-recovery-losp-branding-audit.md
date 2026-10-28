# Android 17 LOSP recovery branding port audit

**Scope:** Read-only audit of the Android 17 `bootable/recovery` checkout against the supplied Android 16 reference patch, `.codex-reference/0001-recovery-rebrand-UI-to-LOSP-identity.patch`. No tracked files were modified and no commit was created.

## Executive findings

- The reference patch contains one source change (`recovery_ui/screen_ui.cpp::ScreenRecoveryUI::SetColor`) and replaces five image assets in each of five density directories (25 PNGs total).
- The Android 17 source still has the old Android 16 palette in `SetColor`, plus a new `UIElement::MENU_BG`. The new element and menu drawing behavior came from Lineage commit `48e92395` (July 2026); `8f64ad16` later restored background drawing only for selected menu rows (August 2026). The enum case remains because the menu calls it in its all-rows drawing implementation, even though the recent follow-up only invokes it for selected rows.
- There is no general new teal/dark-teal full-screen recovery background variable or resource. Standard recovery menu and progress rendering clear the framebuffer to opaque black. The new “background colour/surface” is the `MENU_BG` row fill (a darkened 20% accent), which has no visible area when unselected rows skip drawing it. It can become visible in menu implementations that do draw all rows (including `GraphicMenu`) and on any remaining code path invoking `MENU_BG`.
- The reference patch applies cleanly as a source-level palette mapping only after adapting for `MENU_BG` and the new item layout. Its old color-case assumptions do not match all current callers.
- Android 17 uses the same names and density directory set for the relevant five branded assets. Their current canvases match the reference-era canvas dimensions at all densities; the metadata audit found RGB opaque images, including black canvas backgrounds. This means swapping in old assets should not require geometric regeneration based on dimensions alone. They should be visually reviewed against the current menu placement and black canvas, and only regenerated if the intended new composition must incorporate the menu’s `MENU_BG` surface.

## 1. Android 16 patch changes and Android 17 equivalents

The reference patch header describes the visual intent. Its actual diff contains exactly the following:

| Android 16 patch change | Android 17 equivalent / disposition |
|---|---|
| `recovery_ui/screen_ui.cpp`: palette comment and `SetColor()` replacements | Same function and color switch remains in `recovery_ui/screen_ui.cpp` near line 588. Directly port color values with semantic adaptation described below. |
| Recovery `INFO` and `HEADER`: `#10C8C8` | `INFO` and `HEADER` still exist. Current callers use `INFO` for title/help/battery percent and `HEADER` for menu headers. Android 16 merges `INFO` and `HEADER` to the same recovery/fastboot accent; preserve that behavior if matching baseline. |
| Fastboot `INFO` and `HEADER`: `#FFD835` | Same use sites, but current `MENU_BG` adds a separate fastboot tint. Keep fastboot values mode-dependent via `fastbootd_logo_enabled_`. |
| Battery low red `#FD3535` for both modes | Same `BATTERY_LOW` case. Android 16 removed the recovery-only magenta distinction; directly port the uniform red. |
| Menu text gray `#D8D8D8` | Same `MENU` case for ordinary menu text. Direct. Selected foreground differs: Android 16 changes `MENU_SEL_FG` to black in both modes, while A17 baseline currently uses gray for recovery and black for fastboot. |
| Recovery selected background / scrollbar `#007A7A`; fastboot `#E65100` | `MENU_SEL_BG` and `SCROLLBAR` are still the same enum values. Direct accent mapping. New selected long-press case is separate and needs a deliberate choice. |
| Active/long-press selection: Android 16 makes it mode-dependent `#10C8C8` / `#FFD835` | `MENU_SEL_BG_ACTIVE` remains. It is still hard-coded green (`#009C64`) in Android 17. Adapt to the LOSP mode accent as in the reference. |
| `TEXT_FILL` remains black with alpha 160 | Same case/value; it is used by Wear UI’s full-screen clear, not normal `ScreenRecoveryUI` menu fill. Preserve unless expanding the scope to Wear. |
| Replace `fastbootd.png`, `ic_back.png`, `ic_back_sel.png`, `logo_image.png`, `logo_image_switch.png` in `res-{mdpi,hdpi,xhdpi,xxhdpi,xxxhdpi}/images` | Exact same file paths exist. Reference patch assets can be applied directly at the file level; assess composition and opaque black canvas behavior below. |

The patch does **not** change animations (`loopNNNNN`), text images, font, error icon, progress bars, or stage markers. Its “reduced glow” claim is embodied in the supplied replacement logo/back PNG content; there is no source rendering/glow parameter change.

## 2. Direct ports versus semantic adaptations

**Direct:** Recovery/fastboot primary accent values, battery-low red, menu text color, selected background and scrollbar accent, and the five sets of replacement PNGs. `SetColor()` still branches on `fastbootd_logo_enabled_`, and the core `gr_color()` RGB(A) interface is unchanged.

**Adapt:**

1. Add explicit `MENU_BG` handling. It did not exist in the Android 16 patch’s enum or SetColor switch. Android 17 currently calculates it as 20% of the purple recovery selection color (`#1910 0?` precisely channel-wise `0x7c*0.20`, `0x4d*0.20`, `0xff*0.20`) or 20% of fastboot dark amber. In integer channel terms this is approximately `#180F33` recovery and `#2E1000` fastboot. The current `MENU_BG` color is used for unselected menu-row fills in `48e92395`’s layout; `8f64ad16` then changed the text-menu caller to skip those fills for unselected entries. Set it to a compatible dark teal in option A, or remove its use/neutralize it with option B.
2. Keep `MENU_SEL_BG_ACTIVE` mode-aware as the reference intended. Android 17’s menu row loop calls it for `long_press` regardless of selected row in the original `48e92395` implementation; in that state every drawn row gets the active fill. In current `screen_ui.cpp`, only selected text-menu rows get a bar after `8f64ad16`, and it uses the active color when long pressed.
3. `MENU_SEL_FG`: Android 16 explicitly sets black in both modes. Android 17’s current default is gray for recovery and black for fastboot, so this case needs explicit harmonization if preserving the Android 16 identity. Contrast should be considered alongside selected row teal.
4. `INFO` and `HEADER` are separate enum cases but Android 16 folds them together. Same values should be applied to both cases.
5. The menu header is now laid out separately from options, with a horizontal rule, vertical spacing, row insets, and margin-aware positioning. Avoid changing layout/touch behavior as part of the branding port.

## 3. Background rendering and affected surfaces

### Where it is defined and rendered

`UIElement::MENU_BG` is declared in `recovery_ui/include/recovery_ui/screen_ui.h` near line 38. Its RGB value is assigned in `ScreenRecoveryUI::SetColor()` in `recovery_ui/screen_ui.cpp` near line 611. `TextMenu::DrawItems()` sets this color for the unselected path (near line 203); `DrawHighlightBar()` calls `gr_fill()` using the current color. The initial implementation came from `48e92395 recoveryui: Adjust menu items style` (July 11 author date, merged July 26, 2026). `8f64ad16 recovery_ui: only draw background for selected menu item` (August 12, 2026) updated the text menu to draw its bar only when selected. `37c5d17b` (September 14 author date) only applied horizontal margins to these bars and did not change their color semantics.

There is no new full-screen background color in `ScreenRecoveryUI`. In normal menu display, `draw_screen_locked()` calls `gr_color(0, 0, 0, 255); gr_clear();` before drawing content. In non-menu progress/status rendering, `draw_background_locked()` likewise clears opaque black. The background text art and logos are drawn as images over that black clear. The progress foreground erases only its progress rectangle to black.

### Surface-by-surface map

| Surface | Android 17 rendering and current color behavior |
|---|---|
| Normal recovery | Full-screen opaque black from `draw_screen_locked()` or `draw_background_locked()`. No broad teal fill. LOSP cyan appears in menu header/info text and selected highlights/scrollbar via `SetColor()`. |
| Fastbootd | Same black clear and same menu renderer; mode changes by `fastbootd_logo_enabled_`, selecting `fastbootd.png` and `logo_image_switch.png` and the fastboot palette. It is not a separately filled screen color. |
| Menus | Black canvas, opaque logo at top, then title/header and rows. `MENU_BG` is the dark accent row fill for non-selected rows in the 2026 revised menu drawing. Selected row uses `MENU_SEL_BG`, or `MENU_SEL_BG_ACTIVE` during long press. In the current text-menu follow-up, bars are drawn only for selected rows, so unselected `MENU_BG` is not visible there; `GraphicMenu` and other callers can still request it. |
| Selected menu rows | Solid rectangle (`DrawHighlightBar` → `gr_fill`) in dark accent; text color controlled by `MENU_SEL_FG`. On long press, active color is used. |
| Headers | Menu header strings use `HEADER`; title/help/battery capacity text use `INFO`. Both are cyan recovery / amber fastboot under the reference palette. These are glyph colors, not a filled header panel. |
| Text/log surfaces | Log lines use `LOG` (`#C4C4C4`). `TEXT_FILL` is translucent black (`alpha=160`) when used by `WearRecoveryUI` to clear/fill its screen. Standard `ScreenRecoveryUI` menu canvas itself is black; it has no distinct log panel background. Background status phrases such as “installing” are grayscale localized image masks tinted white, not a colored surface. |
| Image/logo backgrounds | Logos and back arrows are drawn by `gr_blit()` from RGB images. Current Android 17 logo and back assets are opaque RGB with black canvas corners, so black is baked into the image; framebuffer/menu teal does not show through. Animation/status icons are separate assets and are not changed in the reference patch. |

`gr_blit` directly copies display surfaces; it does not composite alpha. The resource loader supports RGB, grayscale, and RGBX/RGBA input conversion, but its display surface path is not an alpha blend. The Android 17 assets audited here are RGB and opaque in all densities.

## 4. Two implementation options for the new row background

### A. Retain the surface and change it to a darker LOSP teal-compatible color

Keep `MENU_BG` and its row geometry, but use a darker teal derived from recovery `#10C8C8` (and a dark amber-compatible value in fastbootd if the system retains mode-specific surfaces). This makes every call site defined and preserves the 2026 menu treatment for implementations that draw unselected bars. It provides a subtle menu-row surface with clear color-family identity and preserves distinction between normal and selected rows. It also means the menu no longer exactly matches the flatter Android 16 appearance, and the dark surface must be checked against black logo canvas and both ordinary and selected text colors. TextMenu currently hides this surface for unselected rows because of `8f64ad16`; therefore a color change alone may have no visible effect there today.

### B. Remove or neutralize the surface to reproduce the flatter Android 16 appearance

Remove the `MENU_BG` bar use from the menu drawing loop (and then remove the enum case if no other callers remain), or make that fill equivalent to the black canvas and preserve existing geometry only if still required. This leaves ordinary rows on the black framebuffer and uses solid highlight colors only for selection/long press, matching the flatter Android 16 concept. It avoids an extra teal panel competing with the logos and LOSP accents. If neutralizing rather than removing, use an opaque black value: `DrawHighlightBar()` is a solid fill, so setting alpha low is not a translucent overlay because `gr_fill()` renders to the framebuffer rather than compositing with an underlying surface. Removing the draw is cleaner than keeping a redundant black fill, provided all UIElement callers are checked.

**Recommendation:** Option B is technically cleanest for faithful Android 16 LOSP appearance. There is no independent Android 17 full-screen surface to recolor; the visual delta is specifically an unselected-row fill, introduced by a newer menu style. The clean implementation should remove the unused unselected-row bar branch and, if a repository-wide search confirms no other consumer, remove `MENU_BG` from the enum and SetColor. Preserve row spacing, touch selection, margins, and selected/active bars. If product intent instead wants the 2026 Lineage row treatment, choose A and explicitly document the deliberate visual evolution.

## 5. `UIElement` inventory and semantics

Android 16 reference patch’s source context has: `BATTERY_LOW`, `HEADER`, `MENU`, `MENU_SEL_BG`, `MENU_SEL_BG_ACTIVE`, `MENU_SEL_FG`, `SCROLLBAR`, `LOG`, `TEXT_FILL`, `INFO`.

Android 17 has all those names and additionally `MENU_BG`. No existing element was renamed or removed. `MENU_BG` is the only new enum element. Current case semantics remain broadly the same, with these differences:

- `INFO` still separate in the enum, but the Android 16 patch’s switch deliberately groups its color with `HEADER`.
- `MENU_SEL_FG` was explicitly black for both modes in the Android 16 patch; A17 source still uses light gray for recovery, black for fastbootd.
- `MENU_SEL_BG_ACTIVE` remains one case, but the Android 16 patch made it mode-specific; A17 source retains a green value.
- `MENU_BG` is a new dark accent. Its menu caller exists in `TextMenu::DrawItems()` and `GraphicMenu::DrawItems()`; current text menu only draws the bar on selected rows as of `8f64ad16`, but GraphicMenu still draws bars on all items using the background color for ordinary rows.
- The current row layout and selected foreground behavior changed after the Android 16 baseline (extra vertical spacing, inset bar, title/header separation). These are layout changes, not new color elements.

## 6. Resource directories, composition and reuse

All five directories (`res-mdpi`, `res-hdpi`, `res-xhdpi`, `res-xxhdpi`, `res-xxxhdpi`) contain the same 35 image names: five branded candidates plus text art, `font_menu`, 30 loop frames, error icon, stage markers, and progress bar images. Android.bp packages `res-*/images/*.png` per density directory. There are no added Android 17 recovery branding image names or additional density directories.

Dimensions checked from current Android 17 resource metadata:

| Asset | mdpi | hdpi | xhdpi | xxhdpi | xxxhdpi |
|---|---:|---:|---:|---:|---:|
| `logo_image`, `logo_image_switch`, `fastbootd` | 256×160 | 384×240 | 512×320 | 768×480 | 1024×640 |
| `ic_back`, `ic_back_sel` | 34×34 | 52×52 | 80×80 | 120×120 | 160×160 |

These are the same canvases as the reference patch’s pre-change Android 16 images (as shown by the binary sizes and reference patch assets), so no new sizing constraint prevents direct reuse. Each current branded candidate is RGB, opaque, and has a black canvas corner. The use of RGB and black canvas is compatible with current `gr_blit()` semantics. The images are not transparent cutouts and will obscure whatever color is behind their full rectangle. They are drawn centered at the top of the menu with no scaling; back arrows are also drawn directly.

Asset recommendations:

- **`logo_image.png`**: reusable as the recovery logo. Source selection: used when dynamic partitions/fastbootd support is unavailable. Keep intended background compatible with the framebuffer black unless a redesigned full-canvas background is intentional.
- **`logo_image_switch.png`**: reusable as the recovery logo for devices with dynamic partitions/fastbootd available. It is used for both modes as the common switch-capable lineage logo.
- **`fastbootd.png`**: reusable as the mode-specific fastbootd logo. Loaded only when fastbootd capability is enabled and selected as the displayed menu logo while fastbootd mode is active. Same dimensions as the two logo images.
- **`ic_back.png`, `ic_back_sel.png`**: reusable as custom normal and selected back arrows. They are drawn for non-main menus and selection `-1`; their same-canvas density set exists. Ensure their dark canvas corners are not interpreted as transparent when composited next to future nonblack surface colors.

Because current files are all opaque and have exact reference-era dimensions, regeneration is **not required merely to port the assets**. Regenerate/redesign only if choosing a new teal-backed composition, changing canvas footprint/placement, or if visual comparison shows the Android 16 art was designed for a different aspect/layout than these 256:160/34:34 slots. The 2026 menu adds vertical spacing and a header rule but does not resize or scale logos; it shifts the subsequent content down by the same fixed logo canvas height. No evidence in source says Android 17 changed the expected logo dimensions or transparency contract.

## 7. Specific asset answers

| Asset | Loaded / drawn | Reuse assessment |
|---|---|---|
| `logo_image` | `ScreenRecoveryUI::Init()` selects it when neither dynamic partitions nor `ro.fastbootd.available` is true; drawn at centered x and `margin_height_` y in menu display. | Reuse exact A16 variant. Maintain expected 256:160 density-scaled canvas and account for opaque black background. |
| `logo_image_switch` | Selected when dynamic partitions or fastbootd availability is true; used as the recovery identity image in switch-capable devices. | Reuse exact A16 variant. It remains the shared recovery logo for the fastboot-capable path. |
| `fastbootd` | Loaded for capable devices; selected in `draw_menu_and_text_buffer_locked()` when `fastbootd_logo_enabled_` is true. | Reuse exact A16 variant; no source-side dimension or alpha contract change. |
| `ic_back` | Loaded at init and rendered next to a submenu logo when menu is not main. | Reuse exact A16 normal arrow. Current RGB opaque corners mean background-color change won’t show behind its canvas. |
| `ic_back_sel` | Used in place of `ic_back` when back row (`selection() == -1`) is selected. | Reuse exact A16 selected arrow. Confirm contrast against black logo canvas and new row position. |

## 8. Other resources and possible branding scope

No Android 17-added LOSP-specific resource name appears. The image families that are visually present but untouched by the reference patch include:

- `loop00000.png`–`loop00029.png`: progress animation frames, displayed only in graphical/status mode.
- `erasing_text.png`, `error_text.png`, `installing_text.png`, `installing_security_text.png`, `no_command_text.png`: localized status word art/masks.
- `icon_error.png`, `stage_empty.png`, `stage_fill.png`, `progress_empty.png`, `progress_fill.png`, and `font_menu.png`.

These are existing upstream recovery resources, not Android 17 additions, and the Android 16 baseline intentionally left them alone. For a minimal LOSP identity port, do not theme them absent a concrete visual inconsistency; avoid introducing changes to recovery status communication or animations.

## 9. Relevant local history

- `f0188ad1` (April 27, 2026): Android 16 baseline LOSP rebrand represented by the supplied patch. The patch itself records exact source palette and asset substitutions.
- `48e92395` (author date July 11, merged July 26, 2026), `recoveryui: Adjust menu items style`: added `UIElement::MENU_BG`, spacing, row insets, and filled all item-row backgrounds with a darkened 20% mode accent. Its stated purpose was distinguishing header text from clickable options.
- `8f64ad16` (August 12, 2026), `recovery_ui: only draw background for selected menu item`: reverted background-bar drawing for non-selected text rows while keeping spacing and touch changes. It did not remove `MENU_BG` or alter `SetColor`; in the current text-menu path the element is consequently not visible for unselected rows.
- `37c5d17b` (author date September 14, 2026), `recoveryui: Apply margin_width_ when drawing menu items`: fixed row geometry for devices with horizontal display margins. This should be preserved when adapting the palette or simplifying the unselected-row surface.

The history indicates the background behavior is an Android 17-era menu style change layered after the A16 port, not a new recovery framebuffer clear or an Android 17 change to `gr_clear()`.

## 10. Scope constraints for implementation

The eventual branding patch should touch only `recovery_ui/screen_ui.cpp` palette/menu-surface logic and the five branded image files across five density folders. Do not alter unrelated recovery strings, device-specific logic, recovery behavior, touch/key selection, menu spacing/margins, package logic, status animations, or non-brand assets.

## 11. Minimal reviewable commit structure

Recommended two-commit series:

1. **`recovery: apply LOSP palette to Android 17 UI`** — port `SetColor()` values, explicitly handle the `MENU_BG` decision (recommended: remove its redundant unselected text-row fill and enum case only if all consumers can be removed safely; otherwise neutralize the current call path without changing geometry), preserve Android 16 recovery/fastboot semantics, and leave menu layout/touch code intact. Keep the source diff limited to `screen_ui.cpp` and, only if necessary after caller audit, `screen_ui.h`.
2. **`recovery: restore LOSP recovery and fastboot artwork`** — replace `fastbootd`, both logo variants, and both back arrows across all five density directories in one asset-only commit. This makes binary review/rollback separate from palette review.

If the selected background policy is independently contentious, split that policy into its own small source commit before the palette commit. Otherwise, two focused commits are sufficient. No unrelated recovery changes should be included.
