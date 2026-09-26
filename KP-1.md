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

## Hand motion (branch `kp-1/motion`)

The webcam acts as a controller, and a video file (or the camera itself) fills the cube.
Wherever the camera sees motion, that part of the picture is pressed into the past, and it
relaxes back the same way a pointer press does. Toggle with the **hand motion** key or `h`.
Pointer presses still work alongside it.

```
camera → 64×64 luma, mirrored, cover-cropped to cube aspect → |Δ| vs previous frame
       → mean per 4×4 block = one map cell → over the sens floor → gaussian splat (radius)
       → dmap = max(dmap, depth · level)
```

- **It uses frame differencing, not hand tracking.** Hand landmarks (e.g. MediaPipe) would
  be a new dependency, which the ground rules say to ask about first. Differencing reacts
  to any motion, not only hands, so it works best against a still background.
- **The camera is mirrored,** so a hand on the right presses the right of the picture, as
  in a mirror.
- **It runs on the CPU, at camera rate.** That's 64×64 samples and at most 256×256 splat
  terms, well under a millisecond. It feeds the same 16×16 map the state layer will
  replace.
- **The motion camera has its own stream,** separate from the picture source, so either
  one can stop without killing the other.
- **Known sensitivities:** auto-exposure or white-balance shifts register as whole-frame
  motion. Raise the `sens` floor (turn the knob down) if it presses when you're still.
- **Depth in seconds depends on the picture source.** The cube counts source frames, so a
  60 fps video file holds 2.1 s where a 30 fps camera holds 4.3 s.

Verified in the same headless Chrome/Metal setup. The fake camera played the sweeping bar
as the "hand", and the picture was a video file. The map's peak tracked the bar's position
to within one cell (≤ 0.07 of frame width) across the full sweep. A test-pattern clip
visibly bent into the past along the motion trail. Display ran at 60 fps with the motion
driver on. Not verified: a real hand in front of a real webcam.

## Grain (branch `kp-1/grain`)

This is CT–1's grain formula moved into the display shader. It's a per-pixel hash,
re-jittered every frame and weighted toward the midtones the way film grain is. There's
no extra pass: it's applied after the cube sample and the map overlay, and it stays out
of the letterbox.

- **`grain`** (0–0.5, default 0.12) is its strength.
- **`size`** (1–4 screen points, default 1.5) is the size of one grain, and scales with
  `devicePixelRatio`. That keeps retina grain from shrinking to invisible single-device-
  pixel noise.

Verified at 2× DPR: the grain is visible at the default and the frame is clean at 0.
Display holds 60 fps and there are no GL errors.

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
