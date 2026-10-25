# RK3568/RK3566 DMC devfreq through the vendor BL31 (research notes)

Status: research only, no code. The goal is a mainline devfreq driver that scales
DDR on RK3566/RK3568 through the Rockchip vendor BL31 (rkbin), so 4K decode is not
held at the DDR rate the loader leaves behind.

References used:

- vendor kernel `rockchip/develop-6.6`: `drivers/devfreq/rockchip_dmc.c`,
  `drivers/devfreq/rockchip_dmc_common.c`, `drivers/firmware/rockchip_sip.c`,
  `include/linux/rockchip/rockchip_sip.h`, `include/soc/rockchip/rockchip_sip.h`,
  `include/soc/rockchip/rockchip_dmc.h`, `drivers/clk/rockchip/clk-ddr.c`,
  `arch/arm64/boot/dts/rockchip/rk356x.dtsi`, `rk3568.dtsi`, `rk3566.dtsi` and
  `rk3568-dram-default-timing.dtsi`
- mainline (`torvalds/master`, `rockchip-7.3.y`): `drivers/devfreq/rk3399_dmc.c`,
  `drivers/devfreq/event/rockchip-dfi.c`, `include/soc/rockchip/rockchip_sip.h`,
  `drivers/pmdomain/rockchip/pm-domains.c`, `arch/arm64/boot/dts/rockchip/rk356x-base.dtsi`
- rkbin `74213af1e952` (the linux-6.1-stan-rkr6 tag that LibreELEC ships): `RKTRUST/RK3568TRUST.ini`
  selects `bin/rk35/rk3568_bl31_v1.45.elf`
  (`v2.3-896-g70d3deb59 ... fwver: v1.45`)
- upstream TF-A master (Oct 2026)

All `rockchip_dmc.c` line numbers below refer to `rockchip/develop-6.6`.

## 0. Check first: is there any headroom above the boot rate?

The DDR loader (TPL `ddr.bin`) trains a set of frequency set points (FSPs). BL31
reads those points back from the loader: the BL31 strings include
`ddr dmc_fsp already initialized in loader.`, `dfs DDR fsp_param[%d].freq_mhz= %dMHz`,
and `loader&trust unmatch!!! Please update loader/trust if need enable dmc`. The kernel
can only choose between these points (section 3). `ddrbin_tool.py -g` decodes the two
loaders that rkbin `74213af1e952` uses:

| loader blob | F0 (boot/final) | F1/F2/F3 |
|---|---|---|
| `rk3568_ddr_1560MHz_v1.23.bin` | DDR4/LP4/LP4X 1560, DDR3/LP3 1056 | 324/528/780 |
| `rk3566_ddr_1056MHz_v1.23.bin` | 1056 for every DRAM type | 324/528/780 |

LibreELEC `packages/tools/rkbin/package.mk` copies `RK3566MINIALL.ini` to
`RK356XMINIALL.ini` and uses the per-SoC ini when `uboot_helper` names a SoC. As a
result, rock-3b (RK3568) gets the 1560 MHz loader and rock-3c (RK3566) gets the
1056 MHz loader.

Consequences:

- A board that sits at 1056 MHz with the RK3566 loader is already at its top FSP.
  A DMC driver would only add 324, 528 and 780 MHz, which saves power but adds no
  bandwidth. Vendor `rk3566.dtsi:12-20` also caps RK3566 LPDDR4/4X `freq_0` at 1056,
  and rkbin ships no RK3566 loader above 1056 MHz. Treat 1056 MHz as the RK3566 ceiling.
- On RK3568 with LPDDR4/DDR4 and the 1560 MHz loader, DDR should boot at 1560 (F0 is the
  "final" frequency). If an RK3568 board reports `clk_scmi_ddr` at 1056 MHz, check which
  loader it boots and which DRAM type it has before writing any driver. Look at the TPL
  banner on the UART (DDR type and MHz), and at BL31's `fsp_param[]` print if it appears.
- If the board is RK3568 at 1056 because of the loader, the fastest fix is the
  1560 MHz loader, with no kernel driver at all. The driver is then only needed to drop
  the rate when idle (and to boost again).

Sections 1 to 6 assume a loader with at least one FSP above the current rate.

## 1. SMC protocol

### Function IDs (vendor `include/linux/rockchip/rockchip_sip.h:19-63`)

| ID | name | use |
|---|---|---|
| `0x82000008` | `SIP_DRAM_CONFIG` (= `ROCKCHIP_SIP_DRAM_FREQ` in `include/soc/rockchip/rockchip_sip.h:9`) | all DDR commands |
| `0x82000009` | `SIP_SHARE_MEM` | allocate the BL31/kernel shared pages |
| `0x82000010` | `SIP_SCMI_AGENT0` | SCMI doorbell (mainline already uses it, `rk356x-base.dtsi:137-147`) |

`sip_smc_dram(arg0, arg1, arg2)` is
`arm_smccc_smc(0x82000008, arg0, arg1, arg2, 0, 0, 0, 0)`
(`drivers/firmware/rockchip_sip.c:38-53`). In SMCCC terms: x1 = arg0 (share-page type,
`SHARE_PAGE_TYPE_DDR` = 2, or 0), x2 = arg1 (always 0 on RK3568), x3 = arg2 (the
sub-command). The register layout is the same as rk3399's
`arm_smccc_smc(ROCKCHIP_SIP_DRAM_FREQ, param, index, cmd)`. Only the meaning of x1
differs.

`sip_smc_request_share_mem(pages, type)` is `SMC(0x82000009, pages, type, 0)`. It returns
a0 = 0 on success and a1 = the **physical** address of the pages
(`rockchip_sip.c:168-184`). The vendor code then maps the pages: `sip_map()`
(`rockchip_sip.c:130-166`) uses `ioremap()` when the address is not kernel RAM, and a
non-cached `vmap()` when it is. Mainline U-Boot starts DRAM bank 0 at 2 MiB
(`arch/arm/mach-rockchip/sdram.c:321-322`, "Reserve 2M for ATF bl31"), so the page
should sit outside kernel RAM and need `ioremap()` or `memremap()`. Confirm by
printing a1 on hardware.

### DRAM sub-commands (x3)

From vendor `include/soc/rockchip/rockchip_sip.h:10-28`:

| value | name | used on RK3568 |
|---|---|---|
| 0x00 | `DRAM_INIT` | yes, once at probe |
| 0x01 | `DRAM_SET_RATE` | yes |
| 0x02 | `DRAM_ROUND_RATE` | no |
| 0x03 | `DRAM_SET_AT_SR` | `set_auto_self_refresh`, only on system-status changes (suspend) |
| 0x05 | `DRAM_GET_RATE` | not by the RK3568 path; it reads the SCMI clock instead |
| 0x08 | `DRAM_GET_VERSION` | yes. **Mainline `include/soc/rockchip/rockchip_sip.h:21` names 0x08 `SET_ODT_PD` (rk3399), so the header needs a second name for the same value** |
| 0x09 | `DRAM_POST_SET_RATE` | yes, from the complete IRQ and on timeout |
| 0x0c | `MCU_START` | yes, with x1 = 0 |
| 0x0e | `DRAM_GET_FREQ_INFO` | yes |
| 0x11 | `DRAM_GET_STALL_TIME` | RK3588 only |

Return convention (vendor `rockchip_sip.h:86-92`): a0 = 0 means success, otherwise a
negative `SIP_RET_*` (-1 unknown SMC, -2 not supported, -3 invalid parameters,
-4 invalid address, -5 denied, -6 `SIP_RET_SET_RATE_TIMEOUT`). SET_RATE returns the
"timeout" code in **a1**, not a0 (`rockchip_dmc.c:353`). GET_VERSION returns the
version in a1.

The BL31 strings also show `ddr_plat_smc_handler` and `%s: unhandled DDR SMC id(0x%llx)`.
The command set is a superset of the above (ECC, addrmap, ddrdbg).

### Shared pages

`rk3568_dmc_init()` (`rockchip_dmc.c:1878-1891`) requests 2 pages of type
`SHARE_PAGE_TYPE_DDR` (= 2; enum at vendor `include/linux/rockchip/rockchip_sip.h:192-204`).
The comment says page 0 holds the interface parameters and page 1 the "dts
parameters". The code then clears both pages (`memset_io(..., 0, 8192)`, line 1891)
and **writes nothing to page 1 on RK3568**. The `of_get_*_timings()` and de-skew
helpers that fill page 1 (`DTS_PAR_OFFSET`, lines 661-1091) belong to the
px30/rk1808/rk3128/rk3228/rk3288/rk3328/rv1126 init paths only. On RK3568 the
per-board DRAM parameters (drive strength, ODT, FSP list) come from the loader,
which is what BL31's "dmc_fsp already initialized in loader" refers to. The vendor
`dmc-fsp` DT node (`rk356x.dtsi:2298-2309`) and `rk3568-dram-default-timing.dtsi`
feed vendor U-Boot's fallback path. No kernel driver reads them, and mainline needs
neither.

Page 0 layout, `struct share_params` (`rockchip_dmc.c:70-98`, u32 fields at these
byte offsets):

| off | field | RK3568 kernel use |
|---|---|---|
| 0x00 | `hz` | target rate in **Hz**, written before SET_RATE |
| 0x04 | `lcdc_type` | `SCREEN_*` of the active connector (HDMI = 6, NULL = 0; `soc/rockchip/rockchip_dmc.h:19-32`) |
| 0x08 | `vop` | unused on RK3568 |
| 0x0c | `vop_dclk_mode` | unused on RK3568 |
| 0x10 | `sr_idle_en` | used by `DRAM_SET_AT_SR` |
| 0x14 | `addr_mcu_el3` | unused |
| 0x18 | `wait_flag1` | set to 1 before every SET_RATE |
| 0x1c | `wait_flag0` | set to 1 before every SET_RATE |
| 0x20 | `complt_hwirq` | left 0 on RK3568 |
| 0x24 | `update_drv_odt_cfg` | 0 |
| 0x28 | `update_deskew_cfg` | 0 |
| 0x2c | `freq_count` | **output** of GET_FREQ_INFO |
| 0x30 | `freq_info_mhz[6]` | **output** of GET_FREQ_INFO, MHz |
| 0x48 | `wait_mode` | RK3588 only (`wait-mode` DT property) |
| 0x4c | `vop_scan_line_time_ns` | written before SET_RATE (line duration of the active CRTC) |

Before DRAM_INIT the kernel only has to clear the pages. Before each SET_RATE it
writes `hz`, `lcdc_type`, `vop_scan_line_time_ns`, `wait_flag0` = 1 and `wait_flag1` = 1
(`rockchip_ddr_set_rate()`, lines 340-357; also lines 483-487).

### Init sequence (`rk3568_dmc_init()`, lines 1864-1934)

1. `SMC(0x82000008, 0, 0, GET_VERSION)`: requires a0 == 0 and a1 >= 0x101, otherwise
   "trusted firmware need update to V1.01".
2. `SMC(0x82000009, 2, SHARE_PAGE_TYPE_DDR)`: map the pages and clear 8 KiB.
3. `wait_ctrl.dcf_en = 2` (start the MCU with an SMC), timeout 85 ms (`17 * 5`).
4. Request the `complete` IRQ with handler `wait_dcf_complete_irq`, then
   `disable_irq()` straight away. The IRQ is only enabled while a rate change waits.
5. `SMC(0x82000008, SHARE_PAGE_TYPE_DDR, 0, DRAM_INIT)`: a0 must be 0.
6. `rockchip_get_freq_info()` (section 3).
7. `is_set_rate_direct = true`; `set_auto_self_refresh = SET_AT_SR`.

## 2. How a rate change works on RK3568

### SIP, not SCMI

`is_set_rate_direct` makes `rockchip_dmcfreq_opp_set_rate()` call
`rockchip_ddr_set_rate()` (the SIP SET_RATE) instead of `clk_set_rate()`
(`rockchip_dmc.c:489-492`). The SCMI DDR clock (`clocks = <&scmi_clk 3>`, `"dmc_clk"`,
`rk356x.dtsi:2259-2260`; BL31 names it `clk_scmi_ddr`) serves only as the readback:
`dmcfreq->rate = clk_get_rate(clk)` after the change, and a mismatch means failure
(lines 507-516). `clk-scmi` registers with `CLK_GET_RATE_NOCACHE` in both vendor and
mainline (mainline `drivers/clk/clk-scmi.c:184`), so the readback reflects the real rate.
Nothing shows that BL31's SCMI clock 3 implements a usable set_rate, and the vendor
never tries it. Mainline `include/dt-bindings/clock/rk3568-cru.h:488-490` defines
SCMI clocks 0-2 only, so a mainline DT needs `SCMI_CLK_DDR 3` (or the raw 3).

The vendor `clk-ddr.c` SIP_V2 ops (`drivers/clk/rockchip/clk-ddr.c:118-176`) wrap the
same share-page SET_RATE/GET_RATE/ROUND_RATE in a clock. They are used on older SoCs,
not on RK3568.

### Sequence of one change (`rockchip_dmcfreq_opp_set_rate()`, lines 380-557)

1. `cpus_read_lock()` (line 425) blocks CPU hotplug. BL31 needs every CPU to respond.
2. Optionally, `min-cpu-freq` pins cpufreq. RK3568 DT does not set it.
3. Raise the voltage first when scaling up (`center` supply; section 3).
4. `rockchip_dmcfreq_write_trylock()` in a spin loop (line 479) takes the write side
   of a global rwsem (`rockchip_dmc_common.c:18-48`). Its readers are:
   - VOP2 `vop2_lock()` (vendor `rockchip_drm_vop2.c:1131-1141`), held across
     `vop2_crtc_atomic_enable()` and `vop2_crtc_atomic_disable()`. No DDR change can
     overlap a CRTC enable or disable (modeset).
   - `rockchip_pmu_lock()` (vendor `pm-domains.c:232-242`). No DDR change can overlap
     a power-domain transition. Mainline has the same guard as `rockchip_pmu_block()`
     and `rockchip_pmu_unblock()` (`drivers/pmdomain/rockchip/pm-domains.c:274-372`,
     used by `rk3399_dmc.c:102`), and it is generic, so it covers RK3568 too.
5. Fill the share page and issue `SMC(SIP_DRAM_CONFIG, 2, 0, SET_RATE)`.
6. If **a1 == -6** (`SIP_RET_SET_RATE_TIMEOUT`), run `rockchip_dmcfreq_wait_complete()`
   (lines 1140-1186):
   - `wait_flag = -1`; `enable_irq(complete)`
   - `cpu_latency_qos_update_request(0)`. The vendor comment says CPUs must stay in
     shallow WFI so that BL31's FIQ is serviced quickly: BL31 parks the other CPUs
     during the switch.
   - `SMC(SIP_DRAM_CONFIG, 0, 0, MCU_START)`
   - `wait_event_timeout(..., 85 ms)`
   - The IRQ handler `wait_dcf_complete_irq()` (lines 1125-1138) issues
     `SMC(SIP_DRAM_CONFIG, 2, 0, POST_SET_RATE)` and wakes the waiter.
   - On timeout, the waiter itself issues `POST_SET_RATE` to clear the IRQ and stop
     the MCU (lines 1175-1181).
   - Restore the QoS and `disable_irq()`.
7. Release the write lock, read back `clk_get_rate()`, then lower the voltage if scaling down.

My reading of a1 == -6: BL31 has armed the DDR frequency-change engine to run at the
right moment and needs an explicit start plus a completion interrupt. BL31 has
`vop idle time %lld us`, `dcf_configure_parameter`, `post_set_rate` and
`PARAM_DCF_STATUS timeout` strings, which point to the DCF executing the switch inside
the VOP blanking window that it sizes from `lcdc_type` and `vop_scan_line_time_ns`. A
driver must handle both outcomes: a0 == 0 with a1 == 0 (done synchronously) and
a1 == -6 (start the MCU and wait).

### The "complete" interrupt

- Vendor DT: `interrupts = <GIC_SPI 10 IRQ_TYPE_LEVEL_HIGH>; interrupt-names = "complete";`
  (`rk356x.dtsi:2256-2257`).
- The RK3568 TRM interrupt list gives IRQ ID 42 (SPI 10) as **dcf** and ID 43 (SPI 11)
  as **ddrmon**. Mainline already uses SPI 11 for the DFI node (`rk356x-base.dtsi:1074-1079`).
  The DCF has a secure register window `DCF_S` at 0xfe3a8000, so only BL31 programs
  it. The kernel just takes the interrupt.
- Mainline has no node with SPI 10. A new `dmc` node must own it.

### Display interaction

- `lcdc_type` and `vop_scan_line_time_ns` come from vendor DRM helpers
  (`rockchip_drm_drv.c:422-459`). The scan-line time is `htotal * 1e6 / crtc_clock` of
  the first connector that has a CRTC. Mainline has no equivalent export. Options:
  1. A small DRM-side helper. This is hard to upstream.
  2. Compute it in the devfreq driver from the DRM state of the connected display.
  3. Pass 0 / `SCREEN_NULL`, which tells BL31 there is no display. The switch then
     happens without waiting for blanking. Expect a one-frame glitch or underflow on
     every change, but probably no hang. This needs testing.
- The vendor uses the VOP bandwidth tables (`vop-bw-dmc-freq` and
  `vop-frame-bw-dmc-freq`, `rk356x.dtsi:2262-2271`; logic in
  `rockchip_dmc_common.c:79-144`) only to set a **floor**: VOP line bandwidth above
  287 MB/s needs at least 528 MHz, frame bandwidth above 620 MB/s needs at least
  780 MHz, and a 4K plane needs at least `vop_4k_rate`. A 4K UI scanout at 324 MHz would
  underflow. A mainline driver without the DRM hook should keep a static floor (for
  example 780 MHz) whenever a display is active, or always.
- Modeset exclusion (step 4 above) has no mainline equivalent. A first version can
  hold off DMC changes by policy (for example, no changes while a refresh-rate switch
  is in progress), or add a DRM-side lock later.

## 3. Frequencies and voltages

### `rockchip_get_freq_info()` (lines 1188-1260)

1. `SMC(SIP_DRAM_CONFIG, 2, 0, GET_FREQ_INFO)`: a0 must be 0. Then read `freq_count`
   (1..6) and `freq_info_mhz[]` from the share page.
2. Disable every DT OPP that is not in the list (`dev_pm_opp_disable`, line 1238).
3. For each listed rate missing from the DT table, `dev_pm_opp_add()` it with the
   voltage of the next higher DT OPP (line 1246).

The level mapping (`rockchip_dmcfreq_level_rate_init()`, lines 3031-3064) assumes the
list is in **ascending** order: `rate_low = freq_info[0]` and so on. With 4 points
(324/528/780/1560), LOW = 324, MID_LOW = 528, MID_HIGH = 780, HIGH = 1560. Check the
order on hardware.

### OPPs and supply

- Vendor `rk356x.dtsi:2311-2343` has a **single** DMC OPP: 1560 MHz at
  900000/900000/1000000 µV (L1 leakage bin 875 mV). `rk3568.dtsi:309-316` adds
  RK3568J/M at 875 mV. Every point below it is added at runtime with the same voltage.
  **In practice there is no DDR-related voltage scaling on RK3568**. vdd_logic simply
  has to be at 900 mV (875 mV on the low-leakage bin) or higher before going to 1560 MHz.
- Supply name `center` (`rockchip_init_opp_table(dev, ..., "dmc_clk", "center")`,
  line 3339). Boards use `center-supply = <&vdd_logic>` (for example vendor
  `rk3568-evb.dtsi:384-387`), and some RK3566 boxes use a fixed regulator.
- Mainline boards leave vdd_logic at its boot value (for example rock-3b
  `vdd_logic: DCDC_REG1`, range 0.5-1.35 V, always-on, no consumer that sets a voltage).
  Read `regulator_summary` on the board before the first 1560 MHz attempt. A minimal
  driver can take `center-supply`, set it to 900 mV once (or per OPP), and leave it there.

## 4. Governor and load

Vendor:

- Private governor `dmc_ondemand` (lines 2757-2852), polling every 50 ms (line 648),
  `upthreshold` 40 and `downdifferential` 20 (`rk356x.dtsi:2278-2279`). Load comes from
  the DFI devfreq-event (`devfreq-events = <&dfi>, <&nocp_cpu>`). The target is raised to
  `max(status_rate or auto_min_rate, nocp_cpu table, vop_req_rate)`, then the
  simple-ondemand style maths runs on DFI busy/total.
- "System status" boosts (lines 2533-2622; levels at `rk356x.dtsi:2280-2291`):
  NORMAL/4K/4K_10B = MID_HIGH (780 MHz with the 1560 loader), BOOST/PERFORMANCE/ISP/
  DUALVIEW/REBOOT = HIGH, SUSPEND = LOW. `auto-min-freq` = 324 MHz. Userspace sets
  VIDEO_4K by writing `1,width=...,height=...` to `/sys/class/devfreq/dmc/system_status`
  (parser in vendor `drivers/soc/rockchip/rockchip_system_monitor.c`, around line 350).
  The Android media stack does this. Kernel decoders do not, and Kodi would not.
- Also: touch/input boost, thermal monitor (`rockchip_system_monitor`), and a NoC CPU
  bandwidth probe (`nocp_cpu`).
- The vendor comment in `get_dev_status` (line 612) says the CPU and the MCU must not
  touch DFI/DMC registers at the same time (seen on RK3588), so DFI reads are taken
  under the DVFS lock. A mainline driver should serialise `devfreq_event_get_event()`
  with SET_RATE as well.

Mainline already has the DFI event driver with RK3568 support
(`drivers/devfreq/event/rockchip-dfi.c:740` `rk3568_dfi_init`, compatible at line 812,
`get_event` at lines 288-322; RK3568 has no kernel-controllable PCLK). It plugs straight
into `simple_ondemand`, as `rk3399_dmc.c` does.

Essential for a first version:

1. SIP init (version, share page, DRAM_INIT, GET_FREQ_INFO), the complete IRQ,
   SET_RATE with the -6 path, `rockchip_pmu_block()` around the change, CPU hotplug lock,
   and `cpu_latency_qos` while waiting.
2. OPPs from GET_FREQ_INFO, `center-supply` held at 900 mV or more, and the
   `userspace` or `performance` governor. Fixed 1560 MHz is the first useful result,
   because it is exactly the bandwidth win.
3. A static display floor (`min_freq`), for example 780 MHz, so no scan-out underflow
   appears.

Can follow later: `simple_ondemand` on DFI (upthreshold 40 / downdifferential 20),
a DRM scan-line/lcdc hook and modeset lock, VOP bandwidth floors, suspend (vendor sets
`devfreq->suspend_freq = deep_suspend_rate`, line 3154, and calls SET_AT_SR), the reboot
rate (vendor moves to HIGH on `SYS_STATUS_REBOOT`, lines 2551-2556), and thermal cooling.

## 5. Mainline status

- No RK3568/RK3588 DMC driver exists in `torvalds/master` or `rockchip-7.3.y`.
  `drivers/devfreq` has only `rk3399_dmc.c`. The bindings have only
  `memory-controllers/rockchip,rk3399-dmc.yaml` and `devfreq/event/rockchip,dfi.yaml`.
  `rockchip-7.3.y` carries nothing extra under `drivers/devfreq` or `include/soc/rockchip`.
- Upstream work so far covers DFI only: the "Add perf support to the rockchip-dfi
  driver" series (v8, 2023: RK3568 and RK3588 DFI, perf PMU) and "RK3588 rockchip-dfi
  enhancements" (2025, LPDDR5). Nobody has posted an RK3568/RK3588 DMC series.
- **Upstream TF-A** (`plat/rockchip/rk3568`, master Oct 2026) handles only
  `RK_SIP_SCMI_AGENT0` in `plat_sip_calls.c`. It has no 0x82000008 or 0x82000009 handler,
  and its SCMI clock table (`drivers/scmi/rk3568_clk.c:626-641`) has only cpu, gpu and
  npu (no DDR clock). There is no DRAM DVFS code for rk3568, rk3576 or rk3588.
  **The driver can only work with the vendor rkbin BL31.** It must probe-fail cleanly
  when GET_VERSION returns `SMC_UNK`. Upstream acceptance is uncertain for a driver
  that depends on a closed BL31, although `rk3399_dmc.c` sets a precedent for SIP-driven
  DMC. That case differs because rk3399 DFS lives in open TF-A.
- A new binding is needed: `rockchip,rk3568-dmc` with `interrupts`/`interrupt-names`
  ("complete"), `clocks` (SCMI DDR), `devfreq-events` (dfi), `operating-points-v2` and
  `center-supply`.

## 6. Risks and a first experiment

Hazards:

- **Display or bus traffic during the switch.** If `lcdc_type` and the scan-line time
  are wrong or zero, BL31 cannot fit the switch into blanking. Expect a glitch at
  minimum. A CRTC enable or disable during the switch is what the vendor rwsem
  prevents. Keep modesets away from the experiment.
- **A PMU power-domain transition during the switch** can hang the bus. Wrap the change
  in `rockchip_pmu_block()` and `rockchip_pmu_unblock()`.
- **CPU hotplug or deep idle during the switch.** BL31 parks CPUs by FIQ. Hold
  `cpus_read_lock()` and set `cpu_latency_qos` to 0 while waiting.
- **A missing complete IRQ.** The waiter times out after 85 ms and must issue
  POST_SET_RATE, or the MCU and IRQ stay armed. Never return without POST_SET_RATE once
  MCU_START has succeeded. The IRQ must be requested before DRAM_INIT and kept disabled
  outside a change. It is level-high and cleared only by POST_SET_RATE, so an enabled
  IRQ with no change in flight would storm.
- **DRAM_INIT twice.** No vendor path calls it twice; it runs once per boot in probe.
  BL31 behaviour on a second call is unknown, so module reload is unsafe. Make the
  module non-removable, or skip INIT when a previous instance has already done it
  (a static flag only covers the life of the module).
- **Undervolt.** vdd_logic below 900 mV at 1560 MHz risks silent memory corruption.
  Check the regulator first.
- **Loader/BL31 mismatch.** "loader&trust unmatch" means no DMC. Loader v1.23 with
  BL31 v1.45 come from the same rkbin tag, so they should match. Check the UART log.
- **The mainline SIP header name clash** (0x08 `SET_ODT_PD` vs `GET_VERSION`) is
  harmless for the code but must be resolved in the header.
- **Warm reboot or kexec from a non-F0 rate.** The vendor sets HIGH before reboot. Do
  the same, or restore the boot rate in `.shutdown`.

A safe first experiment, as an out-of-tree test module with no devfreq:

1. Boot without Kodi running (display idle, ideally a static console or the display
   off), and with no modeset during the test. Record `regulator_summary` (vdd_logic)
   and `clk_scmi_ddr`.
2. `GET_VERSION`: print a0 and a1. Stop unless a0 == 0 and a1 >= 0x101.
3. `SHARE_MEM(2, 2)`: print the physical address. Map with `ioremap()` and clear 8 KiB.
4. Request SPI 10 (needs a DT node or overlay with the interrupt) and disable it.
5. `DRAM_INIT`: print a0. `GET_FREQ_INFO`: print `freq_count` and `freq_info_mhz[]`.
   Unload-safe up to this point? Unknown. Do not reload the module on the same boot.
6. One `SET_RATE` to a **lower** listed point first (for example 780 MHz), with
   `lcdc_type` = 0 and scan-line time = 0, wait flags 1, under `rockchip_pmu_block()`,
   `cpus_read_lock()` and QoS 0. Print a0 and a1, take the -6 path if needed, read back
   `clk_get_rate()` on the SCMI DDR clock (or `GET_RATE` 0x05), then
   return to the boot rate the same way.
7. Only after that, with vdd_logic confirmed at 900 mV or higher, try the top point (1560)
   and run a memory test (for example `memtester`) plus the 4K decode benchmark.
8. Then repeat with the display active to see whether `lcdc_type` and scan-line time
   need wiring before building the real devfreq driver.
