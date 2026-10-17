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
   - Contains filtered dmesg plus `/proc/cmdline`:
     `CTS-|ILITEK|ilitek|icnl9916|ili9882|chipone|active panel|not found|Unknown symbol|version magic|disagrees about version|Exec format|insmod`
     (`CTS-` and `ILITEK` matter: the drivers' printk tags are `CTS-I2CDrv`, `CTS-Core`,
      `CTS-Plat` and `ILITEK: …`, so a grep for the module file names alone misses them.)

3. **Full dmesg (if ADB is working):**
   - `adb shell dmesg > dmesg.log`

## Symptom → cause mapping

The drivers print their own reason. Find the line and read the cause off it:

| dmesg line | Meaning |
|------------|---------|
| `<I>CTS-I2CDrv active_panel_name=…` / `… ilitek …` | cmdline gate **passed** for that driver |
| `CTS-I2CDrv chipone active panel not found!` | no `mipi_mot_vid_*` token in `/proc/cmdline` at all |
| `CTS-I2CDrv Not chipone panel!` | token present but is not the chipone panel — expected, harmless, for the non-matching driver (same for `Not ilitek panel!`) |
| `CTS-Core Device hwid: %06x fwid: %04x not found` | chip answered but is not a supported part |
| `CTS-Core Enter program mode failed` / `CTS-Core Probe device failed` | chip did not answer SPI in program mode → panel power/reset state, not a module problem |
| `ILITEK Driver probe failed` / `ILITEK … dummy check` | same for the ilitek SKU |
| `Unknown symbol` | missing dependency (neither module has any — unexpected) |
| `version magic` / `disagrees about version of symbol` | the **stock** modules were loaded, i.e. `/vendor` was mounted over the ramdisk copies before `insmod` |
| `Exec format error` | wrong firmware build / arch mismatch |
| no `CTS-`/`ILITEK` line at all | the module never loaded → check `insmod` in dmesg |

No `CTS-`/`ILITEK` line and no insmod error at all means the `touch-diag` service itself
did not run (it is `class late_start`, started from `on boot`).

## What to report back

1. **Firmware build number:** `ro.build.display.id` (get via `adb shell getprop ro.build.display.id`)
2. **Logs:** `/tmp/touch_dbg.log` and `/tmp/recovery.log`
3. **`cat /proc/cmdline`** — must contain the `mipi_mot_vid_…` token matching the panel
   (`…icnl9916…` or `…ili9883a…`). This is the single most useful line; it is already appended to
   `/tmp/touch_dbg.log` after the `CMDLINE:` marker.
4. **USB status:** `ls -l /sys/class/udc/`, `cat /config/usb_gadget/g1/UDC` (expect `musb-hdrc`),
   and whether ADB comes up.
5. **Touch status:** does touch input work?

## Known limitations

- Load order is **known to be irrelevant** (both drivers self-select from the cmdline token and the
  non-matching one returns `-1` before binding) — do not reorder `insmod` as a fix.
- `musb_hdrc.kernel_init_done` is still not written in recovery (stock writes it at `early-init`);
  its semantics were not decompiled.
- The CDP/SDP meaning of `write /sys/class/udc/musb-hdrc/device/cmode 2` is unverified.
- Whether the TDDI chip answers program-mode SPI in recovery (panel power/reset state left by the
  bootloader) cannot be judged offline.
- A/B dynamic partition sizes: the earlier "contradiction" is resolved (Virtual A/B, see
  `stock-audit.md` §2).
- mkbootimg args still not diffed against stock boot.img.
