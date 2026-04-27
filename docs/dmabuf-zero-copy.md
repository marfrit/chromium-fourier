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
