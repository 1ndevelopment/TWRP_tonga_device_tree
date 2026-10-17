# Enabling decryption (TWRP crypto) on tonga

Audit + applied change set for `TW_INCLUDE_CRYPTO`. Every claim is tagged **confirmed**
(artifact path + literal text/code read), **inferred** (reasoning stated) or **unknown**
(needs the device). Firmware paths under `fw/work/fs/` are the stock extraction (gitignored);
all firmware strings quoted here were read from it.

## 1. What the device actually does (stock facts)

**/data is FBE, metadata-encrypted, with TEE-bound keys.** Confirmed,
`fw/work/fs/vendor_a/etc/fstab.mt6765:22`:

```
/dev/block/platform/bootdevice/by-name/userdata /data f2fs ... ,checkpoint=fs,fileencryption=aes-256-xts:aes-256-cts:v2,,keydirectory=/metadata/vold/metadata_encryption
```

* fstype **f2fs**, `fileencryption=aes-256-xts:aes-256-cts:v2`, `keydirectory=/metadata/vold/metadata_encryption`.
* `/metadata` is a separate 32 MiB partition (`md_udc`, ext4, `first_stage_mount`, no crypto options
  of its own) — `fstab.mt6765:19`; it only *holds* the key directory.
* Golang-free reading of the empty field between the two commas in `fstab.mt6765:22`: it is the
  positional metadata-cipher slot; TWRP ignores it (its flags are `;`-separated, see §3) and vold
  takes the real values from the stock fstab that TWRP re-parses (see §3).

**The TEE/keymaster is Trustonic.** Confirmed, `fw/work/fs/vendor_a/build.prop`:

| line | value |
|---|---|
| 279 | `ro.hardware.kmsetkey=trustonic` |
| 280 | `ro.vendor.mtk_trustonic_tee_support=1` |
| 281 | `ro.hardware.gatekeeper=trustonic` |
| 312 | `ro.crypto.volume.filenames_mode=aes-256-cts` |

and the matching binaries exist in `fw/work/fs/vendor_a/`:

| path | size | sha256 (first 16) |
|---|---|---|
| `bin/hw/android.hardware.keymaster@4.1-service.trustonic` | 106096 | `105e41fbd29b3a04` |
| `bin/hw/android.hardware.gatekeeper@1.0-service` | 11160 | `1a42b18a0496174b` |
| `bin/hw/vendor.mediatek.hardware.keymaster_attestation@1.1-service` | 11128 | `877c75031341a330` |
| `bin/mcDriverDaemon` (Trustonic Kinibi driver daemon) | 732384 | `ab42943c8c04ac1f` |

**Inferred:** decryption is impossible without the TEE. Reasoning: the keys live in
`/metadata/vold/metadata_encryption` and `/metadata` is flagged `wrappedkey`, i.e. the file encryption
keys are sealed by the keymaster HAL, which on this SoC is backed by the Trustonic TEE
(`ro.hardware.kmsetkey=trustonic`). A software keymaster (TWRP's own `libsoftkeymasterdevice.so`)
cannot unwrap them. This is why the vendor stack has to run *inside* recovery.

## 2. What TWRP android-12.1 needs (source, not folklore)

Verified by reading `TeamWin/android_bootable_recovery` **android-12.1**
(HEAD `5c3d206a5eeb3d446bcda8248a405a4b278bab5c`, clone kept in `/tmp/opencode/twrp-a121`):

* `Android.mk:335` `ifeq ($(TW_INCLUDE_CRYPTO), true)` → `:336` adds `-DTW_INCLUDE_CRYPTO -DUSE_FSCRYPT`,
  `:339` sets `TW_INCLUDE_CRYPTO_FBE := true`, `:340` adds `-DTW_INCLUDE_FBE`, `:358` adds
  `-DTW_INCLUDE_FBE_METADATA_DECRYPT`, and the block also pulls the keystore/keymint libraries.
  **One flag is enough**: `TW_INCLUDE_CRYPTO` is the only one to set; `TW_INCLUDE_CRYPTO_FBE`,
  `TW_INCLUDE_FBE` and `TW_INCLUDE_FBE_METADATA_DECRYPT` are derived (nothing reads them as make
  variables — `Android.mk:339` is their only writer).
* `partition.cpp:1099` `char source_separator = ';';` — an fstab whose first column is the mount point
  (`fstab_ver == 1`, our `recovery.fstab`) splits its flag column on **`;`**, not on `,`
  (`PARTITION_MANAGER` passes the version; AOSP-style lines that start with `/dev/` use `,`).
* `partition.cpp:188`/`:216`/`:217` — TWRP parses `fileencryption=`, `keydirectory=` and
  `wrappedkey` itself; `:1068-1070` sets `Key_Directory` from `keydirectory=`, which is what its
  metadata-decrypt path needs. `TWFLAG_WRAPPEDKEY` itself is a no-op (`:996`).
* `partitionmanager.cpp:409-441` — TWRP **mounts `/vendor` and `/odm` only while reading their props
  and the vendor fstab, then unmounts them**; `twrp.decrypt.done` is set at `:1978`. Consequence:
  HALs referenced as `/vendor/bin/...` cannot be started from the real vendor partition at decrypt
  time — their binaries must be in the ramdisk.
* `Get_Service_From_Manifest("/vendor", ...)` runs while vendor is still mounted, and our stock
  vendor has a real `etc/vintf/manifest.xml` with keymaster 4.1, so the keymaster version is detected
  normally.

## 3. Change set applied

1. **`BoardConfig.mk`** — `TW_INCLUDE_CRYPTO := true` (was `false`).
2. **`system.prop`** (new) + `TARGET_SYSTEM_PROP += $(DEVICE_PATH)/system.prop` in `device.mk`
   (`DEVICE_PATH` is set in `BoardConfig.mk:8`; same mechanism as the reference tree in §4):
   `ro.crypto.volume.filenames_mode=aes-256-cts` — stock's value (`vendor/build.prop:312`), read by
   `fw/work/fs/system_a/system/bin/vold`.
3. **`recovery/root/etc/recovery.fstab`** — `/data` gains, verbatim from stock `fstab.mt6765:22`:
   `;fileencryption=aes-256-xts:aes-256-cts:v2;keydirectory=/metadata/vold/metadata_encryption`
   (`;`-separated per §2). TWRP re-parses the stock vendor fstab as `/etc/additional.fstab` anyway,
   so this is the fallback that keeps the metadata path intact even if the vendor fstab is not found.
4. **`recovery/root/init.decrypt.rc`** (new, imported by `init.recovery.mt6765.rc`) — the four stock
   Trustonic services, stock's own service name/args/user/group/class/interface lines, plus
   `seclabel u:r:recovery:s0` (same domain as the existing `mtk.plpath.utils.link` service) and
   `disabled` (recovery starts them explicitly instead of via `class_start`).
5. **Blobs** under `recovery/root/vendor/` (3.17 MiB total), all copies of stock files, sha256-checked:

   * `bin/hw/` — the three services above; `bin/mcDriverDaemon`.
   * `lib64/` — the `readelf -d` closure of those binaries (30 files in `bin/`+`lib64/`):
     `android.hardware.{gatekeeper@1.0,keymaster@3.0,keymaster@4.0,keymaster@4.1}.so`,
     `lib{McClient,TEECommon,keymaster4,keymaster41,keymaster4support,keymaster4_1support,keymaster_messages,keymaster_portable,puresoftkeymasterdevice,soft_attestation_cert,cppbor_external,cppcose_rkp,kmsetkey}.so`,
     `vendor.mediatek.hardware.keymaster_attestation@1.{0,1}.so`.
   * `lib64/hw/` — the dlopened (not `NEEDED`) hardware modules, so that whichever variant the loader
     asks for is present, exactly as stock ships them: `android.hardware.gatekeeper@1.0-impl.so`,
     `gatekeeper.default.so`, `gatekeeper.trustonic.so`, `libMcGatekeeper.so`, `libSoftGatekeeper.so`,
     `kmsetkey.default.so`, `kmsetkey.trustonic.so`.
   * `app/mcRegistry/` — the 11 TAs of stock's 29-item `-r` list that this device's vendor image
     actually contains (245 KiB); the 18 missing ones are media/sensor/ISP TAs that upstream stock rc
     lists for other products. The rc's `-r` list is exactly those 11.

   The AOSP libraries the closure does not provide (`libbase, libc++, libc, libcrypto, libcutils,
   libdl, libhardware, libhidlbase, liblog, libm, libutils`) are shipped by TWRP's own recovery
   build — verified in the clone's `prebuilt/Android.mk` (`libhidlbase:369`, `libhardware:169`,
   `libutils:104`, `libbase:143`, `libcutils:101`, `libcrypto:138`, `libbinder:82`).

### Deliberate deviations from stock / from the reference tree

* Stock `trustonic.rc:2` passes `--sfs-reformat`. **Kept** — it is what stock runs on every boot, so
  it is the vendor's normal mode, not a recovery-specific risk.
* Stock starts these in `on late-fs` after `wait_for_prop hwservicemanager.ready "true"`
  (`init.mt6765.rc:188`). Recovery starts them on `hwservicemanager.ready=true` **and** on
  `crypto.ready=1`, the latter set in `on post-fs` *after* the `/mnt/vendor/persist` mount — both
  actions are idempotent, so whichever fires first wins.
* `--P1` is stock's path (`/mnt/vendor/persist/mcRegistry`), so `init.decrypt.rc` mounts `/persist`
  there itself (recovery.fstab mounts it at `/persist`). The reference tree in §4 does the same.
* **Not** copied from the reference tree: `ro.hardware.kmsetkey=trustonic`,
  `ro.hardware.gatekeeper=trustonic`, `ro.vendor.mtk_trustonic_tee_support=1` (no reader found for any
  of them in stock `bin/`, `lib64/`, `lib64/hw/` or `lib/` — the only hits for `ro.hardware.kmsetkey`
  are `liboemcrypto.so`/`libvcodecdrv.so`, i.e. DRM), `ro.crypto.support_metadata_encrypt=true`, and
  `keymaster_ver=4.1` + `TW_FORCE_KEYMASTER_VER` (the `keymaster_ver` property has **no verified
  consumer**; the string in `libkeymaster4_1support.so` is `keymaster_version`, a different thing).
  These are the first knobs to try if decryption fails.
* No `install_keyring` / `start vold`: neither exists in stock's `init.rc` nor in TWRP's recovery
  `init.rc` (grepped both), so there is no evidence they are needed here.

## 4. Reference implementation used for the shape

`Moto-Mediatek-Unified/android_device_motorola_penangf-twrp`, branch `android-12.1` — a Motorola
MT6768 with the same Trustonic stack and the same TWRP branch. Its `recovery/root/init.decrypt.rc`
is stock's Trustonic fragment plus recovery seclabels, and its `system.prop` carries the crypto/TEE
props; its `recovery/root/vendor/` holds the same class of files (services + `lib64` + `lib64/hw` +
`mcRegistry`). That confirmed both the approach (ramdisk-resident HALs, not `/vendor` paths) and the
candidate-module set.

## 5. Unverified — needs the device (see `TESTING.md`)

1. **Ramdisk/boot budget.** `boot_a` is **32 MiB** (`fw/out/PGPT`: `boot_a 65536 sectors @512`), stock
   `boot.img` is ~24 MiB, and this adds 3.17 MiB plus TWRP's own crypto stack. **Check the built
   `recovery.img` size before flashing**; if it does not fit, drop `lib64/libpuresoftkeymasterdevice.so`
   and `lib64/libsoft_attestation_cert.so` (software fallbacks, 92 KiB) first, then the media TAs in
   `app/mcRegistry` (the hdcp/widevine/m4u/H264 ones, 172 KiB).
2. **Which gatekeeper module the loader picks** (`gatekeeper.default.so` vs `gatekeeper.trustonic.so`)
   is decided inside `libhidlbase`/`libhardware` at runtime; no prop name and no class-name string
   exists in the service binary, and no prop reader was found in vendor libs, so it cannot be settled
   offline. Both candidates ship, as in stock.
3. **Whether the 11-TA `-r` list is sufficient for the keymaster TA.** The `sec`/`utils`/`cmdq` ones
   are present (`0512…`, `020f…`, `020b…`); the removed 18 are media/sensor TAs. If `mcDriverDaemon`
   proves to require the full list, the remaining TAs are not in this device's vendor image at all
   (only the 11 exist here).
4. **SELinux**: recovery domains must be able to talk to `/dev/mobicore` and the TEE. `dmesg | grep -i
   avc` will show it if not; TWRP recovery is permissive in the usual configuration.
5. `--sfs-reformat` touches Trustonic secure storage on `/persist` (see §3); it is stock's own
   invocation, but it is the one line not to keep if the device ever misbehaves around `/persist`.
