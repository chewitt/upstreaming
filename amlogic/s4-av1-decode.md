# AV1 hardware decode on S4 (S905Y4) in amvdec

Status: research only (2026-10-23). Nothing has been written or tested on hardware.
Base: the amvdec stateless decoder on `amlogic-7.3.y`, which runs H.264, HEVC, VP9 and
MPEG-2 on S4 with the microcode loaded through the secure world.
Vendor reference: Amlogic `media_modules` for 5.15 (the yocto tree, and the
`5.15.170_202508` branch, which agree on everything here). It has three drivers:

| Vendor driver | Path |
|---|---|
| V4L2 AV1 | `drivers/frame_provider/decoder_v4l/vav1/` |
| Firmware loading | `drivers/frame_provider/decoder/utils/amvdec.c` |
| Chip capability table | `drivers/common/chips/decoder_cpu_ver_info.c` |

In this note, `vav1.c:N` and `av1_bufmgr.c:N` refer to the V4L2 variant.

## Summary

- **The hardware:**
  - S4 decodes AV1 Main profile, 8 and 10-bit, level 5.1, up to 4Kx2K at 60 fps (S905Y4 datasheet).
  - It runs on the same single HEVC core that amvdec already uses for HEVC and VP9.
  - None of our other SoCs (GX, G12A, G12B, SM1) has AV1 in the vendor capability table or any AV1 firmware, so this work is S4 only.
- **Firmware is not a blocker:**
  - The vendor ucode is in the `video_ucode.bin` pack that the secure-world loader already uses on S4: `s4_av1_mmu` is an alias of the t5d/SC2 build.
  - It loads with the same SMC as the other codecs, on core 2. The tags are `'AV1S'` (0x41563153) for the swap build and `'AV1M'` (0x4156314d) for the non-swap build.
- **The uAPI already exists:**
  - V4L2 has a stateless AV1 uAPI (`V4L2_PIX_FMT_AV1_FRAME`, controls `V4L2_CID_STATELESS_AV1_{SEQUENCE,FRAME,TILE_GROUP_ENTRY,FILM_GRAIN}`), used by hantro (vpu981) and mediatek.
  - The FFmpeg `v4l2_request_av1` hwaccel is in the LE FFmpeg tree.
- **The main problem:** the ucode is a bitstream parser.
  - It must see the sequence header OBU, the frame header OBU and the tile-group OBUs.
  - FFmpeg passes only the tile data in the OUTPUT buffer (`decode_slice` receives `raw_tile_group->tile_data`). The headers arrive as controls.
  - So amvdec has to **re-synthesise the OBUs from the controls**, as the HEVC codec already does for SPS/PPS (`codec_hevc_synth.h`). VP9 does not need this, because `VP9_FRAME` buffers carry the uncompressed header.
- **Estimated size:** a `codec_av1.c` of about 2.7k-3.0k lines, comparable to `codec_vp9.c` (2470). About a third of that is the frame-header writer.

## Firmware

| Firmware name | TEE tag | Core | Swap |
|---|---|---|---|
| `av1_mmu_swap` | `'AV1S'` 0x41563153 | 2 (HEVC) | 1 |
| `av1_mmu` | `'AV1M'` 0x4156314d | 2 (HEVC) | 0 |

- The vendor tries the swap build first and falls back to the non-swap build (`vav1.c:11686-11700`).
- Swap is used with ucode newer than 4.157 (`vav1.c:192-194`). All current packs are newer: 4.184, 4.221 and 4.222.
- For amvdec this is `.tee_format = 'AV1M'` and `.tee_format_swap = 'AV1S'`, the same pattern as HEVC (`'265S'`/17).
- In the plaintext packs, `s4_av1_mmu[_swap]` are header-only duplicates of `t5d_av1_mmu[_swap]`:
  - the non-swap image is 16384 bytes and the swap image 20480 bytes;
  - the t5d images are byte-identical to `sc2_` and `t7_`.
- The encrypted per-SoC S905Y4 pack carries real `s4_av1_mmu*` entries.

## Ucode and host split

### What the ucode does

The ucode parses the sequence header, frame header and tile groups.

**Applied internally:** tile layout, quantisation, CDEF, loop restoration, TX mode, global motion and CDF adaptation.

**Returned to the host:** a parameter block (RPM, `union param_u` in `av1_global.h:2200-2316`). It contains:
- sequence fields;
- frame type, show flags and refresh flags;
- sizes and superres;
- order hints and `ref_frame_idx`;
- loop-filter levels and deltas;
- per-segment loop-filter features;
- colour info and film-grain-present.

**Copy order:** with efficiency mode on (the default since ucode 4.157), the RPM is a plain copy. It is not the VP9 reversal in groups of 4 (`vav1.c:8998-9006`).

### What the vendor host does

`av1_bufmgr.c` is a 3420-line port of libaom's header logic, driven by the RPM. It handles:
- reference map and refresh;
- frame refs;
- size with refs;
- scale factors;
- sign bias;
- motion-field projection (`av1_bufmgr.c:1262-1435`);
- skip mode;
- segmentation bookkeeping.

`vav1.c` adds:
- loop-filter thresholds and delta inheritance from the primary reference;
- superres upscale steps;
- CDF and segment-map buffer selection per slot;
- the film-grain parameter handshake.

### Host to ucode

The only host-to-ucode path is a parser-memory table, written before every run. `HEVC_PARSER_MEM_WR_ADDR`/`RW_DATA` take 12 words at 0x11a0 on SC2 and later, so S4 (`vav1.c:7549-7566`):

| Words | Content |
|---|---|
| 0-1 | order hints of ref slots 0-7 (8 bits each) |
| 2-9 | `width << 16 \| height` per slot |
| 10-11 | frame-buffer index per slot (8 bits each) |

Per-buffer segment info goes at 0x11b0 + index (`vav1.c:8436`).

There is no register path that lets the host hand the ucode an already-parsed header.

## Mapping onto stateless V4L2

The V4L2 controls replace the libaom bufmgr: refs, order hints, sizes, skip-mode frames and the primary ref frame all come from userspace. Per CAPTURE buffer, the driver still keeps:
- size, `order_hint` and the order hints of its own refs (for motion-field projection);
- frame type and intra-only;
- global-motion parameters;
- segment features (`REF_SEG_INFO`);
- loop-filter deltas;
- the 39 film-grain words.

### Per request

Each request feeds the ucode three OBUs. Each one gets the vendor's 20-byte prefix (described after the list).

1. **Sequence header**, synthesised from `AV1_SEQUENCE`:
   - kept minimal: no frame ids, no timing or decoder model, one operating point;
   - re-sent every frame. The ucode answers `SEQ_HEAD_PARSER_DONE` (0xf2) and the host continues with `SEARCH_HEAD`.
2. **Frame header**, synthesised from `AV1_FRAME`. It uses the freedoms the syntax allows so it doesn't need history:
   - `frame_refs_short_signaling = 0` with an explicit `ref_frame_idx[7]`;
   - `found_ref = 0` with an explicit size;
   - loop-filter delta and segmentation data always sent in full;
   - explicit tile sizes from `tile_info`;
   - global motion coded as subexp deltas against the primary ref's saved parameters;
   - film grain with `update_grain = 1`.
3. **Tile group:** one `OBU_TILE_GROUP` covering every tile, rebuilt in a bounce buffer from `TILE_GROUP_ENTRY` offsets with `tile_size_minus_1` fields.

The 20-byte prefix (vendor `vdec_av1_if.c:806-835`):

| Bytes | Content |
|---|---|
| 0-3 | BE32(L + 8) |
| 4-7 | ~bytes 0-3 |
| 8-11 | `00 00 00 01` |
| 12-15 | `AMLV` |
| 16-19 | 4-byte leb128(L + 4) |

The vendor also has a raw-OBU `no_head` mode (`DECODE_MODE` 0x80000004, `vav1.c:5297-5353`). It is not known whether the S4 ucode supports it. If it does, the prefix can go, but the synthesis is still needed.

### Alternative: userspace passes the raw header OBUs

The uAPI wording ("AV1 parsed frame, including the frame header") would allow this. But FFmpeg doesn't do it, and the other drivers ignore the headers, so it would need an FFmpeg change and a uAPI clarification. It is not a good basis for upstream. Synthesis keeps amvdec compatible with existing userspace.

## Hardware programming on S4

Status codes in `HEVC_ASSIST_SCRATCH_0` (`vav1.c:114-149`):

| Direction | Codes |
|---|---|
| Host to ucode | 5 `DECODE_SLICE`, 6 `SEARCH_HEAD`, 8 `FGS_PARAM_CONT`, 0xff `ACTION_DONE` |
| Ucode to host | 0xe0 picture end, 0xe1 film-grain request, 0xf2 sequence header done, 0xf3 frame header done, 0xf4 `OBU_FRAME` done |
| Errors and buffer state | 0x20-0x25: buffer empty, timeout, oversize, EOS, NAL done |

**Interrupts and the film-grain handshake** (HEVC mailbox 0):
- The film-grain request is handled in the hard IRQ (`vav1.c:10101-10115`). The ucode blocks until it is answered, so it must be answered even if grain is not implemented.
  - `HEVC_FG_STATUS` bit 11 means load: 39 `HEVC_FGS_DATA` words from the reference named in bits 10:8, then set `FGS_CTRL` bit 0.
  - Otherwise, save the current frame's 39 words.
  - Then write 8 to both registers.

**Init** (`vav1.c:5035-5430`, `10369-10440`):
- Frame-based mode: `DECODE_MODE` = 0x80000002, `NAL_SEARCH_CTL` = 8, `HEVC_WAIT_FLAG` = 1.
- `HEVC_COMPATIBILITY` bit 0 enables efficiency mode, where the ucode programs the parser, CABAC and IPP itself. Bit 1 means no RDMA.
- `HEVC_DBLK_CFG3` = 0x404010.
- `MPRED_CTRL4` bit 1 enables compressed co-located MVs (T5D and later).
- **The MMU map address goes in `HEVC_SAO_MMU_DMA_CTRL` (byte offset 0xd8f8)**, and the DW-MMU map in `..._CTRL2`. This is not `HEVC_ASSIST_MMU_MAP_ADDR`, which our HEVC and VP9 code use. The G12A-style write in `vav1.c` sits under `#ifdef CHANGE_REMOVED`.

**Each run:**
- core reset, full re-init, parser-memory ref table, then start (`vav1.c:11571-11785`);
- no AV1 state save or restore on S4 (that is T6W/S6 only).

**Per frame:**
- On the frame-header interrupt, the vendor runs `config_pic_size`, `config_mc_buffer` (ANC canvases with order hints, per-ref scale table at `AV1D_MPP_REFINFO_TBL_*`), `config_mpred_hw`, `config_sao_hw` and `config_dblk_hw`, then writes `AV1_SKIP_MODE_INFO` and `DECODE_SLICE` (`vav1.c:8139-8471`).
- `config_mpred_hw`: `MPRED_CTRL3` = 0x10151015, `REF00_POC` low bits 0x1010, and motion-field refs in `REF06-11`.
- `config_dblk_hw` covers CDEF, upscale, the CDF write/read slots and the segment-map slots.
- `config_sao_hw` sets AV1 mode (`SAO_CTRL5` bit 14) and the per-buffer film-grain table.

**Workspace** (4K, vendor spec index 1, `vav1.c:2215-2306`): about 5.4 MiB in total.

| Buffer | Size |
|---|---|
| Segment maps | 0x360000 (16 slots) |
| CDF | 0x80000 (16 x 0x8000) |
| Global motion | 0x800 |
| Deblock data | 0x60800 |
| CDEF | 0x28200 |
| Upscale | 0xc3000 |
| Film-grain tables | 16 x 8 KiB |
| MMU VBH | 0x5000 |
| DW MMU VBH | 0xa000 |
| MPRED above | 0x5400 |

Outside the workspace:
- per-frame MV buffers, about 0xa0000 at 4K (`vav1.c:1453-1480`);
- 16 frame buffers;
- per frame: FBC header 0x48000 and MMU map 0x4800.

**Output and bit depth:**
- References use MMU/FBC, with 1:1 double write for output, as for HEVC and VP9 on S4.
- S4 has no P010 output, so 10-bit streams come out as 8-bit NV12 unless we add an AM21C path.
- 12-bit is rejected.

**Film grain:**
- The decoder writes a grain table per picture.
- The S4 VPU has film grain disabled in the vendor display driver.
- For grain streams the vendor switches S4 to DW mode 0x21, a second write through its own MMU (`vav1.c:9832-9846`), and takes the grain-applied picture from it.
- It is not known whether plain linear DW output carries grain.
- First step: answer the handshake and ignore grain.

**Clocks and stream input:**
- The HEVC clock always runs at 800 MHz on S4.
- There is no ESPARSER, so the software VIFIFO writer is needed, as for HEVC and VP9.

## Work breakdown

1. amvdec: an S4 format entry, `V4L2_PIX_FMT_AV1_FRAME` plus the controls, TEE tags, a workspace allocator, and core init in efficiency mode.
2. `codec_av1.c` hardware side: picture size, MC/ref scale tables, MPRED and motion-field projection, SAO/DW/MMU, deblock/CDEF/upscale, CDF and segment slots, the IRQ state machine, and the film-grain handshake. About 1.2k lines; much of the MMU and DW code can be shared with `codec_hevc_common.c`.
3. V4L2 state: ref mapping, scale factors, sign bias, order-hint helpers and per-buffer saved state. About 0.35k lines.
4. OBU synthesis: a bit writer, the sequence header, the frame header (tile info, quant, segmentation, delta q/lf, loop filter, CDEF, LR, TX, reference modes, global-motion subexp, film grain), the tile-group rebuild and the AMLV prefix. About 1.0-1.2k lines. This can be checked off-target: feed FFmpeg's own controls through the writer and parse the result with FFmpeg's `av1_frame_split`/CBS against the original OBUs.
5. Testing: fluster `AV1-TEST-VECTORS` and `CHROMIUM-8bit/10bit-AV1-TEST-VECTORS` on VIM1S, compared with the libdav1d md5s.
6. Later: film grain (DW-MMU 0x21) and 10-bit AM21C output.

## Open risks

1. Whether the ucode accepts a sequence header re-sent every frame after the core reset.
2. Whether frame headers that are equivalent but not bit-identical are accepted (explicit refs, sizes and tiles). This should work, because the ucode is a parser, but it is untested.
3. RPM byte order with the TEE-loaded 4.222 ucode.
4. Whether linear DW output carries film grain.
5. Whether the ucode needs anything from OBUs we drop (metadata, padding, temporal delimiters).
