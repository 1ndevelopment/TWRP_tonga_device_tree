# Final Report — tonga TWRP touch/USB fix

> **Partly superseded** — a later firmware audit (see `docs/stock-audit.md`) disproved the
> "module load order" item below: both drivers self-select from the `mipi_mot_vid_*` token in
> `/proc/cmdline` and the wrong one returns `-1` before binding, so order is inert. The
> vermagic item was also resolved (the recovery copies match the recovery kernel). The
> firmware audit is authoritative where the two disagree.

## Summary

Branch `fix/touch-usb-recovery` with 6 commits. FIRMWARE_DIR is not set,
so the stock firmware audit (Phase 1) was skipped. All dependent items
are marked `TODO(verify)`. Nothing is described as "fixed" or "tested" —
all changes are "intended to fix" and "unverified on hardware".

## Commits

1. `75f6e2e` — fix: remove duplicate VENDOR_SECURITY_PATCH and fix misleading comment
2. `9295233` — fix: remove dead build variables
3. `0a2212c` — docs: add A/B consistency and mkbootimg TODO(verify) markers
4. `03fc19f` — fix: load touch modules via explicit insmod and add diagnostic logger
5. `73c45d1` — docs: mark USB configfs setup as unverified on hardware
6. `855add8` — docs: add stock audit report and testing guide

## What was changed

### BoardConfig.mk
- Removed duplicate `VENDOR_SECURITY_PATCH := 2021-08-01` (overridden)
- Fixed misleading "prevent anti rollback" comment → accurate description
- Removed `DEXPREOPT_GENERATE_APEX_IMAGE` (dead)
- Removed `BOARD_RECOVERYIMAGE_PARTITION_SIZE` (recovery-as-boot)
- Removed `TARGET_KERNEL_CONFIG`/`TARGET_KERNEL_SOURCE` (dead with prebuilt kernel)
- Removed `TW_CRYPTO_SYSTEM_VOLD_*` block (no-op without services)
- Removed `TW_INPUT_BLACKLIST := "hbtp_vm"` (Qualcomm, not MTK)
- Removed `TW_LOAD_VENDOR_MODULES` (replaced by explicit insmod)
- Added TODO(verify) markers for A/B consistency and mkbootimg args

### recovery/root/init.recovery.mt6765.rc
- Added `insmod` for ili9882_mmi.ko and icnl9916_tddi_mmi.ko under `on boot`
- Added `touch-diag` service (filtered dmesg → /tmp/touch_dbg.log)
- Added TODO(verify) markers for USB configfs and UDC setup

### docs/
- `docs/stock-audit.md` — audit skip report, unverified items list, A/B contradiction
- `docs/TESTING.md` — flashing instructions, log retrieval, symptom mapping

## What was verified statically

| Check | Result |
|-------|--------|
| All paths referenced by makefiles exist | ✓ |
| No duplicate variable assignments remain | ✓ |
| No tabs in rc files | ✓ |
| Module files are valid ELF aarch64 | ✓ |
| shellcheck on new/modified scripts | N/A (no new scripts) |
| rc file syntax (stray tabs, unknown actions) | ✓ |

## What was NOT verified (TODO(verify) items)

| Item | Why |
|------|-----|
| ili9882_mmi.ko vermagic vs prebuilt kernel | FIRMWARE_DIR not set |
| icnl9916_tddi_mmi.ko vermagic vs prebuilt kernel | FIRMWARE_DIR not set |
| Full transitive dependency chain | FIRMWARE_DIR not set |
| Correct ro.hardware for init.recovery.<hw>.rc | FIRMWARE_DIR not set |
| Post-load sysfs setup for touch panels | FIRMWARE_DIR not set |
| TW_BRIGHTNESS_PATH sysfs path | FIRMWARE_DIR not set |
| sys.usb.controller property value | FIRMWARE_DIR not set |
| UDC name for MT6765 | FIRMWARE_DIR not set |
| Configfs gadget setup from stock *usb*.rc | FIRMWARE_DIR not set |
| ADB function ordering | FIRMWARE_DIR not set |
| mkbootimg args (offsets, page size, header version, cmdline) | FIRMWARE_DIR not set |
| slotselect in vendor/etc/fstab.* | FIRMWARE_DIR not set |
| ro.hardware / ro.boot.hardware | FIRMWARE_DIR not set |
| A/B dynamic partition sizes | Contradiction documented |
| Module load order | FIRMWARE_DIR not set |
| Prebuilt kernel version string | Not found in Image.gz |

## Known risks

1. **Module load order may be wrong** — ili9882 before icnl9916 is a guess
2. **vermagic mismatch possible** — .ko files may not match prebuilt kernel
3. **USB UDC name is a guess** — `musb-hdrc` may not exist on MT6765
4. **A/B partition sizes may be wrong** — dynamic group nearly equals super size
5. **mkbootimg args unverified** — offsets/header version may not match stock
6. **No post-load sysfs setup** — touch may need additional sysfs triggers

## Next steps (require hardware)

1. Run `tools/audit-stock.sh` with FIRMWARE_DIR set to verify all TODO(verify) items
2. Flash via `fastboot boot` and check dmesg for touch module load
3. Verify touch input works
4. Verify USB ADB works
5. Adjust module order / sysfs setup based on dmesg output
