# Stock Firmware Audit — tonga (Moto G Power 2022)

## Status: SKIPPED

`FIRMWARE_DIR` is not set in the build environment. The stock firmware
audit (Phase 1) was skipped. All items that depend on it are marked
`TODO(verify)` throughout the tree.

## What was NOT verified

| Item | Status |
|------|--------|
| `ili9882_mmi.ko` vermagic vs prebuilt kernel | TODO(verify) |
| `icnl9916_tddi_mmi.ko` vermagic vs prebuilt kernel | TODO(verify) |
| Full transitive dependency chain for touch modules | TODO(verify) |
| Correct `ro.hardware` value for `init.recovery.<hw>.rc` | TODO(verify) |
| Post-load sysfs setup for touch panels | TODO(verify) |
| `TW_BRIGHTNESS_PATH` sysfs path | TODO(verify) |
| `sys.usb.controller` property value | TODO(verify) |
| UDC name for MT6765 | TODO(verify) |
| Configfs gadget setup from stock `*usb*.rc` | TODO(verify) |
| ADB function ordering | TODO(verify) |
| mkbootimg args (offsets, page size, header version, cmdline) | TODO(verify) |
| `slotselect` in `vendor/etc/fstab.*` | TODO(verify) |
| `ro.hardware` / `ro.boot.hardware` | TODO(verify) |

## A/B layout contradiction

`recovery/root/etc/recovery.fstab` shows `slotselect` on 33 partitions,
confirming A/B. However:

- `BOARD_SUPER_PARTITION_SIZE := 9126805504` (~8.5 GB)
- `BOARD_MOTOROLA_DYNAMIC_PARTITIONS_SIZE := 9122611200` (~8.5 GB)

These are nearly equal. For classic A/B, the dynamic partitions group
should be at most half of super minus overhead (to fit two copies of
system/vendor/product). This contradiction is documented in
`BoardConfig.mk` with a `TODO(verify)` marker.

## Prebuilt kernel version

`zcat prebuilt/Image.gz | strings | grep -m1 'Linux version'` returned
no output. The kernel version string could not be extracted, so module
vermagic matching is impossible without the stock firmware.

## Files examined (Phase 0)

- `BoardConfig.mk` — cleaned up (see commits)
- `device.mk` — no changes needed
- `twrp_tonga.mk` — no changes needed
- `recovery/root/init.recovery.mt6765.rc` — extended with insmod + diagnostic
- `recovery/root/etc/recovery.fstab` — read-only, has slotselect
- `recovery/root/vendor/lib/modules/` — contains ili9882_mmi.ko, icnl9916_tddi_mmi.ko
- `prebuilt/Image.gz` — 11446295 bytes, no version string found
- `prebuilt/dtb.img` — 109735 bytes
