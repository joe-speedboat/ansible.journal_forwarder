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

## Role Task Layout

The role follows the Bitbull role-template task fallback model:

- `tasks/shared/` contains validation and OS-independent Fluent Bit/audit configuration (`05_validate.yml`, `25_configure.yml`).
- `tasks/Ubuntu/` contains Ubuntu apt repository and package installation tasks (`10_prep.yml`, `20_setup.yml`).
- `tasks/rhelAll/` contains Rocky/Alma/RHEL-compatible repository and package installation tasks (`10_prep.yml`, `20_setup.yml`).

`tasks/main.yml` and `tasks/include-file.yml` are kept unchanged from the template control flow.

## Graylog Search Fields

The role keeps the raw `message` and adds normalized fields for common security and package events:

- Package changes: `audit_type`, `audit_msg`, `package_action`, `package_name`, `package_type`, `package_gpg_result`, `process_comm`, `process_exe`, `terminal`, `source_ip`, `result`
- SSH success/failure/invalid-user attempts: `auth_result`, `auth_method`, `auth_user`, `source_ip`, `source_port`
- Login/logout sessions: `auth_service`, `auth_session_state`, `auth_user`, `auth_uid`, `auth_actor`, `auth_actor_uid`
- Sudo commands: `auth_actor`, `auth_user`, `sudo_pwd`, `sudo_command`, `terminal`

### Graylog Search Patterns and Lab Examples

Use `source:<hostname>` to scope searches to one host. The examples below were validated against the lab hosts `test-hermes1` through `test-hermes5`.

Common search patterns:

```text
# Login/logout sessions, including SSH
source:<host> AND journal_forwarder:true AND auth_service:sshd AND _exists_:auth_session_state

# sudo command execution
source:<host> AND journal_forwarder:true AND _exists_:sudo_command

# su and su - sessions
source:<host> AND journal_forwarder:true AND auth_service:su* AND _exists_:auth_session_state

# Package install, portable across Ubuntu and Rocky/RHEL
source:<host> AND journal_forwarder:true AND package_action:install AND package_name:<package>*

# Package remove, portable across Ubuntu and Rocky/RHEL
source:<host> AND journal_forwarder:true AND package_action:remove AND package_name:<package>*

# Package update/upgrade, portable search shape.
# Set the Graylog time range wide enough to include an actual update transaction.
source:<host> AND journal_forwarder:true AND (package_action:update OR package_action:upgrade OR package_action:dist-upgrade OR package_action:full-upgrade)

# RHEL/Rocky native audit package events
source:<host> AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:install AND package_name:<package>*
```

Notes:

- Rocky/RHEL package modifications use `audit_type=SOFTWARE_UPDATE`.
- Ubuntu apt/dpkg package modifications use `audit_type=EXECVE`; therefore `audit_type:SOFTWARE_UPDATE` is not portable to Ubuntu.
- `su - <user>` is logged by PAM as `auth_service=su-l`; use `auth_service:su*` to match both `su` and `su -`.
- Package update events require an actual package update transaction and must be searched over a time range that includes that transaction. The lab update examples below were visible with a 24h/7d Graylog range, not with the default short range. If no updates were available during a lab run, the search pattern is documented but no verified sample is shown for that OS.

### Example Matches by OS

| OS / lab host | Event | Search pattern | Example fields from Graylog |
|---|---|---|---|
| Ubuntu 24.04 / `test-hermes1` | login/logout | `source:test-hermes1 AND journal_forwarder:true AND auth_service:sshd AND _exists_:auth_session_state` | `auth_service=sshd`, `auth_session_state=closed`, `auth_user=root` |
| Ubuntu 24.04 / `test-hermes1` | sudo | `source:test-hermes1 AND journal_forwarder:true AND _exists_:sudo_command` | `auth_actor=jfmatrix`, `auth_user=root`, `sudo_command=/usr/bin/id` |
| Ubuntu 24.04 / `test-hermes1` | su | `source:test-hermes1 AND journal_forwarder:true AND auth_service:su* AND auth_user:jfmatrix` | `auth_service=su-l`, `auth_session_state=opened/closed`, `auth_user=jfmatrix` |
| Ubuntu 24.04 / `test-hermes1` | package add | `source:test-hermes1 AND journal_forwarder:true AND package_action:install AND package_name:nano*` | `audit_type=EXECVE`, `package_action=install`, `package_name=nano`, `process_comm=apt-get` |
| Ubuntu 24.04 / `test-hermes1` | package remove | `source:test-hermes1 AND journal_forwarder:true AND package_action:remove AND package_name:nano*` | `audit_type=EXECVE`, `package_action=remove`, `package_name=nano:amd64`, `process_comm=dpkg` |
| Ubuntu 24.04 / `test-hermes1` | package update | `source:test-hermes1 AND journal_forwarder:true AND (package_action:upgrade OR package_action:dist-upgrade OR package_action:full-upgrade)` | No package update sample was present in the lab query window. Expected Ubuntu package updates are parsed from apt/dpkg `EXECVE` records. |
| Ubuntu 26.04 / `test-hermes2` | login/logout | `source:test-hermes2 AND journal_forwarder:true AND auth_service:sshd AND _exists_:auth_session_state` | `auth_service=sshd`, `auth_session_state=closed`, `auth_user=root` |
| Ubuntu 26.04 / `test-hermes2` | sudo | `source:test-hermes2 AND journal_forwarder:true AND _exists_:sudo_command` | `auth_actor=jfmatrix`, `auth_user=root`, `sudo_command=/usr/bin/id` |
| Ubuntu 26.04 / `test-hermes2` | su | `source:test-hermes2 AND journal_forwarder:true AND auth_service:su* AND auth_user:jfmatrix` | `auth_service=su-l`, `auth_session_state=opened/closed`, `auth_user=jfmatrix` |
| Ubuntu 26.04 / `test-hermes2` | package add | `source:test-hermes2 AND journal_forwarder:true AND package_action:install AND package_name:nano*` | `audit_type=EXECVE`, `package_action=install`, `package_name=nano`, `process_comm=apt-get` |
| Ubuntu 26.04 / `test-hermes2` | package remove | `source:test-hermes2 AND journal_forwarder:true AND package_action:remove AND package_name:nano*` | `audit_type=EXECVE`, `package_action=remove`, `package_name=nano:amd64`, `process_comm=dpkg` |
| Ubuntu 26.04 / `test-hermes2` | package update | `source:test-hermes2 AND journal_forwarder:true AND (package_action:upgrade OR package_action:dist-upgrade OR package_action:full-upgrade)` | No package update sample was present in the lab query window. Expected Ubuntu package updates are parsed from apt/dpkg `EXECVE` records. |
| Rocky 8.10 / `test-hermes3` | login/logout | `source:test-hermes3 AND journal_forwarder:true AND auth_service:sshd AND _exists_:auth_session_state` | `auth_service=sshd`, `auth_session_state=closed`, `auth_user=root` |
| Rocky 8.10 / `test-hermes3` | sudo | `source:test-hermes3 AND journal_forwarder:true AND _exists_:sudo_command` | `auth_actor=jfmatrix`, `auth_user=root`, `sudo_command=/usr/bin/id` |
| Rocky 8.10 / `test-hermes3` | su | `source:test-hermes3 AND journal_forwarder:true AND auth_service:su* AND auth_user:jfmatrix` | `auth_service=su-l`, `auth_session_state=opened/closed`, `auth_user=jfmatrix` |
| Rocky 8.10 / `test-hermes3` | package add | `source:test-hermes3 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:install AND package_name:nano*` | `audit_type=SOFTWARE_UPDATE`, `package_action=install`, `package_name=nano-2.9.8-3.el8_10.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 8.10 / `test-hermes3` | package remove | `source:test-hermes3 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:remove AND package_name:nano*` | `audit_type=SOFTWARE_UPDATE`, `package_action=remove`, `package_name=nano-2.9.8-3.el8_10.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 8.10 / `test-hermes3` | package update | `source:test-hermes3 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:update` with a 24h/7d time range | `timestamp=2026-06-05T16:18:09.584Z`, `audit_type=SOFTWARE_UPDATE`, `package_action=update`, `package_name=sudo-1.9.5p2-1.el8_10.5.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 9.7 / `test-hermes4` | login/logout | `source:test-hermes4 AND journal_forwarder:true AND auth_service:sshd AND _exists_:auth_session_state` | `auth_service=sshd`, `auth_session_state=closed`, `auth_user=root` |
| Rocky 9.7 / `test-hermes4` | sudo | `source:test-hermes4 AND journal_forwarder:true AND _exists_:sudo_command` | `auth_actor=jfmatrix`, `auth_user=root`, `sudo_command=/usr/bin/id` |
| Rocky 9.7 / `test-hermes4` | su | `source:test-hermes4 AND journal_forwarder:true AND auth_service:su* AND auth_user:jfmatrix` | `auth_service=su-l`, `auth_session_state=opened/closed`, `auth_user=jfmatrix` |
| Rocky 9.7 / `test-hermes4` | package add | `source:test-hermes4 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:install AND package_name:nano*` | `audit_type=SOFTWARE_UPDATE`, `package_action=install`, `package_name=nano-5.6.1-7.el9.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 9.7 / `test-hermes4` | package remove | `source:test-hermes4 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:remove AND package_name:nano*` | `audit_type=SOFTWARE_UPDATE`, `package_action=remove`, `package_name=nano-5.6.1-7.el9.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 9.7 / `test-hermes4` | package update | `source:test-hermes4 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:update` with a 24h/7d time range | `timestamp=2026-06-05T16:18:28.202Z`, `audit_type=SOFTWARE_UPDATE`, `package_action=update`, `package_name=sudo-1.9.17p2-3.el9_8.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 10.1 / `test-hermes5` | login/logout | `source:test-hermes5 AND journal_forwarder:true AND auth_service:sshd AND _exists_:auth_session_state` | `auth_service=sshd`, `auth_session_state=closed`, `auth_user=root` |
| Rocky 10.1 / `test-hermes5` | sudo | `source:test-hermes5 AND journal_forwarder:true AND _exists_:sudo_command` | `auth_actor=jfmatrix`, `auth_user=root`, `sudo_command=/usr/bin/id` |
| Rocky 10.1 / `test-hermes5` | su | `source:test-hermes5 AND journal_forwarder:true AND auth_service:su* AND auth_user:jfmatrix` | `auth_service=su-l`, `auth_session_state=opened/closed`, `auth_user=jfmatrix` |
| Rocky 10.1 / `test-hermes5` | package add | `source:test-hermes5 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:install AND package_name:nano*` | `audit_type=SOFTWARE_UPDATE`, `package_action=install`, `package_name=nano-8.1-3.el10.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 10.1 / `test-hermes5` | package remove | `source:test-hermes5 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:remove AND package_name:nano*` | `audit_type=SOFTWARE_UPDATE`, `package_action=remove`, `package_name=nano-8.1-3.el10.x86_64`, `package_type=rpm`, `process_comm=dnf`, `result=success` |
| Rocky 10.1 / `test-hermes5` | package update | `source:test-hermes5 AND journal_forwarder:true AND audit_type:SOFTWARE_UPDATE AND package_action:update` | No package update sample was present in the lab query window. The expected Rocky/RHEL shape is `audit_type=SOFTWARE_UPDATE`, `package_action=update`, `package_type=rpm`, `process_comm=dnf`. |

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
git clone https://github.com/joe-speedboat/ansible.journal_forwarder /etc/ansible/roles/joe-speedboat.journal_forwarder
```

## License

GPLv3
Copyright (c) Chris Ruettimann <chris@bitbull.ch>
