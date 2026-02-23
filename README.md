# Proxmox USB Backup Manager

A single-script installer that turns a removable USB SSD into a safe, rotation-ready backup storage for Proxmox VE — with SMART health monitoring, sha256 manifest tracking, and clean mount/unmount lifecycle management.

## The Problem

Using a USB drive for Proxmox backups sounds simple until you hit the sharp edges:

- **Emergency mode on reboot** — a missing USB drive with a standard fstab entry drops Proxmox into emergency mode, requiring console access to recover.
- **Device path roulette** — `/dev/sdb` today might be `/dev/sdc` tomorrow. Hardcoded paths in fstab or scripts break silently.
- **No safe disconnect workflow** — Proxmox shows a `?` on the storage, vzdump writes fail mid-backup, or you corrupt the filesystem by unplugging at the wrong time.
- **Silent drive failure** — SSDs degrade without warning. By the time you discover your backup drive has bad sectors, your backups are already corrupted.
- **Drive rotation is painful** — swapping between onsite/offsite drives means editing fstab and re-registering storage every time.

This tool solves all of that with one setup script and five management commands.

## Features

| Feature | What it does |
|---|---|
| **Auto-detect USB devices** | Scans for removable storage, excludes boot disk, handles manual selection as fallback |
| **Safe fstab configuration** | Uses `nofail` + `noauto` + `x-systemd.automount` + `x-systemd.device-timeout=5` — system boots clean even if drive is absent |
| **LABEL-based mounting** | Fstab uses `LABEL=pve-backup` instead of UUID — any drive with the right label works, enabling seamless rotation |
| **Pre-format detection** | Detects existing `pve-backup` filesystems and offers to re-register without formatting (preserves data when moving drives between nodes) |
| **SMART health monitoring** | Checks SSD health on every mount — temperature, wear level, reallocated sectors, CRC errors, unexpected power losses. Supports USB-SATA bridge passthrough |
| **Backup manifests** | Generates sha256 checksums of all vzdump files on unmount. Archives dated copies for offline cross-drive comparison |
| **Integrity verification** | Verify files against manifests (size + existence), re-hash for bit rot detection, compare manifests across rotation drives |
| **Vzdump lock protection** | Refuses to unmount while a vzdump backup is running |
| **Disk space warnings** | Warns at 80% usage, alerts critically at 90% |
| **Safe eject** | Syncs, unmounts, stops systemd automount, powers off USB device via udisksctl or sysfs fallback |
| **Comprehensive logging** | All operations timestamped in `/var/log/usb-backup.log` with caller identification |
| **Clean uninstall** | Removes storage registration, fstab entries, systemd units, scripts, and config — with fstab backup before any change |

## Requirements

- Proxmox VE 7.x or 8.x
- Root access
- USB SSD/HDD (ext4 will be created by the script)
- `smartmontools` (auto-installed if missing)

## Quick Start

```bash
# Download
wget -O /root/setup-usb-backup.sh https://raw.githubusercontent.com/YOURUSER/proxmox-usb-backup/main/setup-usb-backup.sh

# Connect your USB SSD and run
bash /root/setup-usb-backup.sh
```

The script will:

1. Scan for removable USB devices and let you select one
2. Check if the drive is already set up (skip format if so)
3. Format as ext4 with GPT partition table (with double confirmation)
4. Configure fstab with safe mount options
5. Register as Proxmox storage (`usb-backup` with `is_mountpoint=1`)
6. Install five management commands
7. Run an initial SMART health check

## Usage

### Daily Workflow

```bash
# 1. Connect USB SSD

# 2. Mount and enable (SMART check runs automatically)
usb-backup-mount

# 3. Run backups via Proxmox UI
#    Datacenter → any VM → Backup → Storage = usb-backup

# 4. When done — generates manifest, disables storage, unmounts, powers off
usb-backup-unmount --eject

# 5. Physically disconnect the drive
```

### Commands Reference

#### `usb-backup-mount`

Mount the USB backup drive and enable it in Proxmox.

```
usb-backup-mount              # Mount with SMART health check
usb-backup-mount --no-smart   # Mount without SMART check (faster)
```

What happens:
- Resolves device by filesystem label
- Runs SMART health check (warns on degradation)
- Mounts to `/mnt/usb-backup`
- Enables the `usb-backup` storage in Proxmox
- Checks disk space utilization

#### `usb-backup-unmount`

Safely disconnect the backup drive.

```
usb-backup-unmount                # Manifest + disable + unmount
usb-backup-unmount --eject        # + power off device for safe physical removal
usb-backup-unmount --no-manifest  # Skip manifest generation (faster)
usb-backup-unmount --force        # Skip vzdump/open-file safety checks
```

What happens:
- Checks for running vzdump processes (blocks if found)
- Checks for open files on the mount (warns and asks)
- Generates sha256 manifest of all backup files
- Disables the Proxmox storage
- Syncs and unmounts the filesystem
- Stops the systemd automount unit
- With `--eject`: powers off the USB device

#### `usb-backup-status`

Show comprehensive status overview.

```
usb-backup-status              # Device, mount, storage, manifests
usb-backup-status --smart      # + live SMART health check
usb-backup-status --log        # + last 20 log entries
usb-backup-status --log=50     # + last 50 log entries
```

Example output:

```
╔══════════════════════════════════════════════════════════════╗
║              USB Backup Storage Status                     ║
╚══════════════════════════════════════════════════════════════╝

  Device:     CONNECTED
  Partition:  /dev/sdb1  (disk: /dev/sdb)
  UUID:       a1b2c3d4-e5f6-7890-abcd-ef1234567890
  Label:      pve-backup
  Hardware:   sdb  465.8G  Samsung_T7  usb

  Mount:      MOUNTED at /mnt/usb-backup
  Disk:       89G used / 458G total (348G free, 21%)
  Backups:    12 vzdump file(s)
  Latest:     vzdump-qemu-100-2026_02_23-03_00_01.vma.zst
  Manifest:   2026-02-23T14:30:00+01:00

  Proxmox:    usb-backup — active

  Manifests:  7 archived (in /etc/usb-backup/manifests)
  SMART:      last check 2026-02-23 14:25  (--smart for live check)
```

#### `usb-backup-verify`

Verify backup integrity and compare drives.

```
usb-backup-verify                          # Check files vs manifest (size + existence)
usb-backup-verify --rechecksum             # Re-hash all files and compare to manifest
usb-backup-verify --compare <file>         # Compare drive manifest vs a saved manifest
usb-backup-verify --compare <a> <b>        # Compare two manifest files directly
usb-backup-verify --list                   # List all archived manifests
```

**Typical use cases:**

```bash
# After mounting a drive you haven't used in a while — check nothing is corrupt
usb-backup-verify

# Full checksum verification (slower, catches bit rot)
usb-backup-verify --rechecksum

# Compare what's on Drive A vs Drive B (for rotation verification)
usb-backup-verify --list
usb-backup-verify --compare \
  /etc/usb-backup/manifests/manifest-a1b2c3d4-20260220-140000.txt \
  /etc/usb-backup/manifests/manifest-e5f6a7b8-20260215-030001.txt
```

Comparison output shows:
- Files present on one drive but not the other
- Files with the same name but different checksums (changed/corrupted)
- Identical file count

#### `usb-backup-uninstall`

Cleanly remove all components.

```
usb-backup-uninstall          # Remove config, scripts, storage, fstab entry
usb-backup-uninstall --wipe   # + erase filesystem signatures on the drive
```

Removes: Proxmox storage registration, fstab entry, systemd units, all `usb-backup-*` scripts, `/etc/usb-backup/` directory.

Preserves: log file, fstab backups, drive data (unless `--wipe`), the uninstall script itself (with instructions to remove it).

## Drive Rotation

The tool is designed for offsite backup rotation with multiple USB SSDs.

### Setup

Run the setup script on each drive:

```bash
# Drive A
# (connect Drive A)
bash /root/setup-usb-backup.sh
usb-backup-unmount --eject

# Drive B
# (connect Drive B)
bash /root/setup-usb-backup.sh
usb-backup-unmount --eject
```

Both drives get label `pve-backup`. The fstab entry uses `LABEL=` so either drive mounts to the same path without any configuration change.

### Rotation Workflow

```bash
# Week 1: Drive A onsite, Drive B offsite
usb-backup-mount
# ... backups run all week ...
usb-backup-unmount --eject

# Swap drives physically

# Week 2: Drive B onsite, Drive A goes offsite
usb-backup-mount
# ... backups run ...

# Verify both drives have the expected content
usb-backup-verify --compare \
  /etc/usb-backup/manifests/manifest-AAAA-latest.txt \
  /etc/usb-backup/manifests/manifest-BBBB-latest.txt
```

### How Manifests Enable Comparison

Every `usb-backup-unmount` generates a manifest containing:

```
# USB Backup Manifest
# Generated: 2026-02-23T14:30:00+01:00
# Drive UUID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
# Drive Label: pve-backup
# Hostname: pve-node1
#
# Format: sha256  size_bytes  mod_date  filename
# ───────────────────────────────────────────────────────────
3a7f2c...  4831838208  2026-02-23 03:00:15  vzdump-qemu-100-2026_02_23-03_00_01.vma.zst
b8e41d...  2415919104  2026-02-23 03:45:22  vzdump-qemu-101-2026_02_23-03_45_10.vma.zst
# ───────────────────────────────────────────────────────────
# Total files: 2
# Disk usage: 6.8G
```

For files over 10 GB, a partial checksum (first 100 MB + last 100 MB) is used for speed. Manifests are stored both on the drive and archived locally in `/etc/usb-backup/manifests/` — so you can compare even when the other drive is offsite.

## SMART Health Monitoring

SMART checks run automatically on every `usb-backup-mount` and report:

| Check | Source | Warning threshold |
|---|---|---|
| Overall health | SMART self-assessment | Anything other than PASSED |
| Temperature | Attr 194 / NVMe field | > 65°C (configurable) |
| Reallocated sectors | Attr 5 | > 10 (configurable) |
| Pending sectors | Attr 197 | > 0 |
| Wear level | Attr 173/177/231 / NVMe % | > 80% used (configurable) |
| CRC errors | Attr 199 | > 0 (cable/connection issue) |
| Unexpected power loss | Attr 174 / NVMe | Informational |
| Power-on hours | Attr 9 | Informational |
| Power cycles | Attr 12 | Informational |

The script tries multiple SMART transport modes (`-d sat`, `-d auto`, direct) because USB-SATA bridges frequently block SMART passthrough. If your enclosure doesn't support it, the check is skipped with a warning — it doesn't block mounting.

Full SMART output is saved to `/etc/usb-backup/last-smart-sdX.txt` for reference.

## Installed Files

```
/etc/usb-backup/
├── config                    # Shared configuration (mountpoint, thresholds, etc.)
├── functions                 # Shared bash library (logging, SMART, manifests)
├── manifests/                # Archived manifest copies
│   ├── manifest-a1b2c3d4-20260223-140000.txt
│   └── manifest-e5f6a7b8-20260220-030001.txt
└── last-smart-sdb.txt        # Most recent full SMART output

/usr/local/sbin/
├── usb-backup-mount
├── usb-backup-unmount
├── usb-backup-status
├── usb-backup-verify
└── usb-backup-uninstall

/mnt/usb-backup/              # Mount point
├── dump/                     # Proxmox vzdump backup directory
├── .backup-disk-info         # Drive identity marker
└── .backup-manifest-latest.txt  # Current manifest

/var/log/usb-backup.log       # Operation log
/etc/fstab                    # One LABEL-based entry added
```

## Configuration

All settings are in `/etc/usb-backup/config`:

```bash
MOUNTPOINT="/mnt/usb-backup"
STORAGE_ID="usb-backup"
LABEL="pve-backup"

# SMART thresholds
SMART_WEAR_WARN=80        # Warn at this % wear
SMART_REALLOC_WARN=10     # Warn above this many reallocated sectors
SMART_TEMP_WARN=65        # Warn above this temperature (°C)

# Disk space thresholds
DISK_WARN_PCT=80          # Warning at this % full
DISK_CRIT_PCT=90          # Critical at this % full
```

Edit this file to adjust thresholds. All companion scripts source it on every run.

## Safety Design

**Boot safety:** The fstab entry uses `noauto,nofail,x-systemd.device-timeout=5` — the system never enters emergency mode regardless of whether the USB drive is connected.

**Unmount safety:** Before unmounting, checks for running `vzdump` processes and open files on the mountpoint. Blocks if a backup is in progress. Falls back to lazy unmount if a normal unmount fails.

**Automount control:** After unmounting, explicitly stops the systemd automount unit to prevent accidental re-mount from background processes accessing the mountpoint.

**Idempotent operations:** Running setup twice doesn't duplicate fstab entries. Re-registering a drive with existing data is a supported workflow.

**Destructive safeguards:** Format requires typing `YES` then `DESTROY`. Size range checks prevent accidentally formatting the wrong device. Boot disk is auto-excluded from device selection.

**Manifest integrity:** Checksums are generated before unmount and archived locally, so you always have a reference even if the drive is offsite or damaged.

## FAQ

**Q: What happens if I reboot with the USB drive disconnected?**
The system boots normally. The `nofail` and `noauto` fstab options prevent any hang or emergency mode. The Proxmox storage shows as inactive until you plug the drive back in and run `usb-backup-mount`.

**Q: Can I use this with a spinning HDD instead of an SSD?**
Yes. The script works with any block device. SMART checks will report HDD-specific attributes. Adjust `MIN_SIZE_BYTES`/`MAX_SIZE_BYTES` at the top of the setup script if needed.

**Q: My USB enclosure doesn't support SMART passthrough.**
The mount command will log a warning and continue without blocking. Use `usb-backup-mount --no-smart` to suppress the warning.

**Q: How do I move a backup drive to a different Proxmox node?**
Connect the drive and run the setup script. It will detect the existing `pve-backup` filesystem and offer to re-register without formatting.

**Q: How big can the manifest checksums get?**
Files over 10 GB use partial checksums (first + last 100 MB) for speed. A drive with 50 backup files generates a manifest in under a minute.

**Q: Can I change the storage name or mount point?**
Edit the variables at the top of `setup-usb-backup.sh` before running, or edit `/etc/usb-backup/config` after installation (you'll also need to update the fstab entry and Proxmox storage manually).

## Troubleshooting

```bash
# Check what's happening
usb-backup-status --smart --log=50

# Drive connected but not detected?
lsblk -f | grep pve-backup
blkid -L pve-backup

# Storage shows ? in Proxmox UI?
usb-backup-mount

# Can't unmount — "device busy"?
lsof +D /mnt/usb-backup
# Then either close the files or:
usb-backup-unmount --force

# Full log history
cat /var/log/usb-backup.log

# Nuclear option — remove everything and start fresh
usb-backup-uninstall --wipe
```

## License

MIT

## Contributing

Issues and pull requests welcome. Please test on a non-production Proxmox node first.
