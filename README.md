# Proxmox USB Storage Manager

A single-script installer that turns a removable USB SSD into safe, rotation-ready Proxmox VE storage — with selectable content types, SMART health monitoring, sha256 manifest tracking, and clean mount/unmount lifecycle management.

## The Problem

Using a USB drive for Proxmox storage sounds simple until you hit the sharp edges:

- **Emergency mode on reboot** — a missing USB drive with a standard fstab entry drops Proxmox into emergency mode, requiring console access to recover.
- **Device path roulette** — `/dev/sdb` today might be `/dev/sdc` tomorrow.
- **No safe disconnect workflow** — vzdump writes fail mid-backup, or you corrupt the filesystem by unplugging at the wrong time.
- **Silent drive failure** — SSDs degrade without warning. By the time you discover it, your backups are already corrupted.
- **Drive rotation is painful** — swapping onsite/offsite drives means editing fstab and re-registering storage every time.
- **One-trick storage** — most guides configure USB drives for backups only, but you often need ISOs, templates, or disk images on portable media too.

This tool solves all of that with one setup script and five management commands.

## Features

| Feature | What it does |
|---|---|
| **Selectable content types** | Choose per-drive: backup, ISO, templates, disk images, containers, snippets, import — with presets for common scenarios |
| **Auto-detect USB devices** | Scans for removable storage, excludes boot disk, handles manual selection |
| **Safe fstab configuration** | `nofail` + `noauto` + `x-systemd.automount` + `errors=remount-ro` — boots clean if drive absent, goes read-only on fs error |
| **LABEL-based mounting** | Fstab uses `LABEL=pve-backup` — any drive with the right label works, enabling seamless rotation |
| **Multi-label collision guard** | Refuses to operate if two drives with the same label are connected simultaneously |
| **Pre-format detection** | Detects existing setup, offers to re-register without formatting (change content types while preserving data) |
| **SMART health monitoring** | Checks SSD health on every mount — temperature, wear, reallocated sectors, CRC errors |
| **Backup manifests** | sha256 checksums of all static content (backups, ISOs, templates, snippets) on unmount |
| **Read-only verification** | `--check` verifies checksums against manifest without rewriting it — can't accidentally "bless" corrupted data |
| **Vzdump lock protection** | Refuses to unmount while a backup is running |
| **Safe eject** | Syncs, unmounts, stops automount, powers off USB device |
| **Comprehensive logging** | All operations timestamped in `/var/log/usb-backup.log` |
| **Clean uninstall** | Removes everything with fstab backup |

## Content Types

During setup, you choose what each USB drive will store. Different drives can serve different purposes.

| Content type | Proxmox UI name | Directory | Description |
|---|---|---|---|
| `backup` | VZDump backup file | `dump/` | VM and container backups |
| `iso` | ISO image | `template/iso/` | Installation media |
| `vztmpl` | Container template | `template/cache/` | LXC container templates |
| `images` | Disk image | `images/` | VM disk images (qcow2, raw, vmdk) |
| `rootdir` | Container | `images/` | Container root directories |
| `snippets` | Snippets | `snippets/` | Hook scripts, cloud-init configs |
| `import` | Import | `import/` | Import staging area |

### Setup Presets

```
[1] Backup only                               → backup
[2] Backup + ISO storage                      → backup, iso
[3] Backup + ISO + Templates                  → backup, iso, vztmpl
[4] Portable node storage (backup/restore/ISO) → backup, iso, vztmpl, images, import
[5] Everything (all content types)             → all
[6] Custom (pick individually)                 → interactive toggle
```

### Changing Content Types on Existing Drives

Re-run the setup script with the drive connected. It detects the existing `pve-backup` filesystem and offers to re-register with new content types without formatting — preserving all data.

## Requirements

- Proxmox VE 7.x or 8.x
- Root access
- USB SSD/HDD (ext4 will be created by the script)
- `smartmontools` (prompted during setup if missing)

## Quick Start

```bash
# Download
wget -O /root/setup-usb-backup.sh https://raw.githubusercontent.com/YOURUSER/proxmox-usb-backup/main/setup-usb-backup.sh

# Connect your USB SSD and run
bash /root/setup-usb-backup.sh
```

The script will:

1. Scan for removable USB devices (with boot disk and non-USB safety checks)
2. Let you select content types (presets or custom)
3. Check if the drive is already set up (skip format if so)
4. Format as ext4 with GPT partition table (with double confirmation: `YES` + `DESTROY`)
5. Configure fstab with safe mount options (`errors=remount-ro`)
6. Create Proxmox-expected directory structure for selected content types
7. Register as Proxmox storage with `is_mountpoint=1`
8. Install five management commands
9. Run an initial SMART health check

## Usage

### Daily Workflow

```bash
# 1. Connect USB SSD

# 2. Mount and enable (SMART check runs automatically)
usb-backup-mount

# 3. Use in Proxmox UI — storage = usb-backup
#    • Backup VMs/CTs        (if backup enabled)
#    • Upload/use ISOs        (if iso enabled)
#    • Download templates     (if vztmpl enabled)
#    • Store VM disk images   (if images enabled)

# 4. When done — generates manifest, disables storage, unmounts, powers off
usb-backup-unmount --eject

# 5. Physically disconnect the drive
```

### Commands Reference

#### `usb-backup-mount`

```
usb-backup-mount              # Mount with SMART health check
usb-backup-mount --no-smart   # Mount without SMART check (faster)
```

Resolves device by label (with multi-label collision guard), runs SMART check, mounts, verifies the filesystem isn't in read-only error state, ensures content directories exist, enables Proxmox storage, shows per-content-type file counts.

#### `usb-backup-unmount`

```
usb-backup-unmount                # Manifest + disable + unmount
usb-backup-unmount --eject        # + power off for safe physical removal
usb-backup-unmount --no-manifest  # Skip manifest generation (faster)
usb-backup-unmount --force        # Skip vzdump/open-file safety checks
```

Checks for running vzdump and open files (using `fuser` for speed on large directories), generates manifest, disables storage, syncs and unmounts (lazy unmount fallback), stops automount unit. With `--eject`: powers off USB device.

#### `usb-backup-status`

```
usb-backup-status              # Full overview with per-content breakdown
usb-backup-status --smart      # + live SMART health check
usb-backup-status --log        # + last 20 log entries
usb-backup-status --log=50     # + last 50 log entries
```

Detects and warns if the filesystem is mounted read-only (triggered by `errors=remount-ro` on disk error).

#### `usb-backup-verify`

```
usb-backup-verify                          # Size + existence check (fast)
usb-backup-verify --check                  # Full checksum verify — READ-ONLY
usb-backup-verify --regenerate             # Rewrite manifest from current files
usb-backup-verify --compare <file>         # Drive manifest vs saved manifest
usb-backup-verify --compare <a> <b>        # Two manifest files
usb-backup-verify --list                   # List archived manifests
```

**Important design:** `--check` and `--regenerate` are deliberately separate operations:

- `--check` verifies every file's checksum against the existing manifest **without rewriting it**. If corruption has occurred, `--check` will detect and report it — it cannot accidentally "bless" corrupted data by overwriting the manifest.
- `--regenerate` explicitly rewrites the manifest from the current state of the drive. Use this after adding or removing files, or after running backups.

The unmount workflow calls `--regenerate` automatically. Use `--check` any time you want to verify data integrity.

Manifests cover static content types: `backup`, `iso`, `vztmpl`, `snippets`. VM disk images (`images/`) and import staging (`import/`) are excluded because they contain live or transient data.

#### `usb-backup-uninstall`

```
usb-backup-uninstall          # Remove config, scripts, storage, fstab entry
usb-backup-uninstall --wipe   # + erase filesystem signatures on the drive
```

## Drive Rotation

### Setup

```bash
# Drive A — backup + ISO (preset 2)
bash /root/setup-usb-backup.sh
usb-backup-unmount --eject

# Drive B — backup only (preset 1)
bash /root/setup-usb-backup.sh
usb-backup-unmount --eject
```

Both drives get label `pve-backup`. The fstab entry uses `LABEL=` so either drive mounts to the same path.

> **Safety:** If both drives are connected simultaneously, all operations will refuse to proceed and report the collision. Unplug one drive first.

### Comparing Rotation Drives

```bash
usb-backup-verify --list

usb-backup-verify --compare \
  /etc/usb-backup/manifests/manifest-a1b2c3d4-20260220-140000.txt \
  /etc/usb-backup/manifests/manifest-e5f6a7b8-20260215-030001.txt
```

### Manifest Format

```
# USB Storage Manifest
# Generated: 2026-02-23T14:30:00+01:00
# Drive UUID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
# Content types: backup,iso,vztmpl
#
# Format: sha256  size_bytes  mod_date  content_type/filename
3a7f2c...  4831838208  2026-02-23 03:00  backup/vzdump-qemu-100-2026_02_23-03_00_01.vma.zst
b8e41d...  1048576000  2026-02-20 15:30  iso/debian-12.5.0-amd64-netinst.iso
```

Files over 10 GB use partial checksums (first + last 100 MB) for speed, clearly marked as `partial:` in the manifest.

## SMART Health Monitoring

SMART checks run automatically on every `usb-backup-mount`:

| Check | Source | Warning threshold |
|---|---|---|
| Overall health | SMART self-assessment | Not PASSED |
| Temperature | Attr 194 / NVMe | > 65°C |
| Reallocated sectors | Attr 5 | > 10 |
| Pending sectors | Attr 197 | > 0 |
| Wear level | Attr 173/177/231 / NVMe % | > 80% used |
| CRC errors | Attr 199 | > 0 (cable issue) |
| Unexpected power loss | Attr 174 / NVMe | Informational |

Tries multiple transport modes (`-d sat`, `-d auto`, direct) for USB-SATA bridge compatibility. If passthrough is blocked, the check is skipped without blocking mount.

**SATA wear values:** The normalized VALUE starts at 100 (new drive) and decreases toward 0 (end of life). The script computes `wear_used = 100 - VALUE` and warns when it exceeds the threshold.

## Safety Design

**Boot safety:** `noauto,nofail,x-systemd.device-timeout=5` — never enters emergency mode.

**Filesystem error handling:** `errors=remount-ro` causes the kernel to switch the filesystem to read-only on I/O errors, preventing further damage. The mount and status scripts detect this state and alert you. This is safer than `errors=continue` which would silently write into a corrupt filesystem.

**Device resolution:** Uses `lsblk -ndo PKNAME` to resolve partitions to parent disks — safe for all naming schemes including NVMe (`/dev/nvme0n1`), eMMC (`/dev/mmcblk0`), and SCSI/SATA (`/dev/sdb`). No fragile regex stripping.

**Non-USB protection:** If the selected device is not on the USB transport, requires explicit confirmation. If it matches the boot disk, refuses outright.

**Multi-label collision guard:** If two drives with label `pve-backup` are connected simultaneously, all operations abort with an error. Prevents operating on the wrong rotation drive.

**Unmount safety:** Checks for running vzdump and open files (via `fuser -vm` for speed). Falls back to lazy unmount if normal unmount fails.

**Manifest integrity:** `--check` verifies against the manifest read-only and never overwrites it. This means verification cannot accidentally normalize corruption. `--regenerate` is a separate explicit action.

**Idempotent:** Running setup twice doesn't duplicate fstab entries. Re-registering preserves data.

**Destructive safeguards:** Format requires `YES` + `DESTROY`. Size range checks. Boot disk excluded.

## Configuration

All settings are in `/etc/usb-backup/config`:

```bash
MOUNTPOINT="/mnt/usb-backup"
STORAGE_ID="usb-backup"
LABEL="pve-backup"
CONTENT_TYPES="backup,iso,vztmpl"    # Selected during setup

SMART_WEAR_WARN=80
SMART_REALLOC_WARN=10
SMART_TEMP_WARN=65

DISK_WARN_PCT=80
DISK_CRIT_PCT=90
```

## Installed Files

```
/etc/usb-backup/
├── config                    # Shared configuration
├── functions                 # Shared bash library
├── manifests/                # Archived manifest copies
└── last-smart-sdb.txt        # Last SMART output

/usr/local/sbin/
├── usb-backup-mount
├── usb-backup-unmount
├── usb-backup-status
├── usb-backup-verify
└── usb-backup-uninstall

/mnt/usb-backup/              # Mount point (directories per content type)
├── dump/                     # backup
├── template/
│   ├── iso/                  # iso
│   └── cache/                # vztmpl
├── images/                   # images, rootdir
├── snippets/                 # snippets
├── import/                   # import
├── .backup-disk-info         # Drive identity + content types
└── .backup-manifest-latest.txt
```

## FAQ

**Q: What happens if I reboot with the USB drive disconnected?**
System boots normally. Storage shows inactive until you run `usb-backup-mount`.

**Q: What happens if the filesystem has errors?**
With `errors=remount-ro`, the kernel switches to read-only mode. The mount script detects this and refuses to proceed. Run `fsck.ext4` to repair.

**Q: Can I change content types on an existing drive?**
Yes. Re-run setup, select "Re-register only", choose new content types. Data preserved, directories created.

**Q: What if both rotation drives are plugged in at once?**
All operations will abort with a "COLLISION" error. Unplug one drive and retry.

**Q: Which content types are included in manifests?**
Static content: `backup`, `iso`, `vztmpl`, `snippets`. VM disk images and import staging are excluded (live/transient data).

**Q: My USB enclosure doesn't support SMART.**
Mount proceeds with a warning. Use `--no-smart` to suppress.

**Q: Can I use NVMe or non-USB drives?**
Yes. The device normalization is safe for all naming schemes. Non-USB transport requires explicit confirmation during setup.

## Troubleshooting

```bash
# Full status
usb-backup-status --smart --log=50

# Drive connected but not detected?
blkid -L pve-backup

# Filesystem went read-only?
dmesg | tail -30
fsck.ext4 -n /dev/sdb1

# Storage shows ? in Proxmox UI?
usb-backup-mount

# Can't unmount?
fuser -vm /mnt/usb-backup
usb-backup-unmount --force

# Verify backup integrity (without rewriting manifest)
usb-backup-verify --check

# Start fresh
usb-backup-uninstall --wipe
```

## License

MIT

## Contributing

Issues and pull requests welcome. Test on a non-production Proxmox node first.
