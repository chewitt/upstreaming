# upstreaming
Patch tracking for Amlogic and Rockchip upstreaming efforts

## Amlogic

- [Mainline meson DRM modernisation](amlogic/drm-meson-modernisation.md): audit of
  upstream meson DRM against current DRM helpers and todo.rst, with a series plan.
- [dw-hdmi and meson HDMI patches vs the HDMI helper framework](amlogic/dw-hdmi-helper-evaluation.md):
  how the out-of-tree Amlogic HDMI patches map onto the HDMI connector/bridge helpers.
- [AV1 hardware decode on S4](amlogic/s4-av1-decode.md): what the vendor AV1 decoder does
  and what amvdec needs to drive it through the stateless V4L2 AV1 API.

## Rockchip

- [RK3576 VDPU383 decode performance](rockchip/rk3576-vdpu383-performance.md): cover-letter
  notes for the stride and read cache patches (20 to 86 fps 4K60 H.264).
- [RK3566/RK3568 HDMI audio passthrough and HBR](rockchip/rk356x-hdmi-audio.md): dw-hdmi
  IEC958/NLPCM/HBR over I2S, the hw_params ordering and keep-out fixes, modeset timing at
  playback start (HDR metadata comparison, ELD during a modeset), scrambling order, and 10-bit
  2160p50/60 output.
