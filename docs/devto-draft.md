# 5 decisions that made my Arch Linux setup actually work

I run an ASUS Vivobook with a 13th Gen Intel i5-1334U (Raptor Lake-P, Intel Iris Xe). Full disk encryption, Btrfs, Hyprland on Wayland. Everything works. Getting there required five decisions that were non-obvious and each solved a real problem.

This is what those decisions were and why each one mattered.

---

## 1. LUKS2 with argon2id, not PBKDF2

The default LUKS2 KDF is PBKDF2. It's CPU-bound, which means its calibrated cost on your hardware has no bearing on the cost an attacker pays using a GPU cluster. An attacker with a modern GPU can evaluate PBKDF2 at rates that dwarf what your laptop calculated for "2 seconds of work." The math works against you.

argon2id is memory-hard. It requires a large RAM allocation per evaluation. A GPU with thousands of cores cannot run thousands of simultaneous argon2id evaluations unless it also has thousands of simultaneous large memory windows—which it doesn't. The attacker's parallelism advantage evaporates.

The command is straightforward: `cryptsetup luksConvertKey --pbkdf argon2id /dev/nvme0n1p2`. One flag. The unlock time at boot increases by a fraction of a second. The brute-force resistance increases by orders of magnitude.

---

## 2. Seven Btrfs subvolumes, not one

Most guides tell you to create `@` and `@home`. That's the minimum. It's not enough.

My layout: `@`, `@home`, `@snapshots`, `@var-log`, `@var-cache`, `@var-tmp`, `@docker`.

The reason: a Btrfs snapshot captures the entire state of its subvolume. If Docker image layers, package caches, log files, and temp data all live inside `@`, every system snapshot drags all of that along. Snapshots become bloated. Diffs become meaningless. Rollbacks become unpredictable.

`@snapshots` must be its own subvolume or Snapper recurses into itself. `@docker` must be isolated or Docker's write churn gets snapshotted constantly on every pre/post package operation. `@var-cache` and `@var-tmp` contain data that is either regenerable or intentionally ephemeral—neither should ever appear in a system rollback.

The extra fstab entries are worth it. The snapshot quality difference is significant.

---

## 3. systemd-boot, not GRUB

GRUB is powerful and complex. On a pure UEFI system with one OS and one kernel, all of that power is waste.

systemd-boot is an EFI application that reads `.conf` files from the ESP and passes parameters to the kernel. The complete boot configuration for this machine is five lines:

```
default arch.conf
timeout 3
console-mode max
editor no
```

That's it. No `grub-mkconfig`. No module loading. No theme engine. No `grub.cfg` you don't want to touch by hand.

Updates happen via `bootctl update`, which integrates with pacman hooks. The entire bootloader is auditable at a glance. GRUB has had multiple high-profile CVEs (BootHole and its follow-ons) that trace directly back to its complexity. Reducing attack surface in the component that runs before any OS security is active is a real benefit, not a theoretical one.

---

## 4. Intel Xe driver: you need two kernel parameters, not one

This one took debugging time to figure out.

The Raptor Lake-P GPU (PCI ID `a7a1`) is supported by the new `xe` driver, not the legacy `i915` driver. The parameter to enable it is `xe.force_probe=a7a1`. This is documented. What is not obvious: setting only that parameter does nothing.

`i915` loads first. It claims device `a7a1` because `a7a1` is in its device table. By the time `xe` tries to bind, the device is already owned. `xe.force_probe` has nothing to probe.

The fix is a second parameter: `i915.force_probe=!a7a1`. The `!` prefix is a per-device exclusion directive. It tells `i915` to skip this specific device ID during its probe loop. Now `xe` can bind.

Both parameters must appear in the kernel cmdline together:

```
i915.force_probe=!a7a1 xe.force_probe=a7a1
```

Either one alone does not work. This pattern applies to any situation where a newer driver needs to claim a device that an older driver would otherwise grab first.

---

## 5. Docker overlay2 on Btrfs: explicit config is not enough

Docker's overlay2 storage driver and Btrfs have a well-known conflict. Docker detects the underlying filesystem type and historically selects the `btrfs` storage driver automatically when it finds Btrfs. The `btrfs` driver is slower than overlay2 and less well-maintained.

Setting `"storage-driver": "overlay2"` in `/etc/docker/daemon.json` is the obvious fix. It is also not sufficient by itself—Docker may warn or override depending on the kernel version and Docker version.

The complete fix: mount `/var/lib/docker` from its own Btrfs subvolume (`@docker`). Docker's directory sits on Btrfs at the filesystem level, but overlay2 constructs its own internal directory structure without touching Btrfs snapshot mechanics. The explicit `daemon.json` override locks in the driver selection. Both together eliminate the conflict.

The same session: add log rotation. The default Docker log configuration has no size limit. On a system where `/var/log` is a bounded Btrfs subvolume, unbounded container logs will eventually fill it. `"max-size": "10m"` and `"max-file": "3"` in `log-opts` caps this at 30 MB per container.

---

## What I'd do differently

**Script the install.** Documenting afterward reveals how many ordering decisions were implicit. A reproducible install script forces sequencing to be explicit and produces documentation as a byproduct.

**Set up snapshot rollback before needing it.** Snapper creates snapshots. Actually booting into one requires bootloader entries for the snapshot subvolume. Configure that at install time.

**Validate GPU driver binding immediately post-install.** `lspci -k` takes five seconds. Running it before assuming the GPU is working correctly would have surfaced the `i915`/`xe` conflict before it became a debugging session.

**Get the Snapper subvolume order right the first time.** Create `@snapshots`, add the fstab entry, mount it, *then* run `snapper create-config`. If you run `create-config` first, Snapper creates a plain directory inside `@` and the fstab entry conflicts. Recovery from this is annoying. The correct sequence is not obvious from the documentation.
