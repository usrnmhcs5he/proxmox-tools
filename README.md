<div align="center">

# 🖥️ proxmox-tools

**Two interactive Bash scripts for Proxmox VE: post-install setup, and safe USB SSD storage with rotation support.**

![Bash](https://img.shields.io/badge/bash-4%2B-4EAA25?logo=gnubash&logoColor=white)
![Proxmox VE](https://img.shields.io/badge/Proxmox%20VE-8.x-E57000?logo=proxmox&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue)

</div>

| Script | Purpose |
|--------|---------|
| [`proxmox_starter`](proxmox_starter) | Post-install configuration: VLAN trunk, repositories, syslog, Tailscale |
| [`proxmox_usb_backup`](proxmox_usb_backup) | USB SSD as Proxmox storage: content types, SMART, manifests, safe eject |

Both run as root on the Proxmox host and need no config files.

---

## 🚀 proxmox_starter

Post-install configuration for Proxmox VE 8.3+.

### ✨ Features

- **VLAN trunk**: detects interfaces, sets `bridge-vlan-aware` on the selected bridge. `/etc/network/interfaces` is backed up first.
- **Repositories**: disables the enterprise PVE and Ceph repos, enables no-subscription, runs `apt update`.
- **Syslog**: installs rsyslog if missing and forwards to Graylog or any syslog collector.
- **Tailscale**: installs via the official script and authenticates.
- **Run All**, **Restore** (network config from backup) and **Status** overview.

### 🚀 Quick start

```bash
wget https://raw.githubusercontent.com/usrnmhcs5he/proxmox-tools/main/proxmox_starter
bash proxmox_starter
```

### 📁 Backups

| What | Location |
|------|----------|
| Network config | `/etc/network/backups/interfaces.backup.<timestamp>` |
| Repo files | `*.bak.<timestamp>` next to the originals |
| Rsyslog config | `/etc/rsyslog.d/10-graylog.conf.bak.<timestamp>` |

---

## 💾 proxmox_usb_backup

Turns a removable USB SSD into rotation-ready Proxmox storage and installs five management commands.

### ✨ Features

- **Selectable content types**: backup, ISO, templates, disk images, containers, snippets, import, with presets.
- **Boot-safe fstab**: `LABEL=` + `nofail,noauto,x-systemd.automount,errors=remount-ro`. A missing drive never drops the host into emergency mode, and a failing filesystem goes read-only.
- **Drive rotation**: any drive labelled `pve-backup` mounts to the same path. If two are connected at once, every command aborts with a collision error.
- **SMART checks** on every mount: health, temperature, wear, reallocated/pending sectors, CRC errors.
- **sha256 manifests** of static content, written on unmount.
- **Read-only verify**: `--check` never rewrites the manifest, so it cannot bless corrupted data.
- **Safe unmount and eject**: refuses while vzdump runs or files are open; `--eject` powers the drive off.
- **Guarded formatting**: boot disk excluded, non-USB devices need confirmation, format needs `YES` + `DESTROY`.
- **Idempotent**: re-running setup on a configured drive offers re-register without formatting (change content types, keep data).
- **Logging** to `/var/log/usb-backup.log` and a clean uninstall that backs up fstab.

### 📦 Prerequisites

| Need | For |
|------|-----|
| Proxmox VE 7.x / 8.x, root | Everything |
| USB SSD/HDD | Formatted as ext4 (GPT) by the script |
| `smartmontools` | SMART checks (offered during setup) |

### 🚀 Quick start

```bash
wget https://raw.githubusercontent.com/usrnmhcs5he/proxmox-tools/main/proxmox_usb_backup
bash proxmox_usb_backup
```

Guided flow:

```
detect USB device → content types → existing setup check → format
   → fstab → directory layout → register storage → install commands → SMART check
```

### Content types

| Type | Directory | Holds |
|------|-----------|-------|
| `backup` | `dump/` | VM and container backups |
| `iso` | `template/iso/` | Installation media |
| `vztmpl` | `template/cache/` | LXC templates |
| `images` | `images/` | VM disk images |
| `rootdir` | `images/` | Container root directories |
| `snippets` | `snippets/` | Hook scripts, cloud-init configs |
| `import` | `import/` | Import staging |

Presets: **1** backup · **2** backup + ISO · **3** + templates · **4** portable node (backup, iso, vztmpl, images, import) · **5** everything · **6** custom.

### Commands

| Command | Options |
|---------|---------|
| `usb-backup-mount` | `--no-smart` skip the SMART check |
| `usb-backup-unmount` | `--eject` power off · `--no-manifest` skip manifest · `--force` skip vzdump/open-file checks |
| `usb-backup-status` | `--smart` live SMART · `--log[=N]` last N log lines (default 20) |
| `usb-backup-verify` | *(none)* size/existence · `--check` checksums, read-only · `--regenerate` rewrite manifest · `--compare <a> [<b>]` · `--list` |
| `usb-backup-uninstall` | `--wipe` also erase filesystem signatures |

Daily use:

```bash
usb-backup-mount              # connect drive, mount, enable storage "usb-backup"
usb-backup-unmount --eject    # manifest, disable, unmount, power off
```

### Drive rotation

Run setup once per drive (same label). Compare drives via their archived manifests:

```bash
usb-backup-verify --list
usb-backup-verify --compare <manifest-a> <manifest-b>
```

Manifests cover `backup`, `iso`, `vztmpl` and `snippets`. `images/` and `import/` hold live data and are excluded. Files over 10 GB use partial checksums (first + last 100 MB), marked `partial:`.

### SMART thresholds

| Check | Warns when |
|-------|-----------|
| Overall health | not PASSED |
| Temperature | > 65 °C |
| Reallocated sectors | > 10 |
| Pending sectors | > 0 |
| Wear level | > 80 % used |
| CRC errors | > 0 (cable) |

Tries `-d sat`, `-d auto`, then direct for USB bridge compatibility. If passthrough is blocked, the check is skipped and the mount continues.

### ⚙️ Configuration

`/etc/usb-backup/config`:

```bash
MOUNTPOINT="/mnt/usb-backup"
STORAGE_ID="usb-backup"
LABEL="pve-backup"
CONTENT_TYPES="backup,iso,vztmpl"

SMART_WEAR_WARN=80
SMART_REALLOC_WARN=10
SMART_TEMP_WARN=65
DISK_WARN_PCT=80
DISK_CRIT_PCT=90
```

### 📁 Installed files

```
/etc/usb-backup/          config, functions, manifests/, last-smart-*.txt
/usr/local/sbin/          usb-backup-{mount,unmount,status,verify,uninstall}
/var/log/usb-backup.log
/mnt/usb-backup/          dump/  template/{iso,cache}/  images/  snippets/  import/
                          .backup-disk-info  .backup-manifest-latest.txt
```

### 💡 Troubleshooting

```bash
blkid -L pve-backup                  # drive not detected?
dmesg | tail -30                     # filesystem went read-only?
fsck.ext4 -n /dev/sdX1               # ...then check it (unmounted)
fuser -vm /mnt/usb-backup            # can't unmount? then: usb-backup-unmount --force
usb-backup-status --smart --log=50   # full picture
```

---

## 🕘 Changelog

### proxmox_starter

| Version | Changes |
|---------|---------|
| v1 | VLAN trunk setup with backup/restore |
| **v2** | Enterprise repo disable, syslog/Graylog, Tailscale, unified menu, run-all |

### proxmox_usb_backup

| Version | Changes |
|---------|---------|
| v1–v3 | Auto-detect, `nofail`, companion scripts, SMART, manifests, LABEL rotation, logging, uninstall |
| **v4** | Selectable content types and presets; `lsblk`-based device resolution (NVMe/mmc safe); multi-label collision guard; `verify --check` (read-only) vs `--regenerate`; `errors=remount-ro`; `fuser` instead of `lsof`; USB-boot safety |

Full per-version history is kept in each script header.

## License

MIT
