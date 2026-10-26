# rkvdec: AFBC capture output for VDPU346/VDPU381 (and RK3576)

Status: research and design note. No code yet.

Goal: let the mainline `rkvdec` stateless decoder write frames, and fetch references, in
the decoder's compressed frame-buffer format instead of linear NV12/NV15. 4K decode on
RK3566 and RK3576 is limited by reference-fetch bandwidth. The slowest frames are the small
inter frames, and the output is linear.

Sources and the paths used below:

- MPP: `rockchip-linux/mpp` (develop, at `fcfd59d2`). Paths are relative to the repo root.
- Mainline-based kernel: `rockchip-7.3.y`, `drivers/media/platform/rockchip/rkvdec/` (`rkvdec/` below).
- Vendor kernel: `rockchip/develop-6.6` (`drivers/video/rockchip/mpp/`, `drivers/gpu/drm/rockchip/`).
- FFmpeg v4l2-request: `libavutil/hwcontext_v4l2request.c` (armsoc-9.0.1 branch).

## Summary

- **VDPU346 (RK3566/RK3568) and VDPU381 (RK3588) write ARM AFBC.** It uses 16x16
  superblocks, YUV, no YTR and no block split. RK3568 and RK3588 both use MPP's
  `hal_*_vdpu34x.c` HALs. The mainline VOP2 cluster windows on RK3568 and RK3588 already
  accept `DRM_FORMAT_YUV420_8BIT`/`_10BIT` with `DRM_FORMAT_MOD_ARM_AFBC(16x16 | SPARSE)`.
  This is the target.
- **VDPU383 (RK3576) does not write AFBC 16x16.** Its "FBC" is Rockchip RFBC with 64x4
  blocks (evidence below). The RK3576 VOP2 has no YUV AFBC at all, only linear or RFBC.
  Mainline has no RFBC DRM modifier. The vendor's modifier uses vendor id 0x0b, and
  mainline assigns 0x0b to MediaTek. RK3576 is therefore a separate and larger project
  (new DRM modifier, VOP2 RFBC support, V4L2 fourcc), and it is out of scope here.
- **FFmpeg's `V4L2_PIX_FMT_YUV420_{8,10}_AFBC_16X16_SPLIT` describe the Allwinner Cedrus
  format, not this one.** They carry `SPARSE | SPLIT`, and the Rockchip decoder output is
  not split. rkvdec needs its own pair of non-SPLIT fourccs.
- **The output has a 4-line top offset for H.264 and HEVC.** MPP sets `offset_y = 4` and
  adds 16 lines to the vertical stride. A consumer must crop 4 lines from the top. VP9 has
  no offset.
- H.264 field/MBAFF streams (`frame_mbs_only_flag == 0`) cannot use FBC.
- References are read compressed automatically. The FBC enable bit is global per job and
  there is no per-reference switch, so every DPB buffer must be AFBC. In V4L2 this holds
  anyway, because all CAPTURE buffers share one format.

## 1. Registers

### VDPU346 / VDPU381 (`vdpu34x` in MPP)

Register layout: MPP `mpp/hal/rkdec/inc/vdpu34x_com.h`, mainline `rkvdec/rkvdec-vdpu381-regs.h`.

| Register | Field | Use | MPP | mainline |
|---|---|---|---|---|
| SWREG12 | bit2 `fbc_e` | enable FBC output (and FBC reference reads) | `vdpu34x_com.h:96` | `rkvdec-vdpu381-regs.h:95` |
| SWREG13 | bit12 `fbc_output_wr_disable` | not used by MPP | `vdpu34x_com.h:121` | `:118` |
| SWREG14 | bit0 `fbc_force_uncompress` | not used by MPP (see validation) | `vdpu34x_com.h:141` | `:135` |
| SWREG14 | bit3 `allow_16x8_cp_flag`, bit6 `fbc_h264_exten_4or8_flag` | not used by MPP | `vdpu34x_com.h:144,147` | `:138,141` |
| SWREG18 | `y_hor_virstride` | FBC: header stride / 16 | `vdpu34x_com.h:168` | `:162` |
| SWREG19 | `uv_hor_virstride` | FBC: same value as SWREG18 | | |
| SWREG20 | union: `y_virstride` [27:0] / `payload_st_offset` [31:4] | FBC: byte offset of the payload from the buffer start | `vdpu34x_com.h:178-186` | `:170-173` (only the y_virstride view) |
| SWREG141 | `rcb_fbc_base` | FBC row RCB | `vdpu34x_com.h:310` | `rcb_base[]`; size `{11, PIC_WIDTH}` "fbcr" at `rkvdec.c:1949` |

The decoded buffer address registers do not change. `decout_base`, `error_ref_base` and
`ref_base[]` point at the start of the buffer, which is the AFBC header.

Per codec (MPP), `hdr` = `fbc_hdr_stride` = `ALIGN(width, 64)`. It is in pixels, and it is
also the header row size in bytes, because each 16x16 block has a 16-byte header:

- H.264 `mpp/hal/rkdec/h264d/hal_h264d_vdpu34x.c:338-346`:
  `fbc_e = 1`, `reg018 = reg019 = hdr / 16`,
  `payload_st_offset = ALIGN(hdr * (ver_stride + 16) / 16, 4K) >> 4`.
- HEVC `mpp/hal/rkdec/h265d/hal_h265d_vdpu34x.c:956-963`: the same, with `(ver_stride + 64)`.
- VP9 `mpp/hal/rkdec/vp9d/hal_vp9d_vdpu34x.c:629-637`: `h = ALIGN(height, 64)`,
  `ALIGN(hdr * (h + 16) / 16, 4K)`. VP9 also has per-reference strides
  (`:684-689`): `{last,golden,altref}_hor_virstride = hdr >> 4`, and
  `{last,golden,altref}y_virstride` = the reference's payload offset. MPP writes that
  offset in bytes, while the linear path uses 16-byte units. The value is probably ignored
  on reads, because AFBC headers carry their own body offsets, but copy MPP exactly. The
  mainline equivalents are `rkvdec-vdpu381-vp9.c:325-337` (references) and `:482-484`
  (current frame).
- `ver_stride` comes from `vdpu_afbc_align_calc()` (`mpp/hal/rkdec/common/vdpu_com.c:20-39`):
  `ALIGN(height, 16) + expand`. `expand` is 16 for H.264 and HEVC and 0 for VP9
  (`hal_h264d_vdpu34x.c:978`, `hal_h265d_vdpu34x.c:1396`, `hal_vp9d_vdpu34x.c:1070`).
- AV1 on RK3588 is a separate Hantro-derived block (mainline `verisilicon` VPU981), not
  rkvdec. It is not covered here.

RCB sizes with FBC enabled (MPP):

- H.264, `hal_h264d_vdpu34x.c:630-635`: FBC row = `(chroma_format_idc > 1) ? 2 * W * bitdepth : 0` bits, so 0 for 4:2:0.
- HEVC, `hal_h265d_vdpu34x.c:739-760`: FBC row = `W * (chroma_idc - 1) * 2 * bitdepth` bits plus tile padding. The tile filter column uses the `filterd_fbc_on` table (`:75-80`).
- VP9, `hal_vp9d_vdpu34x.c:314-326`: FBC row = `8 * W * bitdepth` bits. The filter column grows from `16 * bd` to `24 * bd` bits per line.

All of these fit inside mainline's fixed `vdpu381_rcb_sizes` (`rkvdec.c:1940-1951`: fbcr
11 bytes/px of width, filtc 67 bytes/line), so no RCB change is needed.

Colmv: there is no interaction with FBC. Aside: MPP sets `colmv_compress_en` (SWREG12
bit1), for H.264 only when `frame_mbs_only` (`hal_h264d_vdpu34x.c:323`; HEVC
`hal_h265d_vdpu34x.c:1028`; VP9 `hal_vp9d_vdpu34x.c:608`). Mainline never sets it. That is
an independent bandwidth experiment.

10-bit: there is no special register. The output is AFBC YUV420 10-bit with a 15 bpp
superblock body budget (see layout). NV15 packing does not apply.

### VDPU383 (RK3576)

Layout: MPP `mpp/hal/rkdec/inc/vdpu383_com.h`, mainline `rkvdec/rkvdec-vdpu383-regs.h`.

- SWREG9: bit0 `fbc_e`, bit18 `fbc_force_uncompress`, bit19 `fbc_sparse_mode`
  (`vdpu383_com.h:29-41`, mainline `rkvdec-vdpu383-regs.h:47-62`). MPP never sets either
  of the last two.
- SWREG68 `hor_virstride` = `fbc_hdr_stride / 64`. SWREG193 `fbc_payload_offset` = a raw
  byte offset, `hdr * ALIGN(ver_stride, 64) / 16`, not 4K aligned
  (`mpp/hal/rkdec/h264d/hal_h264d_vdpu383.c:156-165`; HEVC
  `mpp/hal/rkdec/h265d/hal_h265d_vdpu383.c:423-430`; VP9
  `mpp/hal/rkdec/vp9d/hal_vp9d_vdpu383.c:433-443`, references `:536-540`).
- Separate payload base registers: SWREG192 for the current frame, SWREG194 for the error
  reference, SWREG195-210 for references (`vdpu383_com.h:363-372`). All take the buffer
  base. Mainline already writes 192 and 195-210 with the buffer address
  (`rkvdec-vdpu383-h264.c:317,331`, `rkvdec-vdpu383-hevc.c:418,432`). The vendor kernel
  only translates fds for 192 and 194-210. SWREG193 is a plain offset
  (`mpp_rkvdec2.c:142-182` tables).

Why this is RFBC 64x4 and not AFBC 16x16:

1. The stride unit is `hdr / 64`, which counts 64-pixel-wide blocks per row. VDPU34x uses
   `hdr / 16`, which counts 16-pixel blocks.
2. VDPU384A, the successor with the same register map, programs the same values under
   explicit 64x4 names: `reg193_dpb_fbc64x4_payload_offset`,
   `reg192_dpb_payload64x4_st_cur_base`, and "dpb data use rkfbc64x4"
   (`mpp/hal/rkdec/h264d/hal_h264d_vdpu384a.c:69-80`, `mpp/hal/rkdec/inc/vdpu384a_com.h:33,49-50`).
3. The RK3576 cluster windows offer YUV only as linear or RFBC. Mainline says so in the
   comment at `rockchip_vop2_reg.c:75-80`: "YUV: linear mode and rfbc ... produced by
   Video decoder". `formats_rk3576_cluster` has no `YUV420_8BIT`, and the AFBC modifiers
   are 32x8 only (`:240-...`). The vendor kernel is the same (`rockchip_vop2_reg.c:78-103`,
   `:329-375`). `DRM_FORMAT_MOD_ROCKCHIP_RFBC(64x4)` exists only in vendor
   `drm_fourcc.h:1587-1620`, under vendor id 0x0b, which mainline assigns to
   `DRM_FORMAT_MOD_VENDOR_MTK`.

MPP still labels the RK3576 output `MPP_FRAME_FBC_AFBC_V2`. The label is generic and says
nothing about the block size.

## 2. Buffer layout and size (VDPU346/381)

Layout: one buffer, with the AFBC header first and the payload (body) after it.

- Header: 16 bytes per 16x16 superblock, in raster order, `hdr` bytes per superblock row.
- Payload: starts at the 4K-aligned `payload_st_offset`.
- MPP's `MPP_FRAME_FBC_AFBC_V2` comment (`inc/mpp_frame.h:209-215`): "stores payload offset
  in first 32-bit in header address. Payload offset is always set to zero". This is the
  ARM AFBC rule: each header's first word is the body offset from the start of the
  header buffer. A consumer only needs the header base, and the gap the decoder leaves
  between header and body does not matter to it. The mainline and vendor VOP2 both
  program `HDR_PTR` = buffer start and take body addresses from the headers
  (`rockchip_drm_vop2.c:1533-1536`). RK3576 also sets `PLD_OFFSET` to the base
  (`:1658-1661`).
- Per-superblock body budget, as MPP and DRM size it (sparse worst case):
  `ALIGN(bpp * 256 / 8, 128)`. That is 384 bytes for 8-bit 4:2:0 (12 bpp) and 512 bytes
  for 10-bit (15 bpp).
- Size formula, used by MPP (`mpp/base/mpp_buf_slot.c:260-281,341-367`,
  `mpp/base/mpp_sys_cfg.c:138-...,204-320`) and copied from DRM's
  `drm_gem_afbc_min_size()`:
  `nb = (W64/16) * (VS/16)`; `size = ALIGN(nb*16, 64) + nb * body`, with `W64 = ALIGN(w, 64)`.
  `VS = ALIGN(h, 16) + 16` for H.264 and HEVC; VP9 aligns the height to 64 in `mpp_sys_cfg`.
- The `+16` lines: `compat_ext_fbc_buf_size` (`osal/mpp_compat.c:14-30`) defaults to 1 on
  Linux and 0 on Android. MPP's comment: "The codecs might need extra lines for deblock
  output". H.264 applies it at `mpp/codec/dec/h264/h264d_init.c:473-474`, HEVC at
  `mpp/codec/dec/h265/h265d_dpb.c:98-99`.
- `fbc_hdr_256_odd` (`osal/mpp_compat.c:40-46`) defaults to 0. When set, the header stride
  becomes `ALIGN(w, 256) | 256`, a DRAM bank-spreading option. Keep the default.

Sizes (W64 = width, VS = aligned height + 16). "decoder end" is the payload offset plus
`ceil((h + 4) / 16)` full rows of worst-case bodies. That is the highest address the
decoder can write if bodies sit in fixed slots.

| Case | MPP/DRM min size | H.264 payload off | H.264 end | HEVC payload off | HEVC end | linear NV12/NV15 |
|---|---|---|---|---|---|---|
| 3840x2160 8-bit | 13,056,000 | 528,384 | 13,062,144 | 540,672 | 13,074,432 | 12,441,600 |
| 3840x2160 10-bit | 17,233,920 | 528,384 | 17,240,064 | 540,672 | 17,252,352 | 15,552,000 |
| 1920x1080 8-bit | 3,312,000 | 135,168 | 3,268,608 | 143,360 | 3,276,800 | 3,133,440 |
| 1920x1080 10-bit | 4,371,840 | 135,168 | 4,313,088 | 143,360 | 4,321,280 | 3,916,800 |

At 4K the worst-case decoder end is 6-18 KB past MPP's buffer size. Either the bodies are
packed rather than slotted, which a header dump will show, or MPP relies on slack. The
kernel should size the buffer as
`PAGE_ALIGN(max(drm_min_size(W64, VS, bpp), payload_off + rows * (W64/16) * body))` and then
append colmv. The buffer is about 5% larger than linear. The win is in traffic, not
footprint.

ARM AFBC compatibility:

- 16x16 superblocks, YUV420 (`DRM_FORMAT_YUV420_8BIT` / `_10BIT`), YTR off. YTR is an RGB
  transform, and VOP2 would apply it.
- No SPLIT. The vendor VOP2 accepts only modifiers listed for the plane
  (`rockchip_drm_is_afbc()`, vendor `rockchip_drm_drv.c:146-161`). On RK3568 and RK3588
  the only SPLIT entry is `16x16 | YTR | SPARSE | SPLIT` (vendor `rockchip_vop2_reg.c:209-245`,
  mainline `:202-237`). VOP2 programs `AFBC_BLOCK_SPLIT_EN` straight from the modifier
  (`rockchip_drm_vop2.c:1654-1657`). The vendor stack displays decoder AFBC correctly, so
  it must use `16x16` or `16x16 | SPARSE`, which means no split. If I remember correctly,
  Rockchip's GStreamer and ffmpeg-rockchip use
  `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_SPARSE | AFBC_FORMAT_MOD_BLOCK_SIZE_16x16)`.
  That is from memory and not checked here.
- SPARSE: VOP2 never programs it into the hardware, and DRM sizes every AFBC buffer as
  sparse. `DRM_FORMAT_MOD_ARM_AFBC(16x16 | SPARSE)` is the safe description. A header dump
  settles whether body offsets advance by a fixed 384/512 per block.
- So the exact modifier is `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16 |
  AFBC_FORMAT_MOD_SPARSE)`. `(16x16 | SPARSE | SPLIT)`, what FFmpeg attaches to the Cedrus
  fourccs, is not in the mainline VOP2 list for these windows, so `ADDFB2` would fail.

What VOP2 needs (mainline `rockchip_drm_vop2.c`, `rockchip_vop2_reg.c`):

- Window: cluster windows only. `formats_cluster` has `YUV420_8BIT`/`10BIT` with the
  `format_modifiers_afbc` list (`rockchip_vop2_reg.c:57-73,202-237`; RK3568 windows `:672-697`,
  RK3588 `:1196-1276`). On RK3568 the cluster windows are AFBC-only
  (`rockchip_drm_vop2.c:398-404`). Decoder AFBC is therefore the only way to put video on a
  cluster window there. Today video has to go on an Esmart window.
- Pitch: `vir_width = pitches[0] * 8 / bpp`, with bpp 12 or 15 (`:1619`, and
  `vop2_afbc_transform_offset()` at `:443-447`). The pitch userspace passes must be
  `W64 * bpp / 8`: 5760 for 3840 at 8-bit, 7200 at 10-bit.
- Crop and offset: the header pointer is always the buffer start (`:1533-1536`). The source
  rectangle origin goes to `AFBC_PIC_OFFSET` and `TRANSFORM_OFFSET` (`:1664-1668`), so
  `SRC_Y = 4` with an fb height of `h + 4` handles the 4-line offset. The vendor kernel has
  no decoder-specific handling. `top_crop` in vendor `vop2_afbc_transform_offset()` is
  always 0, so the crop is a userspace job there too.
- RK3568: AFBC source width must be a multiple of 4 (`:1379-1381`).
- Possible mainline issue for 10-bit: `if (fb->format->is_yuv && bpp == 10) format =
  VOP2_CLUSTER_YUV444_10` (`:1626-1635`) never matches, because `vop2_get_bpp()` returns 15
  for `YUV420_10BIT`. The cluster then gets `VOP2_FMT_YUV420SP_10` (0x14), while the
  comment says the cluster uses 0x10. Check this on the first 10-bit test.
- DRM size check: `drm_gem_fb_afbc_init()` computes the minimum from fb width and height
  and requires the GEM object to be at least that size. A `sizeimage` that follows the
  formula above, plus colmv, is larger.

## 3. Restrictions (when MPP refuses FBC)

- H.264: `!frame_mbs_only_flag && bit_depth < 10` → no FBC
  (`mpp/codec/dec/h264/h264d_init.c:433-434`, comment "field mode can not use FBC"). This
  covers PAFF and MBAFF.
- H.264 with IEP detection mode at ≤1080p 8-bit → no FBC (`:439-441`). This is irrelevant
  to V4L2.
- `cap_fbc` per decoder (`osal/mpp_soc.c`): vdpu34x `:338` = 2, vdpu38x (RK3588) `:356` = 2,
  vdpu383 `:428` = 2. The flag is honoured in `mpp_dec_check_fbc_cap()` (`mpp/codec/mpp_dec.c:74-88`).
- 4:2:2: MPP does not refuse it (bpp 16 or 20 in the size calculation, plus the RCB FBC row
  term). There is no 4:2:2 YUV AFBC DRM format to describe a 4:2:2 buffer, so offer 4:2:0
  only.
- AV1 (VDPU383 and VDPU384B only; not in mainline rkvdec): 4:2:2 10-bit is unsupported
  (`mpp/codec/dec/av1/av1d_parser.c:698-704`).
- Width: the header stride is always `ALIGN(w, 64)`. The mainline frmsize step is already
  64 (`rkvdec.c:740-790`).
- Cropping: H.264 and HEVC have `offset_x = 0, offset_y = 4` (`h264d_init.c:470-472`,
  `h265d_dpb.c:95-96`). VP9 has no offset (`vp9d_parser.c:333-342`). AV1 sets 0
  (`av1d_parser.c:708-709`). On RK3588 AV1 also has `ver_stride = ALIGN(h, 8) + 28`
  (`:710-711`), which does not apply to rkvdec. The 4 lines are a decoder property (the
  deblocking write-back delay), not padding that can be dropped. SWREG14
  `fbc_h264_exten_4or8_flag` may be related. Untested.

## 4. Mainline implementation outline

### UAPI

- `include/uapi/linux/videodev2.h`: add two fourccs that map to
  `DRM_FORMAT_MOD_ARM_AFBC(16x16 | SPARSE)`, for example:
  - `V4L2_PIX_FMT_YUV420_8_AFBC_16X16` (YUV 4:2:0 8-bit, AFBC 16x16, sparse, no split)
  - `V4L2_PIX_FMT_YUV420_10_AFBC_16X16` (10-bit)

  The fourcc codes must not collide with Cedrus's `'AS12'`/`'AS01'`. Neither the Cedrus
  pair nor this pair is in mainline yet. Cedrus is in the LibreELEC Allwinner patches
  (`0416-media-Add-NV12-and-P010-AFBC-compressed-formats.patch`, by Jernej Škrabec). Best
  practice is to upstream both families together, with one documentation section that
  gives each fourcc's DRM modifier. V4L2 has no modifier mechanism, so one fourcc per
  modifier set is the existing precedent (as with `NV12_4L4` and `NV12M_8L128`).
- `v4l2-ioctl.c`: add the descriptions. `Documentation/userspace-api/media/v4l/pixfmt-*.rst`:
  describe the layout, the bytesperline meaning (`W64 * bpp / 8`, the DRM pitch, not a
  line stride), the single plane, and the top offset.
- Top offset: report it through `VIDIOC_G_SELECTION(CAPTURE, V4L2_SEL_TGT_COMPOSE)` as
  `{0, 4, w, h}`, or as `{0, 0, w, h}` for VP9. rkvdec has no `g_selection` today.

### rkvdec

1. `rkvdec.h`: add an `afbc` flag (or bpp) to `struct rkvdec_decoded_fmt_desc`, or an
   `rkvdec_is_afbc(fourcc)` helper. Add `afbc_payload_offset` and the header stride to
   `rkvdec_ctx`, set when the format is set.
2. Decoded format tables: `rkvdec_h264_decoded_fmts` and `rkvdec_hevc_decoded_fmts`
   (`rkvdec.c:392-400,497-513`) are shared with RK3399, which has no FBC. Add
   VDPU381-specific copies with the AFBC entries after the linear ones (same `image_fmt`,
   420_8BIT or 420_10BIT). Add them to `rkvdec_vdpu346_vp9_decoded_fmts` and
   `rkvdec_vdpu381_vp9_decoded_fmts` (`:571-618`) as well. Point `vdpu346_coded_fmts` and
   `vdpu381_coded_fmts` at the new tables. Leave VDPU383 alone (RFBC).
3. Default selection: `rkvdec_reset_decoded_fmt()` uses entry 0, and FFmpeg takes the
   driver default first (`v4l2request_select_capture_format()`). A module parameter, for
   example `rkvdec.afbc=1`, could put the AFBC entries first for 4:2:0 progressive
   streams. That gives A/B testing without touching userspace format choice. Long term,
   decide the default per codec (Cedrus makes AFBC the default for 10-bit HEVC).
4. `rkvdec_fill_decoded_pixfmt()` (`rkvdec.c:134-152`): add an AFBC branch before
   `v4l2_fill_pixfmt_mp()`, which does not know compressed fourccs. Set
   `num_planes = 1`, `bytesperline = W64 * bpp / 8`, and `sizeimage` from the formula in §2
   using the worst codec offset (HEVC `+64`), page aligned. Then `colmv_offset` and
   `+= colmv_size` as now. Fill `fmtdesc.description` in `rkvdec_enum_capture_fmt()` so a
   module-only build does not hit the "Unknown pixelformat" WARN in the built-in
   `v4l_fill_fmtdesc()`.
5. H.264 interlace: when the CAPTURE format is AFBC and the SPS has
   `!(flags & V4L2_H264_SPS_FLAG_FRAME_MBS_ONLY)`, either reject the SPS in try_ctrl, or
   (better) keep the AFBC entries out of the enumeration for that stream and reset to NV12
   via the existing `get_image_fmt` / `rkvdec_reset_decoded_fmt` path (`rkvdec.c:194-220`,
   `rkvdec-h264-common.c:140-160`).
6. Registers (VDPU381 regs file): turn `reg020_y_stride` into a union with a
   `payload_st_offset : 28` at bit 4 (`rkvdec-vdpu381-regs.h:170-173`). Then:
   - `rkvdec-vdpu381-h264.c:259-265`: if AFBC, set `fbc_e = 1`, `reg018 = reg019 = W64/16`,
     `reg020 = ALIGN(W64 * (VS + 16) / 16, SZ_4K)`.
   - `rkvdec-vdpu381-hevc.c:426-432`: the same with `+64`.
   - `rkvdec-vdpu381-vp9.c:482-484` (current) and `:325-337` (references), using the VP9
     offset (`ALIGN(h, 64) + 16`).
   - No change to the reference, colmv or RCB addresses.
7. Effort: about 250-350 kernel lines, including UAPI and docs. Per-codec register changes
   are about 15-30 lines each; the format and size plumbing is about 100.

### Userspace

- FFmpeg `libavutil/hwcontext_v4l2request.c`. The `_16X16_SPLIT` entries were added by
  Jernej Škrabec for Cedrus (commit `7e214a8b3d`, "avutil/hwcontext_v4l2request: Add
  support for AFBC_16X16_SPLIT pix fmts", Dec 2025). Plane setup is generic: one plane
  for ARM/Amlogic vendors, `offset = 0`, `pitch = bytesperline`, object size = `sizeimage`
  (`:200-240`). Add two entries for the new fourccs with
  `DRM_FORMAT_MOD_ARM_AFBC(16x16 | SPARSE)` (`:81-130` pattern). The hwaccel must also add
  the top offset to `AVFrame.crop_top` (read from `G_SELECTION`). Today nothing in
  `v4l2_request*.c` reads the selection. About 40 lines.
- Kodi: `VideoLayerBridgeDRMPRIME` creates the fb with the picture width and height
  (`VideoLayerBridgeDRMPRIME.cpp:168`) and fixes `SRC_X`/`SRC_Y` to 0 (`:242-243`). For
  AFBC it needs `fb height = crop_top + height` and `SRC_Y = crop_top << 16`. The plane
  needs IN_FORMATS that contain the modifier. That is a cluster window, which on RK3568
  is AFBC-only. The EGL/GLES path should not be offered these buffers unless Mesa
  (Panfrost/Panthor) imports YUV AFBC; that is unverified. About 30-60 lines.

## Validation plan

Off-target:

- Build-test the module (`O=` build, LE toolchain).
- Compute `sizeimage`, the payload offsets and the register values with a script, and diff
  them against the MPP formulas for the reference sizes (64x64, 1914x1080, 1920x1080,
  3840x2160, 4096x2304, 7680x4320).

On target, module-only (rebuild and `insmod` rkvdec; videodev unchanged):

1. Structure check, with no display. Decode with AFBC selected, `mmap`/EXPBUF one CAPTURE
   buffer, and dump the headers:
   - body offsets increase and stay inside `sizeimage`;
   - the first non-zero header row is row 0 or 1;
   - the last written row is `ceil((h + 4) / 16) - 1` for H.264 and HEVC;
   - whether offsets step by exactly 384/512 (SPARSE) or are packed.

   This also catches an undersized buffer: corrupted colmv, or IOMMU faults in dmesg.
2. Determinism. `framemd5` of the raw AFBC buffers should be identical across runs and
   across RK3566 and RK3588 (same IP family). Any reference-read bug shows as drift,
   because the references are AFBC too.
3. Performance. `ffmpeg -hwaccel v4l2request -hwaccel_output_format drm_prime -f null`,
   NV12 against AFBC, 4K H.264, HEVC Main and Main10, VP9. This needs no display. It is the
   headline number for RK3566.
4. Correctness against software (MD5). AFBC cannot be compared directly. Options:
   - (a) Display through a VOP2 cluster window (`modetest -P ...@...:WxH+0+4` with the AFBC
     fb, or Kodi once patched) and check visually, including the 4-line crop and the
     bottom rows.
   - (b) Set `fbc_force_uncompress` (SWREG14 bit0, VDPU34x). The HW should then write
     uncompressed AFBC superblocks. The per-superblock pixel order can be solved
     empirically by decoding the same intra frame as NV12 and as forced-uncompressed AFBC.
     After that, a small CPU unpacker gives a byte-exact comparison with FFmpeg's software
     decoder through the existing framemd5 flow. Compressed output can then be checked
     against forced-uncompressed output through VOP2 (identical picture). This is a
     hypothesis: MPP never sets the bit.
   - (c) A GPU import (Panfrost/Panthor) and readback, if Mesa supports YUV AFBC import on
     the Mali-G52 and G610. Unverified.

## Open questions

- Sparse or packed bodies on VDPU34x (header dump, §Validation 1).
- The meaning of `fbc_h264_exten_4or8_flag`, and whether any setting removes the 4-line
  offset.
- Mainline VOP2 10-bit YUV AFBC cluster format value (§2).
- Panfrost/Panthor YUV AFBC import, needed for Kodi's EGL path and for readback-based
  testing.
- RK3576: RFBC 64x4 needs a DRM modifier proposal (vendor id), VOP2 RFBC support and a
  V4L2 fourcc. It is a separate design note if pursued.

## Measured results (RK3566, ROCK 3C, 2026-10-25)

`ffmpeg -hwaccel v4l2request -hwaccel_output_format drm_prime -f null`, first 15 s of
`bbb_sunflower_2160p_60fps_normal.mp4` (H.264 High 4K60). Test module only: FBC enabled
with the MPP VDPU34x register values (SWREG12 `fbc_e`, SWREG18/19 = ALIGN(w,64)/16,
SWREG20 = payload offset >> 4), capture buffers grown to the worst-case AFBC size, output
not displayed.

| Decoder clocks | Output | colmv compression | 4K60 H.264 |
|---|---|---|---|
| 240/250 MHz (mainline today) | NV12 | off | 31 fps |
| 240/250 MHz | NV12 | on | 33 fps |
| 240/250 MHz | AFBC | off | 46 fps |
| 240/250 MHz | AFBC | on | 46 fps |
| 400 MHz | NV12 | off | 37 fps |
| 400 MHz | AFBC | on | 72 fps |

At 400 MHz, HEVC 4K NV12 decodes at 57 fps and VP9 4K 10-bit NV15 at 35 fps. No IOMMU
faults or decoder errors in any run. H.264 NV12 output at 400 MHz is bit-exact against
software decode (300 frames).

The decoder clocks are low because `rk356x-base.dtsi` assigns 297 MHz, which a 1200 MHz
GPLL cannot divide to, so they round down to 240 MHz (CABAC 250 MHz from CPLL). The vendor
kernel runs GPLL at 1188 MHz and lists a 400 MHz decoder OPP at 900 mV.

Conclusion: 4K60 H.264 on RK3566 needs both AFBC output and the 400 MHz clocks. Neither
is enough alone.

## Implementation results (RK3566, ROCK 3C, 2026-10-25)

Correction to the table above: the test module wrote the payload offset into SWREG20 bits
[27:0] in 16-byte units. MPP's `payload_st_offset` field starts at bit 4, so the register
holds the offset in bytes, and the test placed the bodies on top of the headers. The AFBC
rows above are therefore not valid.

With the implementation (V4L2 fourccs `'AF12'`/`'AF01'`, rkvdec VDPU381 support, decoder
clocks at 400 MHz, output not displayed):

| Stream | Linear | AFBC |
|---|---|---|
| H.264 High 4K60 | 37 fps | 70 fps |
| HEVC Main 4K | 58 fps | 114 fps |

Findings:

- **Packed bodies.** Header word 0 bits [27:0] hold the body offset from the buffer start, so
  the first block's body sits exactly at the payload offset. Bodies follow each other packed
  (deltas of 160-208 bytes for content and 16 bytes for flat blocks), not in fixed 384/512-byte
  slots. The DRM modifier is therefore `DRM_FORMAT_MOD_ARM_AFBC(AFBC_FORMAT_MOD_BLOCK_SIZE_16x16)`,
  without SPARSE, and the mainline VOP2 cluster windows list that modifier.
- **Header encoding.** The top nibble of word 0 and the remaining 96 bits do not decode as
  sixteen 6-bit subblock sizes the way Mesa's Panfrost header code reads RGB AFBC: the sums
  do not match the body spacing. Only a VOP2 display test can confirm the picture.
- **VP9 reference motion vectors.** The VDPU381 VP9 code derived the reference colmv address
  from the reference height and line stride instead of the colmv offset where it was written.
  With AFBC that address falls inside the payload, and the decoder hung the SoC (watchdog
  reset). Using the colmv offset fixes it; it was also wrong for linear references smaller
  than the capture format.
- **Determinism.** With destination buffers cleared before each job, per-frame CRCs of two
  120-frame runs were identical for H.264 4K, HEVC 4K and VP9 Profile 2 4K. Without clearing,
  later frames differ only because packed bodies leave stale bytes from earlier frames behind
  the last body.
- **Top offset.** H.264 and HEVC report the 4-line offset through the CAPTURE
  `V4L2_SEL_TGT_COMPOSE` rectangle. FFmpeg adds it to `crop_top` and the frame height, and
  Kodi's DRM PRIME plane uses it as `SRC_Y`. This is unverified until it is displayed.
- **Default.** The AFBC formats are listed after the linear ones unless the `afbc` module
  parameter is set, because consumers that import frames through EGL (such as a Dolby Vision
  GLES path) accept only linear frames, and RK3588 shares the VDPU381 code.

## Display results (RK3566, ROCK 3C, Kodi, 2026-10-26)

4K HEVC Main 10 (HDR10, 24 fps) and 4K60 H.264 play smoothly in Kodi with AFBC frames
scanned out by the VOP2 cluster window, so VOP2 accepts the decoder's header encoding.
The 4-line top offset has not yet been checked separately.

- **One AFBC plane on RK3566.** The RK3566 VOP2 exposes three planes to the video port: a
  primary RGB plane, an Esmart plane (linear, including NV12/NV15) and a Cluster plane, which
  is the only one that takes AFBC and the YU08/YU10 formats. Mesa gives Kodi's GUI an AFBC
  surface, so the GUI takes the Cluster plane and no plane is left for AFBC video
  ("Cannot find a Video Plane ... YU08"). Kodi now checks whether a video plane would be
  free with a linear GUI; if so it recreates the GUI surface linear, moves the GUI to a linear
  plane and repeats the plane search. RK3568 and RK3588 are unaffected because they have
  more cluster windows.
- **Memory bandwidth.** With AFBC disabled, 4K HEVC Main 10 playback through the linear
  Esmart path disturbed audio on RK3566; with AFBC it did not.
- **Black level (not AFBC related).** HDR letterbox bars looked lifted compared with an RPi5,
  with and without AFBC. The cause was a dw-hdmi WIP change that shifted the CSC offset
  terms by the output depth: for 10-bit limited-range RGB the RGB full-to-limited offset
  became `0x80` instead of `0x20`. The CSC tables already express offsets relative to the CSC
  scale (16 is `0x40` at scale 0 and `0x20` at scale 1), independent of depth. Writing `0x20`
  back to the A4/B4/C4 registers during playback made black match the RPi5; the change is
  reverted on the branch.
- **Plane colour properties.** VOP2 exposes `COLOR_ENCODING`/`COLOR_RANGE` on the planes, but
  the driver does not program them: YUV-to-RGB always uses the BT.709 limited-range matrix,
  so BT.2020 content is converted with the wrong matrix. Black is unaffected; hue and
  saturation are not exact. Not yet fixed.
