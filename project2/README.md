# Project 2 — Server Backups Automation (Ansible)

Automates backups of critical system and application data on Linux servers with Ansible, replacing a manual `tar` procedure. Includes missing-source handling, secure permissions, and retention cleanup.

## Scenario
Servers must be backed up regularly to prevent data loss from accidental deletion, system failures, or security incidents. Critical system and application data must be backed up on a scheduled basis.

## Architecture
```
Windows laptop (VS Code Remote-SSH)
        │
        ▼
control node (Ubuntu, Ansible)  ──SSH (private IPs)──▶  ms1, ms2 (Amazon Linux)
                                                            │
                                                            ▼
                                                   /backups/*.tar.gz (local)
```

| Host | Role | Private IP |
|------|------|-----------|
| control node | Ansible controller | 172.31.31.155 |
| ms1 | Backup target | 172.31.17.210 |
| ms2 | Backup target | 172.31.24.205 |

## Project Structure
```
project2/
├── ansible.cfg      # Points to the inventory, Python interpreter settings
├── inventory.ini    # Target hosts, SSH user and key
├── vars.yml         # All backup variables
├── backup.yml       # Playbook
└── README.md
```

## Manual Procedure vs. Automation
| Manual step | Playbook equivalent |
|-------------|---------------------|
| `mkdir -p /backups` + `chmod 700 /backups` | `file` module (directory, root-owned, `0700`) |
| `DATE=$(date +%F_%H-%M-%S)` | `backup_timestamp` built from the `ansible_date_time` fact |
| `tar -czvf /backups/etc_backup_$DATE.tar.gz /etc` | `community.general.archive`, looped over `backup_sources` |
| `tar -czvf /backups/web_backup_$DATE.tar.gz /var/www` | Same task. Skipped with a message if the path doesn't exist |
| `ls -lh /backups` | `command: ls -lh` + `debug` output |
| *(not in manual)* | Retention: deletes archives older than 7 days |

## Variables (`vars.yml`)
| Variable | Purpose |
|----------|---------|
| `backup_dir` | Backup destination (`/backups`) |
| `backup_dir_mode` | Directory permissions (`0700`) |
| `backup_timestamp` | `YYYY-MM-DD_HH-MM-SS` from the server's clock (UTC) |
| `backup_sources` | List of `{name, path}` to back up (`/etc`, `/var/www`) |
| `backup_format` | Compression format (`gz`) |
| `backup_retention_days` | Archives older than this are deleted (`7`) |

## Playbook Flow
1. Create `/backups` (root-owned, `0700`)
2. `stat` each source and register the results
3. Report any missing source instead of failing
4. Archive only sources that exist and are directories (`0600` per archive)
5. Find and delete archives past the retention period
6. List `/backups` to verify

## Usage
```bash
cd ~/ansible-projects/project2
ansible all -m ping                            # connectivity
ansible-playbook backup.yml --syntax-check     # validate
ansible-playbook backup.yml                    # run backups
```

**Requirement:** the `community.general` collection (bundled with the full `ansible` package).
```bash
ansible-galaxy collection list community.general
```

## Verification
```bash
# Archive contents
ansible monitored -m shell -a "tar -tzf /backups/etc_backup_*.tar.gz | head -10" --become

# Integrity check
ansible monitored -m shell -a "gzip -t /backups/etc_backup_*.tar.gz && echo OK" --become

# Test restore of one file (to /tmp, never over /etc)
ansible ms1 -m shell -a "mkdir -p /tmp/restore && tar -xzf /backups/etc_backup_*.tar.gz -C /tmp/restore etc/hostname && diff /tmp/restore/etc/hostname /etc/hostname && echo RESTORE OK" --become
```

## Results
- `/etc` archived on both nodes: `etc_backup_<timestamp>.tar.gz` (~9.9 MB), `-rw-------` root:root
- `/var/www` not present on either node, so it was skipped with a clear message and the play still completed
- `failed=0` on both nodes

## Key Design Decisions
- **Missing-source handling:** `/var/www` doesn't exist on these servers. The manual `tar` would fail; the playbook skips and reports it so the `/etc` backup still completes.
- **List-driven sources:** adding a directory to back up is one entry in `vars.yml`, not a new task.
- **Defense in depth:** the directory is `0700` and each archive is `0600`. Archives contain `/etc/shadow` and SSH host keys.
- **Retention after backup:** old archives are deleted only after the new one is created, so a failed run never leaves zero backups.
- **UTC timestamps:** servers use UTC, which avoids daylight-saving gaps and keeps timestamps consistent across hosts.

## Limitations
- Backups are stored on the same server. They protect against accidental deletion but not disk loss, instance termination, or ransomware.
- Not yet scheduled; runs on demand.

## Future Improvements
- Schedule runs (cron on the control node, or `ansible.builtin.cron` on targets)
- Ship archives off-server to S3 (`amazon.aws.s3_object`) for 3-2-1 compliance
- Tiered retention (daily / weekly / monthly)
- Automated restore testing and alerting on backup failure
- Convert into a reusable `backup` role