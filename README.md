# Rocky / Alma Linux VFX Workstation Setup

An Ansible playbook collection for bootstrapping Linux workstations for VFX work. Uses `ansible-pull` so each machine configures itself directly from this repository — no central Ansible controller required.

---

## GPU / Installer Note (RTX 5080/5090)

On machines with RTX 5080 or 5090 GPUs, the Rocky Linux installer may hang or fail to display with the default nouveau driver. To work around this:

1. At the GRUB boot menu, press **`e`** to edit the boot entry.
2. Find the line beginning with `linux` and append the following to the end:

```
nomodeset rd.driver.blacklist=nouveau
```

3. Press **`Ctrl+X`** to boot with these options.

> This is only required during installation. It is **not** needed for RTX 40-series cards or older.

---

## Overview

This collection automates the setup of a KDE workstation tailored for VFX pipelines. It is split into modular playbooks that can be run individually or all together:

- **Dotfiles** — shell profiles, environment variables, umask, Rez environment
- **Repositories & kernel** — CRB, EPEL, ELRepo, NVIDIA CUDA repo, kernel-ml
- **NVIDIA drivers** — 580 open drivers
- **Desktop** — KDE Plasma Workspaces, graphical target
- **Packages** — general deps, Houdini/DCC dependencies, Cockpit, AD tools, services
- **Rez** — Rez package manager install & config
- **Security** — SELinux permissive mode
- **Tuning** — tuned profile, sysctl tuning, system limits
- **NFS** — ASLON NFS share mount
- **Time sync** — chrony NTP, timezone, locale configuration

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

### Full Bootstrap (All Playbooks)

Run the following on the target machine as root (or with `sudo`):

```bash
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git site.yml
```

This will clone the repository and run all playbooks in order:

| # | Playbook | Purpose |
|---|----------|---------|
| 1 | `01-dotfiles.yml` | Shell profiles, environment variables, umask |
| 2 | `02-repos-kernel.yml` | Repositories (CRB, EPEL, ELRepo, NVIDIA), kernel-ml |
| 3 | `03-nvidia.yml` | NVIDIA 580 open drivers |
| 4 | `04-desktop.yml` | KDE Plasma Workspaces + graphical target |
| 5 | `05-packages.yml` | General packages, Houdini deps, Cockpit, services |
| 6 | `06-rez.yml` | Rez package manager install & config |
| 7 | `07-security.yml` | SELinux, system limits |
| 8 | `08-tuning.yml` | Tuned performance profile, sysctl tuning |
| 9 | `09-nfs.yml` | NFS mounts |
| 10 | `11-ntp.yml` | Chrony NTP, timezone, locale |

### Run a Single Playbook

Re-run any individual stage without re-executing the full bootstrap:

```bash
# Re-apply only packages and services
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 05-packages.yml

# Re-apply only Rez
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 06-rez.yml

# Re-apply only NVIDIA drivers
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 03-nvidia.yml
```

### Verbose Output

Append `-vvv` to any command for detailed output.

---

## What Gets Installed

### Dotfiles (`01-dotfiles.yml`)
| File | Destination | Purpose |
|---|---|---|
| `Files/profile` | `/etc/profile` | System-wide environment variables |
| `Files/bash_profile` | `/etc/skel/.bash_profile` | Default bash profile for new users |
| `Files/umask.sh` | `/etc/profile.d/umask.sh` | System-wide umask setting |
| `Files/rez.sh` | `/etc/profile.d/rez.sh` | Rez environment setup |

### Repositories & Kernel (`02-repos-kernel.yml`)
- CRB (CodeReady Builder / PowerTools) repository enabled
- EPEL repository enabled
- ELRepo — provides the mainline (`kernel-ml`) kernel
- NVIDIA CUDA repository
- `kernel-ml` + `kernel-ml-devel` (mainline kernel from ELRepo)

### NVIDIA Drivers (`03-nvidia.yml`)
- NVIDIA 580 open drivers (`@nvidia-driver:580-open`, `nvidia-open`)

### Desktop (`04-desktop.yml`)
- KDE Plasma Workspaces (`@KDE Plasma Workspaces`, `@base-x`)
- Graphical target set as default

### General Packages (`05-packages.yml`)
`bash`, `curl`, `dbus`, `perl`, `git`, `less`, `zsh`, `python3.11`, `htop`, `btop`, `firefox`, `ark`, `flatpak`, `gdb`, `cifs-utils`, `nfs-utils`, `rasdaemon`, `liberation-fonts`, `google-noto-fonts-common`, `google-noto-sans-fonts`, `dejavu-fonts-all`

### Houdini / DCC Dependencies (`05-packages.yml`)
All shared libraries required by Houdini (and most other commercial DCC tools), including: `alsa-lib`, `cups-libs`, `libXrandr`, `libxkbcommon`, `mesa-libGLU`, `nss`, `xcb-util-*`, and more.

### System Management (`05-packages.yml`)
`cockpit`, `cockpit-storaged`, `cockpit-shell`, plus AD/domain-join tooling: `sssd`, `realmd`, `adcli`, `oddjob`, `oddjob-mkhomedir`, `krb5-workstation`, `samba-common-tools`

Cockpit is enabled and started automatically at boot.

### Pipeline — Rez (`06-rez.yml`)
[Rez](https://github.com/AcademySoftwareFoundation/rez) (v3.4.0) is downloaded, installed to `/opt/rez`, and the `rez-pip` plugin is installed. A shared package cache directory is created at `/opt/rez/package_cache` with sticky group permissions (`2777`).

### Security (`07-security.yml`)
- SELinux set to permissive mode
- SELinux Python bindings installed

### Tuning (`08-tuning.yml`)
- `tuned` installed and running with `balanced` profile
- `vm.max_map_count` set to 1048576 for VFX applications
- NFS buffer sizes (`rmem_max`, `wmem_max`) increased for performance
- `netdev_max_backlog` tuned for high-throughput networking
- Kernel core dump pattern set
- System limits configured for VFX applications (`nofile 65536`, `nproc 65536`, `stack unlimited`)

### NFS (`09-nfs.yml`)
- ASLON NFS share mounted at `/mnt/aslon`
- NFS options: `sec=sys,vers=4,rsize=1048576,wsize=1048576,_netdev,nofail,auto`


### Time Synchronization (`11-ntp.yml`)
- Chrony NTP daemon installed and running
- System timezone configured (default: `Australia/Sydney`)
- System locale configured (default: `en_AU.UTF-8`)
- `/etc/locale.conf` deployed

---

## Repository Structure

```
.
├── site.yml                  # Orchestrator — runs all playbooks in order
├── 01-dotfiles.yml           # Shell profiles, environment variables, umask
├── 02-repos-kernel.yml       # Repositories (CRB, EPEL, ELRepo, NVIDIA), kernel-ml
├── 03-nvidia.yml             # NVIDIA 580 open drivers
├── 04-desktop.yml            # KDE Plasma Workspaces + graphical target
├── 05-packages.yml           # General packages, Houdini deps, Cockpit, services
├── 06-rez.yml                # Rez package manager install & config
├── 07-security.yml           # SELinux, system limits
├── 08-tuning.yml             # Tuned performance profile, sysctl tuning
├── 09-nfs.yml                # NFS mounts
├── 11-ntp.yml                # Chrony NTP, timezone, locale
├── Files/                    # Config files deployed to the system
│   ├── profile               # /etc/profile
│   ├── bash_profile          # /etc/skel/.bash_profile
│   ├── umask.sh              # /etc/profile.d/umask.sh
│   ├── rez.sh                # /etc/profile.d/rez.sh
│   └── resolv.conf           # DNS resolver config
├── local.yml                 # Legacy monolithic playbook (deprecated)
└── README.md
```

---

## Customisation

### Rez version
Change the `rez_version` variable in `06-rez.yml`:

```yaml
vars:
  rez_version: "3.4.0"
```

### Rez group
The playbook includes a `group` setting for the package cache directory. Edit in `06-rez.yml`:

```yaml
rez_group: artists
```

### NFS Mount
Edit `09-nfs.yml` to change the NFS server, share path, or mount options.

### Timezone
Edit `11-ntp.yml` to change the timezone:

```yaml
ntp_timezone: "Australia/Sydney"
```

### NTP Servers
Edit `11-ntp.yml` to change the NTP servers:

```yaml
ntp_servers:
  - 0.au.pool.ntp.org
  - 1.au.pool.ntp.org
```

### Locale
Edit `11-ntp.yml` to change the locale:

```yaml
ntp_locale: "en_AU.UTF-8"
```

---

## Notes

- All playbooks target `localhost` and run with `become: true` (root). Ensure the user running `ansible-pull` has sudo privileges or run as root.
- The legacy `local.yml` monolithic playbook is retained for reference but is deprecated in favour of the modular playbooks.
- The ELRepo package URL is pinned to `elrepo-release-8`; update for RHEL 9-based systems.
- The NVIDIA CUDA repo URL references `rhel{{ ansible_distribution_major_version }}` and adapts automatically.
