# Project 3 — Server Patching Automation (Ansible)

Automates monthly security patching of Linux servers with Ansible, replacing a manual `apt`/`yum` procedure. The playbook adapts to the OS family, applies security updates only, reboots only when required, and writes a compliance report on each server.

## Scenario
All servers must have their security packages updated monthly to stay secure and compliant. Manual patching is time-consuming and error-prone. The task is to automate security updates with Ansible and schedule them with cron.

## Architecture
```
Windows laptop (VS Code Remote-SSH)
        │
        ▼
control node (Ubuntu, Ansible, cron)  ──SSH (private IPs)──▶  ms1, ms2 (Amazon Linux 2023)
                                                                  │
                                                                  ▼
                                                    /var/log/patching/patch_<date>.log
```

| Host | Role | OS | Private IP |
|------|------|----|-----------|
| control node | Ansible controller + cron scheduler | Ubuntu | 172.31.31.155 |
| ms1 | Patch target | Amazon Linux 2023 (dnf) | 172.31.17.210 |
| ms2 | Patch target | Amazon Linux 2023 (dnf) | 172.31.24.205 |

## Project Structure
```
project3/
├── ansible.cfg      # Points to the inventory, Python interpreter settings
├── inventory.ini    # Target hosts, SSH user and key
├── vars.yml         # Patch, reboot, report and cron variables
├── patch.yml        # Playbook
└── README.md
```

## Manual Procedure vs. Automation
| Manual step | Playbook equivalent |
|-------------|---------------------|
| `yum updateinfo security` | `dnf updateinfo list --security` (list advisories) |
| `yum update -y` | `dnf` module, `name: "*"`, `state: latest`, `security: true` |
| `apt update` | `apt` module, `update_cache` with `cache_valid_time` |
| `apt install unattended-upgrades -y` | `apt` module, `state: present` |
| `apt upgrade -y` | `unattended-upgrade -v` (security only) or `apt upgrade` when `patch_security_only: false` |
| `yum check-update` | `dnf check-update --security` (rc `0` = none, `100` = updates remain) |
| `apt list --upgradable` | Same command, output registered |
| *(not in manual)* | Reboot check, conditional reboot, per-host patch report |

## Variables (`vars.yml`)
| Variable | Purpose |
|----------|---------|
| `patch_security_only` | `true` installs security updates only; `false` installs all |
| `patch_releasever_latest` | AL2023: move to the newest repo release before patching |
| `patch_exclude_packages` | Packages never to update (e.g. `kernel*`) |
| `patch_apt_cache_valid_time` | Skip `apt update` if the cache is newer than this (seconds) |
| `patch_install_unattended_upgrades` | Install `unattended-upgrades` on Debian/Ubuntu |
| `patch_allow_reboot` | Permit a reboot when one is required |
| `patch_reboot_timeout` | Seconds to wait for the host to come back |
| `patch_reboot_msg` | Message broadcast before reboot |
| `patch_report_dir` | Where each node stores its patch report |
| `patch_cron_*` | Cron schedule and log path on the control node |

## Playbook Flow
1. **RedHat block** (`ansible_os_family == "RedHat"`): list advisories → optional AL2023 release upgrade → apply security updates → reboot check (`dnf needs-restarting -r`) → verify remaining updates
2. **Debian block** (`ansible_os_family == "Debian"`): update cache → install unattended-upgrades → apply security (or all) updates → reboot check (`/var/run/reboot-required`) → list upgradable
3. **Reboot decision**: combines both checks with `set_fact`; reboots only if required **and** allowed
4. **Report**: writes `/var/log/patching/patch_<date>.log` on each node and prints a summary

## Usage
```bash
cd ~/ansible-projects/project3
ansible all -m ping                            # connectivity
ansible-playbook patch.yml --syntax-check      # validate
ansible-playbook patch.yml                     # patch now
```

## Verification
```bash
# Compliance report on each node
ansible monitored -m shell -a "cat /var/log/patching/patch_*.log" --become

# Remaining security updates (rc 0 = fully patched)
ansible monitored -m shell -a "dnf check-update --security; echo rc=\$?" --become

# Reboot required? (exit code 0 = no, 1 = yes)
ansible monitored -m shell -a "dnf needs-restarting -r; echo rc=\$?" --become

# Newer AL2023 release available?
ansible monitored -m command -a "dnf check-release-update" --become
```

## Results
- Both nodes ran the RedHat path. All Debian tasks were skipped as expected.
- No outstanding security advisories within the locked AL2023 release: `Packages changed: False`
- `dnf needs-restarting -r` returned `0`, so no reboot was performed
- Patch report written on both nodes. `failed=0`

## Key Design Decisions
- **Security only:** the manual `apt upgrade -y` / `yum update -y` upgrade *everything*. The playbook filters to security fixes to match the requirement and reduce risk.
- **OS-aware with `block` + `when`:** one playbook serves both OS families. Unused branches are skipped.
- **Explicit exit-code handling:** `failed_when` treats `dnf needs-restarting` rc `1` and `dnf check-update` rc `100` as valid results, not errors.
- **Conditional reboot:** reboots only when the kernel, glibc, or systemd changed, using the `reboot` module, which waits for the host to return.
- **Compliance evidence:** a dated, root-owned report on every node.
- **AL2023 versioned repos:** patching stays within the locked release unless `patch_releasever_latest: true`, for predictable, reproducible updates.

## Scheduling
Patching runs monthly from the control node via cron (1st of the month, 03:00), logging to `patch_cron.log`. *(Configured in Step 5.)*

## Limitations
- Only covers software installed through the package manager (not containers, tarballs, or language packages)
- No batching yet. All hosts patch and reboot at the same time.
- "Fully patched" is relative to the locked AL2023 release

## Future Improvements
- Rolling patching with `serial` and `max_fail_percentage`
- Dev → prod promotion with pinned `releasever` per environment
- Send patch reports to a central location (Datadog logs / S3)
- Pre- and post-patch health checks and load balancer draining
- Pair with a vulnerability scanner (Amazon Inspector) for full coverage