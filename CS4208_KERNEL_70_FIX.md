# CS4208 driver fix for Ubuntu kernel 7.0 (linux-hwe-7.0)

## Problem

The `snd-hda-codec-cs420x` module crashed the audio stack (kernel oops in
`cs_4208_build_pcms_explicit`) on Ubuntu Noble running
`linux-hwe-7.0` (`7.0.0-28-generic`). Symptoms:

- Module loaded, HDA codec probe started, then a general protection fault /
  oops ~30 instructions into `cs_4208_build_pcms_explicit`.
- No soundcards appeared in `/proc/asound/cards`.
- UBSAN `array-index-out-of-bounds` reports from the *distro's own*
  `snd_hda_codec_generic` (CS4208 reports more than 18 input pins vs
  `AUTO_CFG_MAX_INS`) — non-fatal, but noisy.

## Root cause

The installer script (`install.cirrus.driver.sh`) downloads the **vanilla
kernel.org** source tarball (e.g. `linux-7.0.tar.xz`) and builds the module
against its `sound/hda/` headers.

The running kernel, however, is Ubuntu's **patched** `linux-hwe-7.0`, which
carries an Ubuntu HDA patch that adds a member to `struct hda_multi_out`
(in `sound/hda/common/hda_local.h`):

```c
struct snd_kcontrol *share_spdif_kctl;
```

That grows `hda_multi_out` by 8 bytes and shifts **every later field of
`struct hda_gen_spec`** — most importantly the `pcm_rec[]` pointers:

| tree | `spec->pcm_rec[0]` offset |
|---|---|
| vanilla 7.0.0 (what the old module was built against) | `0x840` |
| Ubuntu `linux-hwe-7.0` 7.0.0-28 (what the running kernel uses) | `0x848` |

So the old module read `spec->pcm_rec[0]` at offset `0x840` — landing on an
unrelated int field, interpreting its small value as a `struct hda_pcm *`,
and writing 88 bytes to it. → invalid pointer dereference → oops.

### How it was verified

1. Disassembled the distro's running `snd-hda-codec-generic.ko` and
   fingerprinted the offsets its `snd_hda_gen_build_pcms` uses for the
   `hda_gen_spec` fields.
2. Downloaded the exact source diff for the running kernel version
   (`linux-hwe-7.0_7.0.0-28.28~24.04.1.diff.gz` from Launchpad, SHA256
   verified: `b11f88c0a3100c83ea91e5d3af3135ffabe7c629d975cfd22ebf61153a9fe56d`).
3. Applied its `sound/hda/` hunks to pristine vanilla 7.0.0 source
   (`/tmp/hwefix28/linux-hwe-7.0-7.0.0/sound/hda`).
4. Compiled that tree's `generic.c` as a standalone module and confirmed its
   `snd_hda_gen_build_pcms` disassembly is **byte-identical** (modulo
   addresses) to the running distro module → the patched tree is provably
   the exact source of the running kernel's HDA code.
5. An `offsetof()` probe compiled under the same conditions reports
   `pcm_rec[0] = 0x848`, matching the new module's disassembly.

## What was changed

1. **Replaced the vendored HDA tree** used for building:
   - `build/hda` — now the vanilla-7.0.0 + Ubuntu-7.0.0-28 `sound/hda` patch
     set (verified against the running kernel).
   - Old vanilla tree preserved at `build/hda.bak-vanilla`.
2. **Re-applied the project's CS4208 files** into the new tree
   (`codecs/cirrus/`): `cs420x.c`, `patch_cirrus_a1534_{setup,pcm,verbs}.h`,
   `Makefile` (from `patch_cirrus/`), plus the installer's
   `.free` → `.remove` rename in `patch_cirrus_a1534_pcm.h` (required by the
   post-6.17 `hda_codec_ops` layout).
3. **Rebuilt** the module against `7.0.0-28-generic` headers:
   `make -f Makefile_cs420x all`
   → `build/hda/codecs/cirrus/snd-hda-codec-cs420x.ko`
   (verified via objdump that it now reads `pcm_rec[0]` at `0x848`).

## Install / reload (requires sudo)

```bash
cd ~/buildspace/macbook12-audio-driver-fork
sudo cp build/hda/codecs/cirrus/snd-hda-codec-cs420x.ko \
        /lib/modules/7.0.0-28-generic/updates/
sudo depmod -a
sudo modprobe -r snd-hda-codec-cs420x snd-hda-codec-generic 2>/dev/null
sudo modprobe snd-hda-codec-cs420x
sleep 2 && cat /proc/asound/cards
```

Expected: a soundcard entry for the CS4208 codec, no new oops in `dmesg`.

## Known caveats

- **`install.cirrus.driver.sh` will regress this**: it re-downloads the
  vanilla kernel.org tree and will rebuild a broken module on any re-run.
  On Ubuntu (or any distro that patches `sound/hda`), the script must be
  changed to fetch/apply the distro's `sound/hda` diff, or build against the
  distro's kernel headers.
- **DKMS**: `/usr/src/macbook12-audio-0.1` still contains the old vanilla
  tree. It needs the same replacement before any automatic DKMS build
  (e.g. after a kernel update) can succeed.
- **UBSAN noise**: the distro's `snd_hda_codec_generic` can emit
  `array-index-out-of-bounds` during probe because CS4208 exposes more input
  pins than `AUTO_CFG_MAX_INS` (18). Appears non-fatal; monitor with
  `dmesg` after reload.

## Key artifacts

| path | what |
|---|---|
| `build/hda/` | verified Ubuntu-patched 7.0.0-28 `sound/hda` build tree |
| `build/hda.bak-vanilla/` | previous (broken) vanilla tree |
| `build/hda/codecs/cirrus/snd-hda-codec-cs420x.ko` | fixed module |
| `/tmp/hwefix28/linux-hwe-7.0-7.0.0/` | reference: vanilla + -28 diff tree |
| `/tmp/hwefix/linux-hwe-7.0-7.0.0/` | reference: -38 variant (for future kernel updates) |
| `/tmp/hwe28.diff` / `/tmp/hwe38.diff` | Ubuntu source diffs for 7.0.0-28 / -38 |
| `/tmp/gtest/` | standalone compile of the -28 `generic.c` (verification) |
| `/tmp/oft2/` | `offsetof()` probe module (verification) |

## Future kernel updates

When a new `linux-hwe-7.0` version (e.g. `-39`, `-40`) ships:

1. Download the version's `.diff.gz` from Launchpad
   (`https://launchpad.net/ubuntu/+source/linux-hwe-7.0`) and verify SHA256.
2. Re-extract the `sound/hda/` + `include/sound/hda*` hunks onto a fresh
   vanilla tarball, or — simpler — just replace `build/hda` with the
   distro tree of that version.
3. Re-verify by compiling its `generic.c` and diffing the disassembly of
   `snd_hda_gen_build_pcms` against the freshly installed distro module.
4. Re-apply the `patch_cirrus/` files, rebuild, reinstall.
