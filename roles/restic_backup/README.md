# Role: restic_backup

Deploys systemd service + timer units for scheduled Restic backups. Works in
tandem with the `restic` role (which runs the actual backup logic).

## What it does

1. Deploys `restic-backup.service` — a oneshot systemd service that calls the backup script
2. Deploys `restic-backup.timer` — triggers the service daily with a randomized delay
3. Creates a `user-alignment.conf` drop-in to run the service as the configured user
4. Applies host-specific overrides (start time offsets, additional exclude paths)
5. Verifies the service and timer are active after deployment

## Files

| File | Purpose |
|------|---------|
| `tasks/main.yml` | Main task list (deploys units, enables timer) |
| `tasks/30-root-and-host-overrides.yml` | Per-host overrides for backup time, excludes |
| `tasks/99-verify.yml` | Post-deploy verification |
| `templates/restic-backup.service.j2` | Systemd service unit template |
| `templates/restic-backup.timer.j2` | Systemd timer unit template |

## Variables

Inherits all variables from the `restic` role. Additional scheduling variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `restic_timer_base_calendar` | `03:00` | Base time for backup window |
| `restic_timer_increment_minutes` | `10` | Per-host time offset (stagger fleet) |
| `restic_timer_randomized_delay_sec` | `30m` | Systemd RandomizedDelaySec |
| `restic_env_file` | `/root/.restic/restic.env` | Path to env file with RESTIC_PASSWORD |

## Timer staggering

When deploying to a fleet, backups are staggered by `restic_timer_increment_minutes`
per host to avoid hammering the backup server simultaneously. Host index is derived
from the play order.

## Example: add to a host

```yaml
- hosts: backup_servers
  roles:
    - role: restic_backup
      vars:
        restic_repo_path: "sftp:{{ ansible_user }}@{{ pbs_host_ip }}:/mnt/backup/restic"
        restic_password: "{{ vault_restic_password }}"
```

## Verifying after deploy

```bash
systemctl status restic-backup.timer
systemctl list-timers restic-backup.timer
journalctl -u restic-backup.service --since today
```
