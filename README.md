# linux-pc-crash-fixes

Crash / freeze fixes for this PC (Intel Core i5 650), collected from the
running machine on 2026-10-10.

Machine: `Intel(R) Core(TM) i5 CPU 650 @ 3.20GHz` (family 6, model 37 —
Nehalem/Clarkdale, known for buggy deep C-states).
See `boot/lscpu.txt`.

## Fix 1 — `intel_idle.max_cstate=1` (freezes / hard lockups)

Limits the `intel_idle` driver to C1 only, keeping the CPU out of the
buggy deep C-states that freeze this box.

Kernel cmdline fragment:

```
intel_idle.max_cstate=1
```

Verified active on the running Puppy (ResolutePup64 26.04, kernel 7.0.14):

```
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz pmedia=atahd pfix=fsck,fsckp intel_idle.max_cstate=1

$ cat /sys/module/intel_idle/parameters/max_cstate
1

$ cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
POLL
C1
```

Only `POLL` + `C1` are exposed — deep states are off, as intended.
Evidence copies in `boot/` (`proc-cmdline-puppy.txt`, `max_cstate.txt`,
`available-idle-states.txt`).

### Where it is persisted — Ubuntu GRUB owns the boot

GRUB is handled by Ubuntu 26.04.1 LTS on `/dev/sda2`. The flag is persisted
in two places there (evidence in `boot/ubuntu-default-grub.txt`,
`boot/ubuntu-40_custom.txt`):

1. `/etc/default/grub`:
   `GRUB_CMDLINE_LINUX_DEFAULT="intel_idle.max_cstate=1"` — covers all
   Ubuntu kernels (`/boot/vmlinuz-7.0.0-*`).
2. `/etc/grub.d/40_custom` — custom entries for the other OSes:
   - `ResolutePup64 26.04 (on /dev/sda3)`:
     `linux /vmlinuz pmedia=atahd pfix=fsck,fsckp intel_idle.max_cstate=1`
   - `ResolutePup64 26.04 - Copy SFS to RAM (on /dev/sda3)`: same + `,copy`
   - `EasyOS Excalibur (on /dev/sda6)`: `... rw intel_idle.max_cstate=1
     snd_hda_intel.power_save=0 ...` (full entry in
     `boot/easyos-excalibur-entry.txt`)

History: `/etc/grub.d/40_custom.bak.20261003` shows the Puppy entries
*without* the flag, so the flag was added to the Puppy entries after
2026-10-03 and has survived reboots since. The Puppy-local boot files
(`/initrd/mnt/dev_save/grub.cfg`, `/boot/grub/menu.lst`) were never the
boot path — they don't carry the flag and don't need it while Ubuntu's
GRUB chainloads `/vmlinuz` off `/dev/sda3` directly.

## Fix 2 — `snd_hda_intel.power_save=0` (audio power-save hangs/pops)

Disables HDA audio power-saving, which can hang or pop on wakeup.

```
snd_hda_intel.power_save=0
```

### Status: applied to Puppy 2026-10-10 (was EasyOS-only before)

- EasyOS entry (`40_custom:29`, `grub.cfg:352,384`): present since before.
- Ubuntu kernels (`/etc/default/grub`): absent (unchanged).
- Puppy entries: flag **added 2026-10-10** to both the source
  (`/dev/sda2:/etc/grub.d/40_custom:12,21`) and the generated
  (`/dev/sda2:/boot/grub/grub.cfg:335,344`) active entries:
  `linux /vmlinuz pmedia=atahd pfix=fsck,fsckp intel_idle.max_cstate=1
  snd_hda_intel.power_save=0` (and `,copy` variant).
  Backups: `40_custom.bak.20261010-presndhda`,
  `grub.cfg.bak.20261010-presndhda` on `/dev/sda2`.
  The stale duplicate section from `40_custom.bak.20261003`
  (`grub.cfg:367,376`) was left untouched.
- Running Puppy: applied at runtime the same day
  (`echo 0 > /sys/module/snd_hda_intel/parameters/power_save`;
  readback `0`). Previously at defaults (`power_save=1`,
  `power_save_controller=Y`, card `HDA Intel MID`).

Note: `grub.cfg` was patched directly so the next boot picks the flag
up; still advisable to run `update-grub` from Ubuntu so the generated
file and sources re-converge (also cleans the stale `.bak.20261003`
duplicate entries).

## Re-verify after any change

```sh
cat /proc/cmdline
cat /sys/module/intel_idle/parameters/max_cstate   # expect 1
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name  # expect POLL + C1 only
dmesg | head -3   # "Command line: ..." should show the flag
```
