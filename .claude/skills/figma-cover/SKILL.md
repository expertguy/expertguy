---
name: figma-cover
description: Build a professional, readable cover frame (1920×1080) for a Figma design file with the Figma MCP use_figma tool. Use whenever the user asks for a Figma cover, file cover, thumbnail cover, title page or "cover page" for a design file, or asks to fix/improve an existing Figma cover.
---

# Figma cover

A cover is the first page of a design file. It must be readable at thumbnail size, look
intentional (no dead space) and reuse the file's own tokens. Follow this skill exactly.
Do not ship a cover until every item in the checklist at the bottom passes.

## 0. Before you build

1. Load the `figma-use` skill (mandatory before any `use_figma` call).
2. Read the file first with one read-only `use_figma` call. Collect:
   - variable collections and exact variable names (primitives vs semantic),
   - text styles and the font families actually available (`figma.listAvailableFontsAsync()`),
   - the main deliverable frame(s) you can preview (e.g. the primary screen).
3. Page order: the cover lives on the **first page**, named `Cover`, with one frame named
   `Cover · 1920×1080` at (0,0). Figma uses the first frame of the first page as the thumbnail
   (`setFileThumbnailNodeAsync` is not supported through MCP — do not call it).

## 1. Canvas and grid

- Frame: **1920 × 1080**, `clipsContent = true`, dark brand background (or a strong brand colour).
- Side margins: **128px** left and right. Top bar at y = 64. Footer rule at y = H − 152.
- Two zones:
  - **Left content column** (x = 128, width ≈ 800): all text lives here.
  - **Right visual zone** (≈ 760–900px wide): a real preview of the work
    (a scaled clone of the key screen) or brand geometry. Never leave it empty and never
    leave the left zone half empty.
- Nothing may overlap the footer. Visuals must end at least 48px above the footer rule.

## 2. Content blocks (top → bottom)

| Block | Content | Size / weight |
|---|---|---|
| Top bar | Brand wordmark · divider · product/area name | 28px ExtraBold, letter-spacing 6% · 24px SemiBold |
| Status pill | "Ready for review · v1.0" (or current status) | 20px Bold on accent-filled pill |
| Title | 2–3 words, can break onto 2 lines | **112–128px** ExtraBold, line-height 100%, letter-spacing −2% |
| Subtitle | One sentence: what the file contains | **28–32px** Regular, line-height 140%, max width 740 |
| Summary tiles | 2–4 tiles (segments, platforms, flows) with accent bar, name, count | name 28px Bold · meta 18px |
| Footer meta | 3–4 label/value pairs (Platform, Design system, Version, Date) | label 16px Bold UPPER 8% · value 22px SemiBold |

Minimum text size anywhere on the cover: **16px**. Body/meta values: **≥ 22px**.
Vertical rhythm between blocks: 40 / 32 / 48px (use spacer rectangles or auto-layout gaps).

## 3. Colour rules (the mistake to never repeat)

- Every text fill must be **bound to a variable that actually exists**. Guard it:
  ```js
  const fill = (n) => { if (!V[n]) throw new Error('Missing variable ' + n);
    return figma.variables.setBoundVariableForPaint({type:'SOLID',color:{r:0,g:0,b:0}}, 'color', V[n]); };
  ```
  Binding to an undefined variable silently leaves the placeholder colour (black) — this made
  a title and chip labels black-on-navy and unreadable. The guard makes that impossible.
- Check which collection holds the colour: primitives are often prefixed (e.g. `p:white`),
  semantic tokens are not. Never guess a name — read the map first.
- On a dark background use only: white, the lightest neutral tints (10–20%) for secondary
  text, and one brand accent (e.g. yellow) for eyebrow/status. Mid-grey (40%) only for small
  UPPERCASE labels. Never dark text on the dark background.
- Contrast: body/meta ≥ 4.5:1, large title ≥ 3:1 against the background.

## 4. Visual zone recipe (preview of the work)

```js
const pv = figma.createFrame(); pv.name = 'Preview · <screen>';
pv.resize(760, 712); pv.cornerRadius = 12; pv.clipsContent = true;
pv.effects = [{type:'DROP_SHADOW',color:{r:0,g:0,b:0,a:.35},offset:{x:0,y:24},radius:64,spread:0,visible:true,blendMode:'NORMAL'}];
const shot = keyScreen.clone(); pv.appendChild(shot); shot.rescale(760 / keyScreen.width); shot.x = 0; shot.y = 0;
cover.appendChild(pv); pv.x = 1920 - 760 - 96; pv.y = 160;   // ends at 872, footer rule at 928
```
- Clone the screen *before* switching to the cover page (fetch it by id first).
- Background geometry (circles, rings) sits behind everything, low opacity (≤ 30% for
  outlines), `locked = true`, and must not touch text or the footer.

## 5. Build order in one `use_figma` call

1. Fetch preview source node → switch to the Cover page → load fonts (every style you use,
   including ExtraBold) → build the variable map `V`.
2. Remove the old cover frame if you are rebuilding.
3. Frame → background geometry → preview → brand band → top bar → content column
   (status, title, subtitle, tiles) → footer rule → footer meta.
4. Run the self-audit below in the same script and return the results plus a screenshot
   (`await cover.screenshot({scale:0.5})`).

## 6. Self-audit (put this in the script)

```js
const texts = cover.findAll(n => n.type === 'TEXT');
const unbound = texts.filter(t => !t.fills[0] || !t.fills[0].boundVariables).map(t => t.name);
const tooSmall = texts.filter(t => typeof t.fontSize === 'number' && t.fontSize < 16).map(t => t.name);
const rule = cover.findChild(n => n.name === 'Footer rule');
const overlapsFooter = cover.children.filter(n => !['Footer rule','Footer meta','Background geometry'].includes(n.name) && n.y + n.height > rule.y - 48).map(n => n.name);
return { unbound, tooSmall, overlapsFooter };   // all three must be empty
```

## 7. Checklist before you report the cover as done

- [ ] Title reads clearly at 25% zoom; no text under 16px; meta values ≥ 22px.
- [ ] No black or dark text on the dark background (audit `unbound` is empty).
- [ ] Left column is filled top to bottom (status → title → subtitle → tiles), no big hole.
- [ ] Right zone shows the actual work (scaled preview) or deliberate brand geometry.
- [ ] Nothing overlaps the footer rule; decorative shapes don't touch text.
- [ ] Layers are named (`Top bar`, `Title`, `Segments`, `Footer meta`, `Preview · …`).
- [ ] Cover is the first frame on the first page.
- [ ] No confidential client names in anything outside the client's own file.
- [ ] Screenshot reviewed after the last change — describe what you checked.
