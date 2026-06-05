# Ansible Role: joe-speedboat.journal_forwarder

This role installs and configures Fluent Bit to forward systemd journal logs to Graylog via GELF input. It preserves structured journal metadata such as `hostname`, `priority`, `systemd_unit`, and `message` by using Fluent Bit's native `systemd` input.

The role can also forward auditd package-change events from `/var/log/audit/audit.log`. When audit forwarding is enabled, the role installs and starts `auditd`, deploys package-manager execution audit rules, loads them with `augenrules`, and configures Fluent Bit parsers for common audit fields.

## Requirements

* Ansible 2.9 or higher
* systemd-based Linux distribution
* Supported/tested targets:
  * Ubuntu 24.04 LTS
  * Ubuntu 26.04 LTS
  * Rocky Linux 8.10
  * Rocky Linux 9.7
  * Rocky Linux 10.1
* Network access from the target to `https://packages.fluentbit.io/` so the role can configure the official Fluent Bit package repository
* Graylog server with GELF input enabled, by default port `12201/tcp`

## Role Variables

Variables are defined in `defaults/main.yml`.

- `journal_forwarder_install`: `true` — Toggle installation on/off.
- `graylog_gelf_host`: `graylog1.sun.bitbull.ch` — Graylog GELF input host.
- `graylog_gelf_port`: `12201` — Graylog GELF input port.
- `graylog_gelf_mode`: `tcp` — GELF transport mode, `tcp` or `udp`.
- `graylog_fluent_bit_workers`: `2` — Number of Fluent Bit output workers.
- `journal_forwarder_tag`: `journald` — Tag for the systemd journal input.
- `journal_forwarder_read_from_tail`: `true` — Start reading journal entries from the tail.
- `journal_forwarder_storage_path`: `/var/lib/fluent-bit` — Fluent Bit state database directory.
- `journal_forwarder_audit_enabled`: `true` — Enable auditd package-change forwarding.
- `journal_forwarder_audit_tag`: `audit` — Tag for audit log input.
- `journal_forwarder_audit_rules_file`: `/etc/audit/rules.d/package.rules` — Audit rules destination.

## Package Repositories

The role configures the official Fluent Bit repository for the target OS:

- Ubuntu: `https://packages.fluentbit.io/ubuntu/<codename>` with the Fluent Bit key in `/usr/share/keyrings/fluentbit-keyring.gpg`
- Rocky/RHEL-compatible systems: `https://packages.fluentbit.io/rockylinux/<major-version>/`

Audit package-change rules are rendered per OS family:

- Rocky/RHEL-compatible systems watch `rpm`, `dnf`, `yum`, and `/usr/libexec/platform-python`
- Ubuntu/Debian-compatible systems watch `dpkg`, `apt`, `apt-get`, `apt-cache`, and `/usr/bin/python3`

## Example Playbook

```yaml
- hosts: all
  become: true
  vars:
    graylog_gelf_host: "graylog.example.com"
    graylog_gelf_port: 12201
    graylog_gelf_mode: tcp
  roles:
    - joe-speedboat.journal_forwarder
```

## Graylog Input

Create a GELF TCP input in Graylog listening on the configured port. For the default port on a firewalld-managed Graylog host:

```bash
firewall-cmd --add-port 12201/tcp --permanent
systemctl restart firewalld
```

## Installation

### From Galaxy

```bash
ansible-galaxy install joe-speedboat.journal_forwarder
```

### From Git

```bash
git clone https://github.com/joe-speedboat/ansible.journal_forwarder
ansible-galaxy install -r requirements.yml
```

## License

GPLv3
Copyright (c) Chris Ruettimann <chris@bitbull.ch>
