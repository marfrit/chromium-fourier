# Dmabuf zero-copy on Rockchip — what `external_only` means and why
# every Linux video stack hits the same wall

When a V4L2-decoded NV12 frame from the Rockchip `hantro` (or `rkvdec2`)
driver should reach the screen without a CPU detour, it usually
doesn't. mpv `--vo=gpu` legacy, gstreamer `glimagesink`, chromium's
NV12 native-pixmap pipeline, and pretty much every other GLES-based
consumer in the Linux media graph all stumble on the same fact, which
is rarely written down anywhere visible:

> **All NV12 dmabuf modifiers exposed by mesa `panfrost` and
> `panthor` are `external_only`.**

This page documents the wall, the empirical evidence on PineTab2
(RK3566 / Mali-G52 / panfrost) and CoolPi 4 (RK3588 / Mali-G610 /
panthor), and the upstream-targetable patches that follow from it.

## What `external_only` is

`EGL_EXT_image_dma_buf_import_modifiers` lets a GLES client ask the
driver "for fourcc X, what dmabuf modifiers can you import?" The
return value per modifier carries a flag: **external_only**.

- **external_only = false**: the resulting EGLImage can be bound to a
  regular `GL_TEXTURE_2D` and sampled with a normal `sampler2D` in
  GLES shaders. Drop-in compatible with most existing engines.
- **external_only = true**: the EGLImage can ONLY be bound to
  `GL_TEXTURE_EXTERNAL_OES`, requires `samplerExternalOES` in shaders
  (which means a `#extension GL_OES_EGL_image_external_essl3 :
  require` line and the format-conversion-aware sampler), and various
  GLES operations on the texture (mipmaps, FBO render, `glCopyTexImage`
  etc.) are unavailable.

`external_only` exists because GLES specifies single-plane
`GL_TEXTURE_2D` format conversion semantics. NV12 is multi-plane (Y
plane + interleaved CbCr plane) — sampling it correctly requires the
external-image extension's conversion logic. The mesa drivers
correctly mark NV12 imports as `external_only` because that's what
GLES says they have to be.

## What the boards actually report

Probed via `tools/dmabuf-modifiers.c` (build with
`gcc -O2 -o dmabuf-modifiers dmabuf-modifiers.c -lEGL -lgbm`):

```
== ohm (PineTab2 / RK3566 / Mali-G52 / panfrost / mesa 26.0.5) ==
NV12 (0x3231564e) — 4 modifiers:
    0x0800000000000341 (external_only)   # ARM AFBC, 16x16 split, YUV
    0x0800000000000041 (external_only)   # ARM AFBC, 16x16 split
    0x0000000000000000 (external_only)   # DRM_FORMAT_MOD_LINEAR
    0x0b00000000000001 (external_only)   # ARM AFRC

== ampere (CoolPi 4 / RK3588 / Mali-G610 / panthor / mesa 26.0.5) ==
NV12 (0x3231564e) — 4 modifiers:
    0x0800000000000341 (external_only)
    0x0800000000000041 (external_only)
    0x0000000000000000 (external_only)
    0x0b00000000000001 (external_only)
```

Every NV12 modifier is `external_only`, on both panfrost and panthor,
on the same mesa version. **There is no NV12 → `GL_TEXTURE_2D` import
path on Rockchip-Mali GLES today.** Same answer on Pinebook Pro RK3399
/ panfrost is expected (untested in this snapshot — please file an
issue with the dmabuf-modifiers output if you have one).

## Why this kills zero-copy in chromium specifically

Chromium's `ui/ozone/common/native_pixmap_egl_binding.cc` is the
Linux/Wayland import path for NV12 dmabufs from `media/gpu/v4l2/`. The
binding itself takes a `GLenum target` parameter — so the choice of
`GL_TEXTURE_2D` vs `GL_TEXTURE_EXTERNAL_OES` is made by the *caller*,
in the SharedImage / GPU service layer. The current callers hardcode
`GL_TEXTURE_2D` for NV12; the modifier table is never consulted.

When the underlying mesa driver only advertises `external_only` NV12
modifiers (panfrost / panthor / et al), the eglImage creation
silently produces an EGLImage that can't be sampled correctly via
`sampler2D`, the video pipeline detects the mismatch in subsequent
SharedImage code, and the frame ends up going through the
`media/gpu/chromeos/video_decoder_pipeline.cc` NV12-to-AR24 VPP path.
That's the ~40 % CPU cost we see in the chromium-fourier playback
measurement.

The chromium side fix is non-trivial: query
`eglQueryDmaBufModifiersEXT` at the import site, pick
`GL_TEXTURE_EXTERNAL_OES` whenever the matching modifier is
`external_only`, propagate the choice into the SharedImage
representation, and either:

1. Adopt `samplerExternalOES` in the chromium video compositor's
   sampler shader (touches `cc/output/` shader infrastructure), or
2. Convert via a one-shot internal `EXTERNAL_OES → 2D` blit on the
   GPU (cheaper than NV12-to-AR24 VPP because the sampler does the
   YUV→RGB internally; one-pass instead of two-pass).

Chromium already handles `GL_TEXTURE_EXTERNAL_OES` for video on
Android (where every camera and decoder dmabuf is external_only), so
the patches mostly retarget existing code. But it's a real upstream
patch, not a one-line gn flag flip.

## Why this kills zero-copy in mpv `--vo=gpu` legacy

`vo_gpu` (the legacy renderer) imports the dmabuf via
`hwdec_drmprime_overlay` and binds to `GL_TEXTURE_2D`. When the
modifier is `external_only`, the bind silently gives a black /
mis-tiled output, and most users end up either dropping back to
`--hwdec=no` (full software) or switching to `--vo=drm` (KMS direct
scanout, no GL).

`vo_gpu_next` (libplacebo) handles `external_only` correctly — it has
been written from the start to support both target types. That is why
the [playback HOWTO](playback-howto.md) recommends `--vo=gpu-next`.

## Why this kills zero-copy in gstreamer

`glimagesink` and `gtkglsink` use `GL_TEXTURE_2D` by default. Set
`GST_GL_TEXTURE_TARGET=external-oes` in the environment and bind the
caps `texture-target=external-oes` on the source pad to switch them
to `samplerExternalOES`. Most gst pipelines don't, and silently
fall back to `glupload`-via-CPU.

## The Vulkan escape hatch

`VK_EXT_image_drm_format_modifier` does **not** have an `external_only`
concept. Vulkan's `VkSamplerYcbcrConversion` is the spec-defined way
to sample multi-plane formats, and mesa source includes the
`panvk` driver targeting Mali-Valhall (RK3588 G610+). On a system
where panvk is actually installed and exposes Vulkan 1.3+,
ANGLE-on-Vulkan plus Vulkan-aware video pipelines (libplacebo,
vkd3d-video) zero-copy NV12 with no `external_only` gymnastics.

In practice the escape hatch is currently closed by **packaging**
rather than capability:

- **Arch Linux ARM** (CoolPi 4 / Rock 5 / OrangePi 5) — the stock
  `mesa` package is built without `-Dvulkan-drivers=panfrost`. No
  panvk ICD ships, no `/usr/share/vulkan/icd.d/panvk_*.json` is
  installed, and ANGLE-on-Vulkan returns `VK_ERROR_INCOMPATIBLE_DRIVER`
  straight from the loader. Local fix: rebuild mesa with the
  vulkan-drivers meson arg, or AUR a `vulkan-panfrost` package.
- **Debian / Ubuntu 24.04+** — `mesa-vulkan-drivers` includes panvk
  for RK3588 since mesa 24.x. ANGLE-on-Vulkan should work out of
  the box.
- **Mali-G52 r1 / Bifrost-gen2 / RK3566** — even with panvk
  installed, the driver returns `VK_ERROR_INCOMPATIBLE_DRIVER` on
  probe because Bifrost-gen2 is below the Vulkan support floor
  panvk currently advertises. The GLES + `external_only` workaround
  is all we have on that SoC today.
- **Mali-T860 / Midgard / RK3399** — no Vulkan support in mesa at
  all (panvk targets Bifrost+ and Valhall only). GLES-only forever.

The chromium-fourier `rk3588/` launcher leaves Vulkan enabled
optimistically — once panvk is installed, it lights up without
re-packaging the binary.

## Upstream-targetable next moves

This finding is broader than chromium-fourier; bullet list of what
would help the most consumers:

1. **chromium**: teach `NativePixmapEGLBinding` to honor
   `external_only`, retarget to `GL_TEXTURE_EXTERNAL_OES` when
   advertised, route through the existing Android-side compositor
   shader path. (Big patch; would unblock zero-copy on every
   Mali-on-Linux setup.)
2. **mpv `--vo=gpu` legacy**: make `hwdec_drmprime` honor the
   `external_only` modifier flag and bind `GL_TEXTURE_EXTERNAL_OES`
   accordingly. (Smaller patch; mostly already done in `--vo=gpu-next`.)
3. **mesa**: nothing to fix — current behavior is spec-correct. The
   only question is whether the modifier list could expose AFBC YUV
   variants that decompose to single-plane Y + UV pairs, which mesa
   could then mark non-external. That is a longer-term mesa-internal
   investigation.
4. **kernel V4L2**: confirm what modifier `hantro` actually emits on
   its capture queue. Probably `DRM_FORMAT_MOD_LINEAR` on RK3566,
   possibly AFBC on RK3588 hardware variants. Document that in the
   driver's KConfig help, since today nobody knows without reading
   `drivers/staging/media/hantro/`.

The chromium-fourier project's contribution is the documentation
above (this file) and the probe tool (`tools/dmabuf-modifiers.c`).
The chromium upstream patch is on the followup list.

## Status update (2026-04-28) — patch 3/3 landed and validated

Item #1 of the "Upstream-targetable next moves" list is now done in
this repo as
`{pinetab2,rk3399,rk3588}/patches/nv12-external-oes-on-modifier-external-only.patch`.

The fix is small and surgical. `OzoneImageGLTexturesHolder::GetBinding`
already had a branch that picks `GL_TEXTURE_EXTERNAL_OES` when the
SharedImageFormat carries `PrefersExternalSampler` — but that flag is
only set for the generic Linux multi-plane case, not for NV12 dmabufs
arriving from V4L2 producers via the standard ozone pixmap path. The
patch adds a second predicate: also pick `EXTERNAL_OES` when the EGL
driver advertises the pixmap's actual modifier as `external_only` for
that fourcc. A new helper
`NativePixmapEGLBinding::ModifierRequiresExternalOES` queries
`eglQueryDmaBufModifiersEXT` and caches the answer per
`(fourcc, modifier)` tuple — the EGL round-trip happens once per
unique format+modifier combination in the GPU process lifetime, not
once per frame. Skia Ganesh handles `GL_TEXTURE_EXTERNAL_OES`
natively via `GrGLTextureInfo.fTarget`, so no shader changes are
required. ~+90 lines, zero deletions.

### Validation: ohm (PineTab2 / RK3566 / hantro mainline 6.19.10)

`bbb_1080p30_h264.mp4` played through `chromium-fourier-149-r2` (with
patch 3/3) on Wayland, `--use-gl=angle --use-angle=gles
--enable-features=AcceleratedVideoDecoder`. Visual integrity was
clean — no garble, no color regression, no blank frames.

```
=== chrome process CPU during steady-state 1080p30 H.264 playback ===
PID  PCPU  COMMAND
4951  11.9  chrome (browser)
5005   9.0  chrome --type=gpu-process
5017   5.8  chrome --type=utility (NetworkService)
5087   5.5  chrome --type=renderer (the bbb tab)
5204   0.9  chrome --type=utility (AudioService)

chrome combined: 34.7%   (vs. pre-patch baseline ~131% — ~3.8× reduction)
```

Decoder log confirms the V4L2 stateless path stayed engaged:
```
V4L2VideoDecoder()
InitializeBackend(): Using a stateless API for profile: h264 main and fourcc: S264
SetupInputFormat(): Input (OUTPUT queue) Fourcc: S264
AllocateInputBuffers(): Requesting: 17 OUTPUT buffers of type V4L2_MEMORY_MMAP
SetupOutputFormat(): Output (CAPTURE queue) candidate: NV12
ContinueChangeResolution(): Requesting: 6 CAPTURE buffers of type V4L2_MEMORY_MMAP
```

The GPU process held 19 live dmabuf fds during steady playback (V4L2
capture rotation + compositor pipeline depth) — bounded, not leaking.

### Caveat — KWin 6.6.4 GLES backend on this hardware

Both pre-patch and post-patch builds stall after a few seconds of
playback under a KWin Wayland session on this box. Symptom: renderer
+ GPU processes both park in `futex_do_wait`, the `<video>` element
keeps its ⏸ icon, currentTime advances on the audio clock, and audio
outputs static (last ALSA buffer recycled) then silence. No D-state,
no `vb2`/`v4l2`/`dma_fence` wchan, no error in chrome's log.

The journal pinpoints it:
```
kwin_wayland: GL_INVALID_VALUE in glTexImage2D(internalFormat=GL_ALPHA)
kwin_wayland: GL_INVALID_OPERATION in glTexSubImage2D(invalid texture level 0) × N
```
First occurrence on this box: **2026-03-06** — entirely preexisting,
unrelated to chromium-fourier. KWin asks for an internal format that
doesn't exist in modern GLES (`GL_ALPHA` is GLES1.x legacy, not valid
for `glTexImage2D` with GLES3 contexts), the allocation fails, every
subsequent `glTexSubImage2D` errors at level 0, and KWin keeps
retrying the same broken upload every frame, never recovering. The
frame-callback ack to wayland clients stalls → chrome's renderer
parks waiting for present-feedback that never lands.

Triangulation: VLC on the same compositor fails with `cannot convert
decoder/filter output to any format supported by the output` followed
by `could not initialize video chain`. mpv `--vo=null
--hwdec=v4l2request` returns `Could not create device.` ffmpeg
`-hwaccel v4l2request -f null` plays through clean. The decode path
is healthy on this box; the wall is the compositor's GL backend.

The chromium-fourier patch series is the right thing on the right
hardware; the residual stall is a KWin bug that this campaign cannot
fix from inside chromium. A pivot to identify and patch the offending
`glTexImage2D(GL_ALPHA)` site in KWin is on the followup list.

### Update 2026-04-28 — GL_ALPHA was Qt 6, not KWin; the stall remains

Source-grep traced the `glTexImage2D(internalFormat=GL_ALPHA)` calls
to **Qt 6** rather than KWin. Three sites in qtbase 6.11.0 hard-code
`GL_ALPHA` whenever `QT_CONFIG(opengles2)` is built in (every aarch64
Linux distro), with no runtime check for the actual context's ES
version:

- `src/opengl/qopengltextureglyphcache.cpp:111-117` — text glyph
  cache, the primary KDE-decoration trigger.
- `src/gui/rhi/qrhigles2.cpp:1373-1378` — Qt-Quick-RHI's
  `RED_OR_ALPHA8` path.
- `src/opengl/qopengltextureuploader.cpp:253-275` — `Format_Alpha8`
  and `Format_Grayscale8` cases short-circuit on `isOpenGLES()`
  before reaching the existing TextureSwizzle fallback.

A patch series correcting all three (`qt6-base-fourier`, three small
runtime-checks gated on `caps.gles && caps.ctxMajor >= 3` or
`format().majorVersion() >= 3`) was built and installed on ohm. Fresh
session, post-relogin, idle: zero `GL_INVALID_VALUE` in the journal.

**The chrome stall, however, persisted** — at ~6 s instead of ~3 s,
so the GL_ALPHA churn was contributing some load but wasn't the
primary cause.

A definitive A/B against weston closed the loop:

| Path | Result |
|---|---|
| `ffmpeg -hwaccel v4l2request -f null` | ✓ 36 fps, clean |
| `mpv --vo=null --hwdec=v4l2request` | ✓ decode-only, clean |
| `mpv --vo=drm --hwdec=v4l2request` (KMS scanout, no compositor) | ✓ 0.7 % drops in 19 s |
| chrome v4 under **weston** | ✓ plays through, ~96 % CPU |
| chrome v4 under KWin (post-Qt-fix) | ✗ stall @ ~6 s |
| `mpv --vo=gpu-next --hwdec=v4l2request` under KWin | ✗ 76 % drops, slideshow |

Same chrome v4 binary, same panfrost mesa, same V4L2 driver, same
hardware — only swapping KWin → weston turns the stall off. The
Wayland *protocol* is fine; KWin's *implementation* of it on
panfrost ES 3.2 + fast video clients has a second, structurally
deeper bug that the GL_ALPHA noise was masking.

The chromium-fourier patch series stands on its own merits — under
weston, the patched binary plays the bbb sample end-to-end with the
V4L2VideoDecoder + S264 + NV12 pipeline engaged. The lower-CPU
fast-tile result (~34.7 %) is reachable on KWin too once KWin's
remaining bug is fixed; weston advertises only the LINEAR modifier,
so chrome falls back to LINEAR composite there — still zero-copy,
just less efficient than the AFBC path KWin enables.

The KWin investigation continues in
[`KWIN_PIVOT.md`](https://git.reauktion.de/marfrit/marfrit-packages/src/branch/main/arch/chromium-fourier/KWIN_PIVOT.md)
(local). Headline experiment: WAYLAND_DEBUG on chrome + KWin during
the stall window, looking for the missing `wl_buffer.release` or
`wp_presentation_feedback` event around the 6-second mark.
