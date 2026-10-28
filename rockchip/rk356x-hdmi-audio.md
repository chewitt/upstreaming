# RK3566/RK3568 dw-hdmi audio passthrough and HBR (notes)

Status: working on Rock 3C (RK3566) with branch `rockchip-7.3.y`. E-AC3, Dolby TrueHD and
DTS-HD MA pass through cleanly, including when a title starts straight after another HDR
title. 2160p50/60 HDR is output at 10 bits per component. The sync-loss fix in section 4 is
not yet confirmed, and multichannel PCM channel placement (section 8) is open.

RK356X uses the older dw-hdmi (v2.11a, `DWC HDMI 2.0 TX PHY`) fed by `i2s0_8ch`
(`rockchip-i2s-tdm`). RK3588 and RK3576 use dw-hdmi-qp, which behaves differently in
several places below.

## 1. Compressed audio was sent as PCM

With insert_pcuv set (`AUD_CONF2` bit 2, reset default 1), dw-hdmi generates the IEC 60958
channel status itself from `FC_AUDSCHNL0-8`. Those registers have no non-audio bit, so
IEC 61937 streams go out flagged as PCM and decode as noise. dw-hdmi-qp sets the non-audio
bit itself, so this only affects dw-hdmi.

Setting NLPCM (`AUD_CONF2` bit 1) by itself stopped the noise but gave silence: in NLPCM mode
the controller takes the PCUV bits from the I2S stream, in this per-sample layout with
`AUD_CONF1` width 21:

| Bits | Field |
|---|---|
| 31 | B (block start) |
| 30-27 | P, C, U, V |
| 26-11 | 16-bit IEC 61937 payload |

ALSA's `IEC958_SUBFRAME_LE` uses a different layout: preamble in bits 0-3, audio 4-27, then
V28, U29, C30 and P31.

The changes:

- **dw-hdmi:** Jernej Skrabec's "drm/bridge: dw-hdmi: add IEC958 and HBR I2S support" (from
  the LibreELEC Allwinner series) configures width 21 for `IEC958_SUBFRAME_LE`, selects NLPCM
  for non-audio streams or HBR for 8 channels at 192 kHz, and keeps insert_pcuv for PCM.
  "ASoC: hdmi-codec: name the component driver" lets the CPU DAI recognise an HDMI link.
- **i2s-tdm (WIP):** for playback runtimes whose codec is hdmi-codec, translate
  `IEC958_SUBFRAME_LE` into the layout above in the dmaengine copy path (the payload moves
  right by one bit, PCUV moves to 30-27, B is set on the Z preamble). Because the translation
  runs in the copy path, mmap access is refused for IEC958. Only the `rk3568-i2s-tdm` match
  data enables it; `rk3588-i2s-tdm` is unaffected.
- **alsa-lib conf (LibreELEC):** for non-audio AES0 values (2 and 6), route the `hdmi:` device
  through `iec958` (status from AES0-3, `hdmi_mode`) and `mmap_emul` to `hw`. The `iec958`
  plugin requires mmap access on its slave, and the `hw` plugin has no mmap emulation. PCM
  keeps the direct path.

The Rockchip 6.6 kernel uses the same I2S route on RK3566 boxes: `IEC958_SUBFRAME_LE` with
NLPCM or HBR set in `dw-hdmi-i2s-audio.c`. That makes the I2S route valid on RK356X, unlike
Amlogic, where the vendor never sends compressed audio over I2S. The S/PDIF route also exists
in hardware (RK3568 TRM table 20-2 wires `spdif_8ch` into the HDMI TX S/PDIF input, and
`CONFIG0_ID` reads `0xBF`, audspdif present) and remains a fallback.

## 2. The non-audio flag arrived after hw_params

With the above in place, the hardware received `IEC958_SUBFRAME_LE` and the
`IEC958 Playback Default` control read `AES0=0x06` during playback, but `AUD_CONF2` stayed
`0x00`.

alsa-lib's hooks plugin runs `ctl_elems` hooks after the slave `hw_params`
(`src/pcm/pcm_hooks.c`, `snd_pcm_hooks_hw_params()`), and the hook restores the previous
value on close (`preserve true`). dw-hdmi took the PCM/NLPCM/HBR decision in the hdmi-codec
`hw_params` callback, so it always saw the PCM channel status. dw-hdmi-qp configures audio
from the `prepare` callback, which runs after the hook.

Fix: "drm/bridge: dw-hdmi: configure I2S audio in prepare" moves the I2S sampler setup from
the hdmi-codec `hw_params` op to `prepare`. E-AC3 then passed through. This changes the
dw-hdmi I2S path for every SoC that uses it (i.MX, Allwinner, Amlogic, older Rockchip).

## 3. HBR glitches: HDCP keep-out window

TrueHD and DTS-HD MA then played with glitches and dropouts. `FC_INVIDCONF` read `0x78`:
HDCP keep-out (bit 7) off. Mainline enables the keep-out window only for HDCP or scrambled
links, so every mode below 340 MHz runs without it. The same bug caused the HBR dropouts on
Amlogic GXBB, GXM and G12B, where setting this bit alone stopped them.

Fix: "drm/bridge: dw-hdmi: always enable the HDCP keep-out window"
(`Fixes: 9aaf880ed4ee ("imx-drm: Add mx6 hdmi transmitter support")`), ported from the
Amlogic branch. With it, E-AC3, TrueHD and DTS-HD MA all play cleanly on RK3566.

The HBR dropouts on RK3588/RK3576 (dw-hdmi-qp) are a separate issue: the qp keep-out
setting made no difference there.

## 4. Lost sync after a scrambled to unscrambled modeset (fix not yet confirmed)

Stopping 2160p23.976 10-bit HDR playback (scrambled, about 371 MHz TMDS) and returning to the
1080p60 GUI sometimes leaves the display showing "no signal". In that state:

- every dw-hdmi register read, the AVI infoframe and the mode are identical to a working
  1080p60 state;
- the VOP2 is scanning out (the `SYS_STATUS0` scan line moves);
- the sink's SCDC reports TMDS bit clock ratio 1/10, scrambling off, clock and all channels
  locked, and 0 character errors.

Mainline `dw_hdmi_bridge_atomic_disable()` calls `drm_connector_hdmi_disable_scrambling()`,
which writes the sink's SCDC scrambling and TMDS_Bit_Clock_Ratio bits, before
`dw_hdmi_poweroff()`. The old scrambled signal is still on the wire at that point. HDMI 2.0
requires TMDS clock and data to be suspended while the clock ratio changes. The Rockchip 6.6
driver never touches sink SCDC in disable; it updates it from `hdmi_av_composer()` during the
next setup, with the PHY off, followed by a TMDS software reset.

Candidate fix: "drm/bridge: dw-hdmi: disable sink scrambling after the PHY is off" powers the
PHY off first. dw-hdmi-qp has the same ordering problem; deferring the disable to
`atomic_enable()` fixed an AVR freeze on RK3588 scrambled to unscrambled switches. With the fix,
stopping 2160p HDR playback has not lost sync since, but the fault was intermittent.

Other sequencing differences in the Rockchip 6.6 dw-hdmi driver, not ported:

| Vendor behaviour | Applies when |
|---|---|
| Sets AVMUTE, waits 180 ms, reconfigures without a full modeset, waits 50 ms, clears AVMUTE | Pixel and TMDS clocks are unchanged |
| Forces a test pattern (`FC_DBGFORCE`) for 50 ms after each full modeset (Konka TV workaround) | Every full modeset |
| Enables the DRM (HDR) infoframe 300 ms after the AVI infoframe (Huawei TV workaround) | HDR output |

## 5. Silent passthrough at start: a second modeset

Passthrough sometimes started silent, with the AVR reporting PCM, while every transmitter
register was correct for IEC 61937. On a start with refresh rate switching, Kodi changes the
mode, then about 1 to 1.5 s later the renderer configures and sets the HDR metadata again.
Passthrough opens at about the same time.

Kodi creates a new `HDR_OUTPUT_METADATA` blob on every call, and the branch's HDMI state helper
compared the old and new blobs by pointer (from a WIP HDR patch), so identical metadata forced a
full modeset. When it landed after the bitstream had started, the sink never detected it.
dw-hdmi's own check, and i915, amdgpu and meson, compare the contents with
`drm_connector_atomic_hdr_metadata_equal()`.

Fix: a `fixup!` of that WIP patch compares by content. Kodi PR #29744 also sends the colorspace
and HDR metadata with the refresh-rate modeset.

## 6. Passthrough at 48 kHz: empty ELD during a modeset

After that fix, E-AC3 sometimes still started silent with heavy stutter: the device ran at 48 kHz
although Kodi requested 192 kHz, and dmesg showed `hdmi-audio-codec: HDMI: Unknown ELD version 0`
at the moment the stream opened.

`hdmi_codec_startup()` turns the ELD into rate constraints. With no SADs, `eld_limit_rates()`
allows only 32, 44.1 and 48 kHz. dw-hdmi's `hdmi_audio_get_eld()` returns the ELD of
`curr_conn`, which `atomic_disable()` clears and `atomic_enable()` sets again, so a stream opened
during a modeset sees an empty ELD. This happens when the renderer's update is a real modeset,
for example when the HDR metadata in the decoded frames differs from the container's. Mainline
dw-hdmi reads the ELD the same way.

Fix: "drm/bridge: dw-hdmi: keep the audio ELD across a modeset" reads the ELD from the connector
set by the last `atomic_enable()`, which is kept while the bridge is disabled. A real unplug
still clears the ELD in the connector.

## 7. 10-bit output at 2160p50/60

2160p50/60 RGB at 10 bpc needs a TMDS character rate of 742.5 MHz, above the HDMI 2.0 limit, so
the HDMI state helper's automatic format selection fell back to 8 bpc RGB. It tries YCbCr 4:2:0
only when RGB fails entirely, although the sink here advertises 10 and 12-bit 4:2:0 for VIC
96/97 and the RK3568 platform data includes YCbCr 4:2:0.

- "drm/display: hdmi-state-helper: prefer YCbCr 4:2:0 for higher bpc": when RGB only fits below
  the maximum bpc, also try YCbCr 4:2:0 and use it if it carries more bits per component. Modes
  where RGB reaches the maximum bpc, such as 1080p60 and 2160p24 at 10 bpc, are unchanged.
- "drm/rockchip: vop2: keep 10-bit output for 10-bit bus formats": `vop2_dither_setup()` enables
  pre-dither-down (10 to 8 bits, `VP_DSP_CTRL` bit 16) for every output mode other than AAAA,
  including the 10-bit 4:2:0 bus `UYYVYY10_0_5X30`. The vendor driver leaves it off for all
  10-bit bus formats. This also applies to RK3588 4:2:0 output.

Result at 2160p50: `output_format` YCbCr 4:2:0, `output_bpc` 10, dw-hdmi `TX_INVID0` 0x0b and
`VP_PR_CD` 0x50, `VP_DSP_CTRL` 0x800E. The 8 bpc RGB path still truncates without dithering;
mainline has no RGB888 case in `vop2_dither_setup()`, while the vendor driver pre-dithers it.

## 8. Open: multichannel PCM channel placement

Stereo PCM is heard on FC and FR instead of FL and FR, and 5.1 PCM has no FL. The transmitter
metadata is correct: Kodi sends `FL FR` (or `FL FR LFE FC RL RR`), the audio infoframe carries
the matching channel count and allocation and is sent automatically, the sample layout matches,
and the channel numbers in the channel status are 0 (unspecified). The remaining suspect is the
sample data alignment on I2S: the vendor `rockchip_i2s_tdm` driver notes that the HDMI controller
ignores the first frame-sync cycle and writes one padding frame before starting transmission,
but only enables that for the RK3588 HDMI I2S. A per-channel `speaker-test` is pending.


## Cover letter notes

- **Other RK33XX chips:** the same IEC958 to BPCUV translation is ported to `rockchip_i2s` for
  the older chips that feed dw-hdmi from it. It is enabled only for `rk3328-i2s` and
  `rk3399-i2s` (dw-hdmi v2.11a), and the LibreELEC alsa conf routes "RK3328 HDMI" and
  "RK3399 HDMI" the same way as RK356X. The change probably extends to other RK32XX/RK33XX chips with
  dw-hdmi on `rockchip_i2s`, such as RK3228/RK3229 and RK3288. These are untested, so they are
  left out. Enabling one is a match-data flag plus the alsa conf mapping, and is a follow-up
  for someone with hardware to test on. RK3288 needs a check first: it has dw-hdmi v2.00a,
  which predates the insert_pcuv control, and NLPCM/HBR support there is unconfirmed.
- **RK3399 DisplayPort:** cdn-dp also registers hdmi-codec, and the translation is keyed on an
  hdmi-codec link, not on dw-hdmi. The Gru Chromebooks (`rk3399-gru-sound`, `i2s0`) link
  cdn-dp, so IEC958 subframes to DP there would be rewritten as well. Before posting, restrict
  the check to a dw-hdmi codec, or state why it cannot happen.
