# KP–1

The recent past held as a cube, `N` frames deep. Every pixel on the display reads the
cube some number of frames behind the present. Pressing on the display pushes that
region into the past, and it relaxes back.

Single file: `index.html`. No dependencies, no build. It needs a secure context for the
camera, so serve it from `localhost` (e.g. `python3 -m http.server`) or open it in Chrome
from `file://`. You can also drop a video file instead.

The KP–1 brief was not available when milestone 1 was built. This work follows the build
prompt and its corrections only.

## Milestone 1 — cube (branch `kp-1/cube`)

```
source frame → srcTex (2D) → cover-crop pass → layer `head` of cube (3D, RGBA8)
display      → texture(cube, vec3(uv, (head − d(uv) + ½) / N)), d from a 16×16 map
```

### Decisions

- **The ring is written on the GPU.** Each new source frame is uploaded to a 2D texture
  and drawn into one layer of the 3D texture via `framebufferTextureLayer`. That pass also
  handles the crop and the mirror, and there's no CPU readback or resample.
- **The ring seam is handled by `TEXTURE_WRAP_R = REPEAT`.** The present sits at layer
  `head`, and `d` frames back is `head − d` mod `N`. With REPEAT, linear filtering across
  layer `N−1 → 0` blends two frames that really are adjacent in time. The only true seam
  is between the newest and oldest frames, and `d ≤ N−1` never crosses it.
- **The time axis counts captured frames, not display frames.** A frame is pushed only
  when the source delivers one (`requestVideoFrameCallback`, with a `currentTime` poll as
  fallback). At 30 fps capture, `N = 128` holds about 4.3 s.
- **Until the ring fills, `d` is clamped to the frames held so far.** Otherwise early
  presses would read uninitialised layers.
- **Allocation is proved, not assumed.** `texStorage3D` is followed by a `getError` check,
  a framebuffer-completeness check, and a clear-and-read-back of the deepest layer. If any
  of these fail, resolution steps down (320×180 → 240×135 → 160×90) and `N` stays at 128.
  The readout says which size you got and why.
- **The map is 16×16 so it matches the state that will replace it.** Milestone 2 has
  n = 8 qubits, which gives 2⁸ cells. The interleaved index `x3 y3 x2 y2 x1 y1 x0 y0`
  decodes to a 16×16 grid, so the state layer only has to change where the numbers come
  from. The map is an R16F texture with bilinear filtering. Map row 0 is the bottom (GL
  convention).
- **The classical map works like this.** Each frame, cells decay by `exp(−dt/τ)`. While
  the pointer is down, each cell is set to `max(cell, depth · gaussian)`. That makes a
  press predictable: it sits at `depth` frames under the pointer, then relaxes to the
  present.
- **Contort's zoom/pan is not carried over.** In this instrument the pointer paints the
  surface instead.

### Known artefacts

- **The bilinear 16×16 map puts kinks at cell boundaries.** Contours of equal delay come
  out polygonal. Milestone 2 decides how state cells should interpolate.
- **Linear filtering on `t` cross-fades between adjacent frames.** A fast-moving edge
  therefore shows a sawtooth along its delay contour, one tooth per captured frame. The
  prompt calls for linear-on-t, and this is what it looks like.

### Open question for milestone 2

Correction 1 gives `d = clamp(gain · 2ⁿ · |a_k|², 0, N−1) − gain`. That makes the uniform
state flat (d = 0), but any cell below uniform (2ⁿ|a_k|² < 1) comes out negative, which is
the future. Normalisation guarantees such cells exist whenever any cell is above uniform.
The cube shader already clamps `d` to `[0, fill−1]`, so negative values currently read as
the present. The decision needed: is that clamp intended, or should it be
`clamp(gain · (2ⁿ|a_k|² − 1), 0, N−1)`?

## Verified (milestone 1)

Tested in Chrome 153 headless with ANGLE Metal on an Apple M5 Max. The camera was
Chrome's fake capture device, fed a synthetic 640×360 30 fps video of a bright bar
sweeping over a checker field.

- **The cube allocated at target size:** 320×180×128 RGBA8, 29.5 MB, deepest layer read
  back. `MAX_3D_TEXTURE_SIZE` is 2048.
- **Frame rate:** 60 fps display and 30 fps capture, at both 1400×900 @1× and
  3142×2162 @2× canvas, while pressing. Headless rAF caps at 60.
- **The shear shows on screen.** Dragging across the frame bends the bar into the past
  under the press (up to 91 frames, about 3 s) and leaves a delay contour. The surface
  relaxes on the expected curve (91 → 34 after 2.5 s, → 1.3 after 10.5 s at τ = 2.5 s) and
  returns flat to the present.
- **Oversize allocations fail cleanly:** they are refused with a reason and no GL error.

**Not verified:** a real webcam and a real hand, and frame rate on other GPUs or in
Safari/Firefox.
