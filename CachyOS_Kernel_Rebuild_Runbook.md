# CachyOS Kernel Rebuild And Test Runbook

This runbook is for testing the current Linux-side iMac 5K bring-up patch on CachyOS with `makepkg`.

The current patched files live here in the WT6A_INF workspace:

- [amdgpu_dm.c](/C:/proj/reverse/WT6A_INF/AMD_Linux/display/amdgpu_dm/amdgpu_dm.c)
- [amdgpu_dm.h](/C:/proj/reverse/WT6A_INF/AMD_Linux/display/amdgpu_dm/amdgpu_dm.h)
- [amdgpu_dm_quirks.c](/C:/proj/reverse/WT6A_INF/AMD_Linux/display/amdgpu_dm/amdgpu_dm_quirks.c)

## What This Patch Is Trying To Do

Stage 1 only:

- preserve the known-good probe-boot split state through the first real DRM userspace takeover
- keep the secondary `2560x2880` tiled half from vanishing when X11/Wayland starts
- add `IMAC5K:` debug logging so we can see exactly where the state is kept or dropped

It is not the final Apple-free cold-boot solution yet. It is the first Linux-side preservation step.

## Recommended Workflow

Use a custom CachyOS kernel package, but keep your current working kernel installed as fallback.

The safest loop is:

1. use the modified PKGBUILD as the base
2. let it pull the kernel from `https://github.com/mcirsta/linux-imac-5k`
3. let it pull the CachyOS-provided `config` from the CachyOS package repository
4. build with `makepkg -sri`
5. keep the stock kernel entry installed in the boot menu

## Prerequisites On CachyOS

Install the normal packaging/build dependencies you already use for kernel packages. At minimum:

```bash
sudo pacman -S --needed base-devel git
```

The current [PKGBUILD.txt](/C:/proj/reverse/WT6A_INF/PKGBUILD.txt) already pulls the CachyOS-provided kernel `config`, so you do not need to provide or customize a local `config` file for this path.

## Step 1: Put The PKGBUILD On A Large Filesystem

You need enough free space for a full kernel build. Put the build directory on a large Linux filesystem, not on a small root partition and not on a case-insensitive Windows checkout.

Copy the PKGBUILD into that build directory:

```bash
mkdir -p /big-disk/build/linux-imac-5k
cd /big-disk/build/linux-imac-5k
cp /path/to/WT6A_INF/PKGBUILD.txt ./PKGBUILD
```

The PKGBUILD fetches:

- kernel source from `https://github.com/mcirsta/linux-imac-5k.git#branch=main`
- CachyOS kernel config from `https://raw.githubusercontent.com/CachyOS/linux-cachyos/master/linux-cachyos/config`

## Step 2: Build And Install With makepkg

From the directory containing the custom PKGBUILD:

```bash
makepkg -sri
```

Recommended on the first pass:

- do not remove your old kernel package
- do not remove your old initramfs or boot entry
- keep at least one known-good fallback kernel in the boot menu

## Step 3: Boot Strategy For Stage 1 Testing

For Stage 1, still boot through the known-good probe/diagnostics path first.

That means:

1. boot through the USB/probe path that already gave us the good pre-GUI split state
2. select the new custom kernel from `systemd-boot`
3. test whether the graphical takeover still collapses to one visible `2560x2880` tile

Why:

- Stage 1 is about preserving the good state Linux already receives under probe boot
- it is not yet the plain cold-boot Apple-free synthesis test

## Step 4: What To Check After Boot

Immediately after booting the patched kernel, capture the new kernel logs:

```bash
dmesg | grep IMAC5K
```

The most important messages to watch are:

- `IMAC5K: boot_detect`
- `IMAC5K: connector`
- `IMAC5K: detect_preserve_secondary`
- `IMAC5K: crtc_update`
- `IMAC5K: commit_streams_begin`
- `IMAC5K: commit_streams_end`
- `IMAC5K: atomic_commit_tail_pre`
- `IMAC5K: atomic_commit_tail_post`

Then collect the usual display state:

```bash
drm_info > ~/imac5k_drm_info.txt 2>&1
modetest -M amdgpu -c -p -f > ~/imac5k_modetest.txt 2>&1
sudo sh -c 'cat /sys/kernel/debug/dri/0000:01:00.0/state > ~/imac5k_state.txt'
sudo sh -c 'cat /sys/kernel/debug/dri/0000:01:00.0/framebuffer > ~/imac5k_framebuffer.txt'
```

## What Success Looks Like For Stage 1

Success for this patch is not "plain cold boot now works."

Success is:

- probe boot still reaches the known-good pre-GUI split state
- after X11 or Wayland starts, Linux no longer collapses to one visible `2560x2880` tile
- the second half stays present in DRM/DC state, or Linux presents the panel as the correct full tiled display

## What Failure Looks Like

Common outcomes and what they mean:

- no `IMAC5K:` lines at all
  - the patched kernel is probably not the one you booted

- `IMAC5K:` lines appear, but the second half still vanishes
  - the quirk is active, but the preservation hook is too late or too weak

- boot failure or black screen
  - use the stock kernel entry and roll back the custom package

## Rollback

Because this is packaged as a separate custom kernel, rollback should be simple:

- boot the known-good stock kernel entry
- uninstall the custom package if needed
- or leave it installed and just stop selecting it

## Recommended Next Iteration Loop

For each patch revision:

1. push the updated Linux kernel changes to `https://github.com/mcirsta/linux-imac-5k`
2. rebuild with `makepkg -sri`
3. boot through the probe USB
4. capture `dmesg | grep IMAC5K` and DRM/DC state
5. compare against the last run

## Important Limitation Right Now

This runbook gets you to a real testable kernel, but I have not build-tested the kernel package itself from this Windows workspace.

That is normal for this stage. The safest path is still:

- keep the stock kernel installed
- build a renamed custom package
- use the probe USB only for Stage 1 validation
