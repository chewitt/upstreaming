# dw-hdmi and meson HDMI patches vs the HDMI helper framework

Status: research. Covers the out-of-tree Amlogic DRM/HDMI patches carried on the
`amlogic-7.3.y` kernel branch, after the HDMI 2.0 scrambling framework series
(Nicolas Frattaroli, Cristian Ciocaltea; FROMGIT in that branch) was added.
Commits are named by subject because branch hashes change on rebase.

See also `drm-meson-modernisation.md` for the mainline meson driver itself.

## Key fact

On Amlogic, dw-hdmi advertises `DETECT | EDID | HPD` plus the HDR infoframe and
CEC notifier ops, but not `DRM_BRIDGE_OP_HDMI`. Meson already uses
`drm_bridge_connector` with `DRM_BRIDGE_ATTACH_NO_CONNECTOR`, but because no bridge
in the chain sets `OP_HDMI`, none of the HDMI state helper runs on Amlogic: no
format/bpc selection, infoframe generation, Broadcast RGB, TMDS validation, SCDC
sync or scrambling management. dw-hdmi keeps doing its own SCDC handling.

So the new scrambling framework cannot double-handle scrambling on Amlogic today,
and using the helpers means converting dw-hdmi to `OP_HDMI` for meson. A WIP series
on the Rockchip kernel branch already does that conversion for Rockchip; none of the
Amlogic dw-hdmi patches are in it.

## Our patches

**Keep, independent of any conversion** (several are upstream candidates):

- dw-hdmi: use the DDC address for E-DDC segment reads
- dw-hdmi: latch the bus formats at enable
- dw-hdmi: always enable the HDCP keep-out window (GX HBR dropout fix; must survive
  any conversion, the Rockchip series keeps it conditional on scrambling)
- dw-hdmi: SPDIF audio input, HBR on SPDIF, program the audio path at prepare
- hdmi-codec: register the sink controls once per codec
- bridge_connector: restore the max bpc default
- meson vclk/PLL table and fraction fixes, VENC wide DMT modes, PHY keyed on TMDS
  rate, `dw_hdmi_get_tmds_clock()`

**Duplicated by the Rockchip WIP series, pick one:**

- CSC offset scaling for deep colour: ours sign-extends, clamps and keeps the spare
  MSB clear; the Rockchip version is a plain shift. Prefer ours.
- GCP default phase: ours signals it only when it holds; the Rockchip version always
  clears it.

**Dropped on conversion** (the helper or the Rockchip series provides them):

- dw-hdmi: honour the connector colorspace in the AVI infoframe
- dw-hdmi: list deep colour output formats before RGB888
- dw-hdmi: implement the HDR DRM infoframe callbacks

**Reworked on conversion:**

- meson deep colour output: the VIC/PLL-table gating must move into a meson
  `tmds_char_rate_valid`, otherwise the helper picks a bit depth that the vclk tables
  then reject, with no fallback.
- meson HDR infoframe without a modeset: the helper only writes infoframes on enable,
  so the flush-time update stays, but writes the helper's infoframe state. The
  Rockchip dw-hdmi forces a modeset on HDR changes, which would have to go.
- meson Colorspace property: keep the modeset-on-change, drop the attach if dw-hdmi
  attaches it.

## Behaviour changes on conversion (inferred, need testing)

- Output format default becomes RGB first with a YUV420 fallback only. 4K60 on an
  HDMI 2.0 sink becomes 8-bit RGB at 594 MHz instead of 10-bit 4:2:0, unless
  userspace (Kodi) sets the colour format property.
- GXBB/GXL with `max_bpc` 8 lose the HDR property and HDR infoframe; mainline meson
  signals HDR on GXL today.
- 480i/576i: the Rockchip series enables dw-hdmi pixel repetition, and meson VENC
  already doubles pixels for these modes, so they risk being doubled twice.
- The helper ignores the sink's low-rate scrambling setting, which dw-hdmi's own code
  honours.
- SCDC sync on hotplug may add modesets on replug or TV power-cycle.

## Suggested order

1. Shared dw-hdmi fix base for the Amlogic and Rockchip branches from the "keep"
   patches, settling the two duplicates. Send generic fixes upstream.
2. Prepare meson for the helper: per-SoC `supported_formats`, per-SoC `max_bpc`
   (12 for G12/S4, 10 for GXM, 8 for GXBB/GXL), and a `tmds_char_rate_valid`
   wrapping the meson vclk tables.
3. Bring the Rockchip dw-hdmi conversion series onto Amlogic; drop the three
   duplicated patches; re-apply the HDCP keep-out and bus-format latch fixes; remove
   meson's manual max bpc and HDR property attach.
4. Rework the HDR-without-modeset patch on top of the helper's infoframe state.
5. Port the dw-hdmi-qp "defer sink scrambling disable to atomic_enable" fix to
   dw-hdmi (same timing problem), then drop meson's own
   `dw_hdmi_set_high_tmds_clock_ratio` call so SCDC is not written twice.

Steps 2-5 change HDMI behaviour on every Amlogic SoC and need hardware testing:
HBR passthrough (including 1.001 and deep colour rates), 4K60, HDR on GXL,
480i/576i, CEC.

## Open questions

- Output format policy at 4K60 on HDMI 2.0 sinks (RGB 8-bit vs YUV420 10-bit).
- Whether GXBB/GXL should keep HDR signalling at 8 bpc.
