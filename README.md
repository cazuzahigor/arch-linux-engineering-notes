# Arch Linux — ASUS Vivobook X1504VA

A documented Arch Linux installation where every technical decision has a recorded rationale.

This is not a dotfiles collection or an install guide. It is an engineering reference: full disk encryption, Btrfs snapshots, Wayland compositing, and kernel driver configuration on a 13th Gen Intel laptop — with the reasoning behind each choice preserved for future reference.

> **Note:** This system has since been migrated to Fedora Silverblue. The repository is maintained as an archive of the decisions and configurations described below.

---

## Key Decisions

Five non-obvious problems came up during this build. Each one required understanding something deeper than the surface-level fix.

### 1. LUKS2 with argon2id, not PBKDF2

The default LUKS2 key derivation function is PBKDF2. It is CPU-bound, which means its calibrated cost on the target hardware has no bearing on the cost an attacker pays using a GPU cluster. A modern GPU evaluates PBKDF2 guesses at rates that dwarf what a laptop calibrates for "2 seconds of work."

argon2id is memory-hard. Each evaluation requires a large RAM allocation, and a GPU with thousands of cores cannot run thousands of simultaneous argon2id evaluations without thousands of simultaneous large memory windows — which it does not have. The parallelism advantage collapses.

One command, one flag: `cryptsetup luksConvertKey --pbkdf argon2id /dev/nvme0n1p2`. The unlock time increases by a fraction of a second. The brute-force resistance increases by orders of magnitude.

TRIM passthrough (`discard=async` in fstab, `rd.luks.options=discard` in the kernel cmdline) was deliberately enabled, trading minor metadata leakage for SSD health and write performance. On a laptop with no deniability requirement, this is the correct trade-off.

### 2. Seven Btrfs subvolumes, not one

Most guides create `@` and `@home`. That is the minimum. It is not enough.

Layout: `@`, `@home`, `@snapshots`, `@var-log`, `@var-cache`, `@var-tmp`, `@docker`.

A Btrfs snapshot captures the entire state of its target subvolume. If Docker image layers, package caches, log files, and temporary data all live inside `@`, every system snapshot drags that along. Snapshots become bloated, diffs become meaningless, and rollbacks become unpredictable.

The isolation logic:

- `@snapshots` must be its own subvolume. Without it, Snapper recurses into itself — the snapshot directory appears inside the snapshot it is creating.
- `@docker` must be isolated. Docker's write-heavy layer churn would otherwise be captured in every pre/post package operation snapshot.
- `@var-log`, `@var-cache`, and `@var-tmp` contain data that is either regenerable or intentionally ephemeral. None of it should contribute to OS snapshot deltas.
- `@home` enables independent rollback: reverting root after a broken update does not discard in-progress work in the home directory.

The extra fstab entries cost nothing. The snapshot quality difference is significant.

### 3. systemd-boot, not GRUB

On a pure UEFI system with one OS and one kernel, GRUB's power is entirely wasted. Its configuration language, module loading system, theme engine, and `grub-mkconfig` tooling are complexity without benefit.

systemd-boot is an EFI application that reads `.conf` files from the ESP and passes parameters to the kernel. The entire boot configuration for this machine is five lines:

```
default arch.conf
timeout 3
console-mode max
editor no
```

Updates happen via `bootctl update`, which integrates with pacman hooks. The bootloader is auditable at a glance. GRUB has had multiple high-profile CVEs (BootHole and its follow-ons) that trace directly to its complexity. Reducing attack surface in the component that runs before any OS-level security is active is a practical benefit, not a theoretical one.

### 4. Intel Xe driver: two kernel parameters, not one

The Raptor Lake-P GPU (PCI device ID `a7a1`) is supported by the newer `xe` driver, not the legacy `i915`. The documented parameter to enable it is `xe.force_probe=a7a1`. Setting only that parameter does nothing.

`i915` loads first during the kernel's device probe phase. It claims `a7a1` because the device ID exists in its table. By the time `xe` attempts to bind, the GPU is already owned. `xe.force_probe` has nothing to probe.

The fix is a second parameter: `i915.force_probe=!a7a1`. The `!` prefix is a per-device exclusion directive. It tells `i915` to skip this specific device ID during its probe loop. Now `xe` can bind.

Both parameters must appear together in the kernel command line:

```
i915.force_probe=!a7a1 xe.force_probe=a7a1
```

Either one alone does not work. This is a case where understanding Linux kernel driver binding order matters more than knowing which driver to use.

### 5. Docker overlay2 on Btrfs: explicit config is not enough

Docker detects the underlying filesystem and historically selects the `btrfs` storage driver when it finds Btrfs. The `btrfs` driver is slower and less maintained than overlay2.

Setting `"storage-driver": "overlay2"` in `/etc/docker/daemon.json` is the obvious fix. It is not sufficient by itself — Docker may warn or override depending on the kernel and Docker versions.

The complete fix: `/var/lib/docker` is mounted from its own Btrfs subvolume (`@docker`), and overlay2 is explicitly declared in `daemon.json`. Docker's directory sits on Btrfs at the VFS level, but overlay2 constructs its own internal structure without conflicting with Btrfs snapshot mechanics.

Log rotation was configured at the same time. The default Docker log configuration has no size limit. On a system where `/var/log` is a bounded subvolume, unbounded container logs will eventually fill it. `"max-size": "10m"` and `"max-file": "3"` caps this at 30 MB per container.

---

## Snapper: why the order of operations matters

Snapper requires the `@snapshots` subvolume to exist and be mounted at `/.snapshots` before creating its root configuration. Running `snapper -c root create-config /` without the subvolume in place causes Snapper to create a plain `.snapshots` directory inside `@`. This defeats the isolation goal: snapshots end up nested inside root snapshots, and the fstab entry conflicts with what Snapper created.

The correct sequence:

1. Create the `@snapshots` subvolume manually
2. Add the mount entry to `/etc/fstab`
3. Mount it at `/.snapshots`
4. Run `snapper -c root create-config /`

If `create-config` has already been run without the subvolume, recovery requires removing the directory Snapper created, creating the subvolume, mounting it, and re-running `create-config`. This sequence is not obvious from the documentation.

---

## Hardware

| Component | Specification |
|-----------|---------------|
| Model | ASUS Vivobook X1504VA |
| CPU | Intel Core i5-1334U (13th Gen, Raptor Lake-P) |
| GPU | Intel Iris Xe Graphics (PCI ID: `a7a1`) |
| RAM | 16 GB |
| Storage | 476.9 GB NVMe SSD |
| Keyboard | ABNT2 (Brazilian Portuguese) |
| Network | Intel Wi-Fi 6 (`iwlwifi`) |

## Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Filesystem | Btrfs | Native snapshots, transparent compression, subvolume isolation |
| Encryption | LUKS2 + argon2id | Memory-hard KDF, resistant to GPU/ASIC brute-force |
| Bootloader | systemd-boot | Minimal EFI-native loader, no GRUB complexity |
| Initramfs | mkinitcpio + systemd hooks | `sd-encrypt` integrates cleanly with LUKS2 |
| GPU driver | Intel Xe (`xe.force_probe`) | Required for Raptor Lake-P; `i915` blocked per-device |
| Snapshots | Snapper | Automated pre/post pairs, timeline cleanup, Btrfs-native |
| Firewall | nftables | Kernel-native netfilter, stateless ruleset |
| Containers | Docker (overlay2 on Btrfs) | overlay2 forced over btrfs driver via `daemon.json` |
| Display | Wayland / Hyprland | Wayland-native tiling compositor |
| Terminal | Kitty | GPU-accelerated, Wayland-native |
| Audio | PipeWire + WirePlumber | Modern audio graph, low-latency routing |

## What I Would Do Differently

- **Script the installation end-to-end.** Documenting after the fact reveals how many decisions were made in an undocumented order. A reproducible install script forces sequencing to be explicit and produces documentation as a byproduct.
- **Use a detached LUKS header.** Storing the header on a separate device means the encrypted partition is not recognizable as LUKS without it. The threat model here did not require it, but the cost is low.
- **Configure snapper-rollback before needing it.** Setting up bootloader entries for snapshot boots should happen during installation, not after the first broken update.
- **Evaluate ext4 for `/var/lib/docker`.** The overlay2/Btrfs interaction works but requires ongoing awareness. A dedicated ext4 partition for Docker's data directory would eliminate the friction entirely.
- **Validate GPU driver binding immediately post-install.** `lspci -k` takes five seconds and would have surfaced the `i915`/`xe` conflict before it became a debugging session.

## Reference Configs

| File | Path |
|------|------|
| systemd-boot entry | [`config/arch.conf`](config/arch.conf) |
| systemd-boot loader | [`config/loader.conf`](config/loader.conf) |
| mkinitcpio | [`config/mkinitcpio.conf`](config/mkinitcpio.conf) |
| fstab | [`config/fstab`](config/fstab) |
| nftables firewall | [`config/nftables.conf`](config/nftables.conf) |
| Docker daemon | [`config/docker-daemon.json`](config/docker-daemon.json) |
| Snapper root config | [`config/snapper-root.conf`](config/snapper-root.conf) |
| Hyprland | [`config/hyprland.conf`](config/hyprland.conf) |
| Package list | [`config/pkglist.txt`](config/pkglist.txt) |
