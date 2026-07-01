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

- **Kernel & drivers** — mainline kernel via ELRepo, NVIDIA open drivers (580 series)
- **Desktop environment** — KDE Plasma Workspaces
- **VFX dependencies** — all shared libraries required by Houdini and similar DCC tools
- **Pipeline tooling** — [Rez](https://github.com/AcademySoftwareFoundation/rez) package manager
- **System management** — Cockpit web UI, Active Directory/Samba integration, NFS/CIFS mounts
- **System configuration** — environment variables, default shell profiles, umask, sysctl tuning

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
|---|---|---|
| 1 | `01-dotfiles.yml` | Shell profiles, environment variables, umask |
| 2 | `02-repos-kernel.yml` | Repositories (CRB, EPEL, ELRepo, NVIDIA), kernel-ml |
| 3 | `03-desktop.yml` | KDE Plasma Workspaces + graphical target |
| 4 | `04-packages.yml` | General packages, Houdini deps, Cockpit, services |
| 5 | `05-rez.yml` | Rez package manager install & config |
| 6 | `06-security.yml` | SELinux, system limits |
| 7 | `07-network.yml` | NFS mounts, sysctl tuning |
| 8 | `08-tuned.yml` | Tuned performance profile |

### Run a Single Playbook

Re-run any individual stage without re-executing the full bootstrap:

```bash
# Re-apply only packages and services
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 04-packages.yml

# Re-apply only Rez
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 05-rez.yml

# Re-apply only network/sysctl
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 07-network.yml
```

### Verbose Output

Append `-vvv` to any command for detailed output.

---

## What Gets Installed

### Repositories & Kernel (`02-repos-kernel.yml`)
- CRB (CodeReady Builder / PowerTools) repository enabled
- EPEL repository enabled
- ELRepo — provides the mainline (`kernel-ml`) kernel
- NVIDIA CUDA repository
- `kernel-ml` + `kernel-ml-devel` (mainline kernel from ELRepo)

### Desktop (`03-desktop.yml`)
- KDE Plasma Workspaces (`@KDE Plasma Workspaces`, `@base-x`)
- Graphical target set as default

### General Packages (`04-packages.yml`)
`bash`, `curl`, `dbus`, `perl`, `git`, `less`, `zsh`, `python3.11`, `htop`, `btop`, `firefox`, `ark`, `flatpak`, `gdb`, `cifs-utils`, `nfs-utils`, `rasdaemon`, `liberation-fonts`, `google-noto-fonts-common`, `google-noto-sans-fonts`, `dejavu-fonts-all`

### Houdini / DCC Dependencies (`04-packages.yml`)
All shared libraries required by Houdini (and most other commercial DCC tools), including: `alsa-lib`, `cups-libs`, `libXrandr`, `libxkbcommon`, `mesa-libGLU`, `nss`, `xcb-util-*`, and more.

### System Management (`04-packages.yml`)
`cockpit`, `cockpit-storaged`, `cockpit-shell`, plus AD/domain-join tooling: `sssd`, `realmd`, `adcli`, `oddjob`, `oddjob-mkhomedir`, `krb5-workstation`, `samba-common-tools`

Cockpit is enabled and started automatically at boot.

### Pipeline — Rez (`05-rez.yml`)
[Rez](https://github.com/AcademySoftwareFoundation/rez) (v3.4.0) is downloaded, installed to `/opt/rez`, and the `rez-pip` plugin is installed. A shared package cache directory is created at `/opt/rez/package_cache` with sticky group permissions (`2777`).

### Security (`06-security.yml`)
- SELinux set to permissive mode
- System limits configured for VFX applications (`nofile 65536`, `nproc 65536`, `stack unlimited`)

### Network (`07-network.yml`)
- ASLON NFS share mounted at `/mnt/aslon`
- Sysctl tuning: `vm.max_map_count`, NFS buffer sizes, net backlog, core dump pattern

### Performance (`08-tuned.yml`)
- `tuned` installed and running with `balanced` profile

### Dotfiles (`01-dotfiles.yml`)
| File | Destination | Purpose |
|---|---|---|
| `Files/profile` | `/etc/profile` | System-wide environment variables |
| `Files/bash_profile` | `/etc/skel/.bash_profile` | Default bash profile for new users |
| `Files/umask.sh` | `/etc/profile.d/umask.sh` | System-wide umask setting |
| `Files/rez.sh` | `/etc/profile.d/rez.sh` | Rez environment setup |

---

## Repository Structure

```
.
├── site.yml                  # Orchestrator — runs all playbooks in order
├── 01-dotfiles.yml           # Shell profiles, environment variables, umask
├── 02-repos-kernel.yml       # Repositories (CRB, EPEL, ELRepo, NVIDIA), kernel-ml
├── 03-desktop.yml            # KDE Plasma Workspaces + graphical target
├── 04-packages.yml           # General packages, Houdini deps, Cockpit, services
├── 05-rez.yml                # Rez package manager install & config
├── 06-security.yml           # SELinux, system limits
├── 07-network.yml            # NFS mounts, sysctl tuning
├── 08-tuned.yml              # Tuned performance profile
├── Files/                    # Config files deployed to the system
│   ├── profile               # /etc/profile
│   ├── bash_profile          # /etc/skel/.bash_profile
│   ├── umask.sh              # /etc/profile.d/umask.sh
│   ├── rez.sh                # /etc/profile.d/rez.sh
│   └── resolv.conf           # DNS resolver config
├── local.yml                 # Legacy monolithic playbook (deprecated)
├── PLAN.md                   # Refactoring plan
└── README.md
```

---

## Customisation

### Rez version
Change the `rez_version` variable in `05-rez.yml`:

```yaml
vars:
  rez_version: "3.4.0"
```

### Rez group
The playbook includes a commented-out `group` setting for the package cache directory. Uncomment and set `rez_group` to lock down cache access to a specific AD/local group:

```yaml
  rez_group: Domain_Artists
```

### NFS Mount
Edit `07-network.yml` to change the NFS server, share path, or mount options.

---

## Notes

- All playbooks target `localhost` and run with `become: true` (root). Ensure the user running `ansible-pull` has sudo privileges or run as root.
- NVIDIA driver installation uses the `rhel8` CUDA repo. If running Alma/Rocky 9, update the `baseurl` and `gpgkey` in `02-repos-kernel.yml` accordingly.
- The ELRepo package URL is pinned to `elrepo-release-8`; update this for RHEL 9-based systems.
- The legacy `local.yml` monolithic playbook is retained for reference but is deprecated in favour of the modular playbooks.
