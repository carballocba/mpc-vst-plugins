---
name: panel-skin
description: Draw a realistic MPC plugin skin (1280x628 TUI.json art) modelled on a real instrument's front panel, from reference photos, as procedural art with filmstrip knobs and on/off switch images. Use when a port needs a skin "like the real panel", when slicing a panel photo into tabs, knobs, buttons and LEDs, or when making skin assets (backgrounds, sprite strips) for any mpc-vst-plugins port.
---

# Realistic panel skins

A skin that looks like the instrument, built by a script in the port's repo (`skin/make_skin.py`). Every picture is generated, so
the skin can be rebuilt, restyled and checked. Worked examples: Profit-5 (`sd88me/mpc-vst-profit5`, procedural, Rev 2 / Rev 4 styles)
and Model-72 (`sd88me/mpc-vst-model72`, from GPL assets of the CA-72). Profit-08 shows the html-renderer route (`look=prophet`).

## 1. Read the panel (reference photos)
- Use photos of the user's own unit or ones they supply, for **reference only**: never commit a photo, a crop of one, a traced outline or
  a logo. The skin is drawn from scratch.
- List every control: what it is (knob, switch with LED, push button, display, wheel), its section and legend, and its printed scale.
  Note the materials: face colour, cheeks (wood, metal), knob family (knurled skirt, cap, pointer colour), LED colour, and the type style.
- Match each control to a parameter key in `params.json`. A control that doesn't map is drawn into the background as printed (program buttons, for example).

## 2. Fit the panel into 1280 x 628
- The MPC plugin area is 1280 x 628. Layout y = screen y + 86 (Force Shadow coordinates): `Y_OFF = 86`.
- A wide panel won't fit on one page at a usable size, so split it into **tabs by section**, in the panel's own reading order (Model-72: 5 tabs, Profit-5: 3).
  Repeat the shared material on every tab (face, cheeks) so the tabs read as one panel.
- **Sizes that work on the touchscreen:** knob radius 26-32 px (21 is the minimum, for small groups), switch caps 30-36 px, row pitch about 200 px, at least
  80 px between knob centres. A knob's touch box (`bw`) must not reach its neighbour's: `gen_vst.py` warns "TOUCH ... overlap".
- **Q-Link pages:** at most 16 controls each, grouped as the panel groups them (knobs on one page, the switches of a section on another).
  Check `docs/COMMUNITY_SKINS.md` for per-hardware advice (4 Q-Links on a Key 37).

## 3. Draw at 2x, procedurally
- Draw each tab at 2x (`SS = 2`) and downsample with LANCZOS: the silkscreen stays crisp.
- **Materials as functions:**
  - **face:** flat paint plus faint noise
  - **wood:** sine-streak grain plus noise
  - **knob:** skirt, knurl lines lit from the top left, a cap with a radial gradient, a pointer drawn pointing up
  - **switch:** a bevelled square, its LED with a glow when lit
  - **display:** seven-segment glyphs on dark glass
- **Silkscreen:**
  - section boxes with the title breaking the top line
  - legends with rules each side ("— SOURCE AMOUNT —")
  - scale numerals round each knob at the instrument's real travel (Profit-5: 0-10 over ±150°)
  - waveform symbols over the shape switches
- **Fonts:** use one with a licence that allows rendered output (GNU FreeSans, URW Gothic). Don't commit the font unless its licence allows it.
- **The name:** use the port's own wordmark, never the manufacturer's logo or model name (CLAUDE.md rules).

## 4. Controls: cut from their own tab
Draw the tab **without** its controls. Then, for each control, crop its square from the 2x background, composite the control into it,
and downsample. Every frame then sits exactly over its place, shadows and scale included.
- **Knob:** a filmstrip of 64 frames (minimum first) over the printed travel. A stepped param gets one frame per option.
  Layout line: `knob cx= cy= r= label="" key= strip=art/<key>.png frames=N ns=0 vs=14 bw=<touch>`.
- **Switch:** an off/on pair. Layout line: `toggle cx= cy= label="" key= img=art/<key>_off.png img_on=art/<key>_on.png w= h= ns=0`.
- **Background:** `art file=art/<tab>.png fit=stretch` per `[tab NAME]`, then `qlinks "PAGE" = key,...` lines.
- **Labels:** set `label=""`. The silkscreen already carries the name; MPC's own text would double it.
- **Same key twice** (a switch shown on two tabs): give the second its own art name (`<key>_<tab>`).
- **Contact shadows** go into the background, since they don't turn with the knob.

## 5. Styles
- **One generator with a style switch** (an env var or an argument) covers panel revisions: names (Profit-5's rev2: OSCILLATOR 1/2, FREQ 1), wood tone,
  and controls the old panel lacked (keep them, and say where on an info tab).
- **The style is the look only:** the sound's own revision switch stays a parameter.

## 6. Check before a device
- `python3 tools/gen_vst.py <port>/vst/vst.json` must build with no TOUCH / EDGE warnings (it needs Playwright 1.56 for `"art": "html"`).
- `tools/test_port.sh <port>/vst/vst.json` must pass with the real layout.
- Write previews (`vst/preview/<tab>.png`: the background with every control at its default) and look at each one: overlaps, legends cut off,
  a knob scale running into its neighbour.
- `tools/skin_check.py` on the built skin (TOUCH / EDGE / QLINK), and `skin_compat.py` for MPC OS 2.x / 3.x.
- **Size:** 64 frames of a 70 px knob is about 120 KB. A 3-tab skin with about 60 controls comes to about 11 MB of PNG (Profit-5).
  Keep strips under 12288 px tall (frames x height).
- Commit the generator, the generated `vst/art/` and `layout.conf`, and a preview per tab in `docs/img/`. Ignore `vst/preview/`.

## Slicing a photo instead (when the user insists on their own art)
The general workflow for any panel (a base panel stripped of controls, a blank display, tab backgrounds, and a sprite sheet of knobs,
buttons and LEDs in every state) maps onto the same layout lines:
- **base panel:** `art file=`
- **knob sprite:** a 64-frame `strip=`
- **button states:** `img=` / `img_on=`
- **display:** a `meter` filmstrip, or live text with `get=`

Only use art the user has the rights to. Never commit stock MPC skin art or a manufacturer's photos.
