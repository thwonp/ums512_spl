# ums512_spl — Unisoc sharkl5pro (ums512_1h10) first-stage loader

Builds the SPL (first stage / chipram loader, also called fdl1) for the Unisoc
ums512_1h10 (sharkl5pro / T618) platform from source and wraps it into a
flashable `spl_a` style image: a 512 byte DHTB header, the SPL payload, padded
with zeros to 4 MiB. The two slots `spl_a` and `spl_b` hold identical SPLs, so
the build writes both.

The device this targets is the Anbernic style ums512_1h10 board used by the
GammaOS / rotate builds — in practice the Anbernic RG Rotate.

## Provenance

```
TheGammaSqueeze/ums512_spl      original self-contained build tree
  └── beebono/ums512_spl        rotate fixes (efuse/seg7, SD-boot hardening)
        └── crackerjacques/ums512_spl   this fork, branch `armbian`
```

- **[TheGammaSqueeze/ums512_spl](https://github.com/TheGammaSqueeze/ums512_spl)**
  is the upstream: it vendored Unisoc's `chipram` bootloader, reimplemented the
  A/B slot selection from the stock binary, and added the SD-card boot path.
- **[beebono/ums512_spl](https://github.com/beebono/ums512_spl)** (Noxwell)
  contributed the efuse/SEG7 firewall fixes and the SD-boot hangs, merged
  upstream as PR #6.
- This fork adds the unsigned **`open`** variant, drops `tools/`, and keeps the
  2015-era vendor C building under GCC 14. Nothing else is changed.

The full history of all three is preserved in this repository; `git log` gives
the per-commit authorship.

## License

The vendored `chipram/` tree is Unisoc's chipram bootloader, which is derived
from **U-Boot** and carries U-Boot's terms: `SPDX-License-Identifier: GPL-2.0+`
on the files that have an SPDX tag, and the "either version 2 ... or (at your
option) any later version" boilerplate on the rest. So the tree as a whole is
**GPL-2.0-or-later**, and the changes in this fork are offered under the same
terms.

`COPYING` is the GNU General Public License version 2, verbatim from U-Boot's
`Licenses/gpl-2.0.txt`. Individual files keep whatever notice they already
carry; where a file says otherwise, that file's own notice wins.

## What the `armbian` branch is

This branch carries **no `tools/`**. The upstream tree vendors Unisoc's
`imgheaderinsert` / `sprd_sign` binaries and the `rsa2048_*` signing keys; a
public tree that a distribution builds from should not carry a device signing
key, and it does not need to:

- Armbian builds only the **`open`** variant, which is unsigned and packed by
  the in-tree `scripts/dhtb_pack.py`.
- The `signed`, `signed-open` and `secure`-with-signing paths therefore fail
  here with a clear error. Use upstream
  [beebono/ums512_spl](https://github.com/beebono/ums512_spl) if you need them.

This is the branch the Armbian build framework pins and builds; see
`config/sources/families/sprd-ums512.conf` in
[armbian/build](https://github.com/armbian/build).

## Build

```sh
# one time, on Debian or Ubuntu
./scripts/setup-toolchain.sh

# the variant Armbian ships
./build.sh open

# or every variant this branch can build
./build.sh            # == nosec secure open
```

Results:

```
out/open/spl_a_open.img       out/open/spl_b_open.img
out/nosec/spl_a_nosec.img     out/nosec/spl_b_nosec.img
out/secure/spl_a_secure.img   out/secure/spl_b_secure.img
```

Flash the variant you want to the `spl_a` and `spl_b` partitions in the eMMC
boot area. The only external dependency is an aarch64 cross compiler.

### Variants

| variant | `CONFIG_SECBOOT` | RSA image checks | SPL signed | boots Linux | boots stock Android | in this branch |
| ------- | ---------------- | ---------------- | ---------- | ----------- | ------------------- | -------------- |
| `open`        | on  | **stubbed** | no  | yes | **yes** | **yes — this is the one to use** |
| `nosec`       | off | n/a         | no  | yes | no (secure world never comes up) | yes, reference only |
| `secure`      | on  | enforced    | no  | only stock-signed images | yes | yes, for inspection |
| `signed`      | on  | enforced    | yes | only stock-signed images | yes | no (needs `tools/`) |
| `signed-open` | on  | stubbed     | yes | yes | yes | no (needs `tools/`) |

**`open` is the variant to use, and the one Armbian ships.** It is the same SPL
as `signed-open` — `CONFIG_SECBOOT` on, so the secure-DDR firewall and the
secure-world bring-up happen exactly as in stock, with the four RSA image checks
(teecfg / sml / trustos / uboot) stubbed so a self-built U-Boot boots — but it is
**unsigned**, and packed by `scripts/dhtb_pack.py` instead of the vendor tooling.

It is accepted because **this device's BootROM checks the DHTB payload hash, not
the RSA signature**. That was established on hardware: the known-good
`spl_a_patched.img` carries the *stale* signature copied verbatim from stock over
*different* code, with only the DHTB hash recomputed, and it boots. Verified on
an Anbernic RG Rotate: Linux boots from the SD card, and with the card out the
stock Android on eMMC still boots. `nosec` does **not** manage the latter — it
skips the secure-world bring-up, so Android loops.

If your unit *is* secure-boot fused, the BootROM will reject an unsigned SPL and
you need `signed-open` from upstream. Know your unit's fuse state before
flashing. Recovery from a bad SPL is normally still possible through the Unisoc
BootROM download mode.

Two independent things, to keep them straight:

- **Signing the SPL image** (the SIMGHDR + RSA block) decides whether the
  **BootROM accepts the SPL**. Only matters on a fused unit. Key `rsa2048_0`.
- **`CONFIG_SECBOOT`** turns on the SPL's own secure-world bring-up: the
  `sprd_firewall_config_pre` secure-DDR firewall setup (`sml_teecfg_sec`,
  `dmc_sec`, ...) that the SML/trustos secure world needs, plus the RSA checks of
  the images it loads. The stock SPL is a `CONFIG_SECBOOT` build. A build with it
  **off** skips the firewall setup and the SML handoff hangs, which is why
  `nosec` does not complete a stock boot.

## Alternative: binary-patch the stock SPL

If you only need stock behaviour with a patched U-Boot — no SD-card boot — you
can patch the stock image instead of building one. `scripts/patch_stock_spl.py`
finds the four `bl secboot_verify ; cbz w0 ; mov w0,#5 ; bl error` sequences (by
the invariant `cbz w0 ; mov w0,#5` pair), NOPs them, and recomputes the DHTB and
SIMGHDR hashes. Its output is **byte-for-byte identical** to the known-good
on-device `spl_a_patched.img`.

```sh
# you provide the stock spl_a.img pulled from the device or a PAC
python3 scripts/patch_stock_spl.py stock_spl_a.img spl_open.img
# flash spl_open.img to both spl_a and spl_b
```

The stock SPL already does everything correctly (DDR init, the secure-DDR
firewall, A/B slot selection from misc); the only change needed to run a patched
or self-built U-Boot is to disable the four RSA image checks. What this path does
**not** give you is SD-card boot — that is added by the from-source build.

## Layout

```
chipram/        Vendored Unisoc chipram bootloader source (the SPL lives in
                chipram/nand_spl, sharkl5pro board code under
                chipram/nand_spl/board/spreadtrum/sharkl5pro).
board/          Board config fragments appended to the generated config.h,
                one per build variant: nosec, secure, open, diag (open plus
                on-device diagnostics). See "Where the config comes from".
scripts/        setup-toolchain.sh, dhtb_pack.py (portable DHTB packer),
                stock_frame.py, patch_stock_spl.py, parse_spl_trace.py.
build.sh        Top level build driver.
COPYING         GPL-2.0.
```

(Upstream additionally has `tools/` with the vendor `imgheaderinsert`,
`imgheaderinsert_secure`, `sprd_sign` and the signing keys. This branch does
not — see "What the `armbian` branch is" above.)

## Where the config comes from

The board config header `chipram/include/configs/ums512_1h10.h` does not by
itself define the secure memory map (`CONFIG_SML_LDADDR_START`,
`CONFIG_TOS_LDADDR_START`, `CONFIG_SEC_MEM_SIZE`, and the TOS / ATF switches).
In the full Android BSP those are generated at build time by
`chipram/AndroidChipram.mk`, which reads `BOARD_*` variables from the device
tree file `device/sprd/sharkl5Pro/common/security_feature.mk` and appends the
matching `#define` lines to `out/include/config.h`.

This repo reproduces that step. The resolved values for ums512_1h10
(BOARD_TEE_CONFIG=trusty, non feature phone, not low mem) are captured in
`board/ums512_1h10.nosec.config`, `board/ums512_1h10.secure.config` and
`board/ums512_1h10.open.config`, and `build.sh` appends the chosen one right
after `make ums512_1h10_config`, exactly as the BSP does.

## DHTB image format

The 512 byte header prepended to the payload:

```
0x000  "DHTB"                      magic
0x004  0x00000001                  version
0x008  SHA256(payload)             32 bytes
0x030  payload length              uint32 little endian
```

The header is followed by the raw payload, then the whole thing is padded with
zeros to the 4 MiB partition size. `scripts/dhtb_pack.py` produces this in pure
Python, with no 32 bit multilib and no vendor binary; it is the default
(`PACKER=dhtb`) and the only packer available on this branch.

## A/B slot support (reimplemented from the stock binary)

The vendored chipram source predates this device's Android A/B conversion: its
`nand_boot()` loaded non-slotted partitions (`uboot`, `sml`, `trustos`), which do
not exist on this device (the real partitions are `uboot_a`/`uboot_b`,
`sml_a`/`sml_b`, `trustos_a`/`trustos_b`, `teecfg_a`/`teecfg_b`). That is why an
otherwise-correct build did not boot. The stock SPL was disassembled and the
missing behaviour reimplemented in `chipram/nand_spl/emmc_boot.c` to match it:

- **Slot selection** (`spl_select_slot`): reads the `misc` partition and parses
  the Android `bootloader_control` block at offset 0x800 (magic `0x42414342`),
  choosing the highest-priority bootable slot. On this device that resolves to
  slot **B**, matching the BCB's stored suffix. Verified byte-for-byte against
  the stock selector and against the real `misc` dump.
- **Slotted names**: `teecfg_<slot>`, `sml_<slot>`, `trustos_<slot>`,
  `uboot_<slot>` are loaded in the stock order (teecfg, sml, trustos, then uboot
  last).
- **Load addresses** taken from the stock binary: SML `0x94000000`, TRUSTOS
  `0x94060000` (the vendored config had the wrong `0x94020000`), TEECFG
  `0x94040000`, U-Boot `0x9f000000`.
- **SML handoff ABI**: enters SML at `0x94000000` with `x0` = trustos base
  (`0x94060000`) and `x1` = teecfg base (`0x94040000`), exactly like stock.
- **Dual-backup removed** (`#undef CONFIG_DUAL_BACKUP`): stock has no `*_bak`
  partitions, and the old path ran an `sprd_hash_check` on U-Boot that would have
  rejected a patched U-Boot. The remaining secure-path fallbacks now target the
  other A/B slot instead of `*_bak`.

If `misc`/BCB is unreadable the selector falls back to the BCB's advisory suffix,
and if that is unavailable it defaults to slot B (this device's shipped active
slot); both slots always exist, so it never targets a missing partition.

Not reimplemented (deliberate, not boot-blocking): the Virtual-A/B
`merge_status` snapshot-merge fast path, and the teecfg-derived
`sprd_firewall_config_attr` sizing on the secure variant (the vendored
`get_tos_size` uses an older teecfg layout that does not match this device).

## SD-card boot

The SPL can boot U-Boot from an external microSD card instead of eMMC. On every
boot it loads the secure images (teecfg/sml/trustos) from eMMC as usual, then
tries the SD card for U-Boot first, and only if no valid image is found there
does it fall back to the eMMC `uboot_<slot>` partition. This is gated by
`CONFIG_SD_BOOT` (defined in `chipram/include/configs/ums512_1h10.h`, on for this
board) and implemented in `spl_load_uboot_from_sd()` in
`chipram/nand_spl/emmc_boot.c`.

**Placing U-Boot on the card.** Write a plain DHTB-wrapped U-Boot to the raw
card starting at **sector 200** (byte offset `0x19000`):

- sector 200: the 512 byte DHTB header (`"DHTB"` magic `0x42544844` at offset 0,
  payload length at offset `0x30`),
- sector 201 onward: the raw U-Boot payload.

The image on the card is **not** signature/cert checked: a DHTB-wrapped U-Boot
whose magic is present is treated as valid and loaded (exactly `mImgSize` bytes,
no trailing cert), so it must be under `CONFIG_UBOOT_MAX_SIZE` (1 MiB). A card
with no valid DHTB image at sector 200 simply falls through to eMMC, so an
inserted data card does not break booting.

**Why the SPL does the full SD bring-up.** The SPL itself is loaded from eMMC, so
the BootROM only ever initializes the eMMC controller; the SD controller (SDIO0
at `0x71100000`) and the slot are completely cold. The SD path therefore performs
all of the controller and slot bring-up that the kernel/U-Boot pinctrl would
normally do later:

- **card power**: enable the sc2730 `vddsdcore`/`vddsdio` LDOs. The power
  registers are write-protected, so the power-down clears are silently dropped
  unless the unlock magic (`0x6e7f`) is written first, matching `regulator_init()`
  in `sc27xx_regulator.c`.
- **clock**: set `base_clock` and select the 384 MHz SDIO0 2x source in the AP
  clock core (`REG_AP_CLK_CORE_CGM_SDIO0_2X_CFG`), then divide down to the 400 kHz
  init clock.
- **pads**: the SD0 pads come up in func4, where the controller can read the
  lines but cannot drive CMD/CLK, so they are forced to func1; CMD/DAT pull-ups
  are enabled so the removable-slot lines do not float; and the SDIO0 on-die IO
  supply gate (`BIT_AON_APB_AP_SDIO0_IO_POWER_OFF`) is cleared.
- **PHY delay**: load DLL backup mode plus the board legacy phy-delay so the data
  lines sample correctly when the clock is raised for reads.

This bring-up lives in `SD_HOST_Register()` and `SD_Set_init_clk()` in
`chipram/nand_spl/mmc_v40.c`, guarded by `CONFIG_SOC_SHARKL5PRO` /
`CONFIG_ADIE_SC2730`. Validated on hardware: it boots U-Boot straight from the
card, and eMMC boot still works when no card image is present. The binary-patch
path above is the stock SPL and does not include SD boot.

## Secure-world handoff (survives an Android shutdown)

The from-source secure build reproduces the stock SPL's PUB memory-firewall
handoff so that it keeps booting after Android has run and been shut down. An
earlier version sized the SEG_0 secure region as a fixed 32 MB
(`0x94000000..0x95ffffff`), which is broader than stock: stock sizes SEG_0 to the
TEECFG..end-of-TOS extent and leaves the memory above TOS open. A fresh flash
booted fine, but once Android's TOS provisioned memory in the region our
over-broad SEG_0 had marked secure-only, the next boot's TOS faulted and the SML
handoff hung. The fix sizes SEG_0 like stock (via `sml_teecfg_sec` plus a
`tos_sec` call after the teecfg load), reprograms the above-DRAM SEG_7 catch-all
every boot, matches the AON `iram_sec` ranges, and uses the stock 64-bit
`CHIPRAM_ENV` layout. It lives in
`chipram/secure/trustzone/firewall_sharkl5pro.c`.

## Compatibility with the stock spl_a.img

The stock `spl_a.img` from this device was examined byte for byte. Findings:

1. It is **RSA-2048 signed**. After the SPL code (0xed50 bytes) there is a
   Spreadtrum `SIMGHDR` block (magic `SIMGHDR`) carrying an RSA-2048 public key
   (exponent 0x10001) and a 256 byte signature. The DHTB header fields at
   0x3c and 0x40 describe the signed layout. A plain unsigned build does not
   have this block.

2. The signing key is **rsa2048_0**. The modulus embedded in the stock
   signature block matches `rsa2048_0_pub.pem` in the Unisoc BSP
   (`bsp_build-master/packimage_scripts/config/`) exactly.

3. **What this means for flashing.** See the fuse discussion under "Variants"
   above. In short: this device does not enforce the SPL signature, which is why
   the unsigned `open` image boots; a fused unit would need a signed one.

### DDR timing and device tree

The SPL does not use a device tree. All DDR parameters are compile time, taken
from `chipram/include/configs/ums512_1h10.h`:

```
CLK_DDR_FREQ = 1024000000   (1024 MHz)
DDR_MODE     = 0x0002
CFG_DRAM_TYPE = DRAM_LPDDR3
DRAM_SIZE_AUTO_DETECT, DRAM_TYPE_AUTO_DETECT enabled
DCDC_MEM = 1100
```

Because size and type auto detect are enabled, the SPL probes the DRAM at boot
rather than hard coding a single part, which makes it tolerant of DRAM
variations. These values are used exactly as they appear in the vendored source.
The one thing this repo cannot guarantee is that the vendored chipram snapshot is
the identical revision Unisoc used to build the stock image; the payload size
differs (mostly due to the gcc 4.9 vs modern toolchain), but the DDR init path,
load addresses and boot flow are the same.

## Notes on reproducibility

The image is not byte for byte identical to a stock `spl_a.img` built by Unisoc,
because that was compiled with a different (gcc 4.9 Android) toolchain. The
format, load flow, DDR init and boot behaviour are the same. The build is
verified with gcc-aarch64-linux-gnu 11 and 14.

### Local changes to the vendored source

- `chipram/include/linux/compiler-gcc.h` was patched so that modern gcc (5 and
  newer) uses the existing `compiler-gcc4.h` attribute definitions instead of
  trying to include a per version header that does not ship in this tree. This
  does not affect generated code.
- `chipram/config.mk` keeps GCC 14's stricter C diagnostics as warnings
  (`-fpermissive`, `-Wno-error=implicit-function-declaration` and friends, all
  behind `cc-option`). GCC 14 turns several long-standing warnings into errors by
  default and this 2015 vendor tree relies on all of them; the alternative is
  rewriting headers that came from the vendor.
