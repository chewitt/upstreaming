# Mainline meson DRM modernisation

Status: research, no patches posted yet.
Base reviewed: Linux 7.3-rc5+ (`c03c1a14507d`), upstream code only (no out-of-tree patches).
Scope: `drivers/gpu/drm/meson/` and the meson glue to the Synopsys dw-hdmi bridge.

## Why

Mainline meson DRM has had little proactive maintenance and has drifted from the
baseline that DRM core developers assume when they write tree-wide conversions.
Series that touch per-bridge code still reach meson, but series that touch driver
structure or the HDMI connector framework convert rockchip, sun4i, vc4, mediatek,
msm, imx and intel and skip meson. Bringing meson back to the expected baseline
means future core series include it naturally, and gives a clean base for new SoC
support (S4 is not in upstream meson DRM yet).

## Current state

- Activity since 2023-06: 57 non-merge meson commits (rockchip 342, mediatek 362,
  vc4 174, sun4i 99, tidss 84). About 20 are meson-specific; the rest are tree-wide
  sweeps. MAINTAINERS lists the driver as "Supported".
- Sweeps that reached meson: atomic_create_state for bridges, next-bridge get/put,
  `of_drm_find_and_get_bridge`.
- Sweeps that skipped meson:
  - the `drm_mode_size_dumb()` series (rockchip converted; meson still open-codes
    `meson_dumb_create`, `meson_drv.c:83-93`);
  - "remove dependency on DRM simple helpers" (rockchip, sun4i, exynos, gma500, qxl
    converted; meson still uses `drm_simple_encoder_init` for its three encoders);
  - managed KMS polling to fix use-after-free on unbind (vc4 `073a30d75f30`);
  - the HDMI connector framework (vc4, rockchip dw-hdmi-qp, mediatek, msm use it).
- Structure:
  - component master only because the dw-hdmi glue reads the HHI regmap through
    `drm->dev_private` (`meson_drv.c:463-468` TOFIX, `meson_dw_hdmi.c:676`);
  - deprecated `drm_dev_alloc` (`meson_drv.c:201`) plus `dev_private`;
  - CRTC and planes `devm_kzalloc`'d with non-managed init (`meson_crtc.c`,
    `meson_plane.c`, `meson_overlay.c`; the overlay ignores the init return value);
  - unmanaged `drm_kms_helper_poll_init`;
  - no `drm_dev_unplug`/`drm_dev_enter`, no debugfs, no runtime PM, no drm_panic.
- Already modern: `drmm_mode_config_init`, `DRM_FBDEV_DMA_DRIVER_OPS` +
  `drm_client_setup`, aperture helper, `drm_mode_config_helper_suspend/resume`,
  `drm_bridge_connector` with `DRM_BRIDGE_ATTACH_NO_CONNECTOR`.

## Bugs found

1. `meson_drv_bind_master()` error path calls `drm_dev_put(drm)` before the encoder
   removals and `component_unbind_all(dev, drm)`, which still use `drm`.
2. `meson_drv_unbind()` frees the canvases before `drm_dev_unregister()` and
   `drm_atomic_helper_shutdown()`, so they can be freed while still scanned out, and
   drops the device before removing the bridges and unbinding components.
3. `meson_encoder_hdmi_atomic_check()` stores `output_bus_fmt` in the encoder object
   instead of atomic state, so a TEST_ONLY commit can change what the next enable
   uses.
4. `of_find_device_by_node()` in `meson_encoder_hdmi_init()` appears to keep its
   device reference on the success path (reported, not yet confirmed).

1-3 were checked against the source; 4 needs confirming before a patch.

## todo.rst items that apply

- Open-code `drm_simple_encoder_init`.
- Object lifetime fixes (drmm_* for CRTC/planes/encoders).
- Convert logging to `drm_*` (meson has no `drm_err`/`drm_dbg` calls; ask the
  maintainer first, as todo.rst requires).
- Review `max_width`/`max_height` (hard-coded 3840x2160 scanout limit).
- debugfs.
- drm_panic support.
- Replace the deprecated `drm_dev_alloc`.

Also: the CEC FIXME in `meson_encoder_hdmi.c` re-reads the EDID in `hpd_notify`
only to get the physical address.

## Plan

| # | Area | Modern helper / reference | Effort | Risk | Order |
|---|---|---|---|---|---|
| 1 | Bind/unbind order fixes | teardown as in vc4 (`drm_dev_unplug` then `drm_atomic_helper_shutdown`) | S | Low | first, Cc stable |
| 2 | Bus format in encoder object | read `bridge_state->output_bus_cfg.format` in enable | S | Low | early |
| 3 | Dumb buffer size | `drm_mode_size_dumb(dev, args, SZ_64, 0)` (rockchip) | S | Low | any |
| 4 | Simple encoder helper | own `drm_encoder_funcs` / `drmm_encoder_init` | S | Low | before 6 |
| 5 | Logging | `drm_dbg_kms` / `drm_err` | S-M | Nil | any, ask first |
| 6 | Managed device and objects | `devm_drm_dev_alloc`, `drmm_crtc_init_with_planes`, `drmm_universal_plane_alloc`, `drmm_kms_helper_poll_init`, devm action for `component_unbind_all`; drop `dev_private` | M | Medium | after 1, 4 |
| 7 | Hot-unplug safety | `drm_dev_enter/exit` around MMIO and IRQ | M | Low | after 6 |
| 8 | YUV420 flag | `bridge.ycbcr_420_allowed` on the encoder bridge instead of forcing the connector flag | S | Low | any |
| 9 | Remove components | standalone dw-hdmi glue bridge, HHI via syscon/regmap | L | High | after 6, needs RFC |
| 10 | HDMI connector framework | `OP_HDMI` via dw-hdmi; TMDS limits through a platform `tmds_char_rate_valid`; drop manual max bpc / HDR property attach | M-L | High | with the dw-hdmi conversion |
| 11 | CEC | `OP_HDMI_CEC_NOTIFIER`; delete the EDID re-read | S-M | Low-medium | needs 10 |
| 12 | HDMI audio | `OP_HDMI_AUDIO` / `drm_connector_hdmi_audio_init`, keeping the dw-hdmi device as `hdmi_audio_dev` | S (meson side) | Medium | with dw-hdmi work |
| 13 | Kconfig | drop `select CEC_CORE if CEC_NOTIFIER` after 11 | S | Nil | with 11 |
| 14 | Plane properties | COLOR_ENCODING/RANGE on the overlay, alpha/blend on OSD | M | Medium | independent |
| 15 | CRTC colour management | `drm_crtc_enable_color_mgmt` (the OSD matrix and LUT code exists) | M-L | Medium | later |
| 16 | drm_panic | `drm_fb_dma_get_scanout_buffer` on the OSD plane | S-M | Low | after 6 |
| 17 | debugfs | `drm_debugfs_add_file` for VPU/VENC/VCLK state | S | Nil | optional |
| 18 | max_width/height | derive from pitch/canvas limits | S | Low | any |

## Series

- **A, fixes:** 1, 2, 8 and the reference leak (once confirmed). Bind/unbind test on
  one GX and one G12 board. Suitable for stable.
- **B, missed sweeps:** 3, 4, 5, 18 and 13 where independent. Compile and smoke test.
- **C, managed lifetime:** 6 then 7, about 4-6 patches. Bind/unbind and module
  reload on GXBB, GXL, GXM and G12A/SM1, including CVBS-only and DSI setups.
- **D, de-componentise the dw-hdmi glue:** 9. RFC with the maintainer first; may
  need a binding change. Full HDMI regression testing, including suspend/resume.
- **E, HDMI framework glue:** 10, 11, 12, together with the dw-hdmi `OP_HDMI`
  conversion (see `dw-hdmi-helper-evaluation.md`). Heaviest testing: 4K60 YUV420,
  10-bit on GXM/G12, HDR without a modeset, HBR passthrough, CEC on ao-cec and
  ao-cec-g12a.
- **F, features:** 14, 15, 16, 17 as separate small sets.

S4 DRM enablement should be rebased onto the result of C/D/E rather than going in
first.

## Open questions

- Can the HHI regmap come from a syscon so the component framework can go? (D)
- Default output format once the HDMI state helper is used: it picks RGB first with
  a YUV420 fallback only, so 4K60 on an HDMI 2.0 sink becomes 8-bit RGB at 594 MHz
  instead of today's 10-bit 4:2:0. (E)
- AFBC handling for drm_panic. (16)
- Deprioritised: CVBS has no TV-mode property, and `drm_bridge_connector` has no
  TV-mode support to build on (core work). `commit_tail_rpm` is used although the
  driver has no runtime PM.
