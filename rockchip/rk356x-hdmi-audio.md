# RK3566/RK3568 dw-hdmi audio passthrough and HBR (notes)

Status: working on Rock 3C (RK3566) with branch `rockchip-7.3.y`. E-AC3, Dolby TrueHD and
DTS-HD MA pass through cleanly. One display sync-loss fix is still untested.

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

## 4. Lost sync after a scrambled to unscrambled modeset (fix untested)

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
`atomic_enable()` fixed an AVR freeze on RK3588 scrambled to unscrambled switches.

Other sequencing differences in the Rockchip 6.6 dw-hdmi driver, not ported:

| Vendor behaviour | Applies when |
|---|---|
| Sets AVMUTE, waits 180 ms, reconfigures without a full modeset, waits 50 ms, clears AVMUTE | Pixel and TMDS clocks are unchanged |
| Forces a test pattern (`FC_DBGFORCE`) for 50 ms after each full modeset (Konka TV workaround) | Every full modeset |
| Enables the DRM (HDR) infoframe 300 ms after the AVI infoframe (Huawei TV workaround) | HDR output |
