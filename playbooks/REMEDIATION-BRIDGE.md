# Remediation Bridge — Architecture & Operations Guide

The remediation bridge is the flagship component of this stack. It creates a
closed feedback loop: Prometheus detects problems → Alertmanager fires →
bridge runs the right Ansible playbook → ntfy notifies you of the outcome.

No equivalent open-source tool exists that does Alertmanager → Ansible dispatch
with per-alert cooldowns out of the box.

## Architecture

```
Prometheus ──alerts──▶ Alertmanager
                              │
                              │ POST /hook (JSON webhook)
                              ▼
                    remediation-bridge (FastAPI)
                     running on ansible-host:9999
                              │
                    ┌─────────┴──────────┐
                    │  cooldown check    │  (in-memory, per alert+host)
                    └─────────┬──────────┘
                              │ ansible-playbook ...
                              ▼
                    remediation playbook
                    (restart-container, cleanup-disk, etc.)
                              │
                              ▼
                    ntfy notification → your phone
```

## Deployment

```bash
# Deploy the bridge (always --check first)
ansible-playbook playbooks/deploy-remediation-bridge.yml --check --limit ansible-host
ansible-playbook playbooks/deploy-remediation-bridge.yml --limit ansible-host

# Verify
curl http://ansible-host:9999/health
curl http://ansible-host:9999/mappings
```

## Configure Alertmanager

Add a webhook receiver to your `alertmanager.yml`:

```yaml
receivers:
  - name: remediation-bridge
    webhook_configs:
      - url: http://<ansible-host-ip>:9999/hook
        send_resolved: false   # bridge only handles firing alerts

route:
  receiver: remediation-bridge
  # Optionally limit to specific alert groups:
  routes:
    - match_re:
        alertname: "ContainerDown|DiskSpaceHigh|SystemdServiceFailed|TLSCertExpiringSoon"
      receiver: remediation-bridge
```

## Adding a new remediation

1. Write the playbook to `remediation/<name>.yml` (must accept `target_host` var)
2. Add an entry to `/opt/remediation-bridge/remediation-map.yml`:

```yaml
mappings:
  MyNewAlert:
    playbook: remediation/my-new-playbook.yml
    cooldown_minutes: 15
    extra_vars:           # optional
      some_flag: true
```

3. No restart needed — the bridge hot-reloads the map on each webhook.

## Built-in remediations

| Alert | Playbook | Cooldown |
|-------|---------|---------|
| `ContainerDown` | `remediation/restart-container.yml` | 10 min |
| `DiskSpaceHigh` | `remediation/cleanup-disk.yml` | 60 min |
| `SystemdServiceFailed` | `remediation/service-restart.yml` | 15 min |
| `TLSCertExpiringSoon` | `remediation/caddy-reload.yml` | 360 min |

## Cooldown behavior

If the same `alertname:host` combination triggers within the cooldown window,
the bridge logs it and skips execution. This prevents remediation loops when
a service fails repeatedly (e.g., OOMKill loop).

Cooldowns are in-memory — they reset on bridge restart.

## ntfy notifications

The bridge sends three types of notifications via ntfy:

| Event | Priority | Tags |
|-------|---------|------|
| Remediation succeeded | default | `white_check_mark` |
| Remediation failed | urgent | `rotating_light` |
| Playbook timed out | high | `hourglass_flowing_sand` |

## Variables (set in group_vars/all/vars.yml)

| Variable | Description |
|----------|-------------|
| `ntfy_url` | ntfy server base URL (e.g. `http://x.x.x.x:8081`) |
| `ntfy_topic` | ntfy topic name |
| `ansible_dir` | Path to your ansible repo on ansible-host |
| `vault_pass_file` | Path to vault password file on ansible-host |
| `bridge_dir` | Deployment directory (default: `/opt/remediation-bridge`) |

## Monitoring the bridge

```bash
# Service status
systemctl status remediation-bridge

# Live logs
journalctl -u remediation-bridge -f

# Check current cooldowns (restart bridge to clear)
systemctl restart remediation-bridge
```
