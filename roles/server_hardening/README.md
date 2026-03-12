# server_hardening

Standardized server hardening for Ubuntu 24.04. Designed for Hetzner Cloud + Tailscale VPN but works on any Ubuntu server.

## What It Does

- **SSH**: Key-only auth, root login disabled, AllowUsers, no forwarding, max 3 auth tries
- **UFW**: Default deny incoming, SSH from Tailscale CGNAT only, Tailscale UDP allowed
- **fail2ban**: SSH jail, 1h ban, 5 retries, ignores Tailscale range
- **sysctl**: Network hardening (syncookies, no redirects, rp_filter) + kernel hardening (ASLR, dmesg_restrict, ptrace_scope)
- **Unattended upgrades**: Security-only, auto-reboot at 04:00
- **NTP**: systemd-timesyncd (default) or chrony
- **Systemd**: Journal size limits, DefaultLimitNOFILE
- **User management**: Creates ops user with sudo, locks root password
- **Banner**: Login warning on /etc/issue.net

## Usage

```yaml
- hosts: all
  become: true
  roles:
    - role: server_hardening
      vars:
        hardening_ops_ssh_pubkeys:
          - "ssh-ed25519 AAAA... ops@opskern"
```

## Docker Hosts

Docker requires `net.ipv4.ip_forward = 1`. Override in host_vars:

```yaml
hardening_sysctl_params:
  net.ipv4.ip_forward: 1
  # Include other defaults you want to keep
  net.ipv4.tcp_syncookies: 1
  net.ipv4.conf.all.rp_filter: 1
  # ... etc
```

## Homelab Use

For existing homelab hosts where the user is not `ops`:

```yaml
hardening_ops_user: "thewismit"
hardening_ssh_allowed_users: "thewismit"
```

## Variables

See `defaults/main.yml` for all configurable variables. Every hardening feature can be toggled or tuned.

## Dependencies

- `common` role (passwordless sudo)

## Testing

```bash
cd roles/server_hardening
molecule test
```

Note: UFW and fail2ban are disabled in Docker-based Molecule tests (require kernel modules). Full integration testing requires a real VM.

## License

MIT
