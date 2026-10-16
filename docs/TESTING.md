# Testing Guide — tonga TWRP touch/USB fix

## Prerequisites

- Stock firmware build number: `S3RQS32.20-42-10-9-9` (from `twrp_tonga.mk`)
- A/B device with `slotselect` confirmed in recovery.fstab
- Bootloader must support `fastboot boot` for safe testing

## Flashing instructions

**WARNING: Do NOT flash over your only working boot image.**

1. **Preferred:** `fastboot boot twrp_tonga.img`
   - Does not overwrite the current boot partition
   - If the bootloader supports this, use it

2. **Alternative (only if A/B confirmed):**
   - Flash to the **inactive** slot only
   - `fastboot flash boot_b twrp_tonga.img` (or `_a` if currently on `_b`)
   - Verify current slot: `fastboot getvar current-slot`

3. **Do NOT flash to both slots** unless you have a known-good backup.

## Retrieving logs

1. **TWRP Copy Log:**
   - Advanced → Copy Log
   - Saves to `/tmp/recovery.log` on the device
   - Pull via `adb pull /tmp/recovery.log`

2. **Touch diagnostic log:**
   - The `touch-diag` service writes to `/tmp/touch_dbg.log`
   - Pull via `adb pull /tmp/touch_dbg.log`
   - Contains filtered dmesg: `ili9882|icnl9916|ts_mmi|Unknown symbol|Exec format|insmod`

3. **Full dmesg (if ADB is working):**
   - `adb shell dmesg > dmesg.log`

## Symptom → cause mapping

| dmesg symptom | Likely cause | Action |
|---------------|--------------|--------|
| `Unknown symbol` | Missing dependency or wrong load order | Check modules.dep, reorder insmod |
| `Exec format error` | Wrong firmware build / arch mismatch | Verify .ko files match prebuilt kernel |
| `No such file` | Missing module file | Check /vendor/lib/modules/ contents |
| Clean load, no input device | Probe timing or sysfs trigger issue | Check post-load sysfs setup, add delays |
| `insmod: failed to load` (other) | vermagic mismatch | Verify kernel version matches modules |

## What to report back

1. **Firmware build number:** `ro.build.display.id` (get via `adb shell getprop ro.build.display.id`)
2. **Logs:** `/tmp/touch_dbg.log` and `/tmp/recovery.log`
3. **Panel variant:** Which touch panel the device has (ili9882 or icnl9916)
   - Check dmesg for which module successfully probed
   - Or check `ls /sys/class/touchscreen/` after boot
4. **USB status:** Does ADB work in recovery? Does MTP work?
5. **Touch status:** Does touch input work? Any dmesg errors?

## Known limitations

- Module load order and vermagic match are **unverified** (FIRMWARE_DIR not set)
- USB UDC name and configfs setup are **unverified** (stock audit skipped)
- Post-load sysfs setup for touch panels is **unverified**
- A/B dynamic partition sizes may be incorrect (documented contradiction)
- mkbootimg args not diffed against stock boot.img
