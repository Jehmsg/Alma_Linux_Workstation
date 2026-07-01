# Multi-Playbook Refactoring Plan

## Target Structure

```
.
├── README.md
├── site.yml                  # Orchestrator — runs all playbooks in order
├── ansible.cfg               # ansible-pull configuration
├── Files/
│   ├── bash_profile
│   ├── profile
│   ├── resolv.conf
│   ├── rez.sh
│   └── umask.sh
├── 01-dotfiles.yml           # Shell profiles, environment variables, umask
├── 02-repos-kernel.yml       # Repositories (CRB, EPEL, ELRepo, NVIDIA), kernel-ml
├── 03-desktop.yml            # KDE Plasma + graphical target
├── 04-packages.yml           # General packages, Houdini deps, Cockpit, services
├── 05-rez.yml                # Rez package manager install & config
├── 06-security.yml           # SELinux, system limits (limits.d)
├── 07-network.yml            # NFS mounts, sysctl tuning
└── 08-tuned.yml              # Tuned profile
```

## Rationale for Splitting

Each playbook groups logically independent tasks so that:

- **Individual runs** — you can re-run a single stage (e.g. `04-packages.yml`) without re-executing unrelated work.
- **Fault recovery** — if the run fails mid-way, `ansible-pull` can be re-run and each playbook handles its own idempotency.
- **Selective deployment** — skip stages that are already applied (e.g. skip Rez on machines that don't need it).
- **Clear ownership** — each file has a single responsibility, making maintenance and diffs trivial.

The `01-` / `02-` / `03-` prefix ensures filesystem sort order matches execution order when used with `site.yml`.

---

## Playbook Breakdown

### `site.yml` — Orchestrator

A thin playbook that imports all others in order. Run this for a full bootstrap; each sub-playbook can also be run standalone.

```yaml
---
- import_playbook: 01-dotfiles.yml
- import_playbook: 02-repos-kernel.yml
- import_playbook: 03-desktop.yml
- import_playbook: 04-packages.yml
- import_playbook: 05-rez.yml
- import_playbook: 06-security.yml
- import_playbook: 07-network.yml
- import_playbook: 08-tuned.yml
```

### `01-dotfiles.yml` — Shell Profiles & Environment

| Task | Module |
|---|---|
| Copy `/etc/profile` | `copy` (src: Files/profile) |
| Copy `/etc/skel/.bash_profile` | `copy` (src: Files/bash_profile) |
| Copy `/etc/profile.d/umask.sh` | `copy` (src: Files/umask.sh) |
| Copy `/etc/profile.d/rez.sh` | `copy` (src: Files/rez.sh) |

**Idempotency:** `copy` module compares src/dest by default — no change if identical.

---

### `02-repos-kernel.yml` — Repositories & Kernel

| Task | Module |
|---|---|
| Enable CRB/PowerTools repo | `dnf_config_manager` |
| Enable EPEL repo | `dnf_config_manager` |
| Add Elrepo GPG key (v1) | `rpm_key` |
| Add Elrepo GPG key (v2) | `rpm_key` |
| Add NVIDIA CUDA repo | `yum_repository` |
| Install ELRepo release RPM | `dnf` |
| Install `kernel-ml` + `kernel-ml-devel` | `dnf` (enablerepo: elrepo-kernel) |

**Shared vars:** `crb_powertools`

---

### `03-desktop.yml` — Desktop Environment

| Task | Module |
|---|---|
| Install KDE Plasma Workspaces + base-x | `dnf` |
| Set graphical target | `shell` (`systemctl set-default graphical.target`) |

---

### `04-packages.yml` — Packages & Services

| Task | Module |
|---|---|
| Install general dependencies | `dnf` |
| Install Houdini/DCC dependencies | `dnf` |
| Install Cockpit + AD/domain tools | `dnf` |
| Enable & start `cockpit.socket` | `service` |
| Disable & stop `mcelog.service` | `service` |
| Enable & start `rasdaemon.service` | `service` |

---

### `05-rez.yml` — Rez Package Manager

| Task | Module |
|---|---|
| Download Rez archive | `get_url` |
| Extract archive | `unarchive` |
| Run `install.py` | `command` |
| Remove extracted directory | `file` (state: absent) |
| Remove downloaded archive | `file` (state: absent) |
| Create package cache dir (`2777`) | `file` (state: directory) |
| Install `rez-pip` plugin | `command` |

**Shared vars:** `rez_version`, `rez_dir`, `rez_cache_dir`, `rez_python_version`, `rez_download_url`, `rez_download_dir`

---

### `06-security.yml` — Security & System Limits

| Task | Module |
|---|---|
| Install SELinux Python bindings | `dnf` |
| Set SELinux to permissive | `selinux` |
| Create `/etc/security/limits.d/vfx.conf` | `copy` |

---

### `07-network.yml` — Network & Sysctl Tuning

| Task | Module |
|---|---|
| Create `/mnt/aslon` mount point | `file` |
| Mount ASLON NFS share | `mount` |
| Set `vm.max_map_count` | `sysctl` |
| Set `net.core.rmem_max` | `sysctl` |
| Set `net.core.wmem_max` | `sysctl` |
| Set `net.core.netdev_max_backlog` | `sysctl` |
| Set `kernel.core_pattern` | `sysctl` |

---

### `08-tuned.yml` — Performance Tuning

| Task | Module |
|---|---|
| Install `tuned` | `dnf` |
| Enable & start `tuned` service | `service` |
| Set profile to `balanced` | `command` (`tuned-adm profile balanced`) |

---

## Shared Variables

All playbooks that need the common vars should define them locally (since `ansible-pull` runs each playbook independently against `localhost`):

```yaml
vars:
  rez_version: "3.4.0"
  rez_dir: /opt/rez
  rez_cache_dir: "{{ rez_dir }}/package_cache"
  rez_python_version: "python3.11"
  rez_group: Domain_Artists
  rez_download_url: "https://github.com/AcademySoftwareFoundation/rez/releases/download/{{ rez_version }}/{{ rez_version }}.tar.gz"
  rez_download_dir: /tmp
  crb_powertools: "{{ 'powertools' if ansible_distribution_major_version == '8' else 'crb' }}"
```

Each playbook declares only the vars it needs.

---

## Usage

### Full bootstrap (all playbooks in order)

```bash
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git \
  site.yml
```

### Run a single playbook

```bash
# Re-apply only packages and services
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git \
  04-packages.yml

# Re-apply only Rez
ansible-pull -U https://github.com/Jehmsg/Alma_Linux_Workstation.git \
  05-rez.yml
```

### Verbose output

Append `-vvv` to any command.

---

## Execution Order Diagram

```
┌─────────────┐
│ 01-dotfiles │  ← shell profiles (no dependencies)
└──────┬──────┘
       ▼
┌───────────────┐
│ 02-repos-knl  │  ← repos, kernel (no deps on 01)
└──────┬────────┘
       ▼
┌─────────────┐
│  03-desktop │  ← KDE (depends on repos from 02)
└──────┬──────┘
       ▼
┌─────────────┐
│ 04-packages │  ← all packages + services (depends on repos from 02)
└──────┬──────┘
       ▼
┌─────────┐
│ 05-rez  │  ← Rez install (depends on python3.11 from 04)
└────┬────┘
     ▼
┌───────────┐
│06-security│  ← SELinux, limits (no hard deps)
└────┬──────┘
     ▼
┌───────────┐
│07-network │  ← NFS, sysctl (no hard deps)
└────┬──────┘
     ▼
┌──────────┐
│ 08-tuned │  ← performance profile (no hard deps)
└──────────┘
```

---

## Migration Checklist

- [ ] Create `site.yml` orchestrator
- [ ] Create `01-dotfiles.yml`
- [ ] Create `02-repos-kernel.yml`
- [ ] Create `03-desktop.yml`
- [ ] Create `04-packages.yml`
- [ ] Create `05-rez.yml`
- [ ] Create `06-security.yml`
- [ ] Create `07-network.yml`
- [ ] Create `08-tuned.yml`
- [ ] Update `README.md` with new structure and usage examples
- [ ] (Optional) Add `ansible.cfg` for default connection settings
- [ ] Test full run via `ansible-pull` with `site.yml`
- [ ] Test individual playbook runs
