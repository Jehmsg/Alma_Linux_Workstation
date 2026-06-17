# Rocky / Alma Linux VFX Workstation Setup

An Ansible playbook for bootstrapping Linux workstations for VFX work. Uses `ansible-pull` so each machine configures itself directly from this repository — no central Ansible controller required.

---

## GPU / Installer Note (RTX 5080/5090)

On machines with RTX 5080 or 5090 GPUs, the Rocky Linux installer may hang or fail to display with the default nouveau driver. To work around this:

1. At the GRUB boot menu, press **`e`** to edit the boot entry.
2. Find the line beginning with `linux` and append the following to the end:

```
nomodeset rd.blacklist=nouveau
```

3. Press **`Ctrl+X`** to boot with these options.

> This is only required during installation. It is **not** needed for RTX 40-series cards or older.

---

## Overview

This playbook automates the setup of a GNOME/KDE workstation tailored for VFX pipelines. It handles:

- **Kernel & drivers** — mainline kernel via ELRepo, NVIDIA open drivers (580 series)
- **Desktop environment** — KDE Plasma Workspaces
- **VFX dependencies** — all shared libraries required by Houdini and similar DCC tools
- **Pipeline tooling** — [Rez](https://github.com/AcademySoftwareFoundation/rez) package manager
- **System management** — Cockpit web UI, Active Directory/Samba integration, NFS/CIFS mounts
- **System configuration** — environment variables, default shell profiles, umask

---

## Requirements

- Rocky Linux 8 / Alma Linux 8 (or 9 with minor adjustments)
- Internet access from the target machine
- `ansible` and `git` installed on the target machine

Install Ansible on a fresh machine:

```bash
dnf install -y ansible git
```

---

## Usage

Run the following on the target machine as root (or with `sudo`):

```bash
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git
```

This will:
1. Clone the repository to a temporary local directory
2. Run `local.yml` against `localhost`

### Run with verbose output

```bash
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git -v
```

---

## What Gets Installed

### Repositories & Kernel
- CRB (CodeReady Builder / PowerTools) repository enabled
- EPEL repository enabled
- ELRepo — provides the mainline (`kernel-ml`) kernel
- NVIDIA CUDA repository

### Kernel & Drivers
- `kernel-ml` + `kernel-ml-devel` (mainline kernel from ELRepo)
- NVIDIA open drivers (`nvidia-open`, module stream `580-open`)

### Desktop
- KDE Plasma Workspaces (`@KDE Plasma Workspaces`, `@base-x`)

### General Packages
`bash`, `curl`, `dbus`, `perl`, `git`, `less`, `zsh`, `python3.11`, `htop`, `btop`, `firefox`, `ark`, `flatpak`, `gdb`, `cifs-utils`, `nfs-utils`, `redhat-lsb-core`, `rasdaemon`

### Houdini / DCC Dependencies
All shared libraries required by Houdini (and most other commercial DCC tools), including: `alsa-lib`, `cups-libs`, `libXrandr`, `libxkbcommon`, `mesa-libGLU`, `nss`, `xcb-util-*`, and more.

### System Management (Cockpit)
`cockpit`, `cockpit-storaged`, `cockpit-shell`, plus AD/domain-join tooling: `sssd`, `realmd`, `adcli`, `oddjob`, `oddjob-mkhomedir`, `krb5-workstation`, `samba-common-tools`

Cockpit is enabled and started automatically at boot.

### Pipeline — Rez
[Rez](https://github.com/AcademySoftwareFoundation/rez) (v3.4.0) is downloaded, installed to `/opt/rez`, and the `rez-pip` plugin is installed. A shared package cache directory is created at `/opt/rez/package_cache` with sticky group permissions (`2777`).

### System Services
| Service | State |
|---|---|
| `cockpit.socket` | Enabled, started |
| `rasdaemon.service` | Enabled, started |
| `mcelog.service` | Disabled, stopped |

### Dotfiles / System Config
| File | Destination | Purpose |
|---|---|---|
| `Files/profile` | `/etc/profile` | System-wide environment variables |
| `Files/bash_profile` | `/etc/skel/.bash_profile` | Default bash profile for new users |
| `Files/umask.sh` | `/etc/profile.d/umask.sh` | System-wide umask setting |

---

## Repository Structure

```
.
├── local.yml          # Main Ansible playbook
├── Files/             # Config files deployed to the system
│   ├── profile        # /etc/profile
│   ├── bash_profile   # /etc/skel/.bash_profile
│   └── umask.sh       # /etc/profile.d/umask.sh
└── README.md
```

---

## Customisation

### Rez version
Change the `rez_version` variable near the top of `local.yml`:

```yaml
vars:
  rez_version: "3.4.0"
```

### Rez group
The playbook includes a commented-out `group` setting for the package cache directory. Uncomment and set `rez_group` to lock down cache access to a specific AD/local group:

```yaml
  rez_group: Domain_Artists
```

---

## Notes

- The playbook targets `localhost` and runs with `become: true` (root). Ensure the user running `ansible-pull` has sudo privileges or run as root.
- NVIDIA driver installation uses the `rhel8` CUDA repo. If running Alma/Rocky 9, update the `baseurl` and `gpgkey` in the `Nvidia Repo` task accordingly.
- The ELRepo package URL is pinned to `elrepo-release-8`; update this for RHEL 9-based systems.
