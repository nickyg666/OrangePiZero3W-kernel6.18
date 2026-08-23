# Mainlining Strategy
## Orange Pi Zero 3W (Allwinner A733 / T527)

## Current Landscape (August 2026)
### Who supports A733/Sun60i?
1. **Orange Pi** — vendor BSP kernel 6.6.98, closed GPU driver, OMX video
2. **DietPi** — same vendor 6.6.98 kernel, slim image, no GPU
3. **Armbian (community)** — BSP kernel builds for Radxa Cubie A7A/A7Z, active forum thread (190+ replies)
4. **Linux mainline (kernel.org)** — no upstream sun60i CCU/pinctrl drivers yet

### Our Position
- We have the **only known mainline 6.18+ kernel build** for this SoC
- Hybrid approach: linux 6.18.19 base + vendor BSP drivers (CCU, pinctrl, uart-ng)
- AW_UART_NG ported to 6.18 kfifo API — builds clean
- MMC dtb compatibility issue diagnosed and patched (2026-08-10)

## Strategy: Three Horizons

### Horizon 1: Boot (Now)
- Make the 6.18.19 hybrid kernel boot the Zero 3W to userspace
- **ROOT CAUSE (2026-08-11)**: Two DTB blockers found and fixed:
  1. mmc compatibles (sun20i-d1-mmc) — patched 2026-08-10
  2. `vmmc-supply = <0x54>` → axp8191 dc1sw2 (no mainline driver) → EPROBE_DEFER forever → MMC never probes → removed from sdc0 (2026-08-11). Deployed dtb MD5 `b2a702fb…`
- PENDING: User power-cycle test with fixed dtb
- Verification chain (in order): `/boot/mainline/mainline-bootdiag.txt` (initramfs reached + /dev/mmcblk* listing) → `mainline-bootlog.txt` (root mounted/init) → `/var/lib/ili9486-boot-count` > 2 (userspace)
- All other dependencies verified sound: vendor AW CCU binds `allwinner,sun60iw2-ccu` via CLK_OF_DECLARE, clock IDs 0x8f/0x90 = CLK_SMHC0/CLK_BUS_SMHC0, reset 0x23 = RST_BUS_SMHC0, pinctrl-sun60iw2 built-in with gpiochip (cd-gpios OK), wakeupgen IRQ domain built-in (IRQCHIP_DECLARE), sun20i_d1_cfg needs no output/sample clocks
- If boot succeeds: board will NOT return on network (no WiFi/eth drivers in hybrid yet) — MUST check SD evidence
- Console available via ttyS0 (AW_UART_NG port) + LCD (ili9486 overlay needed for display)

### Horizon 2: Stabilize (Weeks 1-4)
- Fix any runtime bugs from the hybrid approach
- Verify Ethernet (sunxi-gmac-210), USB, WiFi (AIC8800 SDIO)
- Fix display: add ili9486 overlay to mainline boot path or port ili9486 driver
- Proper initramfs (currently functional but built via chroot)
- Boot time measurement and optimization
- Submit uart-ng kfifo port as clean patch series

### Horizon 3: Upstream (Ongoing)
- **Target trees**:
  1. Armbian `linux-sunxi` tree (community-maintained sunxi patches)
  2. Collabora sunxi-ng (upstream mainline efforts)
  3. kernel.org (final target)
- **Required mainline work**:
  - sunxi-ng CCU driver for A733 (T527) — clock tree definition
  - sunxi pinctrl driver for A733 (build on existing sunxi pinctrl)
  - Replace vendor DTS with proper mainline DTS (already started in our tree)
  - GPU: PowerVR IMG BXE — needs Imagination powervr driver (linux 6.12+ staging)
  - Display: ili9486 panel driver for mainline DRM
  - WiFi: AIC8800 driver (vendor-only, needs mainlining effort)

## Community Resources
- Armbian forum: "Radxa Cubie A7A/A7Z - Allwinner a733" thread (190 replies, active)
- Armbian forum: "A733 zero copy hardware decoding" — cedar/libvdecoder approach
- GitHub: NickAlilovic/build (Armbian build for Radxa A7A/A7Z)
- Orange Pi SDK: linux-5.15 vendor BSP (older than our 6.6.98 but useful for reference)
- Allwinner platform docs: sun60iw2p1 user manual (if available)

## Key Dependencies for Full Mainlining
| Component       | Status                    | Effort    |
|-----------------|---------------------------|-----------|
| UART (uart-ng)  | Ported to 6.18, builds    | Low (done)|
| MMC             | DTB fix in place          | Low (done)|
| CCU             | Using vendor driver       | Medium    |
| Pinctrl         | Using vendor driver       | Medium    |
| Ethernet        | Vendor driver in kernel   | Low       |
| USB             | Vendor drivers in kernel  | Low       |
| WiFi (AIC8800)  | Not in mainline           | High      |
| GPU (PowerVR)   | Staging in 6.12+          | High      |
| Display (ili9486)| SPI panel, not mainline   | Medium    |
| VPU (cedar)     | libvdecoder reverse-eng   | High      |
| NPU (vip2)      | Vendor driver             | Very High |

## Recommended Next Steps
1. 🔴 User: reinsert SD into board + power-cycle (dtb `b2a702fb…` deployed)
2. 🔴 Reinsert SD into host reader, check `mainline-bootdiag.txt` → `mainline-bootlog.txt` → `ili9486-boot-count`
3. Publish this memory layer to GitHub (user directive 2026-08-11)
4. If boot successful: submit uart-ng port as patch to Armbian sunxi list
5. Fork Armbian build system, add orangepi-zero3w board config
6. Begin mainline CCU driver skeleton for sun60i (A733)
7. Start ili9486 panel DRM driver based on BSP vendor driver