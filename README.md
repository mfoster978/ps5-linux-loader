# PS5 Pro 11.60 Viola Fix

> **Temporary community fix — tested on Bazzite Deck OS v1.8.9**

This fork contains the PS5 Linux loader and initramfs workaround I used to activate the real **Viola** firmware path on a retail **PS5 Pro running firmware 11.60**.

The working result changed the GPU from the 36-CU Cyan Skillfish fallback to the PS5 Pro's factory **60 active CUs**. This is not an overclock and does not use `ps5_cu_unlock`. In my testing it substantially increased GPU performance and stopped repeated crashes or HDMI loss in 007 First Light, GTA V Legacy, Manor Lords, and RV There Yet?

## Read this first

| Status | Details |
| --- | --- |
| Console tested | Retail PS5 Pro |
| PS5 firmware tested | 11.60 |
| Linux tested | Bazzite Deck OS v1.8.9 |
| Ubuntu | **Not tested.** The loader fix likely applies, but Ubuntu's paths and initramfs procedure may differ. Do not install the Bazzite script on Ubuntu without adapting and validating it. |
| Expected topology | 60 active CUs instead of the 36-CU fallback |
| Loader PR | [ps5-linux/ps5-linux-loader #91](https://github.com/ps5-linux/ps5-linux-loader/pull/91) |
| Bazzite bug | [VivaLaVent/ps5-bazzite-deck #27](https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27) |

> [!CAUTION]
> This is a temporary personal-fork build, not an official release. I do not plan to maintain it after the official projects publish their fixes. When the loader developers release the 11.60 fix, use the official [ps5-linux-loader](https://github.com/ps5-linux/ps5-linux-loader). When Bazzite fixes its staging script, stop using the replacement script from this fork.

## Easiest Bazzite instructions

These steps are specifically for **PS5 Pro firmware 11.60 with Bazzite Deck OS v1.8.9**.

### 1. Download the hardware-tested ELF

**[Download ps5-linux-loader.elf](https://github.com/mfoster978/ps5-linux-loader/releases/download/ps5-pro-11.60-viola-test-v1/ps5-linux-loader.elf)**

Release notes:

https://github.com/mfoster978/ps5-linux-loader/releases/tag/ps5-pro-11.60-viola-test-v1

Verify the file:

```text
File:   ps5-linux-loader.elf
Size:   323128 bytes
SHA256: cb412af9ba1b708c5131f80e6ce755f337b688ca5ec1f57bdbbc2ce8c44f1a4b
```

Linux:

```bash
sha256sum ps5-linux-loader.elf
```

Windows PowerShell:

```powershell
Get-FileHash .\ps5-linux-loader.elf -Algorithm SHA256
```

Do not continue if the size or hash differs.

### 2. Add the ELF to Payload Manager once

Import or send `ps5-linux-loader.elf` to Payload Manager once so it is saved there. After that, select the saved entry for each Linux boot; do not keep uploading the file.

For every loader boot:

1. cold boot into PS5 OS;
2. start only the jailbreak/ELF transport required for Payload Manager;
3. do **not** run etaHEN, kstuff, a debugger, HEN, or another unrelated payload; and
4. select the saved `ps5-linux-loader.elf`.

### 3. Launch once: firmware dump and Linux boot happen together

The loader does **not** use a special run-twice flow and does **not** skip existing firmware files.

On each supported PS5 Pro launch it:

1. reconstructs all eight Viola files;
2. displays the firmware dump notifications;
3. enters rest mode in the same launch; and
4. wakes and boots Linux normally.

Wait for the PS5 to enter rest mode. After the orange light becomes steady, wake it normally. A successful wake changes from flashing blue to white and starts Linux.

### 4. On the first Bazzite session after the files are created, repair the next initrd

The loader boots using the initrd already loaded for that launch. During the running Bazzite session, v1.8.9 sees the newly created Viola files and runs `ps5-stage-viola-fw`.

The original script creates:

- `/boot/efi/initrd.img.noviola`, the base backup;
- a replacement `/boot/efi/initrd.img` with the Viola CPIO appended without required boundary padding.

The current session still works because it loaded the previous initrd. The unaligned file would break the **next** boot, so repair it before shutting down or rebooting.

Back up the current files:

```bash
sudo cp -a /boot/efi/initrd.img \
  /boot/efi/initrd.img.before-viola-alignment-repair
sudo cp -a /usr/local/sbin/ps5-stage-viola-fw \
  /usr/local/sbin/ps5-stage-viola-fw.before-alignment-repair
```

Download and inspect the corrected script:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/mfoster978/ps5-linux-loader/main/bazzite/ps5-stage-viola-fw \
  -o /tmp/ps5-stage-viola-fw
less /tmp/ps5-stage-viola-fw
```

Install it:

```bash
sudo install -m 0755 /tmp/ps5-stage-viola-fw \
  /usr/local/sbin/ps5-stage-viola-fw
```

Restore the base initrd and rebuild it with the corrected script:

```bash
sudo cp -a /boot/efi/initrd.img.noviola /boot/efi/initrd.img
sudo /usr/local/sbin/ps5-stage-viola-fw
sync
```

Verify the boundary:

```bash
E=/boot/efi
base_size=$(stat -c %s "$E/initrd.img.noviola")
padding=$(( (4 - (base_size % 4)) % 4 ))
offset=$((base_size + padding))
printf 'base=%s padding=%s cpio_offset=%s\n' \
  "$base_size" "$padding" "$offset"
dd if="$E/initrd.img" bs=1 skip="$offset" count=6 status=none
printf '\n'
```

The exact tested v1.8.9 base reported:

```text
base=100924166
padding=2
cpio_offset=100924168
070701
```

The script calculates the correct zero-to-three-byte padding for any base size. The literal two-byte value applies only to this exact tested base.

### 5. Later boots

After the aligned initrd is ready:

1. power off Bazzite cleanly;
2. cold boot the PS5;
3. start the jailbreak/ELF transport and Payload Manager;
4. launch only the already-saved `ps5-linux-loader.elf`;
5. allow rest mode to complete; and
6. wake into Linux.

The loader will re-dump the files on every launch. This is expected and inexpensive. Bazzite's state hash prevents unnecessary initrd restaging when the staged result is unchanged.

### 6. Confirm Viola and 60 active CUs

```bash
journalctl -b -k --no-pager | grep -E \
  'PS5 Pro: using Viola firmware|RequestActiveWgp|Set BAPM params|active_cu_number'
```

Validated output:

```text
amdgpu 0000:20:00.0: PS5 Pro: using Viola firmware
mp1: Viola: RequestActiveWgp(30) failed: -22
mp1: Set BAPM params (Viola)
amdgpu 0000:20:00.0: SE 2, SH per SE 2, CU per SH 16, active_cu_number 60
```

`RequestActiveWgp(30)` returned `-22`, but the GPU still exposed its factory 30-WGP / 60-CU topology.

## What this fixed in my testing

The post-repair results below were recorded with both the correct 11.60 Viola offset and the aligned initramfs active. The kernel selected Viola firmware with 60 active CUs.

| Game | Resolution | Graphics preset | Observed frame rate | Before both fixes | After both fixes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **007 First Light** | **4K / 3840 × 2160** | **Low** | **Approximately 34 FPS average in cutscenes** | Crashed during loading; lowering resolution had not fixed it | **Working; no crashes in my retest** |
| **007 First Light** | **1440p / 2560 × 1440** | **Ultra — everything set to Ultra** | **53–61 FPS** | Crashed during loading | **Working; no crashes in my retest** |
| **Grand Theft Auto V Legacy** | **1440p / 2560 × 1440** | Not separately re-recorded | **Solid approximately 60 FPS** | Repeated driving crash or HDMI loss after approximately five minutes | **Smooth; the driving crash and HDMI loss did not recur** |
| **Manor Lords** | Not recorded | Not recorded | Not recorded | Failed after **Start Game**, during loading | **Working through the previous failure point** |
| **RV There Yet?** | Not recorded | Not recorded | Not recorded | Failed while loading a game | **Working through the previous failure point** |

The 4K 007 result is a cutscene average; the 1440p result is the observed range. These are on-screen readings, not a controlled benchmark.

### Stable comparisons already confirmed

These were stable before the repairs and are not counted as newly fixed games.

| Game or use | Recorded settings | Recorded frame rate | Result |
| :--- | :--- | :--- | :--- |
| **Grand Theft Auto IV** | Not recorded | Not recorded | Long-term stability confirmed in my testing |
| **Call of Duty: Black Ops III** | **Native 4K / 3840 × 2160; 100% internal render scale; 60-Hz output** | **51–60 FPS; approximately 56 FPS average** | Long-term stability confirmed in my testing |
| **Desktop mode and occasional web browsing** | Normal desktop use | Not applicable | Works normally |

### Awaiting a targeted retest

| Game | Earlier failure | Post-repair status |
| :--- | :--- | :--- |
| **Grand Theft Auto V Enhanced** | Crashed around Michael's funeral sequence | That scene has not yet been retested after the repairs |

These results apply to the tested PS5 Pro 11.60/Bazzite v1.8.9 system and are not guarantees for every game or image.

## Ubuntu status

Ubuntu has **not** been tested.

The loader should still create the eight files on a supported PS5 Pro 11.60 setup, but this fork's script assumes Bazzite paths, service behavior, state file, and `initrd.img` workflow. Ubuntu may use different initramfs tools or locations.

Do not copy the Bazzite script onto Ubuntu and assume it is correct. Back up the active initramfs and independently identify and validate Ubuntu's staging method.

## Recovery

- Keep the official/original loader and initrd backups.
- Do not repeatedly retry a flashing-blue-light hang.
- If the kernel reports `invalid magic at start of compressed archive`, restore the known-working initrd before rebuilding.
- Do not use this guide for a regular PS5 or firmware other than 11.60.
- Firmware dump notifications on every loader launch are normal.

## Upstream status and end of support

- Loader source fix: https://github.com/ps5-linux/ps5-linux-loader/pull/91
- Bazzite alignment report: https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27

This fork exists only until official fixes are available. At that point:

1. download the official loader;
2. use the Bazzite-provided staging script; and
3. stop using the ELF and script from this fork.

---

# Detailed technical record

## Final source scope

The maintainer-requested branch contains only:

- `source/offsets.c`: adds `.PS5_PRO_GPU_FW_OFFSET = 0x257F550` to `off_1160`;
- `source/firmware.c`: requires `sceKernelGetMainSocId() == 0x840fd0` before the existing nonzero-offset GPU dump path.

The complete diff is three additions and one deletion. It does not change versions, headers, the build system, `main()`, `fetch_linux()`, rest-mode flow, or existing firmware reconstruction.

Minimal source commit:

https://github.com/mfoster978/ps5-linux-loader/commit/55fc69fc6cbe38cec48024adefd1d49f5756a59d

## Why the gate is needed

The 11.60 offset table is shared by regular and Pro consoles. Adding a Pro-only GPU table address without a hardware gate could make a regular PS5 enter the wrong path. The SoC ID gate confines the new offset to PS5 Pro hardware.

## Offset derivation

The value was derived independently on a retail PS5 Pro 11.60:

1. a bounded kernel-data window was captured using sequential read-only requests;
2. the capture was searched offline for arrays of eight canonical kernel pointers;
3. candidates were resolved using the loader's existing `gpu_firmware_blob_header`;
4. feature ordering, pointer bounds, tag strings, payload sizes, jump-table bounds, RLC/SDMA checksums, and guarded PFP bytes were validated;
5. exactly one table survived; and
6. a second cold-boot capture reproduced the same kernel-text-relative value.

Result:

```c
.PS5_PRO_GPU_FW_OFFSET = 0x257F550,
```

This is qword 39 in the loader's 61-qword `offset_list`. No proprietary dump or firmware payload is published.

## Reconstructed files

The unchanged upstream reconstruction path writes:

- `viola_ce.bin`
- `viola_me.bin`
- `viola_mec.bin`
- `viola_mec2.bin`
- `viola_pfp.bin`
- `viola_rlc.bin`
- `viola_sdma.bin`
- `viola_sdma1.bin`

The validation set was checked for AMDGPU headers, nonzero sizes, `$PS1` structure and bounds, CRC32 values, and the guarded PFP patch.

## Reproducible build

The final ELF used:

- `ps5-payload-sdk` v0.45;
- SDK archive SHA-256 `5fac64093d0e3e69f2584ce7935f24f703f622d9681f1495c85b9f00d1291a2c`;
- Ubuntu Clang 18.1.3 (`1ubuntu1`);
- Ubuntu LLD 18.1.3.

Untouched baseline `e7954b27ddc27b03ddbb6f49fb94100b0d1f271e` reproduced the known-working loader byte for byte:

```text
323128 bytes
3d2d14ef4779712707b668534eb9fa909570dd14817123a63a984120ff658365
```

Two clean serial builds of the minimal tree were identical:

```text
323128 bytes
cb412af9ba1b708c5131f80e6ce755f337b688ca5ec1f57bdbbc2ce8c44f1a4b
```

## ELF audit

Compared with the reproduced baseline:

- 24 sections and four program headers;
- identical 16 KiB-aligned load/dynamic topology;
- `.gnu.hash` only;
- `mmap` and `mprotect` remain defined in the ELF;
- compiled qword 39 is `50 f5 57 02 00 00 00 00`, or `0x257F550`;
- compiled SoC comparison is `0x840fd0`;
- `.text` grows by 16 bytes for the gate;
- remaining non-code section and segment layouts are unchanged.

## Hardware test of the exact minimal ELF

The 323,128-byte SHA-256-qualified ELF was tested on a retail PS5 Pro 11.60:

1. all GPU firmware dump notifications appeared;
2. the loader entered rest mode in the same launch;
3. wake changed from flashing blue to white in approximately five seconds; and
4. Linux booted successfully.

This validates the one-pass behavior requested by the maintainer.

## Bazzite initramfs defect

The loader writes firmware to the boot filesystem, but AMDGPU requests it from the initramfs. Bazzite v1.8.9 directly concatenates a `newc` CPIO after a base whose size may not be divisible by four.

Tested layout:

```text
base initrd.img.noviola: 100924166 bytes
padding:                 2 zero bytes
newc magic offset:       100924168
newc archive size:       272384 bytes
working initrd.img:      101196552 bytes
working SHA-256:         50aaa95909aacb9e28c5e64622b4ec9b05dd9cdf53266cc8f5b78b2659d6c17e
```

General formula:

```text
padding = (4 - (base_size mod 4)) mod 4
```

The separate issue is:

https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27

## Limitations

- Only retail PS5 Pro system software 11.60 was tested.
- Bazzite Deck OS v1.8.9 was used for Linux validation.
- Ubuntu and other images were not tested.
- The published initrd hash applies only to the exact tested base and firmware set.
- The Bazzite workaround is external to the loader PR.
- No proprietary firmware files are distributed.

## Public evidence

- Upstream PR and source diff: https://github.com/ps5-linux/ps5-linux-loader/pull/91
- Minimal source commit: https://github.com/mfoster978/ps5-linux-loader/commit/55fc69fc6cbe38cec48024adefd1d49f5756a59d
- Hardware-tested release: https://github.com/mfoster978/ps5-linux-loader/releases/tag/ps5-pro-11.60-viola-test-v1
- Bazzite issue and runtime evidence: https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27

Testing and validation were performed by [@mfoster978](https://github.com/mfoster978).
