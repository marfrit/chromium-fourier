# Mainline Rockchip V4L2 video playback HOWTO

This is a small guide to **playing video back on a mainline Linux
Rockchip system** using the in-kernel V4L2 stateless decoder
(`hantro` driver for H.264, `rkvdec2` for AV1 / HEVC / VP9), without
the Rockchip MPP / vendor 5.10 BSP stack.

If you're here because you got `ffmpeg -hwaccel v4l2request` running
at "300 fps with no display" but every attempt to actually watch the
video drops to single-digit fps — yes, you've hit the **dmabuf
zero-copy import wall**, this page is for you.

## The dmabuf zero-copy wall

The hantro / VDPU381 V4L2 stateless decoder produces NV12 frames
sitting in DMA-BUF / `drm_prime` memory on its V4L2 capture queue.
Decoding is genuinely cheap on the VPU; the expensive part is **what
happens between the V4L2 capture queue and your screen**:

```
                  decoder side                       display side
   ┌─────────┐    ┌──────────────┐  dmabuf  ┌──────────────┐    ┌─────┐
   │  H.264  │───▶│  V4L2 cap-q  │─────────▶│  consumer    │───▶│  ?  │
   │  bytes  │    │   (NV12)     │  drm_fd  │ (kms/gl/vk)  │    │     │
   └─────────┘    └──────────────┘          └──────────────┘    └─────┘
                       cheap                  ^^^ this is the wall
```

Three things have to align for the dmabuf to land on screen
zero-copy:

1. The consumer must accept `NV12` (or `MM21`/`P010` if the SoC's
   format requires it) over a `drm_prime` fd. **DRM/KMS scanout** does
   this trivially. **EGL** does it via `EGL_EXT_image_dma_buf_import`
   + `EGL_EXT_image_dma_buf_import_modifiers`. **Vulkan** does it via
   `VK_EXT_external_memory_dma_buf` + `VK_EXT_image_drm_format_modifier`.
2. The userspace driver below the consumer (mesa for GL/GLES, panvk
   for Vulkan) must understand the format and modifier the V4L2
   decoder emitted. Modifier mismatch = silent fallback to readback.
3. On Wayland: the compositor must negotiate a tranche on
   `zwp_linux_dmabuf_v1` v3+ that includes the format the V4L2
   decoder produced. Older negotiation paths just silently choose
   `XRGB8888`, forcing `NV12 → ARGB` conversion.

When any of those misalign, the playback chain falls back to **CPU
readback** — the dmabuf gets `mmap`'d, copied to a CPU buffer, and
shoved through whatever software conversion path the player has for
its non-accelerated VO. That's where the ~300 fps decode collapses
into ~15 fps display.

## Known-working invocations

Tested on PineTab2 (RK3566 / Mali-G52 / panfrost) and validated to
zero-copy. The same flags should apply to RK3399 (Mali-T860 /
panfrost) and RK3588 (Mali-G610 / panthor) — the kernel V4L2 ABI is
unchanged across SoCs.

### mpv, direct KMS (most reliable)

```sh
mpv --hwdec=v4l2request --vo=drm --hwdec-codecs=h264,hevc,vp9 file.mp4
```

`--vo=drm` scans the V4L2 capture-queue dmabuf straight to the display
plane via a DRM atomic commit — **no GL, no Wayland, no compositor**.
This is the single most reliable path and a good baseline to confirm
your kernel-side V4L2 stack actually works. Caveat: takes over the
console (you cannot have X / Wayland running on the same KMS card at
the same time). Switch to a free VT first (`Ctrl-Alt-F3` etc.).

### mpv on Wayland (gpu-next)

```sh
mpv --hwdec=v4l2request --vo=gpu-next \
    --gpu-context=wayland --gpu-api=opengl \
    --hwdec-codecs=h264,hevc,vp9 \
    file.mp4
```

`gpu-next` (libplacebo) handles `drm_prime → EGL_EXT_image_dma_buf_import →
zero-copy GL texture` cleanly. The legacy `--vo=gpu` does not — if you
get frame drops here, it is the dmabuf import gap, not the decoder.
Vulkan path (`--gpu-api=vulkan`) is still flaky on panfrost; on
panthor (RK3588) it works and is sometimes faster, try
`--gpu-api=vulkan` and watch the stats.

### gstreamer (mature)

```sh
gst-launch-1.0 -v \
  filesrc location=file.mp4 ! \
  qtdemux ! h264parse ! v4l2slh264dec ! \
  kmssink
```

`v4l2slh264dec` is the in-tree GStreamer V4L2 stateless H.264
decoder element. `kmssink` does direct scanout, same idea as `mpv
--vo=drm`. Switch to `glimagesink` or `waylandsink` for windowed
output once you've confirmed the decoder side works. Equivalent
elements exist for HEVC (`v4l2slvp9dec` etc).

### ffplay (works in principle, slower in practice)

```sh
ffplay -hwaccel v4l2request -hwaccel_output_format drm_prime file.mp4
```

ffplay can hit the kernel decoder fine, but its SDL2 windowing path
doesn't drive `EGL_EXT_image_dma_buf_import`, so the drm_prime frames
get readback'd into an SDL2 RGB texture before display. Useful as a
sanity check that decode itself works; not what you want for actual
playback.

### Browsers

For watching video in a browser on the same hardware, see the
chromium-fourier project this doc lives in. The browser case has its
own `gpu_feature_info.supports_nv12_gl_native_pixmap` wall on top of
the playback-side dmabuf wall.

## Verifying success

Beyond just "the picture shows up smoothly", a few quick checks:

```sh
# Decoder is open: should show your player as F....m holder
fuser -v /dev/video1 /dev/media0

# CPU is low: ffmpeg/mpv main thread under 30 % at 1080p30
top -p $(pgrep -f mpv)

# DRM plane is direct-scanning the dmabuf (--vo=drm):
sudo cat /sys/kernel/debug/dri/0/state | grep -A4 plane
```

If you see `fuser` empty, your player did not actually engage the
V4L2 path — likely missing `--hwdec=v4l2request` or the player was
built without v4l2_request hwaccel support. Verify:

```sh
ffmpeg -hwaccels 2>&1 | grep v4l2request
mpv --hwdec=help 2>&1 | grep v4l2
```

Both must list `v4l2request` for the path to work.

## Per-SoC quirks

- **RK3399** (Pinebook Pro, Rock Pi 4, etc.) — Mali-T860 / Midgard
  panfrost. H.264 only; no AV1 / HEVC / VP9 hardware. `hantro` is
  the only V4L2 driver. Mali-T860 has no Vulkan support
  (`panfrost` gallium covers Midgard for GL only); use the GL paths.
- **RK3566 / RK3568** (PineTab2, Quartz64, RK35xx series) —
  Mali-G52 / Bifrost gen2 panfrost. H.264 (8-bit) on `hantro`. Mali
  Vulkan via `panvk` exists but currently returns
  `VK_ERROR_INCOMPATIBLE_DRIVER` at chromium / mpv probe time on
  G52 r1; stick to GL paths for now.
- **RK3588** (Rock 5, CoolPi 4, Orange Pi 5, etc.) — Mali-G610 /
  Valhall panthor. H.264 on `hantro`; AV1 / HEVC / VP9 on
  `rkvdec2`. Vulkan on G610 / panthor / panvk works and is the
  recommended path; mpv `--gpu-api=vulkan` should beat the GL path.

## Why the docs gap

If you're frustrated that none of this is in any of the obvious
places: you're not alone. The reason most online tutorials route you
through Rockchip MPP, `libv4l-rkmpp`, the 5.10 BSP kernel, and X11
is that **that combination shipped first** — Rockchip's vendor stack
was working two to three years before the mainline V4L2 stateless
path stabilised. The mainline path's documentation lives across
Bootlin's blog, a few LWN articles, scattered linux-media mailing
list threads, the
[`v4l2-request-test`](https://gitlab.collabora.com/cosproject/v4l2-request-test)
README, and Mesa MR commit messages. There is no single canonical
HOWTO upstream. This page is one attempt at consolidating the
working invocations; PRs welcome.
