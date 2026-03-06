# Arch Linux — ASUS Vivobook X1504VA

Documentation of a completed Arch Linux installation before migration to Fedora Silverblue.
This is a reference archive, not a dotfiles repo for reuse.

---

## Hardware

| Component | Value |
|-----------|-------|
| Model | ASUS Vivobook X1504VA |
| CPU | Intel Core i5-1334U (13th Gen, Raptor Lake-P) |
| GPU | Intel Iris Xe Graphics (PCI ID: a7a1) |
| RAM | 16 GB |
| Storage | 476.9 GB NVMe SSD |
| Keyboard | ABNT2 (Brazilian Portuguese layout) |
| Network | Intel Wi-Fi 6 (iwlwifi) |

---

## Stack

| Component | Choice | Why |
|-----------|--------|-----|
| OS | Arch Linux | Rolling release, full control over every layer |
| Filesystem | Btrfs | Native snapshots, transparent compression, subvolume layout |
| Encryption | LUKS2 + argon2id | Memory-hard KDF, resistant to GPU/ASIC brute-force |
| Bootloader | systemd-boot | Simple EFI-native loader, no GRUB complexity |
| Initramfs | mkinitcpio + systemd hooks | sd-encrypt hook integrates cleanly with LUKS2 |
| GPU Driver | Intel Xe (xe.force_probe) | Required for Raptor Lake-P Xe GPU; i915 blocked per-device |
| Snapshots | Snapper | Automated pre/post pairs, timeline cleanup, Btrfs-native |
| Firewall | nftables | Native kernel netfilter, stateless ruleset |
| Containers | Docker (overlay2 on Btrfs) | Standard tooling; overlay2 forced over btrfs driver |
| Display | Wayland / Hyprland | Wayland-native compositor, tiling WM |
| Terminal | Kitty | GPU-accelerated, good Wayland support |
| Bar | Waybar | Modular status bar for Hyprland |
| Launcher | rofi | Fast application launcher |
| Audio | PipeWire + WirePlumber | Modern audio graph, low-latency |

---

## Key Technical Decisions

### LUKS2 with argon2id over default settings

The default LUKS2 format uses PBKDF2 as the key derivation function. PBKDF2 is purely CPU-bound: its cost is measured in iterations per second on a single core, which means an attacker with access to GPU clusters or ASICs can evaluate guesses at rates orders of magnitude faster than what the target hardware uses for calibration. The extra hardware the attacker brings is directly leveraged against the passphrase.

argon2id is memory-hard. It requires a configurable amount of RAM during key derivation, which breaks the parallelism advantage of GPU attacks. A GPU with thousands of cores cannot run thousands of simultaneous argon2id evaluations unless it has thousands of simultaneous large memory allocations—which it doesn't. The practical effect is that the attacker's hardware advantage collapses: a high-memory, low-parallelism operation is much harder to scale than a low-memory, high-parallelism one.

The TRIM passthrough options (`discard=async` in fstab, `rd.luks.options=discard` in the kernel cmdline) were deliberately enabled, trading minor metadata leakage (an attacker can observe which sectors are in use) for SSD health and write performance on the NVMe device. On a laptop with no threat model requiring deniability of data presence, this is the right call.

---

### Btrfs subvolume layout (7 subvolumes)

The installation uses 7 Btrfs subvolumes: `@` (root), `@home`, `@snapshots`, `@var-log`, `@var-cache`, `@var-tmp`, and `@docker`. The motivation is containment: a Btrfs snapshot captures the entire state of the subvolume it targets. If everything lives in a single subvolume, snapshots capture ephemeral and volatile data—package caches, Docker image layers, log files, temp files—alongside the actual system state. This makes snapshots bloated, diffs noisy, and rollbacks unpredictable.

Separating `@home` from `@` enables independent rollback vectors: rolling back root after a broken update does not discard in-progress home directory changes. `@snapshots` must be its own subvolume to prevent the snapshot directory from appearing inside root snapshots, which would be recursive and breaks Snapper's logic. `@docker` isolation means Docker's write-heavy layer churn never enters system snapshots at all—Docker manages its own state and does not benefit from being snapshot with the OS.

`@var-log`, `@var-cache`, and `@var-tmp` are separated because they contain data that is either intentionally ephemeral, actively rotated, or regenerable—none of which should contribute to OS snapshot deltas. The key insight is that Btrfs subvolumes are the right primitive for this kind of isolation, not bind mounts or directory structure alone.

---

### systemd-boot over GRUB

GRUB is the default recommendation in much of the Arch documentation, but on a pure UEFI system with no legacy boot requirement, it introduces significant unnecessary complexity: its own configuration language (`grub.cfg`), `grub-mkconfig` tooling, module loading system, theme engine, and a non-trivial maintenance surface. systemd-boot is a trivially simple EFI application: it reads `.conf` files from the ESP, passes kernel parameters, and exits. The entire boot configuration for this system is 5 lines of plaintext.

The practical argument: GRUB's flexibility is completely wasted here. There is one OS, one kernel, one set of boot parameters. systemd-boot handles this without any abstraction layer between the config file and what the kernel receives. Updates integrate cleanly via `bootctl update` and the `systemd-boot-update.service` unit, which can be triggered automatically by pacman hooks. GRUB also has a history of high-profile CVEs related to its complexity (BootHole and follow-on vulnerabilities); reducing bootloader complexity reduces attack surface in a component that runs before any OS-level security is active.

---

### Intel Xe driver with force_probe (and why i915 must be blocked)

The Intel Raptor Lake-P GPU (PCI device ID `a7a1`) sits in a transitional position in the Linux driver ecosystem. The `i915` driver is the longstanding Intel GPU driver and will claim this device ID by default at kernel module load time. However, `i915` does not fully support the Xe microarchitecture, resulting in suboptimal performance and potential instability. The newer `xe` driver correctly handles the hardware but is gated behind `xe.force_probe=a7a1` because the device ID had not yet been promoted to stable support in the `xe` driver at setup time.

The critical, non-obvious detail is `i915.force_probe=!a7a1`: the `!` prefix is a per-device blacklist directive. Without it, `i915` loads first (kernel driver priority), claims the GPU, and `xe` never binds—even if `xe` is present and `xe.force_probe` is set. Both parameters must appear together in the kernel command line. Setting only `xe.force_probe=a7a1` without the `i915` exclusion results in `i915` still owning the device and `xe` having nothing to bind to. This is a case where understanding Linux kernel driver binding order matters more than knowing which driver to use.

---

### Docker overlay2 on Btrfs (the conflict and the fix)

Docker has a known storage driver conflict with Btrfs. When Docker detects Btrfs as the underlying filesystem for its data directory, it historically selects the `btrfs` storage driver automatically. The `btrfs` driver is slow, poorly maintained, and lacks the overlay2 feature set. Even when `"storage-driver": "overlay2"` is set in `daemon.json`, Docker performs a filesystem-type check at startup and may override or warn against the configuration.

The fix used here is a combination: Docker's data directory (`/var/lib/docker`) is mounted from its own Btrfs subvolume (`@docker`), and `overlay2` is explicitly declared in `/etc/docker/daemon.json`. The subvolume mount means Docker's directory is on Btrfs at the VFS level, but overlay2 constructs its own directory structure (`overlay2/`, `image/`, `volumes/`) inside that mount without conflicting with Btrfs's own snapshot mechanics. Log rotation (`max-size: 10m`, `max-file: 3`) was configured at the same time—the default is no rotation, which on a system with a bounded `/var/log` subvolume and no automatic log pruning is a slow disk exhaustion waiting to happen.

---

### Snapper setup sequence (why the order of operations matters)

Snapper requires the `@snapshots` subvolume to exist and be mounted at `/.snapshots` before it creates its root configuration. If you run `snapper -c root create-config /` before manually creating the subvolume and adding the fstab entry, Snapper creates a `.snapshots` directory inside the `@` subvolume as a plain directory. This defeats the isolation goal entirely: snapshots end up nested inside root snapshots, and the `/.snapshots` fstab entry conflicts with what Snapper created, resulting in mount errors or boot failures depending on timing.

The correct sequence: (1) create the `@snapshots` Btrfs subvolume manually on the unmounted or live filesystem, (2) add the mount entry to `/etc/fstab`, (3) mount it at `/.snapshots`, (4) then run `snapper -c root create-config /`. Snapper detects the existing mountpoint and uses it correctly. If you have already run `create-config` without the subvolume, recovery requires: removing the `.snapshots` directory Snapper created, creating the subvolume, adding and mounting the fstab entry, and re-running `create-config`. Discovering this after the first snapshot cycle, when Snapper has already written into the wrong location, makes the recovery more involved.

The snapper config enables timeline snapshots (hourly, daily), pre/post pairs for package operations via the `snap-pac` package, and cleanup rules to bound snapshot count. The `ALLOW_GROUPS="wheel"` setting allows non-root users in the wheel group to manage snapshots.

---

## What I Would Do Differently

- **Script the installation end-to-end.** Documenting after the fact reveals how many decisions were made in an undocumented order. A reproducible install script forces you to make sequencing explicit, catches ordering bugs, and produces documentation as a byproduct.
- **Use a detached LUKS header.** Storing the LUKS header on a separate USB device means the encrypted partition is not even recognizable as LUKS without the header device present. The threat model here didn't require it, but the hardware supports it and the cost is low.
- **Configure snapper-rollback before needing it.** Setting up the rollback mechanism (bootloader entries for snapshot boots) should happen during installation, not after the first botched update.
- **Evaluate ext4 for `/var/lib/docker`.** The overlay2/Btrfs interaction works but requires ongoing awareness. A dedicated ext4 partition or loop device for Docker's data directory would eliminate the friction entirely.
- **Test the `xe` driver situation earlier.** The `i915.force_probe=!a7a1` + `xe.force_probe=a7a1` combination took debugging time to arrive at. Running hardware validation immediately post-install rather than assuming the GPU was working would have surfaced this sooner.

---

## Reference Configs

| File | Path in this repo |
|------|------------------|
| systemd-boot entry | [config/arch.conf](config/arch.conf) |
| systemd-boot loader | [config/loader.conf](config/loader.conf) |
| mkinitcpio | [config/mkinitcpio.conf](config/mkinitcpio.conf) |
| fstab | [config/fstab](config/fstab) |
| nftables firewall | [config/nftables.conf](config/nftables.conf) |
| Docker daemon | [config/docker-daemon.json](config/docker-daemon.json) |
| Snapper root config | [config/snapper-root.conf](config/snapper-root.conf) |
| Hyprland | [config/hyprland.conf](config/hyprland.conf) |
| Explicit package list | [config/pkglist.txt](config/pkglist.txt) |
