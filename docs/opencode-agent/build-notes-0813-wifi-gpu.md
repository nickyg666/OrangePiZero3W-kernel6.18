# WiFi + GPU porting notes (2026-08-13)

## WiFi (AIC8800 SDIO)
- Board wifi is SDIO on sdc1 (sdmmc1 @ 0x04021000), NOT USB.
- The mainline bsp's original aic8800 driver was USB-only (registered via usbcore).
- Replaced bsp/drivers/net/wireless/aic8800 with the vendor linux-orangepi
  driver (aic8800_bsp + aic8800_fdrv + aic8800_btlpm/btusb), which has proper
  SDIO support (aicsdio.c / aicwf_sdio.c).
- Config: CONFIG_AIC_INTF_SDIO=y, CONFIG_AIC8800_WLAN_SUPPORT=m (replaced the
  old AIC8800_USB/AIC8800_SDIO symbols).
- 6.6.98 -> 6.18 API fixes applied:
  - wakeup_source_create/add/remove/destroy -> wakeup_source_register/unregister (>=6.0)
  - from_timer -> timer_container_of, del_timer -> timer_delete,
    del_timer_sync -> timer_delete_sync
  - cfg80211_rx_spurious_frame / rx_unexpected_4addr_frame: +link_id,gfp args
  - cfg80211_ch_switch_notify: 4->3 args (link_id only)
  - cfg80211_ch_switch_started_notify: dropped punct_bitmap arg
  - cfg80211_cac_event: +link_id arg
  - cfg80211_ops callbacks: change_beacon(cfg80211_ap_update*),
    set_monitor_channel(+dev), set_wiphy_params(+radio_idx),
    set_tx_power(+radio_idx), start_radar_detection(+link_id)
  - aicsdio.c: added <linux/platform_device.h> and <linux/timer.h>
- Added bsp/drivers/mmc/sunxi-mmc-rescan-shim.c (obj-y): provides
  sunxi_mmc_rescan_card() via of_find_node_by_type("sdcN") + mmc_detect_change,
  since vendor sunxi-mmc-export.c (CONFIG_AW_MMC) is not built (mainline
  sunxi-mmc used instead).
- Built: aic8800_bsp.ko, aic8800_fdrv.ko, sunxi_rfkill.ko (provides
  sunxi_wlan_set_power/get_bus_index). Image rebuilt with shim.
- Firmware needed at /lib/firmware/aic8800d80/ (already present on SD).

## GPU (PowerVR IMG BXM / rogue)
- Mainline powervr.ko (drivers/gpu/drm/imagination) supports img,img-rogue /
  img,img-axe; needs firmware powervr/rogue_*.fw.
- Deployed DTB gpu@1800000 has compatible "img,gpu" (vendor) -> won't match
  mainline powervr. Need to change to img,img-rogue + provide firmware.
- Vendor firmware: rgx.fw.36.56.104.183 + rgx.sh.36.56.104.183 (from Radxa
  image, present in hybrid-opi66 img at /usr/lib/firmware/).
- Vendor userspace: libGLESv2_PVR_MESA, libPVROCL, libVK_IMG.so,
  /usr/share/vulkan/icd.d/img_icd.json (all in hybrid image).
