# Stock Firmware Audit — tonga (Moto G Power 2022, MT6765H)

Status: **PERFORMED** (2026-10-17). Evidence-only: no physical device was used.
Scope: touch, display coupling, USB gadget bring-up, recovery-tree facts.
Out of scope (not analysed): preloader, LK, TEE, AVB, bootloader unlock.

Inputs: `fw/out/` (stock firmware, read-only) → all derived output in `fw/work/` (gitignored).
Tagging: **confirmed** = direct evidence quoted below; **inferred** = stated reasoning;
**unknown** = cannot decide offline.

---

## 1. Method / tooling

| Step | Tool | Notes |
|------|------|-------|
| Sparse super merge |`simg2img $(ls super.img_sparsechunk.* \| sort -V) fw/work/super.img` | multi-input = tile by absolute block offset (verified with a 2-chunk control); output 9,126,805,504 B |
| super introspection | `lpdump` → `fw/work/lpdump.txt`; `lpunpack` → `fw/work/super_parts/` | absolute paths required |
| ext4 extraction | `debugfs -R "rdump / <dir>"` → `fw/work/fs/` | `Operation not permitted while changing ownership` is cosmetic (no root) |
| boot.img | `fw/work/unpack_bootimg.py` (self-written) | `unpack_bootimg` not installed |
| DTB/DTBO | `fw/work/dtbo_parse.py` (self-written minimal FDT parser) | `dtc` not installed |
| Native RE | REA MCP, provider **Ghidra 12.1.4** (arm64) | `objdump`/`readelf`/`nm` also available; `modinfo`/`modprobe`/`depmod` are NOT |

## 2. Super partition layout — RESOLVED: Virtual A/B, not classic A/B

`fw/work/lpdump.txt`:

```
header flags: virtual_ab_device
metadata max size 65536, slot count 3
group main_a max size 9124708352
group main_b max size 9124708352
```

Only slot `_a` is populated (`product_b`/`system_ext_b`/`vendor_b` = 0 bytes; small `system_b`).
**Confirmed: stock is Virtual A/B.** This explains the BoardConfig "contradiction" noted by the
earlier task: `BOARD_SUPER_PARTITION_SIZE 9126805504` equals the real super size, and
`BOARD_MOTOROLA_DYNAMIC_PARTITIONS_SIZE 9122611200` is 2,097,152 B (2 MiB) **below** the stock
group max — not a half-size split. There is no A/B contradiction; the two-copies-per-group concern
does not apply to Virtual A/B.
(No files changed here — BoardConfig is read-only for this task.)

## 3. boot.img, kernel and DTB

Parsed header (`fw/work/mkbootimg_args.txt`): version 2, page size 2048,
cmdline `bootopt=64S3,32N2,64N2 buildvariant=user`, os_version 12.0.0 patch 2023-12,
kernel_addr `0x40080000`, ramdisk_addr `0x51b00000`, tags_addr `0x47880000`, dtb_addr `0x47880000`.

| Artifact | stock `boot.img` | repo `prebuilt/` | verdict |
|---|---|---|---|
| DTB (109,735 B) | sha256 `3715d90f…483c66` | `prebuilt/dtb.img` sha256 `3715d90f…483c66` | **byte-identical (confirmed)** |
| kernel | `Linux version 4.19.191-gbaa59638869e … #1 SMP PREEMPT Tue Dec 19 06:03:53 CST 2023` | `Image.gz` decompresses (twice) to `4.19.191-g2991a7c6cb9d … Fri Sep 29 07:18:28 CDT 2023` | **different build (confirmed)** |

`prebuilt/Image.gz` is **doubly gzipped**: outer gzip has FNAME `kernel` and its payload is another
gzip stream (`1f8b0800`); the ARM64 Image magic `ARMd` appears only after the second gunzip.
Decompressed size is identical (26,796,032 B) but sha256 differs
(stock `b4879c32…b132de`, prebuilt `6f359a10…366c3f`). Flag this as a packaging defect (not a
conclusion about bootability — the built recovery image was not inspected).

DTB container note: both `boot.img`'s appended dtb and `dtbo.img` start with magic `0xd7b7ab1e`
(Motorola/MTK container); the real FDT (`d00dfeed`) starts at +0x40. `boot.img`'s dtb has
**no touch-controller compatible**; `/chosen/bootargs` is
`console=tty0 console=ttyS0,921600n1 vmalloc=500M slub_debug=OFZPU page_owner=on swiotlb=noforce androidboot.hardware=mt…`
— **no panel name** (confirmed).

## 4. Touch

### 4.1 Two panel SKUs, both in the stock kernel

`strings fw/work/boot/Image` contains exactly two Moto LCM driver names:

```
mipi_mot_vid_tm_icnl9916_652_hdp   / tianma_icnl9916_hdp_vid
mipi_mot_vid_tm_ili9883a_652_hdp   / tianma_ili9883a_hdp_vid
```

**Confirmed:** tonga ships in two hardware variants (both Tianma 6.5" HDP):
one with **Chipone ICNL9916** touch, one with **Ilitek ILI9883A** touch. Both LCM names are also
present in the recovery (`prebuilt`) kernel.

### 4.2 Touch node lives in dtbo, not in boot.img's DTB

`dtbo.img` overlay `fragment@22`/`fragment@24` targets the base DTB symbol `touch`:

```
/fragment@24/__overlay__/touch@0
    compatible = "tchip,ilitek", "novatek,NVT-ts-spi", "chipone-tddi"
    status = "okay"
    chipone,irq-gpio / chipone,rst-gpio / chipone,x-res / chipone,y-res
    novatek,irq-gpio / novatek,reset-gpio / touch,irq-gpio / touch,reset-gpio
/fragment@22/__overlay__   tpd-resolution = 0x2d0 0x640, tpd-max-touch-num = 0xa,
                           tpd-panel-match = "" (empty)
```

It is an **SPI** device (`reg = 0x0`, `spi-max-frequency = 0x989680` = 10 MHz).
The base DTB's own `/touch` node is a placeholder with `compatible = "goodix,touch"`.
**Inferred:** the touch IC is selected only after the overlay is applied; the compatible list is the
same for both SKUs, so the compatible strings alone do not identify the IC.

Recovery ships **only the two touch `.ko`** (`recovery/root/vendor/lib/modules/`), no `mmi_info.ko`,
`utags.ko`, `moto_f_usbnet.ko`, etc. They are not needed: `modules.dep` lists **no dependencies** for
either touch module (confirmed), and every imported symbol of both modules resolves in **both**
kernels (116/112 undefined symbols checked byte-wise against `Image` and `Image.prebuilt` — 0 absent).

### 4.3 Modules

| | stock `ili9882_mmi.ko` | stock `icnl9916_tddi_mmi.ko` |
|---|---|---|
| size | 604,152 | 327,040 |
| vermagic | `4.19.191-gbaa59638869e SMP preempt mod_unload modversions aarch64` | same |
| `of_match` | `tchip,ilitek` | `chipone-tddi` (+ `chipone,def-config-id`, `chipone,build_id`) |
| modules.alias | **no `of:` alias emitted** | `alias of:N*T*Cchipone-tddi icnl9916_tddi_mmi` |

Recovery copies of the same two sizes differ from stock in only **19 bytes** each — i.e. the *same*
modules relinked/patched for the older recovery kernel: vermagic
`4.19.191-g2991a7c6cb9d …`. Recovery kernel and recovery modules are therefore mutually consistent
(confirmed).

### 4.4 The selection mechanism — drivers gate on the kernel command line

REA/Ghidra decompilation of the stock `icnl9916_tddi_mmi.ko` (`cts_driver_probe`, .text @ `0x100004`;
string/literal cross-references via `.rela.text`):

```
printk("<I>CTS-I2CDrv enter\n");
if (saved_command_line == NULL)      { printk("... saved_command_line null!\n"); return -1; }
printk("... saved_command_line is %s\n", saved_command_line);
p = strstr(saved_command_line, "mipi_mot_vid_");
if (p == NULL)                       { printk("... chipone active panel not found!\n"); return -1; }
strncpy(panel_name, p, 0x32);
if (strstr(panel_name, "icnl") == NULL) {
                                        printk("... Not chipone panel!\n");
                                        printk("... MTK get chipone panel error\n"); return -1; }
printk("... active_panel_name=%s\n", panel_name);
```

`ili9882_mmi.ko` carries the identical structure with the strings
`ILITEK: … saved_command_line is %s`, `ILITEK: … ilitek active panel not found!`,
`ILITEK: … Not ilitek panel!` and the literal `mipi_mot_vid_`.

**Confirmed:** the two drivers are **mutually exclusive and self-selecting**:
- cmdline token containing `icnl`  → only `icnl9916_tddi_mmi` probes;
- cmdline token containing `ili`   → only `ili9882_mmi` probes;
- no `mipi_mot_vid_` token at all   → **both** return `-1` (no touch).

**Confirmed corollary: insmod order is irrelevant.** `__driver_attach` only probes devices with
`dev->driver == NULL`, and each wrong driver bails out with `-1` before binding. The earlier
"load order" hypothesis (and the previous fix that reordered `insmod`) cannot affect the outcome.

The `mipi_mot_vid_…` token appears in **none** of `boot.img`, the appended DTB, or `dtbo.img`
(0 occurrences) — **inferred:** it is appended to the kernel cmdline at boot by LK. LK analysis is
out of scope, so **unknown:** whether the token is present when booting this recovery image.
On-device check: `cat /proc/cmdline` (see §6).

### 4.5 Firmware requested by each driver

`icnl9916_tddi_mmi.ko` (`cts_request_firmware`, Ghidra @ `0x108a14`, string refs `0x11d99a`,
`0x12325d`, `0x12452c`): maps hw id → `/vendor/firmware/ICNL9916.bin` (hwid `0x990160`) or
`/vendor/firmware/ICNL9911.bin` (hwid `0x991110`), then requests it via
`cts_request_newer_firmware_from_fs()`. If the file is not present/readable the helper returns 0
(no update). **Inferred: the probe does not depend on this file** (it is the *update* path; the
IC keeps its flashed firmware otherwise).

`ili9882_mmi.ko` uses `request_firmware` with a panel-selected name from its table:
`ILITEK_FW_TM_9883A`, `ILITEK_FW_TM_9882H`, `ILITEK_FW_TM_9882N`, `ILITEK_FW_TIANMA_9882N`,
`ILITEK_FW_TM`, `ILITEK_FW_TM_7807S`, `ILITEK_FW_TXD*`, `ILITEK_FW_INX`, `ILITEK_FW_BOE`,
`ILITEK_FW_CSOT`, `ILITEK_FW_AUO`, `ILITEK_FW_HLT`, `ILITEK_FW_DJ`
(plus `/sdcard/ILITEK_FW*` override paths).

Full stock-vs-recovery presence matrix: `docs/touch-firmware-files.txt`.

## 5. USB

### 5.1 Controller

Base DTB `usb0@11200000`, `compatible = "mediatek,mt6765-usb20"` → MediaTek **MUSB** (USB 2.0).
`Musb` strings and the `cmode` / `cmode_show` / `cmode_store` sysfs attributes are present in the
stock kernel Image (confirmed).

### 5.2 Stock bring-up sequence

`vendor/etc/init/hw/init.mt6765.usb.rc` (stock):

| line | statement |
|---|---|
| 5 | `setprop vendor.usb.vid "0x0E8D"` |
| 6–7 | `mkdir /dev/usb-ffs…`, `mkdir /dev/usb-ffs/adb 0770 shell shell` |
| 19–22 | `mkdir /config/usb_gadget/g1/functions/ffs.adb`, `…/functions/mtp.gs0`, `write …/mtp.gs0/os_desc/interface.MTP/compatible_id "MTP"` |
| 41 | `mount functionfs adb /dev/usb-ffs/adb rmode=0770,fmode=0660,uid=2000,gid=2000,no_disconnect=1` |
| 64–71 | `setprop sys.usb.configfs 1` · `setprop vendor.usb.controller "musb-hdrc"` · acm props · `setprop sys.usb.ffs.aio_compat 0` · `write /sys/class/android_usb/android0/f_mtp/cpu_mask 0xF0` |
| 109 | same block again for the second boot path: `vendor.usb.controller "musb-hdrc"` |
| 165–171 | on `sys.usb.ffs.ready=1 && sys.usb.config=adb`: `write …/configs/b.1/strings/0x409/configuration "adb"`, `symlink …/functions/ffs.adb …/configs/b.1/f1`, **`write /config/usb_gadget/g1/UDC ${vendor.usb.controller}`**, `setprop sys.usb.state ${sys.usb.config}` |
| 132, 137–171 | adb/mtp handlers; `setprop vendor.usb.pid 0x2E81` for plain adb |

**Confirmed: the UDC name is `musb-hdrc`, and the property the vendor rc uses is
`vendor.usb.controller` — not `sys.usb.controller`.** No stock `.rc` ever sets
`sys.usb.controller`. The only binary in the stock tree that sets `sys.usb.controller` is the vendor
USB HAL:

```
vendor/bin/hw/android.hardware.usb@1.2-service-mediatekv2
  strings: "sys.usb.controller", "/config/usb_gadget/g1/UDC", "/sys/class/udc/",
           "configfs setup not done yet", "/config/usb_gadget/g1/os_desc/b.1"
vendor/etc/init/android.hardware.usb@1.2-service-mediatekv2.rc
  service vendor.usb-hal-1-2 … class hal
```

**Confirmed:** on stock, `sys.usb.controller` is a runtime value produced by the MTK USB HAL;
`system/etc/init/hw/init.usb.configfs.rc` (AOSP) only *consumes* `${sys.usb.controller}`.

### 5.3 Recovery gap

`recovery/root/init.recovery.mt6765.rc` (whole file) sets only
`sys.usb.configfs 1`, `sys.usb.ffs.aio_compat 1`, writes
`/sys/class/udc/musb-hdrc/device/cmode 2`, and `start adbd`. It **mounts no configfs, creates no
`/config/usb_gadget/g1` gadget, mounts no functionfs, and writes no UDC**. `find recovery -iname '*usb*'`
returns nothing, and no file in the repo creates a gadget or sets `sys.usb.controller`
(confirmed by grep over `recovery/` and `*.mk`).

**Inferred:** in recovery the vendor USB HAL is absent, so `sys.usb.controller` is unset; any
generic recovery USB rc that writes `${sys.usb.controller}` to the UDC writes an empty value and the
gadget is never bound → no ADB/MTP. (The built ramdisk's generic `init.recovery.usb.rc` comes from
the TWRP common tree and is not present in this repo, so that file's content is **unknown**.)

**Unknown:** the effect of `write /sys/class/udc/musb-hdrc/device/cmode 2`. `cmode*` attributes exist
in the kernel and the `[MUSB] cmode=%d, cable_mode=%d` logging is present, but the CDP/SDP semantics
of value `2` were not verified against the driver.

## 6. Checks that require the device (cannot be decided offline)

1. `cat /proc/cmdline` in TWRP **and** in stock — must contain the `mipi_mot_vid_…` panel token, and
   it must match the SKU (`…icnl9916…` or `…ili9883a…`). If absent → touch root cause confirmed.
2. `ls /sys/class/udc/` in TWRP — expect `musb-hdrc`.
3. `ls /config/usb_gadget/g1/` and `cat /config/usb_gadget/g1/UDC` in TWRP.
4. `dmesg | grep -E 'CTS-I2CDrv|ILITEK|mipi_mot_vid_|musb'` — the driver gate prints its reason
   verbatim (`chipone active panel not found!` / `Not chipone panel!` / `Not ilitek panel!`).
5. Which SKU the unit is (Chipone vs Ilitek) — determines whether the missing
   `/vendor/firmware/ICNL9916.bin` in recovery matters.

## 7. Previously `TODO(verify)` rows — now resolved

| Item | Result |
|---|---|
| touch module vermagic vs prebuilt kernel | recovery modules = `…g2991a7c6cb9d` = prebuilt kernel ✓ (stock modules = `…gbaa59638869e`) |
| transitive dependency chain for touch modules | **none** (`modules.dep` has no deps for either; all imported symbols resolve) |
| `ro.hardware` value | still not needed: recovery uses `init.recovery.mt6765.rc`; stock `androidboot.hardware=mt…` in `/chosen/bootargs` |
| post-load sysfs setup for touch | none required by the modules themselves; selection is by cmdline token, not sysfs |
| `sys.usb.controller` | **never set by stock `.rc`**; set at runtime by the MTK USB HAL; value `musb-hdrc` (via `vendor.usb.controller`) |
| UDC name for MT6765 | **`musb-hdrc`** — confirmed (init.mt6765.usb.rc:65,109) |
| configfs gadget setup from stock `*usb*.rc` | captured in `docs/usb-stock-sequence.rc.txt` |
| ADB function ordering | `sys.usb.configfs 1` → gadget dirs → `sys.usb.ffs.ready=1` + `sys.usb.config=adb` → symlink `ffs.adb` → write UDC |
| mkbootimg args | v2, page 2048, cmdline `bootopt=64S3,32N2,64N2 buildvariant=user`, kernel `0x40080000`, ramdisk `0x51b00000`, tags `0x47880000` |
| `slotselect` in fstab | consistent with Virtual A/B (`lpdump` header flag) |
| prebuilt kernel version string | `Linux version 4.19.191-g2991a7c6cb9d` (comes out only after **two** gunzips) |
