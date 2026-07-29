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
- **Repositories & kernel** — CRB, EPEL, ELRepo, NVIDIA CUDA, RPM Fusion, kernel-ml
- **NVIDIA drivers** — 580 open drivers
- **Mainline kernel** — kernel-ml from ELRepo (installed before NVIDIA driver)
- **Desktop** — KDE Plasma Workspaces, graphical target
- **Packages** — general deps, Houdini/DCC dependencies, Cockpit, AD tools, services
- **Rez** — Rez package manager install & config
- **Security** — SELinux permissive mode
- **Tuning** — tuned profile, sysctl tuning *(disabled — under repair)*
- **Domain join** — AD/SSSD realm join (manual)
- **Post-domain** — SSSD LDAP + NFS mounts (run after domain join)

---

## Requirements

- Rocky Linux 9 / Alma Linux 9
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

This will clone the repository and run all playbooks in order (except domain-join/post-domain which are manual):

| # | Playbook | Purpose |
|---|----------|---------|
| 1 | `01-dotfiles.yml` | Shell profiles, environment variables, umask |
| 2 | `02-repos-kernel.yml` | Repositories (CRB, EPEL, ELRepo, NVIDIA, RPM Fusion) |
| 3 | `03-nvidia.yml` | NVIDIA 580 open drivers |
| 3b | `03b-kernel-ml.yml` | Mainline kernel (kernel-ml) from ELRepo |
| 4 | `04-desktop.yml` | KDE Plasma Workspaces + graphical target |
| 5 | `05-packages.yml` | General packages, Houdini deps, Cockpit, services |
| 6 | `06-rez.yml` | Rez package manager install & config |
| 7 | `07-security.yml` | SELinux permissive mode |
| 8 | `08-tuning.yml` | Tuned profile, sysctl tuning *(disabled — under repair)* |

> `08-tuning.yml` is commented out of `site.yml` — under repair.
> `10-domain-join.yml` is commented out of `site.yml` and must be run manually.

### Run a Single Playbook

Re-apply any individual stage without re-executing the full bootstrap:

```bash
# Re-apply only packages and services
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 05-packages.yml

# Re-apply only Rez
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 06-rez.yml

# Re-apply only NVIDIA drivers
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git 03-nvidia.yml
```

### Domain Join + Post-Domain (Combined)

```bash
# 1. Join the domain
export REALM_PASSWORD="your_password"
ansible-playbook 10-domain-join.yml

# 2. Run post-domain tasks (SSSD + NFS)
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git post-domain.yml
```

> `post-domain.yml` combines `11-domain-sssd.yml` and `09-nfs.yml` — both require the machine to be domain-joined first.

### Domain Join (Individual)

Join the machine to the Active Directory domain manually:

```bash
sudo ansible-playbook 10-domain-join.yml
```

You will be prompted for the `Administrator` password (or set `REALM_PASSWORD` env var).

### SSSD LDAP Configuration (Individual)

After joining the domain, configure SSSD LDAP settings:

```bash
sudo ansible-playbook 11-domain-sssd.yml
```

This adds `fallback_homedir`, `use_fully_qualified_names`, `ldap_idmap_range_min`, and `ldap_idmap_default_domain_sid` to the SSSD domain configuration. After updating SSSD settings, the playbook stops the service, flushes all cache files (`/var/lib/sss/db/*` and `/var/lib/sss/mc/*`), then restarts SSSD.

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

### Repositories & Kernel (`02-repos-kernel.yml` + `03b-kernel-ml.yml`)

- CRB (CodeReady Builder / PowerTools) repository enabled
- EPEL repository enabled
- ELRepo — provides the mainline (`kernel-ml`) kernel
- NVIDIA CUDA repository
- RPM Fusion free and nonfree repositories
- `kernel-ml` + `kernel-ml-devel` (mainline kernel from ELRepo)

### NVIDIA Drivers (`03-nvidia.yml`)

- NVIDIA 580 open drivers (`@nvidia-driver:580-open`, `nvidia-open`)

### Desktop (`04-desktop.yml`)

- KDE Plasma Workspaces (`@KDE Plasma Workspaces`, `@base-x`)
- Graphical target set as default

### General Packages (`05-packages.yml`)

`bash`, `curl`, `dbus`, `perl`, `git`, `less`, `zsh`, `python3.11`, `htop`, `btop`, `firefox`, `ark`, `flatpak`, `gdb`, `cifs-utils`, `nfs-utils`, `rasdaemon`, `liberation-fonts`, `google-noto-fonts-common`, `google-noto-sans-fonts`, `dejavu-sans-fonts`, `dejavu-sans-mono-fonts`, `dejavu-serif-fonts`

### Houdini / DCC Dependencies (`05-packages.yml`)

All shared libraries required by Houdini (and most other commercial DCC tools), including: `alsa-lib`, `cups-libs`, `libXrandr`, `libxkbcommon`, `mesa-libGLU`, `nss`, `xcb-util-*`, and more.

Also installs `ffmpeg` (from RPM Fusion with `allowerasing` to resolve conflicts) and `nxagent`.

### System Management (`05-packages.yml`)

`cockpit`, `cockpit-storaged`, `cockpit-shell`, plus AD/domain-join tooling: `sssd`, `realmd`, `adcli`, `oddjob`, `oddjob-mkhomedir`, `krb5-workstation`, `samba-common-tools`

Cockpit is enabled and started automatically at boot. `mcelog` is disabled. `rasdaemon` is enabled.

### Pipeline — Rez (`06-rez.yml`)

[Rez](https://github.com/AcademySoftwareFoundation/rez) (v3.4.0) is downloaded, installed to `/opt/rez`, and the `rez-pip` plugin is installed. A shared package cache directory is created at `/opt/rez/package_cache` with sticky group permissions (`2777`).

### Security (`07-security.yml`)

- SELinux set to permissive mode
- SELinux Python bindings installed

### Tuning (`08-tuning.yml`)

> **Disabled** — under repair.

- `tuned` installed and running with `latency-performance` profile
- `vm.max_map_count` set to 1048576 for VFX applications
- NFS buffer sizes (`rmem_max`, `wmem_max`) increased for performance
- `netdev_max_backlog` tuned for high-throughput networking
- Kernel core dump pattern set
- System limits configured for VFX applications (`nofile 65536`, `nproc 65536`, `stack unlimited`)

### NFS (`09-nfs.yml`)

- ASLON NFS share mounted at `/mnt/aslon`
- NFS options: `sec=sys,vers=3,rsize=1048576,wsize=1048576,_netdev,nofail,auto`

### Domain Join (`10-domain-join.yml`)

- Joins the machine to the `alongsidegroup.com` Active Directory domain via `realm join`
- Ensures SSSD and oddjobd services are running
- Enables oddjob-mkhomedir for automatic home directory creation

### SSSD LDAP (`11-domain-sssd.yml`)

Manually run after domain join to configure SSSD for proper AD user mapping:

- `fallback_homedir = /home/%u`
- `use_fully_qualified_names = False`
- `ldap_idmap_range_min = 1260388352`
- `ldap_idmap_default_domain_sid = S-1-5-21-2080557663-2646592229-2320375442`

After applying changes, SSSD is stopped, all cache files are flushed, and the service is restarted.

---

## Repository Structure

```
.
├── site.yml                  # Orchestrator — runs playbooks in order (domain join & tuning excluded)
├── 01-dotfiles.yml           # Shell profiles, environment variables, umask
├── 02-repos-kernel.yml       # Repositories (CRB, EPEL, ELRepo, NVIDIA, RPM Fusion)
├── 03-nvidia.yml             # NVIDIA 580 open drivers
├── 03b-kernel-ml.yml         # Mainline kernel (kernel-ml) from ELRepo
├── 04-desktop.yml            # KDE Plasma Workspaces + graphical target
├── 05-packages.yml           # General packages, Houdini deps, Cockpit, services
├── 06-rez.yml                # Rez package manager install & config
├── 07-security.yml           # SELinux permissive mode
├── 08-tuning.yml             # Tuned profile, sysctl tuning *(disabled — under repair)*
├── 09-nfs.yml                # NFS mounts
├── 10-domain-join.yml        # AD/SSSD realm join (manual)
├── 11-domain-sssd.yml        # SSSD LDAP config (manual, run after domain join)
├── post-domain.yml           # Post-domain tasks (SSSD + NFS, requires domain join)
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

### Domain

Edit `10-domain-join.yml` to change the realm name, user, or workgroup.

### SSSD LDAP

Edit `11-domain-sssd.yml` to change the SID, range min, or other LDAP mapping settings.

---

## Notes

- All playbooks target `localhost` and run with `become: true` (root). Ensure the user running `ansible-pull` has sudo privileges or run as root.
- The legacy `local.yml` monolithic playbook is retained for reference but is deprecated in favour of the modular playbooks.
- The ELRepo package URL adapts to `{{ ansible_distribution_major_version }}` automatically.
- The NVIDIA CUDA repo URL adapts to `{{ ansible_distribution_major_version }}` automatically.
- NFS uses `vers=3` to avoid mount hangs seen with `vers=4` on the current Synology target.
- `10-domain-join.yml`, `08-tuning.yml`, and `post-domain.yml` are not included in `site.yml` — they must be run manually.