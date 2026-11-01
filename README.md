# PS5 Pro 11.60 Viola Fix

> **Temporary community fix — tested on Bazzite Deck OS v1.8.9**

This fork contains the PS5 Linux loader and initramfs workaround I used to activate the real **Viola** firmware path on a retail **PS5 Pro running firmware 11.60**.

The working result changed the GPU from the 36-CU Cyan Skillfish fallback to the PS5 Pro's factory **60 active CUs**. This is not an overclock and does not use `ps5_cu_unlock`. In my testing it substantially increased GPU performance and stopped the repeated crashes/HDMI loss in GTA V Legacy, Manor Lords, and RV There Yet? My later 007 First Light retests also ran without crashes: about 34 FPS in 4K Low cutscenes and 53–61 FPS at 1440p Ultra.

## Read this first

| Status | Details |
| --- | --- |
| Console tested | Retail PS5 Pro |
| PS5 firmware tested | 11.60 |
| Linux tested | Bazzite Deck OS v1.8.9 |
| Ubuntu | **Not tested.** The loader-side fix will likely be similar, but Ubuntu's initramfs creation, paths, and staging procedure may differ. Do not install the Bazzite script on Ubuntu without adapting and validating it. |
| Expected topology | 60 active CUs instead of the 36-CU fallback |
| Loader PR | [ps5-linux/ps5-linux-loader #91](https://github.com/ps5-linux/ps5-linux-loader/pull/91) |
| Bazzite bug report | [VivaLaVent/ps5-bazzite-deck #27](https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27) |

> [!CAUTION]
> This is a temporary, hardware-tested fork build—not an official release. I do not plan to maintain or update it after the official projects publish their fixes. When the loader developers release the 11.60 fix, use the official [ps5-linux-loader](https://github.com/ps5-linux/ps5-linux-loader). When Bazzite fixes its initramfs staging script, stop using the replacement script from this fork.

## Easiest Bazzite Deck OS instructions

These steps are for **PS5 Pro firmware 11.60 with Bazzite Deck OS v1.8.9**.

### 1. Download the tested loader

Download the exact hardware-tested ELF (this is a custom made ELF with fixes for 11.60 PS5 PRO ONLY):

**[Download ps5-linux-loader.elf](https://github.com/mfoster978/ps5-linux-loader/releases/download/ps5-pro-11.60-viola-test-v1/ps5-linux-loader.elf)**

Full release notes, recovery steps, and technical evidence:

https://github.com/mfoster978/ps5-linux-loader/releases/tag/ps5-pro-11.60-viola-test-v1

Verify the downloaded file before using it:

```text
File:   ps5-linux-loader.elf
Size:   323216 bytes
SHA256: 4fadbfa17761d6de0c1eeb71fae88fa3b375b7b27f8ccb7258ddff11405cf041
```

On Linux:

```bash
sha256sum ps5-linux-loader.elf
```

On Windows PowerShell:

```powershell
Get-FileHash .\ps5-linux-loader.elf -Algorithm SHA256
```

Do not continue if the hash differs.

### 2. Add the ELF to Payload Manager once

Import or send the downloaded `ps5-linux-loader.elf` to the PS5 Payload Manager **one time** so it is saved there.

After it is saved, use the Payload Manager entry to start it. You do not need to keep uploading or sending the ELF from the PC.

For every loader boot:

1. cold boot into PS5 OS;
2. start the jailbreak/ELF transport needed for Payload Manager;
3. do **not** run etaHEN, kstuff, a debugger, HEN, or any unrelated payload; and
4. select the saved `ps5-linux-loader.elf` entry in Payload Manager.

The jailbreak/ELF loader and Payload Manager are the transport. The Viola loader ELF should be the only payload launched.

### 3. First Payload Manager launch: create the eight files

Start the saved `ps5-linux-loader.elf`.

If any Viola file is missing, the loader:

1. creates all eight files on the boot partition;
2. calls `sync()`;
3. tells you the files were installed; and
4. exits before rest mode or Linux wake preparation.

That is the expected first run. The Bazzite initramfs problem does **not** happen before the files exist. It happens when Bazzite first boots after these files have been created and its staging service appends them to `initrd.img`.

If all eight files already exist and are nonempty, the loader silently skips all GPU table reads and firmware reconstruction. It does not display a “reusing” message.

### 4. Launch the saved ELF again and boot Bazzite once

Use Payload Manager to start the already-saved `ps5-linux-loader.elf` again. Do not upload it again and do not launch another payload.

This boot still uses the old initrd, so Bazzite can start normally. During this running Bazzite session, v1.8.9 sees the newly created Viola files and runs `ps5-stage-viola-fw`.

The original v1.8.9 script creates:

- `/boot/efi/initrd.img.noviola`, the base initrd backup; and
- a new `/boot/efi/initrd.img` with the Viola CPIO appended without the required boundary padding.

The unaligned file does not break the Linux session that is already running because that session loaded the previous initrd before the service changed the file on disk. It would break the **next** boot if it is not repaired first.

### 5. Repair and rebuild the on-disk initrd in that Bazzite session

First preserve the current files:

```bash
sudo cp -a /boot/efi/initrd.img \
  /boot/efi/initrd.img.before-viola-alignment-repair
sudo cp -a /usr/local/sbin/ps5-stage-viola-fw \
  /usr/local/sbin/ps5-stage-viola-fw.before-alignment-repair
```

Download and inspect the corrected script from this fork:

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

Restore the unmodified base initrd that the original service saved, then run the corrected script:

```bash
sudo cp -a /boot/efi/initrd.img.noviola /boot/efi/initrd.img
sudo /usr/local/sbin/ps5-stage-viola-fw
sync
```

For the exact v1.8.9 base tested here, the rebuilt file needs two zero bytes before the appended CPIO. Verify the boundary:

```bash
E=/boot/efi
base_size=$(stat -c %s "$E/initrd.img.noviola")
padding=$(( (4 - (base_size % 4)) % 4 ))
offset=$((base_size + padding))
printf 'base=%s padding=%s cpio_offset=%s\n' \
  "$base_size" "$padding" "$offset"
dd if="$E/initrd.img" bs=1 skip="$offset" count=6 status=none
printf '\n'
sha256sum "$E/initrd.img"
```

The tested v1.8.9 layout reports:

```text
base=100924166
padding=2
cpio_offset=100924168
070701
```

Power off Bazzite cleanly after the aligned initrd has been created.

### 6. Boot with Viola using the saved Payload Manager entry

1. Cold boot the PS5 again.
2. Start the jailbreak/ELF transport and Payload Manager.
3. Do not run any other payload.
4. Select the already-saved `ps5-linux-loader.elf`.
5. Let the PS5 enter rest mode.
6. Wait for the orange light to stop blinking and become steady.
7. Wake the PS5 to Linux.

You do not need to upload the ELF again. Use the saved Payload Manager entry for later Linux boots.

### 7. Confirm 60 active CUs

In Linux:

```bash
journalctl -b -k --no-pager | grep -E \
  'PS5 Pro: using Viola firmware|RequestActiveWgp|Set BAPM params|active_cu_number'
```

The validated result was:

```text
amdgpu 0000:20:00.0: PS5 Pro: using Viola firmware
mp1: Viola: RequestActiveWgp(30) failed: -22
mp1: Set BAPM params (Viola)
amdgpu 0000:20:00.0: SE 2, SH per SE 2, CU per SH 16, active_cu_number 60
```

`RequestActiveWgp(30)` returned `-22`, but the GPU still exposed the correct factory topology of 30 WGPs / 60 active CUs.

## Game testing — updated November 1, 2026

I tested this on my **retail PS5 Pro, PS5 system software 11.60, and Bazzite Deck OS v1.8.9**. The post-repair results below were recorded with both repairs active: the loader's correct 11.60 Viola offset and the four-byte-aligned initramfs archive. The kernel selected **Viola firmware with 60 active CUs**.

My PS5 system software is still **11.60**. These repairs corrected Linux GPU firmware extraction and staging.

### Games I retested after both repairs

| Game | Resolution | Graphics preset | Observed frame rate | Before the repairs | After both repairs |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **007 First Light** | **4K · 3840 × 2160** | **Low** | **~34 FPS average in cutscenes** | Crashed during loading; lowering the resolution had not fixed it | ✅ **Working; no crashes in my retest** |
| **007 First Light** | **1440p · 2560 × 1440** | **Ultra — everything set to Ultra** | **53–61 FPS** | Crashed during loading | ✅ **Working; no crashes in my retest** |
| **Grand Theft Auto V Legacy** | **1440p · 2560 × 1440** | Not separately re-recorded after repair | **Solid ~60 FPS** | Repeated driving crash/HDMI loss after about 5 minutes; previously 1440p Very High at ~60 FPS with dips to 48 | ✅ **Smooth; no recurrence of the driving crash or HDMI loss in my retest** |
| **Manor Lords** | Not recorded | Not recorded | Not recorded | Failed after **Start Game**, during loading | ✅ **Working through the previous failure point; no crashes in my retest** |
| **RV There Yet?** | Not recorded | Not recorded | Not recorded | Failed while loading a game | ✅ **Working through the previous failure point; no crashes in my retest** |

The 4K 007 figure is a **cutscene average**; the 1440p figure is the **observed range** in my retest. These are my on-screen readings; I did not record an exact duration for the post-repair tests.

### Stable comparisons I had already confirmed

These were already stable in my testing before the repairs; I am not counting them as newly fixed games.

| Game / use | Recorded settings | Recorded frame rate | My stability result |
| :--- | :--- | :--- | :--- |
| **Grand Theft Auto IV** | Not recorded | Not recorded | ✅ **Long-term stability confirmed in my testing; zero issues** |
| **Call of Duty: Black Ops III** | **Native 4K · 3840 × 2160; 100% internal render scale; 60-Hz output** | **51–60 FPS; ~56 FPS average** | ✅ **Long-term stability confirmed in my testing; zero issues** |
| **Desktop mode / occasional web browsing** | Normal desktop use | Not applicable | ✅ **Works normally; no issues in that use** |

### Earlier failure still awaiting a targeted retest

| Game | Earlier failure | Post-repair status |
| :--- | :--- | :--- |
| **Grand Theft Auto V Enhanced** | Crashed around Michael's funeral sequence | ⏳ **I have not yet confirmed a post-repair retest of that scene** |

The pre-repair GTA V failure logs contained a `gfx_0.0.0` timeout, failed MODE1 recovery, VRAM loss, and KIQ/KCQ/gfx resume errors. With the proper Viola firmware and 60 active CUs, the driving workload I reproduced no longer fails.

## Ubuntu status

Ubuntu has **not** been tested with this fix.

The loader portion should still create the eight files on the boot partition on a supported PS5 Pro 11.60 setup. However, the Bazzite-specific script in `bazzite/ps5-stage-viola-fw` assumes Bazzite's paths, service, state file, and `initrd.img` workflow. Ubuntu may use different initramfs tools or locations.

Do not copy the Bazzite script onto Ubuntu and assume it is correct. Ubuntu users should first identify the active initramfs, back it up, determine how concatenated firmware archives are staged, and independently verify the four-byte boundary.

## If something goes wrong

- Keep the original loader and all initrd/script backups.
- Do not repeatedly retry after a flashing-blue-light hang.
- If the loader says the files were installed, that first run is supposed to stop; run the same ELF again.
- If the loader says nothing about firmware because all eight files already exist, that is normal.
- If the kernel reports `invalid magic at start of compressed archive`, restore the known-working initrd backup before trying to rebuild it.
- Do not use this guide for a regular PS5 or a firmware version other than 11.60.

The detailed release notes include the recovery procedure and exact alignment checks:

https://github.com/mfoster978/ps5-linux-loader/releases/tag/ps5-pro-11.60-viola-test-v1

## Upstream status and end of support

The fixes have been reported to the projects that should own them:

- Loader source fix: https://github.com/ps5-linux/ps5-linux-loader/pull/91
- Bazzite initramfs alignment report: https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27

This fork exists so users can test the validated fix now. I will not maintain it after the official fixes are available. At that point:

1. download the official PS5 Linux loader;
2. use the Bazzite-provided staging script;
3. stop using the ELF and Bazzite script from this fork.

---

# Detailed technical record

The sections below preserve the complete investigation, source behavior, failed-build analysis, reproducible build proof, ELF audit, hardware validation, and limitations from the upstream pull request.

## Summary

This PR completes the missing PS5 Pro Viola firmware path for retail firmware 11.60 and adds a deliberately conservative first-run flow:

- adds the independently derived `off_1160.PS5_PRO_GPU_FW_OFFSET = 0x257F550`;
- enters the new path only on the PS5 Pro SoC;
- silently bypasses all GPU firmware-table and reconstruction work when the complete eight-file Viola set is already installed;
- after creating a missing set, calls `sync()` and returns before Linux staging, wake preparation, hypervisor defeat, or rest-mode entry; and
- allows the next run to boot normally using the installed files.

This is the minimal source delta that was built reproducibly with the original Ubuntu toolchain and successfully hardware-tested on a retail PS5 Pro running 11.60.

## User-visible result

With all eight validated files present, the final loader:

1. displayed no GPU firmware error;
2. displayed no old “reusing” or “found” notification;
3. performed no loader-side GPU table/reconstruction work;
4. entered rest mode normally;
5. woke normally; and
6. successfully booted Linux.

The resulting Linux boot uses the Viola firmware and reports the factory PS5 Pro topology:

```text
amdgpu 0000:20:00.0: PS5 Pro: using Viola firmware
mp1: Viola: RequestActiveWgp(30) failed: -22
mp1: Set BAPM params (Viola)
amdgpu 0000:20:00.0: SE 2, SH per SE 2, CU per SH 16, active_cu_number 60
```

`ps5_cu_unlock` was not used. Although `RequestActiveWgp(30)` returns `-22`, the GPU exposes the correct factory 30-WGP / 60-CU topology.

## Original 11.60 problem

The loader already contained PS5 Pro firmware reconstruction support, but `off_1160` did not initialize `PS5_PRO_GPU_FW_OFFSET`. Because `offset_list` is zero-initialized, the effective value was `0` and the existing guard skipped the GPU dump:

```c
if (env_offset.PS5_PRO_GPU_FW_OFFSET != 0 &&
    dump_ps5_pro_gpu_firmware(boot_file_path) < 0)
  ret = -1;
```

On the tested Bazzite Deck OS v1.8.9 image, the missing files caused AMDGPU to remain on the Cyan Skillfish fallback:

- before: `CU per SH 10`, `active_cu_number 36`;
- after: `CU per SH 16`, `active_cu_number 60`.

## Offset derivation and independent validation

The 11.60 value was derived from a retail PS5 Pro rather than copied from the 13.20/13.60 tables:

1. A read-only debugger established the 11.60 kernel-data base.
2. A bounded kernel-data window was captured with sequential read-only requests.
3. The capture was analyzed offline for aligned eight-pointer arrays.
4. Candidate entries were resolved as the loader's `gpu_firmware_blob_header`.
5. Candidates had to match the exact eight-entry structure and feature sequence.
6. Header bounds, canonical pointers, tag strings, payload lengths, jump-table bounds, RLC/SDMA checksums, and the guarded PFP bytes were validated.
7. Exactly one table survived.
8. Its absolute address was converted to the loader's kernel-text-relative coordinate system.
9. A second capture after a cold boot reproduced the same relative value.

Result:

```c
.PS5_PRO_GPU_FW_OFFSET = 0x257F550,
```

This field is qword index 39 in the 61-qword `offset_list`. No proprietary kernel dump or firmware payload is included in the PR.

## Reconstructed file set

The existing loader path reconstructs:

- `viola_ce.bin`
- `viola_me.bin`
- `viola_mec.bin`
- `viola_mec2.bin`
- `viola_pfp.bin`
- `viola_rlc.bin`
- `viola_sdma.bin`
- `viola_sdma1.bin`

The files used for runtime validation were checked independently for nonzero size, AMDGPU headers, `$PS1` layout and bounds, CRC32 values, and the guarded PFP patch.

## Detailed source behavior

### `source/offsets.c`

Adds the independently validated `0x257F550` value to `off_1160`.

### `source/firmware.c`

- Gates the GPU path with `sceKernelGetMainSocId() == 0x840fd0` because the 11.60 offset table is shared by regular and Pro consoles.
- Builds the boot directory without accessing GPU memory.
- Checks each expected path with `stat()`.
- Requires every entry to be a regular, nonempty file.
- If all eight pass, returns through the normal boot path without calling `dump_ps5_pro_gpu_firmware()`.
- If any entry fails, calls the existing reconstruction path, flushes writes with `sync()`, and returns `DEVICE_FIRMWARE_INSTALLED`.
- Preserves the prior behavior for other supported Pro firmware versions.

### `source/loader.c` and `include/loader.h`

- Converts `DEVICE_FIRMWARE_INSTALLED` into `FETCH_LINUX_FIRMWARE_INSTALLED`.
- Unmaps the temporary initrd and bzImage allocations before returning.
- Does not proceed into VRAM parsing, command-line parsing, 11.60 page alias staging, or ordinary page installation on that first run.

### `source/main.c`

- Handles the dedicated first-run result immediately after `fetch_linux()`.
- Notifies the user that firmware was installed/flushed and that the loader must be run again.
- Returns before `prepare_resume()`, all hypervisor-defeat paths, the five-second rest-mode countdown, and `enter_rest_mode()`.

### Installed-file bypass guarantee

When the set is complete, this path does not:

- read the GPU firmware pointer table;
- read any GPU blob header;
- derive payload sizes;
- copy or patch payloads;
- construct AMDGPU headers;
- truncate or rewrite files; or
- emit a “reusing” notification.

The regular/nonzero check is intentionally the only installed-set check. Reading and revalidating the contents would violate the goal of completely bypassing loader-side firmware work after a set has already been validated.

## Why the first candidate was rejected

An earlier Windows-built candidate wrote/recognized the firmware but left the console at a flashing blue light during the same-run Linux wake. It was preserved for analysis and immediately replaced by the known-working original.

Failed Windows candidate:

- 323,432 bytes
- SHA-256 `72586933f6b083c33eabf15040ad12ab1d2d8f976501cf5f96a9d9cab6336575`

Known-working original:

- 323,128 bytes
- SHA-256 `3d2d14ef4779712707b668534eb9fa909570dd14817123a63a984120ff658365`

ELF comparison proved that the Windows candidate was not boot-equivalent:

| Property | Working original | Failed Windows candidate |
| --- | --- | --- |
| Sections | 24 | 25 |
| Entry | `0x12f28` | `0x12cf8` |
| Dynamic hashes | GNU hash only | GNU hash plus SysV `.hash` / `HASH` |
| `mmap` | statically defined | unresolved |
| `mprotect` | statically defined | unresolved |
| `munmap` | unresolved | unresolved |

The staging path relies heavily on `mmap`/`mprotect`, making that linkage difference boot-critical.

The SDK wrapper ordering also differed: the Linux wrapper supplies `-lc` before project objects and places `crt1.o`, `libkernel_web`, `libSceLibcInternal`, and `libSceNet` after them. The Windows wrapper placed kernel libraries before the objects. Installing upstream LLVM 18.1.3 on Windows did not reproduce the official binary:

- untouched Windows baseline: 323,344 bytes;
- SHA-256: `babf91fb6e4abff429777c345e4448f627f239ffac3026353a08bfec80b57db3`.

No later Windows-produced ELF was sent to the console.

## Exact Linux toolchain reproduction

The final build used a clean Linux Codespace because the local Windows/WSL path could not provide a trustworthy PS5 link:

- `ps5-payload-sdk` v0.45;
- SDK archive SHA-256 `5fac64093d0e3e69f2584ce7935f24f703f622d9681f1495c85b9f00d1291a2c`;
- Ubuntu Clang 18.1.3 (`1ubuntu1`);
- Ubuntu LLD 18.1.3.

Before applying this PR, untouched commit `e7954b27ddc27b03ddbb6f49fb94100b0d1f271e` reproduced the known-working loader **byte for byte**:

- size: 323,128 bytes;
- SHA-256: `3d2d14ef4779712707b668534eb9fa909570dd14817123a63a984120ff658365`.

That exact match proved that the Linux environment reproduced the original compiler, linker, wrapper behavior, and generated shellcode inputs.

The patched tree was then built twice from clean state, serially because the current Makefile has a generated-header race under `make -j2`:

- build 1: 323,216 bytes, SHA-256 `4fadbfa17761d6de0c1eeb71fae88fa3b375b7b27f8ccb7258ddff11405cf041`;
- build 2: 323,216 bytes, same SHA-256;
- byte comparison: identical.

## Final ELF audit

Before hardware testing, the final ELF was compared with the byte-reproduced original:

- 24 sections;
- four program headers;
- compatible 16 KiB-aligned load/dynamic layout;
- GNU hash only, with no added `.hash` or `HASH` tag;
- `mmap` statically defined;
- `mprotect` statically defined;
- `munmap` unresolved, matching the original;
- Ubuntu Clang 18.1.3 provenance;
- Ubuntu LLD 18.1.3 provenance;
- qword 39 bytes `50 f5 57 02 00 00 00 00`, value `0x257F550`.

The first load segment grew by 640 bytes for the added code. The remaining segment layout was unchanged.

## Hardware test of the exact final ELF

The SHA-256-qualified final build was run on the retail PS5 Pro 11.60 with all eight validated files installed:

- no loader error;
- no “found” or “reusing” message;
- normal rest-mode transition;
- normal wake;
- successful Linux boot.

This tests the installed-file bypass and complete normal wake path of the exact final ELF. The missing-file path was not repeated after final build because doing so would have required removing the preserved validated set. The underlying extraction path had already created and validated that set; this PR changes when the process stops after extraction.

## Bazzite Deck OS initramfs packaging bug discovered during validation

The loader writes the files to the EFI/boot filesystem, but AMDGPU needs them inside the initramfs. Bazzite Deck OS v1.8.9's `/usr/local/sbin/ps5-stage-viola-fw` appends a `newc` CPIO, but currently concatenates it without aligning the next archive:

```bash
cat "$E/initrd.img.noviola" "$T/viola.cpio" > "$E/initrd.img.new"
```

The tested base was 100,924,166 bytes, requiring two zero bytes before the next `070701` header. Without them:

```text
Initramfs unpacking failed: invalid magic at start of compressed archive
```

Validated layout:

```text
base initrd.img.noviola: 100924166 bytes
padding:                 2 zero bytes
newc magic offset:       100924168
newc archive size:       272384 bytes
working initrd.img:      101196552 bytes
working SHA-256:         50aaa95909aacb9e28c5e64622b4ec9b05dd9cdf53266cc8f5b78b2659d6c17e
```

The general correction is:

```bash
base_size=$(stat -c %s "$E/initrd.img.noviola")
padding=$(( (4 - (base_size % 4)) % 4 ))
{
  cat "$E/initrd.img.noviola"
  if [ "$padding" -ne 0 ]; then
    dd if=/dev/zero bs=1 count="$padding" status=none
  fi
  cat "$T/viola.cpio"
} > "$E/initrd.img.new" &&
  mv "$E/initrd.img.new" "$E/initrd.img"
```

This external packaging bug and full runtime evidence are tracked at:

https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27

The manual Bazzite script edit is temporary. Once the Deck OS developer aligns appended archives automatically, users should not apply it. Similarly, once this loader fix is merged and released, users should obtain the loader from the official repository instead of the temporary fork prerelease.

## Post-fix runtime and game results

My current before-and-after tables, including 007 First Light at 4K Low and 1440p Ultra, are in [Game testing — updated November 1, 2026](#game-testing--updated-november-1-2026) above.

The combined loader and initramfs corrections activated Viola firmware with 60 factory-active CUs. The previously reproduced GTA V Legacy driving workload no longer fails.

## Published test artifact

The hardware-tested ELF is available only as an explicitly marked prerelease asset on the submitter's fork:

https://github.com/mfoster978/ps5-linux-loader/releases/tag/ps5-pro-11.60-viola-test-v1

It is not committed to this PR. No proprietary firmware file is included.

Users should switch to the official `ps5-linux/ps5-linux-loader` release after this change or an equivalent fix is published upstream.

## Limitations

- The final ELF's installed-file branch was hardware-tested; the final exact binary's first-run branch was not repeated after preserving the validated files.
- The installed-set test intentionally checks regular/nonempty files without reading their contents.
- The working initrd SHA-256 applies only to the exact tested v1.8.9 base and file set.
- Other console models, firmware versions, and distributions were not validated with the fork artifact.
- The Bazzite alignment workaround is external to this repository.
- No proprietary kernel or firmware dump is included.
## Credit request

If this work is merged or included in project/release notes, please credit GitHub user **[@mfoster978](https://github.com/mfoster978)** for:

- independently deriving and cold-boot validating the retail PS5 Pro 11.60 Viola firmware-table offset;
- providing the PS5 Pro hardware and performing the loader, rest-mode, wake, and Linux boot tests;
- isolating the Bazzite Deck OS `newc` initramfs alignment bug and validating the two-byte correction on v1.8.9;
- validating the eight reconstructed Viola files and the resulting 60-CU topology; and
- completing the post-fix GTA V Legacy, Manor Lords, RV There Yet?, and 007 First Light game tests.

## Public evidence and reproduction

Everything needed to inspect the source change and the tested artifact is public:

- exact source commit: https://github.com/mfoster978/ps5-linux-loader/commit/585bf78599d5ca95d3c180c59191a1c3f9320c58
- complete source diff: https://github.com/ps5-linux/ps5-linux-loader/pull/91/files
- hardware-tested prerelease and ELF asset: https://github.com/mfoster978/ps5-linux-loader/releases/tag/ps5-pro-11.60-viola-test-v1
- Bazzite issue with the pre-fix failure logs, initramfs layout, 36-CU/60-CU comparison, kernel Viola messages, and game results: https://github.com/VivaLaVent/ps5-bazzite-deck/issues/27

### Reproduce the compiler and linker baseline

The SDK release used for the build is public:

```text
https://github.com/ps5-payload-dev/sdk/releases/tag/v0.45
ps5-payload-sdk.zip SHA-256:
5fac64093d0e3e69f2584ce7935f24f703f622d9681f1495c85b9f00d1291a2c
```

The Linux environment reported:

```text
Ubuntu clang version 18.1.3 (1ubuntu1)
Ubuntu LLD 18.1.3 (compatible with GNU linkers)
```

Build the untouched baseline serially:

```bash
git checkout e7954b27ddc27b03ddbb6f49fb94100b0d1f271e
make PS5_PAYLOAD_SDK=/path/to/ps5-payload-sdk clean
make PS5_PAYLOAD_SDK=/path/to/ps5-payload-sdk
stat -c '%s' bin/ps5-linux-loader.elf
sha256sum bin/ps5-linux-loader.elf
```

Recorded untouched result:

```text
323128 bytes
3d2d14ef4779712707b668534eb9fa909570dd14817123a63a984120ff658365
```

That is byte-identical to the known-working original loader.

Build commit `585bf78599d5ca95d3c180c59191a1c3f9320c58` twice from clean state with the same commands. Both recorded outputs were:

```text
323216 bytes
4fadbfa17761d6de0c1eeb71fae88fa3b375b7b27f8ccb7258ddff11405cf041
```

`cmp` returned success for the two outputs.

### Independently inspect the ELF

Reviewers can inspect the downloaded asset with standard LLVM/GNU ELF tools:

```bash
sha256sum ps5-linux-loader.elf
readelf -h ps5-linux-loader.elf
readelf -S ps5-linux-loader.elf
readelf -l ps5-linux-loader.elf
readelf -d ps5-linux-loader.elf
nm -D ps5-linux-loader.elf | grep -E '^(.* )?(mmap|mprotect|munmap)$'
objdump -s --section=.data --start-address=0x33850 \
  --stop-address=0x33858 ps5-linux-loader.elf
```

Recorded final output properties:

```text
sections:              24
program headers:       4
segment alignment:     0x4000
dynamic hash:          GNU_HASH only
mmap:                  defined in ELF
mprotect:              defined in ELF
munmap:                unresolved, matching the original
off_1160 qword 39:     50 f5 57 02 00 00 00 00
decoded qword value:   0x257F550
```

The release asset metadata also publishes the same SHA-256 digest, allowing the downloaded ELF to be checked independently before use.
