# chromium-fourier

Mainline-Linux Chromium with V4L2 hardware video decode unlocked for
Rockchip SoCs on Wayland + panfrost / panthor. **No vendor MPP, no Mali
blob, no panfork, no 5.10 BSP kernel.**

## Why this exists

The two existing chromium ports for Rockchip both pull in a vendor
stack:

- **7Ji's `chromium-mpp`** (Arch Linux ARM): forces the Rockchip 5.10
  BSP kernel, X11, and the vendor MPP library. Works, but locks you
  into the BSP world.
- **JeffyCN / igel-oss-style ports**: similar — vendor `libv4l-rkmpp`
  + libva backend, BSP kernel preferred.

Stock upstream Chromium gates the V4L2 video decoder behind
`BUILDFLAG(IS_CHROMEOS)`. On a vanilla `x86_64`/`aarch64` Linux build
the V4L2 path compiles in but the runtime master gate
(`media::kAcceleratedVideoDecodeLinux`) defaults to disabled when
`USE_VAAPI` isn't set, so `<video>` falls all the way through to
`media/filters/ffmpeg_video_decoder.cc` (software).

`chromium-fourier` is a small set of patches and a build recipe that
flips the V4L2 dispatch on for **mainline-only Rockchip systems**:

- mainline upstream Linux kernel (no Rockchip 5.10 BSP)
- Wayland (Plasma / KWin, sway, etc. — no X11 wrapping)
- mesa **panfrost** (Bifrost: Mali-G31 / G52 / G57) or **panthor**
  (Valhall: Mali-G610+) — no proprietary Mali blobs
- in-tree V4L2 stateless decoder (`media/gpu/v4l2`) talking to the
  hantro / VDPU381 driver via `/dev/video1` + `/dev/media0`

The end goal is a Chromium binary that decodes 1080p H.264 (and AV1 /
VP9 / HEVC where the SoC supports it) on the VPU rather than the CPU,
on a "boring upstream Linux" stack.

## Repository layout

```
chromium-fourier/
├── README.md
├── LICENSE
├── docs/
│   └── playback-howto.md      # mainline V4L2 video playback (mpv,
│                              # gstreamer, ffplay) — the dmabuf wall
├── pinetab2/                  # RK3566 (Mali-G52 panfrost) — validated
│   ├── PKGBUILD
│   └── patches/
│       ├── enable-v4l2-decoder-default.patch
│       └── wayland-allow-direct-egl-gles2.patch
└── rk3588/                    # RK3588 (Mali-G610 panthor) — scaffolded
    ├── PKGBUILD
    └── patches/               # same two patches as pinetab2/
        ├── enable-v4l2-decoder-default.patch
        └── wayland-allow-direct-egl-gles2.patch
```

Per-board status:

- **`pinetab2/`** — **validated end-to-end** on PineTab2 (RK3566 /
  Mali-G52 Bifrost / panfrost). 1080p30 H.264 plays at ~46 % combined
  CPU vs ~85 % renderer-only with the stock fallback.
- **`rk3588/`** — scaffolded; same patches, same gn args. **Validation
  pending on RK3588 hardware** (Mali-G610 Valhall / panthor + panvk
  for Vulkan, `hantro` for H.264, `rkvdec2` for AV1 / HEVC / VP9).
  Differences vs PineTab2 are launcher-only: Vulkan stays enabled
  because panvk on Valhall is functional.
- **`rk3399/`** (planned) — Pinebook Pro / Rock Pi 4 / etc.
  Mali-T860 Midgard panfrost; H.264 only via `hantro`. Same patches;
  Vulkan-default-disabled stance correct (Midgard has no Vulkan).

A separate Firefox-side effort (different patch shape — Firefox has
its own `media-rdd` / `RemoteVideoDecoder` plumbing for V4L2) will
follow as a sibling repo.

## Patches

Both patches live under `pinetab2/patches/` and are applied unchanged
to a vanilla upstream chromium tree. They are architecture-independent.

### `enable-v4l2-decoder-default.patch`
Flips `media::kAcceleratedVideoDecodeLinux` (user-visible feature name
`AcceleratedVideoDecoder`) to `FEATURE_ENABLED_BY_DEFAULT` whenever the
build sets `BUILDFLAG(USE_V4L2_CODEC)`, symmetric with the existing
`USE_VAAPI` arm. Without this, a `use_v4l2_codec=true use_vaapi=false`
build silently falls back to ffmpeg software decode at runtime even
though the V4L2 pipeline is fully compiled in.

### `wayland-allow-direct-egl-gles2.patch`
Re-allows `kGLImplementationEGLGLES2` (i.e. `--use-gl=egl`, no ANGLE
shim) in `WaylandSurfaceFactory::GetAllowedGLImplementations()`. The
downstream dispatcher in `CreateViewGLSurface` already handles that
implementation; only the advertised list was tightened upstream.
Restoring direct EGL is the path to surfacing panfrost's
`EGL_EXT_image_dma_buf_import` to chrome's GL display layer, which is
in turn what flips `gpu_feature_info.supports_nv12_gl_native_pixmap`
to true and lets the V4L2-decoded NV12 frames go zero-copy into the
compositor (instead of through the NV12→AR24 VPP shader path).

> **Caveat**: with `is_official_build=false`, the gn default for
> `dcheck_always_on` is `true`, and the direct EGL path FATALs at
> `gl_context_egl.cc:241` (`DCHECK(!global_texture_share_group_)`).
> The PKGBUILD therefore explicitly sets `dcheck_always_on=false`.
> The launcher defaults to `--use-gl=angle --use-angle=gles` (which
> works without DCHECK gymnastics, with a small CPU cost vs direct
> EGL); flip to `--use-gl=egl` once you have the dcheck-off binary.

## Building

### Quickstart on the target board (aarch64, native)

```sh
cd pinetab2
makepkg -si
```

This downloads the upstream chromium release tarball
(`chromium-${pkgver}.tar.xz`, ~5.7 GB compressed), applies the two
patches, runs `gn gen` + `ninja`, and installs into `/usr/lib/chromium/`
plus a launcher shim at `/usr/bin/chromium`. **Plan for 10+ hours** on
an RK3566 (PineTab2) — and roughly twice the RAM-to-disk swap headroom
that pacman would normally suggest. Cross-compiling from an x86_64
host (the `target_cpu="x64"` branch in the same PKGBUILD) is
significantly faster.

### Cross-compiling from x86_64

```sh
cd pinetab2
CARCH=aarch64 makepkg -i
```

Requires the aarch64 toolchain (`aarch64-linux-gnu-gcc` / `clang
--target=aarch64`). Or just build on the x86_64 host first
(`makepkg -si` on x86_64) to validate the patch chain before
committing the long aarch64 run.

### Bumping pkgver

Patch line numbers drift between chromium minor releases. After
bumping `pkgver`:

1. Try `makepkg --noextract` once — if `prepare()`'s `patch -p1`
   complains about offset, the hunk still applies but the diff was
   regenerated; commit the new patch.
2. If `patch -p1` fails outright (`Hunk #N FAILED`), open the target
   file and re-emit the hunk by hand. The two affected files
   (`media/base/media_switches.cc` and
   `ui/ozone/platform/wayland/gpu/wayland_surface_factory.cc`) are
   small and the conditional block being patched is stable.

## Runtime

The launcher shim (`/usr/bin/chromium`) defaults to:

```
--ozone-platform=wayland
--use-gl=angle --use-angle=gles
--enable-features=AcceleratedVideoDecoder
--disable-features=Vulkan
```

Vulkan is disabled by default because **panvk on RK3566 (Mali-G52
Bifrost) returns `VK_ERROR_INCOMPATIBLE_DRIVER`** on chromium's probe
and breaks V4L2 dispatch downstream (chrome falls back to FFmpeg
software). Override paths if you're working on a board where Vulkan
actually works:

| User flag                     | Effect                                |
|-------------------------------|---------------------------------------|
| `--enable-features=Vulkan`    | Enable Vulkan (panthor / others)      |
| `--use-vulkan=native`         | Pick the native Vulkan backend        |
| `--use-vulkan=swiftshader`    | Force the SwANGLE Vulkan backend      |
| `--disable-features=Vulkan`   | Explicit re-disable (rarely needed)   |
| `--use-angle=vulkan`          | Run ANGLE on Vulkan                   |

The launcher detects any of those on the command line and skips its
own `--disable-features=Vulkan`, so the user's intent always wins.

## Adjacent: playing video outside the browser

If you only want to watch H.264 / HEVC / VP9 / AV1 files from the
command line (mpv, gstreamer, ffmpeg), see
[`docs/playback-howto.md`](docs/playback-howto.md). Same hardware, same
underlying V4L2 stack, separate dmabuf wall — that doc captures the
working invocations and what the wall is.

## Validation

Tested end-to-end on **PineTab2** (RK3566 / Mali-G52 / panfrost /
mainline kernel) playing
`bbb_1080p30_h264.mp4` from `file://`:

- `Selected V4L2VideoDecoder for video decoding, codec: h264, profile:
  h264 main, coded size: [1920,1080]`
- `Open(): Using a stateless API for profile: h264 main and fourcc:
  S264`
- `AllocateInputBuffers(): Requesting: 17 OUTPUT buffers of type
  V4L2_MEMORY_MMAP`
- `SetExtCtrlsInit(): Setting EXT_CTRLS for H264`
- `SetupOutputFormat(): Output (CAPTURE queue) candidate: NV12`
- `/dev/video1` + `/dev/media0` open in the GPU process
- Combined Chromium CPU during playback: ~46 % across all chrome
  procs, vs ~85 % on a single renderer with stock Chromium falling
  through to software FFmpeg.

## License

The patches are released under the same BSD-3-Clause as Chromium
itself. See `LICENSE`.
