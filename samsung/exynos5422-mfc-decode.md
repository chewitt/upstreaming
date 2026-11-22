# Hardware video decode on Exynos 5422 (Odroid XU4)

Research notes on what mainline Linux, FFmpeg and Kodi need to play video with the
Exynos 5422 Multi Format Codec (MFC) block. Status: research only, nothing tested.

## Summary

- The kernel side of decode is complete: `s5p-mfc` drives the 5422 MFC (v7) as a
  stateful V4L2 decoder with linear NV12M output and dmabuf export. No codecs were
  removed from the driver.
- The display hardware has no YUV planes on Exynos5. Frames must be converted to RGB
  by the GScaler, or composited by the GPU.
- Samsung's 2019 series exposing the GScalers as DRM planes ("local path") was never
  merged, and it only targeted the Exynos5433 DECON, not the 5422 FIMD and Mixer.
- FFmpeg's `*_v4l2m2m` decoders still only return frames in system memory, not as
  DRM_PRIME (dmabuf) frames, so Kodi cannot use them zero-copy.
- The cheapest route to try first is GPU composition: Panfrost (Mali-T628) importing
  the MFC NV12M dmabufs through EGL, plus DRM_PRIME output in FFmpeg's stateful
  decoder.

## Hardware

| Block | Mainline driver | Compatible | Notes |
|---|---|---|---|
| MFC v7 | `drivers/media/platform/samsung/s5p-mfc` | `samsung,mfc-v7` | stateful V4L2 m2m, firmware `s5p-mfc-v7.fw` |
| GScaler x2 | `exynos-gsc` (V4L2 m2m) or `exynos_drm_gsc` (DRM IPP) | `samsung,exynos5420-gsc` | colour conversion, scaling, rotation |
| Mixer (HDMI) | `exynos_mixer` | `samsung,exynos5420-mixer` | RGB layers only |
| FIMD (LCD/DP) | `exynos_drm_fimd` | `samsung,exynos5420-fimd` | |
| GPU Mali-T628 | `panfrost` | `samsung,exynos5420-mali`, `arm,mali-t628` | |

The 5422 uses `exynos5420.dtsi`.

## MFC v7 in mainline s5p-mfc

Decode formats on MFC v7 (from the `formats[]` table in `s5p_mfc_dec.c`):

- Coded: H.264, H.264 MVC, H.263, MPEG-1, MPEG-2, MPEG-4, XviD, VC-1 (Annex G and L),
  VP8. HEVC and VP9 are MFC v10+ only, so the 5422 never had them.
- Raw (CAPTURE): `NV12M` and `NV21M` (linear, one buffer per plane) and
  `NV12MT_16X16` (16x16 tiled). `YUV420M`/`YVU420M` are MFC v12 only.

The CAPTURE queue supports `VB2_DMABUF` and the decoder implements `VIDIOC_EXPBUF`, so
decoded frames can be exported as dmabufs.

The driver history since 2016 shows no codec removal. The larger reworks were:

- 2016-11, Marek Szyprowski: clock handling rework, MFC v8 (Exynos5433) support.
- 2017-02/03, Marek Szyprowski: memory allocation rework (preallocated block allocator
  for MFC v6+, IOMMU configuration, removal of dead code and unused structures).
- 2018-02, Smitha T Murthy: MFC v10.10 with HEVC and VP9 decode, HEVC encode.
- 2022-2025, Aakarsh Jain and Smitha T Murthy: MFC v12 (Tesla FSD), Exynos3250 v7
  variant data, NV12M/NV21M plane-size fix.

Everything else is treewide or V4L2 core churn.

## The missing YUV planes

On Exynos4 the HDMI Mixer has a Video Processor (VP) layer that accepts NV12/NV21. On
Exynos5 the VP is absent:

```c
static const struct mixer_drv_data exynos5420_mxr_drv_data = {
	.version = MXR_VER_128_0_0_184,
	.is_vp_enabled = 0,
};
```

so the Mixer only exposes ARGB4444/1555, RGB565 and XRGB/ARGB8888 layers. A decoded
NV12 frame cannot be scanned out directly; it has to be converted first.

The GScaler can do that conversion in two ways:

1. **Memory to memory**: read NV12 from memory and write RGB to memory. This works in
   mainline through either the V4L2 `exynos-gsc` driver or the DRM IPP v2 interface
   (`exynos_drm_gsc`). Both bind `samsung,exynos5420-gsc`, so only one of them can own
   each GScaler. The extra pass costs memory bandwidth.
2. **Local path**: the GScaler sends its output straight to the display controller
   with no intermediate buffer, acting as a YUV-capable plane. The vendor kernels use
   this; mainline does not support it.

## Samsung's GScaler planes series (2019, not merged)

Andrzej Hajda, "drm/exynos: add support for GSCALER planes on Exynos5433":

- v1: <https://patchwork.kernel.org/project/linux-samsung-soc/cover/20190301122055.7135-1-a.hajda@samsung.com/>
- v3 RESEND (24 patches): <https://patchwork.kernel.org/project/linux-samsung-soc/cover/20190325071349.22600-1-a.hajda@samsung.com/>

The cover letter describes three parts:

1. Preparatory refactoring of the Exynos `drm_crtc` and `drm_plane` structures, so that
   a plane does not have to be physically bound to a CRTC (embed `exynos_drm_crtc` in
   each display controller context, `exynos_drm_crtc_init()`, plane
   update/disable callbacks, a GSCALER plane capability).
2. Local path support in the GScaler and the Exynos5433 DECON.
3. Fixes for existing bugs found along the way (GSC id assignment, the BGRX8888 →
   XBGR8888 format, YVU420 handling, waiting for a previous DECON update).

The series also depended on a separate "drm/exynos: add support for dynamic zpos in
DECON and FIMD" series, and the author notes the hardware feature is poorly
documented ("trial and error, vendor code analysis, guessing from datasheets"). It was
tested on a TM2 (Exynos5433) only.

None of the 24 patches are in mainline. For the 5422 the series is not enough on its
own: the 5422 has FIMD and the Mixer rather than DECON, and both would need their own
local-path support.

## FFmpeg

### Upstream

The stateful `h264_v4l2m2m`, `mpeg2_v4l2m2m`, `vp8_v4l2m2m`, ... decoders map
`V4L2_PIX_FMT_NV12M`/`NV21M` to `AV_PIX_FMT_NV12`/`NV21` and return frames in system
memory. There is no `AV_PIX_FMT_DRM_PRIME` output, so every frame is copied and Kodi
cannot pass it to the renderer as a dmabuf. Upstream has handled the XU4 in the past:
commit `ccc83ff11e` ("avcodec/v4l2_m2m_dec: Fix decoding on Odroid XU4", 2019) restored
the order in which CAPTURE buffers are set up, after an earlier refactor broke H.264
decode on the XU4.

### memeka's branches (github.com/mihailescu2m/FFmpeg)

Last pushed January 2019.

- `v4l2-drmprime`: FFmpeg 4.0.1, Lukas Rusak's v4l2m2m DRM_PRIME patches, and six
  Exynos commits (July 2018) that add a converter stage behind the MFC decoder:
  - `ee76397093` v4l2_m2m: check for and initialize converter device
  - `31278723b4` v4l2_m2m: implement decoder->converter pipeline
  - `98990e54e4` v4l2_m2m: add support for V4L2_MEMORY_DMABUF
  - `3f75962701` v4l2_m2m: add formats for exynos
  - `bc7061fe19` v4l2_m2m: proper release buffers when using converter
  - `a13b541be7` v4l2_m2m: exynos: force DRM_FORMAT_XRGB8888 when using conveter

  The pipeline is MFC (V4L2 decoder) → GScaler (`exynos-gsc` V4L2 m2m, memory to
  memory) → XRGB8888 dmabuf → DRM_PRIME frame. Kodi then places the RGB frame on a
  Mixer or FIMD layer. This works around the missing YUV planes, at the cost of a
  colour conversion pass in memory and Exynos-specific code in a generic decoder.

- `v4l2-leia-drmprime`: FFmpeg 4.0.3 with the Kodi Leia patches and Rusak's later
  v4l2m2m work (hwcontext, flush, software pixel formats, DRM_FORMAT_MOD_LINEAR),
  without the converter.

## Options for a current stack

### A. GPU composition (try first)

Panfrost supports the Mali-T628, and Mesa can import NV12 dmabufs through
`EGL_EXT_image_dma_buf_import`. Kodi's DRMPRIME GLES renderer could then draw the
MFC frames directly, with no GScaler and no new kernel code.

Needs:
- FFmpeg: DRM_PRIME output from the stateful v4l2m2m decoder (Rusak's patches or the
  newer Raspberry Pi v4l2m2m rework), exporting the CAPTURE buffers with
  `VIDIOC_EXPBUF`.
- Kodi: picks up `*_v4l2m2m` decoders that offer DRM_PRIME already (as used on the
  Raspberry Pi).

Open questions:
- Does Panfrost import a two-buffer NV12M frame (separate dmabufs per plane)?
- Does a T628 have the fill rate and bandwidth for 1080p60 composition plus the GUI?

A quick check: on a mainline image with Panfrost, run
`gst-launch-1.0 filesrc ! parsebin ! v4l2h264dec ! glimagesink` and confirm it
imports the decoder buffers without a copy.

### B. GScaler converter in FFmpeg (memeka's approach)

Proven to work in 2018, but it puts Exynos-specific device discovery and format
choices into FFmpeg's generic v4l2m2m code. That is unlikely to be accepted upstream,
and it costs an extra full-frame memory pass.

### C. GScaler local-path planes in Exynos DRM

The proper kernel solution: GScalers become YUV-capable DRM planes and the decoded
frame is scanned out directly. Hajda's series could be revived for the structural
refactoring, but the 5422 would need new local-path support in FIMD and the Mixer,
for hardware with little documentation. This is the largest piece of work.

## Sources

- Hajda GSCALER planes series: see links above.
- ODROID forum, GScaler zero-copy with v4l2m2m: <https://forum.odroid.com/viewtopic.php?t=31322>
- ODROID forum, `exynos-gsc` m2m device: <https://forum.odroid.com/viewtopic.php?t=26154>
- ODROID forum, XU4 `h264_v4l2m2m` with `s5p-mfc` and `exynos-gsc`: <https://forum.odroid.com/viewtopic.php?t=33671>
- FFmpeg-devel, v4l2_m2m MJPEG on the XU4 4.14 kernel: <https://ffmpeg.org/pipermail/ffmpeg-devel/2018-August/233115.html>
- LibreELEC forum, Odroid XU4 support: <https://forum.libreelec.tv/thread/12409-support-for-odroid-xu4/>
- Armbian, Exynos 5422 media testing script: <https://forum.armbian.com/topic/7343-exynos-5422-odroid-xu4-hc1-hc2-media-testing-script/>
