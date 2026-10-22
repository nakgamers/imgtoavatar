# Example: Rayneshia

A complete worked example of the imgtoavatar workflow.

**Input:** one illustration — Rayneshia, an original character from the
music video *King Of Curse* (ash-blonde bob, red eyes, glasses, navy jacket
with a red-lined hood, ring of fire).

The starting point — the still illustration, exactly as drawn:

![Rayneshia, original still illustration](source.png)

**Output:** a mesh-rigged avatar running a seamless idle loop as a live
lockscreen — blinking, breathing, hair swaying, procedural embers drifting
at the edges of the frame.

![Rayneshia rigged — idle loop preview](preview.gif)

## Files

| File | What it is |
|---|---|
| `source.png` | The original illustration (the only input) |
| `rig.json` | The finished mesh rig produced from the validated draft |
| `avatar.json` | Avatar descriptor used by the studio runtime |
| `rayneshia-live-lockscreen.mp4` | The rendered idle loop (live lockscreen) |
| `preview.gif` | Short preview of the loop |

## Notes from the build

- Eye regions were kept deliberately tight — the single biggest factor in
  a clean blink.
- Hair movement comes from strand physics: side strands and bangs got a
  larger `max` (tip reach), which is what makes the sway visible.
- Ember particles are procedural and were kept off the face.
- The loop seam was checked frame-to-frame; the lockscreen prompt of the
  desktop sits at a fixed center position, so the composition keeps the
  face clear of it.

## Terms

Artwork © Tinandar Hermawan. Published as a worked example — not covered
by this repository's MIT License; please don't redistribute or use the
character commercially without permission. The rig data in this folder was
produced with mesh-avatar-studio (MIT, by Yuki Shindo / shinshin86).
