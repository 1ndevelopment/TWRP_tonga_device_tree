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

Corroborated by Moto's own per-panel configs in the stock vendor image,
`etc/motorola/12m/`: `chipone-tianma_icnl9916_hdp_vid-ICNL9916-tonga.ini`,
`chipone-txd_icnl9911c_hdp_vid-ICNL9911C-tonga.ini`, `TM655_driver_Command_ilitek_tm.ini` — the same
two families and nothing else, so **there is no third (novatek) SKU** to support (inferred from
these two independent sources). Those `.ini` files are factory-test configs and are **not** read at
probe time: neither module references `/vendor/etc/motorola`.

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

### 4.6 What this change did, and the residual unknowns

State of `recovery/root/` (touch):

| item | before | after |
|---|---|---|
| `vendor/lib/modules/ili9882_mmi.ko`, `icnl9916_tddi_mmi.ko` | present, relinked for the recovery kernel | **unchanged** — already correct |
| `vendor/firmware/ICNL9916.bin`, `ICNL9911.bin` | **absent** | added, byte-identical to stock (sha256 in `docs/touch-firmware-files.txt`) |
| `init.recovery.mt6765.rc` | `insmod` both modules under `on boot`; comment claimed load order mattered | same trigger and commands; comment corrected; diag service now also logs `/proc/cmdline` and greps `version magic` |

The two drivers themselves were already in the device tree, so none were added. Two properties of
that placement were verified rather than assumed:

1. **The files do reach the built recovery ramdisk.** TWRP's build copies
   `$(TARGET_DEVICE_DIR)/recovery/root` into it — `TeamWin/android_build` (android-12.1)
   `core/Makefile:2073-2080` and the copy at `:2258`, `cp -rf $(item) $(TARGET_RECOVERY_OUT)/`,
   where `TARGET_RECOVERY_ROOT_OUT = $(TARGET_RECOVERY_OUT)/root` (confirmed from source). The
   commented-out `PRODUCT_COPY_FILES` line at `twrp_tonga.mk:34` is therefore redundant, not a defect.
2. **`/vendor/firmware` is the firmware search path.** The stock DTB's `/chosen/bootargs` ends with
   `firmware_class.path=/vendor/firmware`, so the ramdisk copies are exactly what `request_firmware`
   resolves (confirmed).

Residual unknowns — none of these can be settled from the firmware dump:

- **The cmdline gate (§4.4) is probably satisfied.** The `mipi_mot_vid_*` token is in none of the
  partitions this tree writes, so it must be appended by LK, and the same stock LK boots both stock
  Android and TWRP. Still `unknown` until `cat /proc/cmdline` is read.
- **/vendor shadowing.** `recovery.fstab:3` mounts a `vendor` partition at `/vendor`. If TWRP mounts
  it before the `insmod` lines run, the ramdisk copies are hidden and the *stock* modules
  (vermagic `gbaa59638869e`) get loaded against the recovery kernel (`g2991a7c6cb9d`), which fails
  with a version-magic error. The diag service now greps `version magic` to catch exactly this.
- **Load timing.** Stock loads this same module set from `init.mmi.overlay.rc` under `on early-init`;
  the tree loads it under `on boot`. Left unchanged: a bare `insmod` is proven only at `on boot`,
  and TWRP is expected to pick up a late-registering input device. (inferred, unverified)
### 4.7 Probe path — there is no second gate (REA, both modules)

Full detail: `fw/work/probe-path-notes.md`. What runs **after** the cmdline gate:

`icnl9916` (`cts_driver_probe` → `cts_probe_device`): workqueue alloc → DT `chipone,irq-gpio`/`rst-gpio`/
`x-res`/`y-res` → `input_register_device("chipone-tddi")` → `gpio_request_one("chipone-tddi-int"/
"-rst")` → SPI read of the chip hwid in program mode, which must match an entry of
`cts_device_hwdatas[]` (`0x990100`, `0x990110`, `0x991110`, `0x990160`, matched by hwid **or** fwid) →
`fb_register_client` → `request_threaded_irq("chipone-tddi")`. Every one of those is fatal. The tool
node `/proc/icn85xx_tool`, the sysfs class, the charger-detect path and the firmware request are
warnings or post-bind only — in particular `cts_request_firmware` is only reached from
`cts_firmware_upgrade_work`, queued *after* probe returns 0, which confirms §4.5's inference.

`ili9882` (`ilitek_spi_probe` → `ilitek_plat_probe` → `ili_tddi_init`): the DRM gate
(`of_drm_find_panel`) only runs if the touch node has `panel`/`panel2` phandles — it does not (§4.2),
so that path is **inert on this board** → full-duplex SPI check → `ili_core_spi_setup` → DT
`touch,irq-gpio`/`reset-gpio` + `gpio_request("TP_INT"/"TP_RESET")` + `devm_request_threaded_irq
("ilitek")` → `ili_ic_dummy_check()` → FW kthread. Fatal, same as above; firmware, sysfs, netlink and
pinctrl (`pmx_ts_int_*`, absent from the node) are optional.

**Confirmed:** no gate exists that a normal boot satisfies and recovery does not — both boots use the
same bootloader-applied `dtbo` and both kernels have the needed features (`mediatek,mt6765-spi` is
present in `Image.prebuilt`). The only recovery-specific failure points are:

1. the cmdline token (§4.4) — the only plausible one;
2. the TDDI chip having to answer a SPI read in program mode, i.e. panel power/reset state left by
   the bootloader — not judgeable offline;
3. GPIO `&pio` 10 / `0xa4` already claimed — nothing else in this ramdisk claims them (the two touch
   modules are the only added consumers; `touch-diag` only reads dmesg).

One firmware nuance worth knowing: `is_filesystem_mounted("/vendor/firmware/ICNL9916.bin")` compares
the file's vfsmount with `/`, so the icnl firmware update only runs when `/vendor` is a **separate
mount** — in recovery, i.e. only if TWRP mounts the vendor partition, which is also the condition that
would shadow the ramdisk modules (§4.6). Either way it is non-fatal: it is queued work, not a bind gate.

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

### 5.3 Recovery gap — RESOLVED

`recovery/root/init.recovery.mt6765.rc` originally set only `sys.usb.configfs 1`,
`sys.usb.ffs.aio_compat 1`, wrote `/sys/class/udc/musb-hdrc/device/cmode 2` and `start adbd`.

TWRP's own recovery rc was fetched and read (`TeamWin/android_bootable_recovery`, android-12.1,
`etc/init.rc`), so the previously-unknown part is now confirmed: recovery **already** has the whole
configfs path — `on fs && property:sys.usb.configfs=1` mounts configfs and builds
`/config/usb_gadget/g1` with `functions/ffs.adb` and `configs/b.1` (`:127-138`), `on fs` mounts
functionfs at `/dev/usb-ffs/adb` (`:149-151`), and `on property:sys.usb.config=adb &&
property:sys.usb.ffs.ready=1 && property:sys.usb.configfs=1` symlinks `ffs.adb` into config `b.1`
and writes the UDC (`:201-206`).

The only missing inputs were the two properties the stock vendor USB HAL used to provide:

- **`sys.usb.controller`** — nothing in recovery sets it, so the UDC write above resolved to an
  empty name and no UDC was ever bound. Stock's own rc uses `vendor.usb.controller "musb-hdrc"`
  (`init.mt6765.usb.rc:65`) and `factory_init.rc` writes `musb-hdrc` literally.
- **`sys.usb.config=adb`** — TWRP sets it only when `ro.debuggable=1` (`init.rc:116`), so release
  builds never triggered the attach.

Both are now set in `on post-fs` (post-fs, not `on fs`, because TWRP imports the device rc at
`init.rc:6`, i.e. *before* its own `on fs` blocks build the gadget and mount the functionfs).
Nothing else was added — mounting configfs, the gadget, the ffs mounts and the bind already exist.

**Also applied:** `start adbd` moved from `on fs` to the end of `on post-fs`. This file is imported
at TWRP `init.rc:6`, so its `on fs` actions run before TWRP mounts the adb functionfs (`:149-151`);
adbd started earlier cannot open its functionfs and exits.

**Not changed:** the CDP/SDP meaning of `cmode 2` remains unverified. `musb_hdrc.kernel_init_done`
(stock `init.mt6765.usb.rc:2`, `on early-init`) is deliberately **not** written: decompilation of the
recovery kernel shows it does not gate the UDC — its only readers are six loads in the MUSB
host-role worker, while `/sys/class/udc/<name>` comes from `usb_add_gadget_udc()` in the `musb-hdrc`
platform probe, which never reads it (`docs/musb-kernel-init-done.md`; re-verified in
`usb-stock-sequence.rc.txt` §4). Device checks:
`ls /sys/class/udc/`, `cat /config/usb_gadget/g1/UDC`, `dmesg | grep -iE 'musb|udc|gadget'`.

## 6. Checks that require the device (cannot be decided offline)

1. `cat /proc/cmdline` in TWRP **and** in stock — must contain the `mipi_mot_vid_…` panel token, and
   it must match the SKU (`…icnl9916…` or `…ili9883a…`). If absent → touch root cause confirmed.
   Now also captured automatically in `/tmp/touch_dbg.log` by the `touch-diag` service.
2. `cat /tmp/touch_dbg.log` in TWRP — the drivers print their own reason verbatim:
   `CTS-I2CDrv chipone active panel not found!` / `Not chipone panel!` (icnl9916) and the
   `ILITEK: … ilitek active panel not found!` / `Not ilitek panel!` equivalents, or the hardware
   failures `CTS-Core Device hwid: %06x fwid: %04x not found`, `CTS-Core Enter program mode failed`,
   `ILITEK Driver probe failed`. Which of these appears decides which of the ranked causes it is.
3. `ls /sys/class/udc/` in TWRP — expect `musb-hdrc`.
4. `ls /config/usb_gadget/g1/` and `cat /config/usb_gadget/g1/UDC` in TWRP.
5. Which SKU the unit is (Chipone vs Ilitek) — both are now fully supplied in recovery, so this only
   tells you which firmware was meant to be used.

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
