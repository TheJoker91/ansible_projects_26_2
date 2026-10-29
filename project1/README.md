# Project 1 — Server Monitoring with Datadog (Ansible)

Automates onboarding of Linux servers into Datadog by installing and configuring the Datadog Agent 7 with Ansible, replacing a manual install procedure.

## Scenario
Production servers must be continuously monitored to detect performance issues, outages, and resource exhaustion. The company uses Datadog. New servers must be onboarded through automation instead of by hand.

## Architecture
```
Windows laptop (VS Code Remote-SSH)
        │
        ▼
control node (Ubuntu, Ansible)  ──SSH (private IPs)──▶  ms1, ms2 (Amazon Linux)
                                                            │
                                                            ▼
                                                   Datadog (datadoghq.com)
```

| Host | Role | Private IP |
|------|------|-----------|
| control node | Ansible controller | 172.31.31.155 |
| ms1 | Monitored target | 172.31.17.210 |
| ms2 | Monitored target | 172.31.24.205 |

## Project Structure
```
project1/
├── ansible.cfg      # Points to the inventory, Python interpreter settings
├── inventory.ini    # Target hosts, SSH user and key
├── vars.yml         # All Datadog variables
├── datadog.yml      # Playbook
└── README.md
```

## Manual Steps vs. Automation
| Manual step | Playbook equivalent |
|-------------|---------------------|
| `export DD_API_KEY / DD_SITE / DD_ENV / DD_HOSTNAME` | `environment:` block populated from `vars.yml` |
| `curl ... install_script_agent7.sh \| bash` | `get_url` download + `command` run, guarded by `creates:` |
| `systemctl enable` / `systemctl start` | `systemd_service` (enabled + started) |
| `systemctl status` | `datadog-agent health` with retries |

## Variables (`vars.yml`)
| Variable | Purpose |
|----------|---------|
| `dd_api_key` | Datadog API key |
| `dd_site` | Datadog region (`datadoghq.com`) |
| `dd_env` | Environment tag (`dev`) |
| `dd_hostname` | `{{ inventory_hostname }}`, a unique name per server |
| `dd_script_url` / `dd_script_path` | Installer location and download path |
| `dd_config_file` | Agent config, used as the idempotency guard |
| `dd_service_name` | systemd service name |
| `dd_health_retries` / `dd_health_delay` | Health check wait tuning |

## Usage
```bash
cd ~/ansible-projects/project1
ansible all -m ping                              # connectivity
ansible-playbook datadog.yml --syntax-check      # validate
ansible-playbook datadog.yml                     # deploy
```

## Verification
```bash
ansible monitored -m command -a "systemctl is-active datadog-agent" --become
ansible monitored -m command -a "systemctl is-enabled datadog-agent" --become
ansible monitored -m shell -a "grep -E '^(hostname|env|site):' /etc/datadog-agent/datadog.yaml" --become
ansible monitored -m shell -a "datadog-agent status | grep -A3 'API Keys status'" --become
```
In the Datadog UI, both hosts appear under **Infrastructure → Infrastructure List** with the `env:dev` tag.

## Results
- Both nodes onboarded: `Agent health: PASS`, service `active` and `enabled`
- Each server reports under its own hostname (`ms1`, `ms2`)
- Rerunning the playbook makes no changes (idempotent)

## Key Design Decisions
- **Unique hostnames:** the manual steps hardcoded `student-server`, which would have merged both servers into one Datadog host.
- **Idempotency:** `creates:` prevents reinstalling the agent on every run.
- **Secret handling:** `no_log: true` keeps the API key out of console output and logs.
- **Health check with retries:** absorbs the agent's warm-up time instead of failing on the first check.

## Troubleshooting Notes
| Issue | Cause | Fix |
|-------|-------|-----|
| `no such identity: ~/.ssh/dev-key.pem` | Key not on control node | Used the control node's own `id_ed25519` key |
| `Section [ws1:vars] not valid` | `:vars` used on a host name, not a group | One `[monitored]` group + `[monitored:vars]` |
| Permission denied with `.pub` key | Public key passed as identity | Use the private key (no `.pub`) |

## Future Improvements
- Encrypt the API key with Ansible Vault (`ansible-vault encrypt vars.yml`)
- Replace the install script with the official `datadog.dd` collection for version pinning and GPG verification
- Use the `amazon.aws.aws_ec2` dynamic inventory to discover servers by tag
- Run the playbook from a CI/CD pipeline