# `musb_hdrc.kernel_init_done` — what it gates, and does 0 block `/sys/class/udc/musb-hdrc`

Target: `fw/work/boot/Image.prebuilt` (recovery kernel, raw arm64 `Image`, kernel 4.19.191, MTK MUSB).
All file offsets below are offsets **inside `Image.prebuilt`**.

> Mapping correction (empirically verified, not the value in the brief):
> the brief's `VA = 0xffff800080080000 + file_offset` is missing two `f`s. The real, verified mapping is
> **`VA = 0xffffff8008080000 + file_offset`**.
> Proof: `Image` header at file 0 has `magic = "ARMd"` (0x644d5241) and `text_offset = 0x80000`; pointers stored in
> `.data` (e.g. the literal `0xffffff8008081800` repeated at file 0x5800) show the loaded prefix is `0xffffff80_……`,
> and the string pointer search below only matches with base `0xffffff8008080000`.

---

## 1. Verdict

**confirmed.** `musb_hdrc.kernel_init_done` is a plain 32-bit `int` module parameter (`perm 0644`, `level -1`) of the
built-in MUSB driver. In the whole 26 MB image its value is read by **exactly one function**, the MUSB host-role worker
`do_host_work` (VA `0xffffff80089d3444`; the name is the driver's own `__func__` string at VA `0xffffff800950dc98`), where
it is one of two early-exit conditions of a `while`-loop that waits for the "USB ready" state before proceeding
(the driver logs it literally as `init_done:%d`). Writing `1` to the sysfs node only stores the integer via the generic
`param_set_int()` (a `kstrtoint()` into `kp->arg`) — it does not notify, rescan, re-probe or touch any device.
`kernel_init_done` does **not** gate UDC registration: the sysfs gadget node is created by `usb_add_gadget_udc()`
(single call site, VA `0xffffff80089e6d58`), reached from the `musb-hdrc` platform driver's **probe** (VA
`0xffffff80089da7e4`), and that probe path never reads the variable. Therefore, in a recovery build where nothing writes
`1`, **`/sys/class/udc/musb-hdrc` is still created by the driver probe**; the parameter is not a precondition for it.
(The node's existence still depends on the MUSB probe succeeding at all — clock/`usb_psy` readiness, which is a separate
mechanism that does not consult `kernel_init_done`.)

Confidence: param-table entry, ops/set/get, the exhaustive reader set, and the single `usb_add_gadget_udc` call site are
all directly evidenced below (**confirmed**). The runtime consequence of skipping the host-work wait loop, and the exact
ordering of `usb_add_gadget_udc` versus that loop at boot, are **inferred** from control flow (no device/console trace
was available).

---

## 2. The parameter-table entry and the ops it points at

Locate the `__param` entry by searching the raw image for the little-endian pointer to the name string
(`"musb_hdrc.kernel_init_done"` at file `0x1162AC0` → VA `0xffffff80091e2ac0`). Exactly one 8-byte match exists:

```
$ python3  # search struct.pack('<Q', 0xffffff80091e2ac0)
hits at file off ['0x15c7070']
```

`struct kernel_param` (arm64 4.19, 40 bytes: name@0, mod@8, ops@16, perm@24 u16, level@26 s8, flags@27 u8, arg@32):

```
entry file offset : 0x15c7070   (VA 0xffffff8009647070)
raw 40 bytes      : c02a1e0980ffffff 0000000000000000 3094070980ffffff a401ff00 00000000 c4bebf0980ffffff
  name  = 0xffffff80091e2ac0  -> "musb_hdrc.kernel_init_done"
  mod   = 0x0                 (built-in, no struct module)
  ops   = 0xffffff8009079430
  perm  = 0644   level = -1(0xff)   flags = 0
  arg   = 0xffffff8009bfbec4   (file off would be 0x1b7bec4 > image size 0x198e000 -> it is in .BSS)
```

The module parameter's storage is therefore the 4-byte int at **VA `0xffffff8009bfbec4`** (BSS page `0xffffff8009bfb000`,
offset `0xec4`). Its address appears **nowhere else** in the image as data — the only 8-byte occurrence of
`0xffffff8009bfbec4` is the `arg` field itself (file `0x15c7090`), so no code ever takes `&kernel_init_done` indirectly.
Every access is therefore a direct `adrp`+`ldr/str`, which makes the reader search in §3 exhaustive.

`ops = 0xffffff8009079430` decodes as `struct kernel_param_ops { u32 flags; set; get; free; }`:

```
file 0xff9430: 00000000 00000000 | 00ad0d0880ffffff | 34ad0d0880ffffff | 0000000000000000
  flags = 0 ; set = 0xffffff80080dad00 ; get = 0xffffff80080dad34 ; free = NULL
```

Disassembly of those two functions (inferred identity: the generic `param_set_int` / `param_get_int` pair — behavior
confirmed by the code):

```
ffffff80080dad00 <set>:
  stp  x29, x30, [sp, #-0x20]!
  stp  x20, x19, [sp, #0x10]
  mov  x29, sp
  mov  x19, x1              ; x19 = kp
  mov  x20, x0              ; x20 = val
  bl   0xffffff800809f01c   ; __fentry__ (ftrace trampoline, not semantics)
  ldr  x2, [x19, #0x20]     ; x2 = kp->arg  (== &kernel_init_done)
  mov  x0, x20              ; val
  mov  w1, wzr              ; base = 0
  bl   0xffffff80084b1248   ; kstrtoint(val, 0, kp->arg)   <-- stores the int
  ldp  x20, x19, [sp, #0x10]
  ldp  x29, x30, [sp], #0x20
  ret                       ; returns kstrtoint()'s result; nothing else

ffffff80080dad34 <get>:
  ... ldr x8,[x19,#0x20] ; adrp x2,<"%d"> ; ldr w3,[x8] ; bl <snprintf wrapper> ...
  (writes kp->arg as "%d")
```

`0xffffff800809f01c` is the standard ftrace `__fentry__` call that opens almost every function in this image (also at
the top of the readers in §3); it is patched to a NOP at boot and has no bearing on behavior.

---

## 3. Every reader of the variable

#### Method
Full `llvm-objdump-19` disassembly of the image (wrapped in a synthetic AArch64 ELF, GNU objdump here has no aarch64
target). Then: every memory operand of the form `[xR, #0xec4]` was matched with the nearest prior definition of `xR`
(a 6000-instruction window); a hit counts only when `xR` was defined by `adrp xR, 0xffffff8009bfb000`. Result: **6
accesses, all in one function, all loads, zero stores.**

#### The function
Start `0xffffff80089d3444` (its prologue `sub sp, sp, #0x80`); it is the DRIVER's `do_host_work`
(debug string `"do_host_work"` at VA `0xffffff800950dc98`, passed as the `%s` argument of
`"[MUSB]%s %d: init_done:%d, is_ready:%d, inited:%d, TO:%d, diff:%lld, skip_cnt<%d>\n"` at VA `0xffffff8009480fd1`).
Its address is stored in a table of function pointers at VA `0xffffff80092a72d8` (indirect dispatch), consistent with a
work/scheduler callback rather than a driver `.probe`.

Entry gate (the decisive use):

```
ffffff80089d346c: adrp x8, 0xffffff8009bfb000
ffffff80089d3470: adrp x28, 0xffffff8009bfb000
ffffff80089d3474: ldrb w9, [x8,  #0xe74]      ; "inited"
ffffff80089d3478: ldr  w8, [x28, #0xec4]      ; <-- kernel_init_done
ffffff80089d347c: tbnz w9, #0x0, 0xffffff80089d36ec   ; if inited&1 -> proceed
ffffff80089d3480: cbnz w8,       0xffffff80089d36ec   ; if kernel_init_done != 0 -> proceed
ffffff80089d3484: ...                            ; else enter the wait loop
```

Loop tail (re-checks the same flag after each sleep):

```
ffffff80089d3564: mov  w0, #0x1e
ffffff80089d3568: bl   0xffffff8008149128     ; msleep(30)
ffffff80089d356c: adrp x8, 0xffffff8009bfb000
ffffff80089d3570: ldrb w9, [x8,  #0xe74]      ; inited
ffffff80089d3574: ldr  w8, [x28, #0xec4]      ; <-- kernel_init_done
ffffff80089d3578: tbnz w9, #0x0, 0xffffff80089d36ec
ffffff80089d357c: cbz  w8, 0xffffff80089d34bc ; if 0 -> loop again
ffffff80089d3580: b   0xffffff80089d36ec     ; else -> proceed
```

Proceed block (loop exit) `0xffffff80089d36ec`:

```
ffffff80089d36ec: ldur x23, [x29, #-0x8]      ; saved argument (a per-call context)
ffffff80089d36f0: tbz  w9, #0x0, 0xffffff80089d3700
ffffff80089d36f4: b    0xffffff80089d3724
ffffff80089d3718: mov  w8, #0x1
ffffff80089d371c: adrp x9, 0xffffff8009bfb000
ffffff80089d3720: strb w8, [x9, #0xe74]       ; set "inited" = 1
ffffff80089d3724: mov  w0, #0x1
ffffff80089d3728: bl   0xffffff80089d00b0     ; MUSB state action
   ... reads mtk_musb ([page+0xef0]) and a per-context state field [x23+0x88], etc.
```

The remaining four reads are inside the loop's **debug-logging / timeout** branches; they only feed log arguments, they
do not change control flow:

| instruction (VA) | what it feeds |
|---|---|
| `ffffff80089d3590: ldr w3, [x28, #0xec4]` | `init_done` arg of the `"...skip_cnt<%d>"` log (`[MUSB]%s %d: init_done:%d, …`) |
| `ffffff80089d3604: ldr w4, [x28, #0xec4]` | `init_done` arg of the `"diff_time:%lld"` / `"…diff:%lld<unlimit>"` log |
| `ffffff80089d3650: ldr w4, [x28, #0xec4]` | same, second branch |
| `ffffff80089d36c0: ldr w3, [x28, #0xec4]` | same, third branch |

#### What the value selects
Not a probe-deferral condition, not UDC registration, not a pullup, not log silence. With `kernel_init_done != 0` the
function **skips the "wait until USB is ready" loop** and jumps straight to the `proceed` block (`0xffffff80089d36ec`),
which sets its own `"inited"` flag (`[page+0xe74]`) and runs the next MUSB state action. With `kernel_init_done == 0`
the function instead stays in the loop, where it re-checks the "is_ready" bit (`[mtk_musb + 0x223d]`), a `"TO"` flag
(`[page+0xe78]`) and a `diff` timeout (`cmp x4, #0x7531` = 30001 at `0x89d353c`), sleeping 30 (ms) per iteration.
So `kernel_init_done` is an alternative "go now" signal for the host-work readiness wait.

No other function in the image reads or writes it (verified with the 6-hit, single-function result above, plus the fact
that `&kernel_init_done` is never stored anywhere except the `__param` entry).

---

## 4. Behavior of the sysfs store handler when `1` is written

```
echo 1 > /sys/module/musb_hdrc/parameters/kernel_init_done
```

dispatches to `ops->set = 0xffffff80080dad00`, i.e. the generic `param_set_int()` shown in §2: it calls
`kstrtoint("1", 0, kp->arg)` and returns. **It only writes the integer.** There is no notifier call, no
`device_attach` / `driver_attach` / `bus_rescan_devices` / `usb_gadget_probe_driver` / re-probe, and no workqueue kick —
the handler is 0x34 bytes long and its only call besides `__fentry__` is `kstrtoint`. Whatever reacts to the flag does so
later, by polling it (the `do_host_work` loop of §3 re-reads it after every `msleep(30)`).

---

## 5. In a recovery build that never writes `1`, does `/sys/class/udc/musb-hdrc` exist?

**Yes — expected to exist, and `kernel_init_done` is not what controls it.**

The `/sys/class/udc/<name>` node is created by `usb_add_gadget_udc()` (gadget core), via the `udc` class device add.
Resolve it through `__ksymtab`: the exported-name string `usb_add_gadget_udc` is at file `0x15ae373`; its
`__ksymtab` entry (3×`s32` PREL32 at file `0x15787b0`) yields

```
usb_add_gadget_udc  ->  VA 0xffffff8008ac8b7c
```

There is **exactly one call site** in the whole image:

```
grep 'bl 0xffffff8008ac8b7c'  ->  ffffff80089e6d58
ffffff80089e6d50: ldr  x0, [x19, #0x3b8]   ; parent dev
ffffff80089e6d54: mov  x1, x20             ; x20 = x19 + 0x1db0  (the struct usb_gadget)
ffffff80089e6d58: bl   0xffffff8008ac8b7c  ; usb_add_gadget_udc(parent, gadget)
```

That call lives in a helper function starting at `0xffffff80089e6c50`, whose one caller is `0xffffff80089db7b4`
(inside the region `0xffffff80089da7e4 … 0xffffff80089dc0d8`). The very same helper sets the gadget name:

```
ffffff80089e6c80: adrp x10, 0xffffff80091e2000
ffffff80089e6c84: add  x10, x10, #0xd40      ; x10 = "musb-hdrc"  (str at VA 0xffffff80091e2d40)
ffffff80089e6c88: str  x9,  [x19, #0x1e08]
ffffff80089e6c90: str  x10, [x19, #0x1e18]   ; name inside gadget (gadget base x19+0x1db0, +0x68)
```

and `0xffffff80089da7e4` is the **probe** of the `musb-hdrc` `platform_driver`
(`platform_driver` struct at VA `0xffffff80098fcdb8`: `probe = 0xffffff80089da7e4`,
`remove = 0xffffff80089dc0d8`, `driver.name = "musb-hdrc"`, `driver.bus = 0xffffff800989e530`).
So the chain is: platform probe `0xffffff80089da7e4` → helper `0xffffff80089e6c50` (sets gadget name
`"musb-hdrc"`) → `usb_add_gadget_udc()` → `/sys/class/udc/musb-hdrc`.

**What settles it:**
1. `kernel_init_done` has exactly one reader (`do_host_work`, §3) and its address is never aliased (§2); therefore no
   function on the probe/`usb_add_gadget_udc` path consults it.
2. `usb_add_gadget_udc()` has exactly one call site (§5), and it is in the platform-driver probe, not in `do_host_work`.
   `do_host_work` and the probe are separate functions (`0xffffff80089d3444` vs `0xffffff80089da7e4`).

**What it does *not* settle:**
- It does not prove the MUSB probe always *succeeds*. The image contains `"[MUSB]%s %d: clk not ready"` and
  `"[MUSB]%s %d: usb_psy not ready"` plus the core `"%s requests probe deferral"`; those are real deferral conditions
  of this driver, but they are checked elsewhere and, per §3, `kernel_init_done` is not among them.
- It does not establish exact boot ordering (whether the probe ran before or after any later write of `1`). It only
  establishes that the node's creation is not conditioned on the flag.
- It does not cover the stock `Image`: `Image` and `Image.prebuilt` are different builds (the `__param` entry bytes
  differ, and ~10 M bytes differ overall), although the reader function, the `ops` set/get, the `usb_add_gadget_udc`
  call site and the `.modinfo` string block are byte-identical at the same offsets. All conclusions above are for
  **`Image.prebuilt`** (the recovery kernel). The same reader/UDC structure was re-verified byte-identical in `Image`
  for those specific regions, so the conclusion is very likely the same there too (**inferred**).

---

### Quoted evidence (exact)

```
file off 0x15c7070  __param kernel_param  name=0xffffff80091e2ac0 mod=0 ops=0xffffff8009079430 perm=0644 level=-1 flags=0 arg=0xffffff8009bfbec4
string  0x1162ac0  "musb_hdrc.kernel_init_done"
string  0x1162d40  "musb-hdrc"                     (platform driver .name, pointer held in .data at file 0x187cde0)
ops     0xffffff8009079430 {set=0xffffff80080dad00, get=0xffffff80080dad34}
set     0xffffff80080dad00  ldr x2,[x19,#0x20]; mov x0,x20; mov w1,wzr; bl 0xffffff80084b1248   ; kstrtoint(val,0,kp->arg)
reader  0xffffff80089d3478  ldr w8,[x28,#0xec4]   (adrp x28,0xffffff8009bfb000 @0x89d3470)
reader  0xffffff80089d3574  ldr w8,[x28,#0xec4]
reader  0xffffff80089d3590  ldr w3,[x28,#0xec4]
reader  0xffffff80089d3604  ldr w4,[x28,#0xec4]
reader  0xffffff80089d3650  ldr w4,[x28,#0xec4]
reader  0xffffff80089d36c0  ldr w3,[x28,#0xec4]
udc     0xffffff80089e6d58  bl 0xffffff8008ac8b7c  (= usb_add_gadget_udc), x0=[musb+0x3b8], x1=musb+0x1db0
probe   0xffffff80089da7e4  (musb-hdrc platform_driver .probe)
```
