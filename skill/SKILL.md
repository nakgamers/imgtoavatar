# imgtoavatar — turn one illustration into a living 2D mesh avatar

An agent skill for rigging a single still illustration into an animated 2D
avatar — blinking, breathing, hair swaying, head turning, talking — using
[mesh-avatar-studio](https://github.com/shinshin86/mesh-avatar-studio)
(MIT, by Yuki Shindo / shinshin86).

Use this skill when the user asks to: "rig this illustration", "animate my
character", "make an avatar that blinks / breathes / talks", "img to avatar",
"build a live lockscreen / live wallpaper from artwork", "make a VTuber-style
2D avatar from one image", or shows a character illustration and wants it to
move.

## Ground rules

- The upstream repo is the engine. **Read its `docs/agent-guide.md` end to
  end, in order, before touching a rig** — it is the canonical workflow
  (coordinate calibration, zoomed grids, overlays, pose review). Field
  meanings live in `docs/rig-fields.md`. This skill adds sequencing,
  quality gates, and hard-won lessons on top; it does not replace the guide.
- The user's artwork belongs to the user. Never send their image to an
  external service without asking. Keep all derived files inside the studio
  repo's `projects/` while working.
- Visually inspect every render before reporting success, and report
  remaining defects honestly (torn mesh, dead eyes, particle covering the
  face) instead of declaring victory from exit codes.

## Prerequisites

- Node.js 22.17+ and Python 3.10+ (with `uv`)
- mesh-avatar-studio cloned and installed:

```sh
git clone https://github.com/shinshin86/mesh-avatar-studio
cd mesh-avatar-studio
npm install
```

## First decision: isolated character or full scene?

The upstream engine assumes the easy case: a character illustration
("IP") with **no background** — transparent or plain — which can be rigged
directly. Real commissions are often **full scenes**: the character is
painted into a background with effects in front of and behind them.

For a full scene, **decompose before you rig.** A flat scene can only be
warped as one sheet — the background tears, foreground effects move with
the body, and the result looks cheap. Split into layers, rig each layer
with its own motion, and composite them back: the result is visibly
better, and it is the only way to keep a painted background alive.

### Decomposition method

1. **Cut along occlusion boundaries, back to front.** Produce full-canvas
   transparent PNG layers (same canvas size as the source — never crop),
   ordered exactly as they stack. A typical fine split:
   background → scenic elements → back effects → hair-back → body → face →
   hair-front → eyes → glasses → mouth → front effects.
   (The Rayneshia example used 11 working layers.)
2. **Make a clean background plate.** The background layer must be the
   scene *with the character removed* — inpaint / blur-fill the hole.
   Nobody sees it directly, but any gap in the upper layers will.
3. **Group into composite plates for rendering.** Merge the fine split
   into a few plates with shared motion — e.g. **background / ring /
   character / fire** (4 plates in the example). Character-internal layers
   (hair, eyes, glasses, mouth) stay separate inside the character plate
   so the mesh rig can drive them.
4. **Prove the split before animating.** Re-stack the plates and diff
   against the original. The example recomposite differs by ≈ 0.2/255 —
   if your stack does not rebuild the original almost exactly, the split
   is wrong; fix it now, not after rigging.
5. **Rig per layer, composite in stack order.** The character gets the
   full mesh treatment (blink, breath, hair sway); scenic plates get
   their own subtle profiles (slow pulse, flicker, micro-parallax);
   foreground effects (flames, particles) stay on top, exactly where the
   painter put them.

## Workflow

1. **New project.** Create `projects/<name>/` with the source illustration
   as `source.png` (helper: `tools/new-project.py`). One character, front
   facing, clean background works best. **Full scene? Run the
   decomposition above first** — rig the character layers, keep the other
   plates for compositing.
2. **Draft the rig.** Author `projects/<name>/rig.draft.json` following the
   agent guide: calibrate coordinates on zoomed grids (`tools/grid.py`),
   define parts (eyes, mouth, hair strands, body), and check placements with
   overlays (`tools/overlay.py`).
3. **Validate the draft.**
   `npm run validate-rig -- projects/<name>/rig.draft.json --draft`
   Fix every error and re-run until it passes. Do not build on an invalid
   rig.
4. **Build layers.** `tools/build-layers.py` turns the draft into the final
   `rig.json` + layer images.
5. **Pose review (mandatory).** `npm run render-poses -- projects/<name>`
   produces a contact sheet: blink, mouth open, head turn / look / tilt,
   body tilt, hair sway. Open it and look. No torn mesh, no sliding facial
   features, eyes must read as *closed* in the blink frame.
6. **Animate.**
   - Idle loop (lockscreen / wallpaper / ambient): `tools/render-idle.mjs`
     — blink + breathing + sway timeline, seamless loop.
   - Talking / presenter (voice-over videos): `tools/render-talk.mjs`.
   - Web runtime package: `npm run pack-avatar`.
7. **Optional garnish.** Procedural ember/particle overlays via
   `tools/embers.py`, composited with ffmpeg. Keep particles off the face.
8. **Human fine-tune.** Subtle proportion/weight fixes are fastest in the
   studio's local editor (`npm run dev`) by the owner; hand over the project
   folder with the draft rig intact so edits stay non-destructive.

## Hard lessons (learned the expensive way)

- **Eye ROI must be tight.** A loose eye region is the #1 cause of broken
  blinks. Zoom in, hug the eyelid line, re-check with the overlay.
- **Hair sway is governed by each strand's `max`** (tip reach ≈ 0.9 × max),
  helped by the sway gain — if the hair looks glued, raise `max` on side
  strands and bangs before touching anything else.
- **Loop seam check:** compare frame 1 and the last frame of any idle loop;
  a visible jump ruins a lockscreen.
- **Effects never cover the face.** Particles, glows, and vignettes stay at
  the edges; the face is the product.
- **Lockscreen framing:** desktop lock prompts (e.g. GNOME) sit hardcoded
  at center / lower-center and cannot be moved — compose the character so
  the face clears that zone.
- **A passing validator is not a passing avatar.** Only the pose contact
  sheet and the rendered loop count as proof.
- **Recomposite before you animate.** In the decomposition path, a stack
  that does not diff-clean against the original means torn seams and
  halos later. Cut edges want a 1–2 px feather; check boundaries at 200%
  zoom, especially hair against bright backgrounds.

## Deliverables

- `rig.json` + `avatar.json` (the portable avatar data)
- Pose contact sheet (proof of deformation quality)
- Rendered idle loop (MP4) and/or talk render, packed avatar for web
- A short note of any remaining defects
