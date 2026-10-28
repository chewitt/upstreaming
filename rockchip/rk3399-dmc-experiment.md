# RK3399 DDR rate experiment for 4K 10-bit playback (scope)

Status: scope only, no code. Board: Firefly ROC-RK3399-PC (LPDDR4), LibreELEC with
mainline u-boot TPL, upstream TF-A 2.15 and `rockchip-7.3.y`.

## Problem

With the Kodi OSD shown, 4K 10-bit HEVC at 50/60 fps shows flickering black lines
(VOP `POST_BUF_EMPTY` underflow); 59.94 is worse than 50 and turns green. The same
mode and output (3840x2160, YCbCr 4:2:0 10 bpc) is clean with 8-bit HEVC or H.264
at 59.94/60, with 10-bit at 25 fps, and with 10-bit paused. HDR is not a factor
(10-bit SDR fails too). The fault needs 10-bit decode, an NV15 video plane and the
GUI plane at the same time, which points at DDR bandwidth.

Ruled out earlier: VOP QoS (already priority 3/3), VOP AXI outstanding, rkvdec read
cache (already on), GUI resolution (720p GUI still fails), AFBC on the GUI plane.
The vendor VOP driver sets up 10-bit windows the same way mainline does (`fmt_10`,
line-buffer mode, AXI outstanding 30).

## What the vendor kernel does differently

DDR runs at a fixed 800 MHz on the board (`sclk_ddrc` 800 MHz; mainline TPL
`rk3399-sdram-lpddr4-100.dtsi`); there is no DMC devfreq node. The vendor kernel
(`rockchip/develop-6.6`, `drivers/devfreq/rockchip_dmc.c`) raises DDR for 4K 10-bit
video through `system-status-freq`:

| Board (vendor DT) | DRAM | NORMAL | VIDEO_4K | VIDEO_4K_10B |
|---|---|---|---|---|
| rk3399-evb, excavator-sapphire, videostrong, mid-818 | DDR3/LPDDR3 | 800 | 600 | 800 |
| rk3399-sapphire-excavator-lp4, rk3399pro-evb-lp4 | LPDDR4 | 856 | 856 | 856 |
| rk3399-tve1030g | LPDDR4 | 856 | 666 | 856 |

LPDDR4 boards list a 928 MHz OPP but mark it `status = "disabled"`; 856 MHz is the
highest LPDDR4 rate Rockchip ships. The vendor ROC-PC DT is the mainline copy and
has no DMC node.

The vendor DMC node also has `vop-pn-msch-readlatency` (MSCH read latency by number
of active VOP planes, set through SIP `ROCKCHIP_SIP_CONFIG_DRAM_SET_MSCH_RL` = 0x0a).
The two vendor DTs that set it (rk3399-sapphire-excavator-lp4, rk3399-evb-ind) use
0x20 for every plane count. Mainline u-boot programs
`ddrtiminga0` = 0x80241d22, i.e. read latency 0x80. TF-A 2.15 does not implement the
0x0a SIP call and its DFS code does not change the read latency.

## What mainline already has

- **Kernel:** `drivers/devfreq/rk3399_dmc.c` and `devfreq/event/rockchip-dfi.c`; the
  `dmc` and `dfi` nodes are in `rk3399-base.dtsi` (dmc disabled). Only gru and op1
  enable it. LPDDR4 is handled (`rockchip,lpddr4_odt_dis_freq`; DRAM type comes from
  PMUGRF OS_REG2). All tuning properties are optional. Required: `center-supply`,
  `operating-points-v2`. LE config has `CONFIG_ARM_RK3399_DMC_DEVFREQ=m`,
  `CONFIG_DEVFREQ_EVENT_ROCKCHIP_DFI=y` and the userspace/performance governors.
- **TF-A 2.15:** `plat/rockchip/rk3399/drivers/dram/` (dram.c, dfs.c,
  dram_spec_timing.c with LPDDR4 timings). `dram_init()` reads the DRAM geometry
  from PMUGRF OS_REG2 (written by the u-boot TPL), `dram_dfs_init()` takes the boot
  rate from the DPLL. SIP calls: SET_RATE, ROUND_RATE, GET_RATE, SET_ODT_PD.
  `ddr_set_rate()` regenerates controller/PI/PHY timings and retrains through the M0.
- **TF-A DPLL table:** 928, 800, 732, 666, 600, 528, 400, 300, 200 MHz. A request is
  rounded down to the next entry, so 856 MHz becomes 800 MHz. 856 needs a new entry:
  `{.mhz = 856, .refdiv = 1, .fbdiv = 107, .postdiv1 = 3, .postdiv2 = 1}`
  (24 MHz x 107 / 3).
- **Regulator:** ROC-PC `vdd_center` is RK808 DCDC_REG1, 0.75-1.35 V. The op1/gru OPPs
  use 0.900 V up to 800 MHz and 0.925 V at 928 MHz; vendor LPDDR4 boards use 0.900 V
  at 856 MHz.

## Experiment

1. **TF-A (LE patch):** add the 856 MHz DPLL entry. Nothing else changes.
2. **Kernel DT (WIP, ROC-PC only):** enable `&dmc` with `center-supply = <&vdd_center>`,
   an OPP table 400/666/800/856 MHz at 0.900 V, and `rockchip,lpddr4_odt_dis_freq`
   (gru uses 666 MHz). Leave the idle/ODT/drive properties at driver defaults at first.
3. **Run time:** select the `userspace` governor and set 800 then 856 MHz by hand
   (the DMC entry under `/sys/class/devfreq/`), checking `sclk_ddrc` and the DPLL registers after each
   step. Stress memory at 856 MHz before any video test.
4. **Video test:** Samsung "Ride on Board" (HEVC 10-bit SDR 59.94) and Planet Earth III
   HLG 4K50 with the OSD up, at 800 and at 856 MHz.

Separate, cheaper experiment (one variable at a time): set the MSCH read latency to the
vendor value 0x20 by changing `ddrtiminga0` 0x80241d22 -> 0x20241d22 in the u-boot
LPDDR4 params, with DDR left at 800 MHz.

## Risks

- 856 MHz on this board's LPDDR4 is untested in mainline; TF-A's LPDDR4 DFS path is
  less exercised than LPDDR3 (gru). Failure modes: hang during the switch, or silent
  memory corruption at the new rate. Memory stress testing comes first.
- ODT/drive-strength defaults in the mainline driver may not suit this board's LPDDR4
  layout. The vendor LPDDR4 DTs carry board values that can be compared.
- Bandwidth gain is about 7% (800 -> 856). That may not be enough; if it isn't, DDR
  rate is not the whole story.
- The devfreq `simple_ondemand` governor (DFI load) would also lower the rate when idle;
  keep the experiment on `userspace`/`performance` until the effect on video is known.
