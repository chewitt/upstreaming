# RK3576 VDPU383 decode performance (cover-letter notes)

Status: two WIP commits on `rockchip-7.3.y`, not posted. These notes are the commentary
for the cover letter; the commit messages carry only the problem and the fix.

## Patches

1. `media: rkvdec: pad the VDPU383 frame stride`
2. `media: rkvdec: enable the VDPU383 read cache`

## Problem

On RK3576 (ROCK 4D), the mainline `rkvdec` VDPU383 support decoded 4K60 H.264 at about
20 fps, against a rated 4K60. The CPU was idle and the request pipeline was full (FFmpeg
kept four jobs queued), so the hardware itself was slow: a median of 56.6 ms per frame.
The slowest frames were the small inter frames (median 4.4 KB of bitstream). That pointed
at reference fetches and motion compensation, not entropy decoding.

## Results

`ffmpeg -hwaccel v4l2request -hwaccel_output_format drm_prime -f null`, Kodi stopped,
first 20 s of `bbb_sunflower_2160p_60fps_normal.mp4` (H.264 High) and 15 s of a 4K
59.94 fps HEVC Main stream:

| Kernel | 4K60 H.264 | 4K HEVC Main |
|---|---|---|
| Unpatched | 20 fps | 79 fps |
| Stride padding only | 34 fps | not measured |
| Read cache only | 54 fps | not measured |
| Both patches | 86 fps | 163 fps |

Neither patch reaches 60 fps alone.

## Correctness

Output was bit-exact against FFmpeg's software decoder (`framemd5` of
`hwdownload,format=nv12` frames) for:
- 4K60 H.264: first 120 frames, and 300 frames from 5:00;
- 1080p60 H.264: 300 frames (stride 1984);
- 4K HEVC Main: 240 frames;
- 1914x1080 HEVC Main: 240 frames (odd width, stride 1984).

Kodi playback: 4K60 H.264 smooth; 4K23.976 HEVC Main 10 smooth.

## Where the values come from

- Stride: MPP `mpp_align_128_odd_plus_64()` (`osal/mpp_common.c`), used as `SLOTS_HOR_ALIGN`
  by `hal_h264d_vdpu383.c`, the AV1 and the VDPU384 HALs.
- Cache: vendor kernel `drivers/video/rockchip/mpp/mpp_rkvdec2.c`, `rkvdec_vdpu383_run()`:
  `RKVDEC_REG_CACHE0_SIZE_BASE` = cacheable | read allocate | 64-byte lines,
  `RKVDEC_REG_CLR_CACHE0_BASE` = 1, `RKVDEC_REG_MAX_READS` = 0x1c. The vendor register base
  is 0x27b00100, so these land at 0x27b0061c/610/618, inside the `cache` region that
  mainline `rk3576.dtsi` already declares (0x27b00600).

## Ruled out

- Row cache buffer placement: allocating the RCBs in MPP's SRAM priority order (loop filter
  row first) made no difference. The SRAM pool was in use.
- Clocks and DDR: decoder core 594 MHz, DDR 2736 MHz.
- Control registers (gating, timeout, error handling, QoS): same as MPP.

## Open questions for review

- Should the cache setup be done once at power-up rather than per job? The vendor driver
  writes it per job.
- The RCB sizes in `vdpu383_rcb_sizes[]` are about 7x larger than MPP's per-codec sizes
  (about 2 MB vs about 260 KB at 3840 wide); separate cleanup.
