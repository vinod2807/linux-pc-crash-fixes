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

### Where it is persisted

Persisted for EasyOS, on `/dev/sda1` (`vfat`), file `/grub/custom.cfg`:

```
menuentry 'EasyOS Excalibur 7.4.10' --class easyos --class gnu-linux --class gnu --class os --id easyos-excalibur {
    ...
    search --no-floppy --fs-uuid --set=root e093e860-eddd-4457-88b3-0017b647d6e7
    echo 'Loading EasyOS Excalibur ...'
    linux /easyos/excalibur/vmlinuz rw intel_idle.max_cstate=1 snd_hda_intel.power_save=0 wkg_uuid=e093e860-eddd-4457-88b3-0017b647d6e7 wkg_dir=easyos/excalibur
    initrd /easyos/excalibur/initrd
}
```

Source: `/dev/sda1:/grub/custom.cfg` (EasyOS entry). Full entry saved in
`boot/easyos-excalibur-entry.txt`.

### NOT persisted for Puppy — one-time boot edit

The running Puppy has the flag, but none of the on-disk Puppy boot files do:

- `/initrd/mnt/dev_save/grub.cfg` (`linux /vmlinuz pmedia=cd pfix=fsck,fsckp`)
- `/initrd/mnt/dev_save/boot/grub/menu.lst` (`kernel /vmlinuz pmedia=cd pfix=fsck,fsckp`)

So the current Puppy boot was edited once at the GRUB prompt (`e`).
To make it permanent, append `intel_idle.max_cstate=1` to the `linux` /
`kernel` line of the used entry in both files above (they are two views of
the same Puppy install on `/dev/sda3`, LABEL `ResolutePup`).

## Fix 2 — `snd_hda_intel.power_save=0` (audio power-save hangs/pops)

Disables HDA audio power-saving, which can hang or pop on wakeup.

```
snd_hda_intel.power_save=0
```

Lives in the same EasyOS `linux` line quoted above. Add it next to
`intel_idle.max_cstate=1` if backporting to Puppy.

## Re-verify after any change

```sh
cat /proc/cmdline
cat /sys/module/intel_idle/parameters/max_cstate   # expect 1
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name  # expect POLL + C1 only
dmesg | head -3   # "Command line: ..." should show the flag
```
