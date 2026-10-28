# RK3399 DDR rate experiment for 4K 10-bit playback (scope)

Status: PARKED 2026-10-28. The MSCH read latency change (u-boot) is kept; the DMC
experiment was dropped after the results below (TF-A patches and the ROC-PC DT node removed). Board: Firefly ROC-RK3399-PC (LPDDR4), LibreELEC with
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

## Result 1: hard hang on the first switch (2026-10-28)

With the DMC node enabled, `modprobe rk3399_dmc` returns and about a second later the
board hangs without any kernel or TF-A output; u-boot then reports `Reset cause: WDOG`.
`simple_ondemand` requests 856 MHz straight away, so the first DFS call is the suspect.

### Why: controller and PHY frequency sets are paired differently

The Denali controller (CTL) and PHY each keep several frequency copies. The controller copy
in use is `CTL_REG(ch, 111)` bits 17:16. `DENALI_CTL_19` gives the DFI bus frequency each
controller copy drives (`DFIBUS_FREQ_F0/F1/F2`), which selects the PHY copy.
`PHY_REG(ch, 896)` bits 9:8 (`PHY_FREQ_SEL_INDEX`) only choose which PHY copy register
writes update (TRM Part 1, DENALI_PHY_896).

| Loader | `DENALI_CTL_19` (F0/F1/F2) | Controller copy -> PHY copy |
|---|---|---|
| coreboot (gru, LPDDR3) | 0x00020100 (0/1/2) | 0->0, 1->1 |
| mainline u-boot LPDDR4 | 0x00000102 (2/1/0) | 0->1, 1->0, 2->3 (`lpddr4_phy_fn` = {1, 0, 0xb}) |

The u-boot TPL trains at 400 MHz on controller copy 0 / PHY copy 1 and finishes at 800 MHz
on controller copy 1 / PHY copy 0 (`set_rate_index(dram, params, 1)`).

TF-A `dfs.c` assumes controller copy n uses PHY copy n. `dram_dfs_init()` reads the
current controller copy (1); `prepare_ddr_timing()` picks the other one (0) and calls
`gen_rk3399_phy_params(..., 0)`, which writes PHY copy 0, the copy in use. Memory access
breaks before the M0 even starts the switch.

The DFI-frequency-to-PHY-copy rule itself is not documented in the TRM, and the coreboot
and u-boot pairings for DFI value 1 disagree (PHY 1 vs PHY 0), so it may depend on DRAM
type. TF-A cannot derive the PHY copy from `DENALI_CTL_19` alone with what is known.

### TF-A fix (proposal)

Keep a controller-copy -> PHY-copy table in `rk3399_dram_status`, filled in
`dram_dfs_init()`: identity by default, and {1, 0} when the DRAM is LPDDR4 and
`DENALI_CTL_19` holds the mainline u-boot value 0x00000102. Use it wherever DFS passes the
controller index to PHY code:

- `prepare_ddr_timing()`: `gen_rk3399_phy_params(..., phy_index(index))` (this also covers
  the DLL bypass setup inside it);
- `dram_dfs_init()`: `PHY_FREQ_SEL_INDEX` for the current copy.

The controller side (CTL/PI timings, CIC `PARAM_FREQ_SELECT`, trefi per copy) stays on the
controller index. Suspend/resume (`suspend.c`) also writes `PHY_FREQ_SEL_INDEX` and needs
the same mapping before it could be used; LibreELEC does not suspend.

Check first, from the u-boot console (stop autoboot): `md.l 0xffa8004c 1` (DENALI_CTL_19)
and `md.l 0xffa801bc 1` (DENALI_CTL_111, bits 17:16 = current copy) should read 0x102 and
copy 1 on the ROC-PC. Channel 1 is at 0xffa88000.

## Result 2: TF-A PHY copy mapping (2026-10-28)

The u-boot console confirmed DENALI_CTL_19 = 0x00000102 and DENALI_CTL_111 = 0x01010400
(controller copy 1). With the controller-to-PHY copy table in TF-A (PHY copy 1 written for
controller copy 0), the switch no longer locks up at once: the system keeps running with
corrupt memory (HDMI audio DMA played noise), then the watchdog resets it. Remaining per-copy
state that TF-A does not reconcile with the mainline u-boot setup: PI frequency copies, LPDDR4
FSP-OP/FSP-WR mode register state, and u-boot 400 MHz values left in controller copy 0 that
TF-A does not regenerate. Next diagnostic if resumed: switch 800 -> 400 MHz (a rate the TPL
trained) to test TF-A DFS after mainline u-boot independently of 856 MHz.
