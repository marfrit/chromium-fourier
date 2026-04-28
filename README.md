# fourier

> *(repo currently named `chromium-fourier`; rename to `fourier` is
> pending the dust settling on the wider patch series)*

A campaign of patches that makes mainline-Linux Wayland video playback
on ARM Rockchip SoCs **work end-to-end on a "boring upstream" stack**
— no vendor MPP, no Mali blob, no `panfork`, no 5.10 BSP kernel, no
ANGLE-on-Vulkan-only Mali fallback, no NV12-to-AR24 software
conversion in the renderer.

Started as one patch series for chromium V4L2 hardware decode.
Became four, because each layer of the stack had its own contributing
bug. The patches each live in their own directory of this repo (or
will, once the planned rename + restructure happens — see "Repository
plan" below).

## What we found, layer by layer

| Layer | Bug | Patch |
|---|---|---|
| chromium runtime gate | `media::kAcceleratedVideoDecodeLinux` defaults to disabled when `USE_VAAPI` isn't set, even on a `use_v4l2_codec=true` build. Silent fallback to ffmpeg software decode. | `enable-v4l2-decoder-default.patch` |
| chromium ozone GL surface advert | `WaylandSurfaceFactory::GetAllowedGLImplementations()` strips `kGLImplementationEGLGLES2`, blocking direct-EGL / panfrost native dmabuf import surfacing. | `wayland-allow-direct-egl-gles2.patch` |
| chromium NV12 dmabuf import | `OzoneImageGLTexturesHolder::GetBinding` picks `GL_TEXTURE_2D` for V4L2-produced NV12 dmabufs. ANGLE rejects YUV EGLImage on non-`EXTERNAL_OES` target, force-falls to NV12→AR24 software conversion. Skia Ganesh handles `GL_TEXTURE_EXTERNAL_OES` natively; this branch was just missed. | `nv12-external-oes-on-modifier-external-only.patch` |
| chromium V4L2 capture pool depth | `V4L2VideoDecoder::ContinueChangeResolution` requests `num_codec_reference_frames + 2` = 6 buffers for H.264 main — exactly equal to chrome's wayland pipeline depth. First scheduling jitter exhausts the pool and hard-stalls the decoder. | `v4l2-capture-pool-floor-at-16.patch` |
| Qt 6 (`QOpenGLTextureGlyphCache`, `QRhiGles2`, `QOpenGLTextureUploader`) | `QT_CONFIG(opengles2)` build branch unconditionally picks `GL_ALPHA` as `glTexImage2D` internalFormat. `GL_ALPHA` was deprecated in OpenGL ES 3.0 (only sized formats valid). On Mali ES 3.2 contexts every text-glyph cache upload errors `GL_INVALID_VALUE`; KWin's debug callback fills the journal and adds enough load to mask other bugs. | `qt6-base-fourier` |
| KWin 6.6.4 transaction scheduling | `Transaction::watchDmaBuf` calls `DMA_BUF_IOCTL_EXPORT_SYNC_FILE` on every imported dmabuf and parks the transaction on a `QSocketNotifier(POLLIN)`. For V4L2-produced dmabufs the kernel returns a stub fence (because vb2 doesn't populate `dma_resv` — see kernel layer below). Even when the fence is real, the export-then-poll roundtrip adds milliseconds per frame. | `kwin-fourier` |
| Linux kernel (`videobuf2-core`, `hantro`, `rockchip-rga`) | vb2 doesn't propagate V4L2 buffer-state-done into the dmabuf's `dma_resv` exclusive fence. Wayland compositors (and any other userspace that thinks it's doing implicit-sync the modern way) get a stub fence representing nothing real. The architectural hole that the KWin layer above is papering over. | *open* |

## Validation (2026-04-28)

End-to-end smooth 1080p30 H.264 playback under KDE Plasma 6.6.4
Wayland on ohm (PineTab2 / RK3566 / Mali-G52 / panfrost mesa 26.0.5 /
mainline kernel 6.19.10) with the full patch chain installed:

- `bbb_1080p30_h264.mp4` plays through to EOF, no stall, no slideshow.
- Combined Chromium CPU: **~81 %** across all chrome procs vs **~131 %**
  baseline pre-patch (NV12→AR24 software conversion). Playback at
  ~34 % combined was briefly observed pre-stall and is reachable
  again on a less jitter-prone compositor session.
- KWin Wayland session also feels markedly snappier for unrelated
  video clients (Brave on YouTube, mpv, VLC) — the kwin-fourier
  watchDmaBuf bypass is a general-purpose latency reduction for
  every wp_linux_dmabuf client on this hardware.

## Repository plan

The current github layout still reflects the chromium-only origin:

```
chromium-fourier/               (this repo — renaming planned)
├── README.md                   (this file)
├── LICENSE
├── docs/
│   ├── dmabuf-zero-copy.md     # NV12 external_only investigation,
│   │                           # now also chronicles the rest
│   └── playback-howto.md       # mpv / gstreamer / ffmpeg recipes
├── tools/
│   └── dmabuf-modifiers.c      # tiny EGL modifier-table dumper
├── pinetab2/                   # RK3566 / Mali-G52 / panfrost — validated
├── rk3399/                     # RK3399 / Mali-T860 / panfrost — validated
└── rk3588/                     # RK3588 / Mali-G610 / panthor  — validated
```

Each board directory carries a PKGBUILD plus the four chromium
patches. The PKGBUILDs target pacman / Arch Linux ARM but the
patches themselves are board-agnostic and apply unchanged to a
vanilla upstream chromium tree.

The qt6-base-fourier and kwin-fourier patch series live in a
sibling project repo today — those are the target of the planned
rename + restructure, after which the layout becomes:

```
fourier/                        (renamed)
├── README.md
├── chromium-fourier/           (current pinetab2/ rk3399/ rk3588/)
├── qt6-base-fourier/           (currently in marfrit-packages)
├── kwin-fourier/               (currently in marfrit-packages)
└── docs/, tools/               (kept)
```

In the meantime, the qt6-base-fourier and kwin-fourier sources are
maintained at https://git.reauktion.de/marfrit/marfrit-packages
under `arch/qt6-base-fourier/` and `arch/kwin-fourier/`. They will
move into this repo when the rename happens.

## Building (chromium part, today)

```sh
cd pinetab2          # or rk3399, rk3588
makepkg -si
```

Downloads the upstream chromium release tarball, applies the four
fourier patches, runs `gn gen` + `ninja`, installs into
`/usr/lib/chromium/` plus a launcher shim at `/usr/bin/chromium`.
**Plan for 10+ hours** of build time on an RK3566 — and roughly twice
the RAM-to-disk swap headroom that pacman's normal heuristics
suggest. Cross-compiling from x86_64 (the `target_cpu="x64"` branch
in the same PKGBUILD) is significantly faster.

For the qt6-base-fourier and kwin-fourier components see the
respective READMEs in `arch/qt6-base-fourier/` and
`arch/kwin-fourier/` of the marfrit-packages gitea repo for now.

## Patches (chromium part)

### `enable-v4l2-decoder-default.patch`
Flips `media::kAcceleratedVideoDecodeLinux` (user-visible feature
name `AcceleratedVideoDecoder`) to `FEATURE_ENABLED_BY_DEFAULT` when
the build sets `BUILDFLAG(USE_V4L2_CODEC)`, symmetric with the
existing `USE_VAAPI` arm.

### `wayland-allow-direct-egl-gles2.patch`
Re-allows `kGLImplementationEGLGLES2` (`--use-gl=egl`, no ANGLE
shim) in `WaylandSurfaceFactory::GetAllowedGLImplementations()`. The
downstream dispatcher in `CreateViewGLSurface` already handles that
implementation; only the advertised list was tightened upstream.

> **Caveat**: with `is_official_build=false`, the gn default for
> `dcheck_always_on` is `true`, and the direct EGL path FATALs at
> `gl_context_egl.cc:241` (`DCHECK(!global_texture_share_group_)`).
> The PKGBUILD therefore explicitly sets `dcheck_always_on=false`.

### `nv12-external-oes-on-modifier-external-only.patch`
Extends `OzoneImageGLTexturesHolder::GetBinding` to pick
`GL_TEXTURE_EXTERNAL_OES` whenever the EGL driver advertises the
pixmap's DRM modifier as `external_only` for the given fourcc, in
addition to the existing `format.PrefersExternalSampler()` branch.
Adds a static helper `NativePixmapEGLBinding::ModifierRequiresExternalOES`
that queries `eglQueryDmaBufModifiersEXT` and caches the answer per
`(fourcc, modifier)` tuple.

### `v4l2-capture-pool-floor-at-16.patch`
Floors `v4l2_num_buffers` at 16 in `V4L2VideoDecoder::ContinueChangeResolution`
when V4L2 is the buffer allocator. Default
(`num_codec_reference_frames + 2`) is 6 for H.264 main, equal to
chrome's wayland compositor pipeline depth — first scheduling jitter
event exhausts the pool. The `std::max` preserves correctness for
high-DPB codecs (HEVC, VP9 with deeper reference counts).

## Runtime

The launcher shim (`/usr/bin/chromium`) defaults to:

```
--ozone-platform=wayland
--use-gl=angle --use-angle=gles
--enable-features=AcceleratedVideoDecoder
--disable-features=Vulkan
```

Vulkan is disabled by default because **panvk on RK3566 (Mali-G52
Bifrost)** returns `VK_ERROR_INCOMPATIBLE_DRIVER` on chromium's probe
and breaks V4L2 dispatch downstream. Override paths if you're working
on a board where Vulkan actually works (`rk3588/` defaults Vulkan-on):

| User flag                     | Effect                                |
|-------------------------------|---------------------------------------|
| `--enable-features=Vulkan`    | Enable Vulkan (panthor / others)      |
| `--use-vulkan=native`         | Pick the native Vulkan backend        |
| `--use-vulkan=swiftshader`    | Force the SwANGLE Vulkan backend      |
| `--disable-features=Vulkan`   | Explicit re-disable (rarely needed)   |
| `--use-angle=vulkan`          | Run ANGLE on Vulkan                   |

The launcher detects any of those on the command line and skips its
own `--disable-features=Vulkan`, so the user's intent always wins.

## Adjacent

- `docs/dmabuf-zero-copy.md` — the panfrost / panthor `external_only`
  investigation; the original deep-dive that produced patch 3 and
  pointed at the rest of the chain.
- `docs/playback-howto.md` — mpv / gstreamer / ffmpeg invocations
  that work on the same V4L2 stack outside the browser.
- `tools/dmabuf-modifiers.c` — a 100-line EGL probe that dumps the
  available NV12 modifiers and their `external_only` flags. Useful
  for sanity-checking what your panfrost / panthor / mesa version
  advertises before you start patching.

A separate Firefox-side effort (`firefox-fourier`, in
marfrit-packages) carries the same V4L2 stateless unlock for
Firefox 150 — different patch shape because Firefox's media-rdd /
RemoteVideoDecoder plumbing is independent.

## License

The patches are released under the same BSD-3-Clause as Chromium
itself. See `LICENSE`. The qt6-base-fourier and kwin-fourier
patches are released under the LGPL-2.1+ matching their upstreams.
