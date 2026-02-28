# Role: restic

Runs Restic backups with automatic service pausing, NFS stale-handle recovery,
prune, and repository integrity checks. Designed for `cron` or systemd timer use.

## What it does

1. Deploys the Restic password file to the target host
2. Verifies the NFS/SFTP repo path is accessible (auto-remounts if stale)
3. Optionally pauses systemd services before backup (stops database writes)
4. Runs `restic backup` over the configured paths
5. Restarts paused services in an `always` block (even on failure)
6. Prunes old snapshots per retention policy
7. Runs `restic check` for repository integrity

## Exit code 3 behavior

Restic exits with code 3 when some files were skipped (permission denied,
files in use, etc.). This is **normal** when running as a non-root user or
when system files change during backup. The playbook treats exit code 3 as
success. Only codes other than 0 and 3 indicate real failures.

## Requirements

- Restic binary installed on the target host (`apt install restic` or manual install)
- Accessible backup repository (SFTP, NFS, local, S3, etc.)
- Password file readable by the backup user

## Variables

All variables have defaults in `defaults/main.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `restic_repo_path` | `/mnt/synology/restic` | Path to Restic repository |
| `restic_password_file` | `/root/.restic_pass` | Path where password file is written |
| `restic_password` | `changeme` | Password content — **override via vault** |
| `restic_backup_paths` | `[/etc, /var/lib/postgresql]` | Paths to back up |
| `restic_exclude_paths` | `[/var/cache, /tmp, ...]` | Paths to exclude |
| `restic_pause_services` | `[]` | systemd services to stop during backup |
| `restic_prune.keep_last` | `7` | Daily snapshots to keep |
| `restic_prune.keep_weekly` | `4` | Weekly snapshots to keep |
| `restic_prune.keep_monthly` | `3` | Monthly snapshots to keep |

## Example playbook

```yaml
- hosts: backup_servers
  roles:
    - role: restic
      vars:
        restic_repo_path: "sftp:{{ ansible_user }}@{{ pbs_host_ip }}:/mnt/backup/restic"
        restic_password: "{{ vault_restic_password }}"
        restic_backup_paths:
          - /etc
          - /var
        restic_pause_services:
          - postgresql
```

## Companion role

Use `restic_backup` alongside this role to deploy systemd timers that call
this role on a schedule (daily by default, with randomized delay).

## NFS stale handle recovery

If the repo path is an NFS mount that goes stale (common on homelab NAS restarts),
the role auto-recovers by running `umount -l` + `mount` before attempting backup.
This prevents manual intervention after NAS reboots.
