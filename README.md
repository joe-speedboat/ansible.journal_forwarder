# Ansible Role: joe-speedboat.journal_forwarder

This role installs and configures fluent-bit to forward systemd journal logs to Graylog via GELF input. Unlike traditional syslog forwarding, this approach preserves journal keys (structured metadata) such as _HOSTNAME, _SYSTEMD_UNIT, PRIORITY, and MESSAGE fields.

## Requirements

* Ansible 2.9 or higher
* systemd-based Linux distribution (Rocky, AlmaLinux, RHEL, Ubuntu, Debian)
* Graylog server with GELF input enabled (default: port 12201/tcp)

## Role Variables

Variables are defined in `defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `journal_forwarder_install` | `True` | Toggle installation on/off |
| `graylog_gelf_host` | `graylog1.sun.bitbull.ch` | Graylog GELF input host |
| `graylog_gelf_port` | `12201` | Graylog GELF input port |
| `graylog_gelf_mode` | `tcp` | Transport mode (tcp/udp) |
| `graylog_fluent_bit_workers` | `2` | Number of fluent-bit workers |

## Example Playbook

```yaml
- hosts: all
  vars:
    graylog_gelf_host: "graylog.example.com"
  roles:
    - joe-speedboat.journal_forwarder
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
