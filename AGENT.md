# AGENT.md - Maintainer Guide for `joe-speedboat.journal_forwarder`

This file is for AI agents and human maintainers working on this repository. Keep it accurate when the role behavior changes.

## Role purpose

`joe-speedboat.journal_forwarder` is a Linux-only Ansible role that installs and configures Fluent Bit to forward Linux logs to Graylog over GELF.

The role owns the Linux/Fluent Bit side only. Windows Event Log forwarding is handled by the separate `joe-speedboat.winlogbeat_forwarder` role.

## What the role collects

The role can forward three classes of Linux events:

1. `journald`
   - Source: systemd journal via Fluent Bit `systemd` input.
   - Default tag: `journald`.
   - Adds `log_type=journald`.

2. `auditd`
   - Source: `/var/log/audit/audit.log` via Fluent Bit `tail` input.
   - Default tag: `audit`.
   - Adds `log_type=auditd`.
   - Installs audit package-change watch rules for package manager tooling.

3. `security_file`
   - Source: selected file paths via Fluent Bit `tail` input.
   - Default paths include auth, secure, nginx, Apache, and httpd logs.
   - Default tag: `security_files`.
   - Adds `log_type=security_file`.

All records get:

```text
journal_forwarder=true
```

## Target delivery

The delivery target is a Graylog GELF input.

Default connection variables:

```yaml
graylog_gelf_host: graylog.example.com
graylog_gelf_port: 12201
graylog_gelf_mode: tcp
```

The Graylog side must have a GELF TCP input listening before the role is deployed. The default expected port is:

```text
12201/tcp
```

Fluent Bit output is configured with:

```text
Name                    gelf
Match                   *
Host                    {{ graylog_gelf_host }}
Port                    {{ graylog_gelf_port }}
Mode                    {{ graylog_gelf_mode }}
Gelf_Host_Key           hostname
Gelf_Short_Message_Key  message
Gelf_Level_Key          priority
Gelf_Tag_Key            fb_tag
```

The role sets `hostname ${HOSTNAME}` in Fluent Bit filters and uses `Gelf_Host_Key hostname` so Graylog's `source` field is stable and derived consistently across inputs.

Useful Graylog searches after deployment:

```text
journal_forwarder: true AND log_type: journald
journal_forwarder: true AND log_type: auditd
journal_forwarder: true AND log_type: security_file
journal_forwarder: true AND _exists_: package_action
journal_forwarder: true AND _exists_: sudo_command
```

## Supported target families

The role is intended for systemd-based Linux systems.

Currently documented/tested families:

- Ubuntu 24/26
- Rocky/RHEL/Alma 8-10

The role metadata may list broader Debian/Ubuntu versions. Do not claim support for an OS unless it has been verified on a real target or representative lab VM.

RHEL-family 8 targets need a Python interpreter supported by the controller for fact gathering. The role keeps command-module package handling in `tasks/rhelAll-8/` for target-side compatibility.

## Important variables

### Installation and removal

```yaml
journal_forwarder_install: true
journal_forwarder_remove_repo: true
```

`journal_forwarder_install` gates normal install/configure tasks.

`journal_forwarder_remove_repo` controls repository cleanup during uninstall.

### Graylog GELF

```yaml
graylog_gelf_host: graylog.example.com
graylog_gelf_port: 12201
graylog_gelf_mode: tcp
graylog_fluent_bit_workers: 2
```

### Journald

```yaml
journal_forwarder_tag: journald
journal_forwarder_read_from_tail: true
journal_forwarder_storage_path: /var/lib/fluent-bit
journal_forwarder_systemd_db: "{{ journal_forwarder_storage_path }}/flb_systemd.db"
```

### Audit forwarding

```yaml
journal_forwarder_audit_enabled: true
journal_forwarder_audit_remove: true
journal_forwarder_audit_tag: audit
journal_forwarder_audit_db: "{{ journal_forwarder_storage_path }}/flb_audit.db"
journal_forwarder_audit_rules_file: /etc/audit/rules.d/journal-forwarder-package.rules
journal_forwarder_legacy_audit_rules_files:
  - /etc/audit/rules.d/package.rules
  - /etc/audit/rules.d/logbeat-package.rules
```

Important: `journal_forwarder_audit_remove: true` means uninstall removes the audit package (`audit` on RHEL-family, `auditd` on Debian/Ubuntu-family). If auditd is managed by a baseline/security role, set this to `false` before uninstall.

The role removes only known role-owned audit rule files. It must not delete arbitrary site audit policy such as:

```text
/etc/audit/rules.d/audit.rules
```

### Security/application file logs

```yaml
journal_forwarder_security_files_enabled: true
journal_forwarder_security_files_tag: security_files
journal_forwarder_security_files_db: "{{ journal_forwarder_storage_path }}/flb_security_files.db"
journal_forwarder_security_log_paths:
  - /var/log/auth.log
  - /var/log/secure
  - /var/log/nginx/*.log
  - /var/log/apache2/*.log
  - /var/log/httpd/*.log
```

## Task layout

This role follows the repository OS-dispatcher pattern.

```text
tasks/
  main.yml                 # discovers numeric task files and dispatches by OS
  include-file.yml          # includes OS-specific task files
  uninstall.yml             # uninstall dispatcher
  Ubuntu/
    00_validate.yml
    10_install.yml
    20_configure.yml
    30_service.yml
    uninstall.yml
  rhelAll/
    00_validate.yml
    10_install.yml
    20_configure.yml
    30_service.yml
    uninstall.yml
  rhelAll-8/
    10_install.yml
    uninstall.yml
templates/
  fluent-bit.conf.j2
  parsers-journal-forwarder.conf.j2
  audit-package.rules.j2
```

RHEL 8-specific package workarounds belong in `tasks/rhelAll-8/`. Do not put Python-incompatible module usage into RHEL 8 paths.

## Install behavior

Normal install does the following:

1. Validates Graylog GELF variables.
2. Configures the official Fluent Bit repository.
3. Installs Fluent Bit.
4. Installs audit packages when `journal_forwarder_audit_enabled: true`.
5. Creates Fluent Bit storage/config directories.
6. Deploys parser and Fluent Bit configuration.
7. Removes known legacy role-owned audit rule files.
8. Deploys current audit package-change rules.
9. Starts/enables auditd when audit forwarding is enabled.
10. Loads active audit rules with `augenrules --load`.
11. Starts/enables Fluent Bit.

## Uninstall behavior

Uninstall is invoked with:

```yaml
- ansible.builtin.include_role:
    name: joe-speedboat.journal_forwarder
    tasks_from: uninstall.yml
```

Default uninstall behavior:

1. Stops and disables Fluent Bit.
2. Removes the Fluent Bit package.
3. Removes Fluent Bit configuration and state paths:
   - `/etc/fluent-bit`
   - `/var/lib/fluent-bit`
   - `/opt/fluent-bit`
4. Removes audit packages when both are true:
   - `journal_forwarder_audit_enabled`
   - `journal_forwarder_audit_remove`
5. Removes current and legacy role-owned audit rules:
   - `/etc/audit/rules.d/journal-forwarder-package.rules`
   - `/etc/audit/rules.d/package.rules`
   - `/etc/audit/rules.d/logbeat-package.rules`
6. Removes the Fluent Bit repository when `journal_forwarder_remove_repo: true`.
7. On Ubuntu, removes `/usr/share/keyrings/fluentbit-keyring.gpg` when repository removal is enabled.

## Known pitfalls

### Duplicate audit rules

If `augenrules --load` fails with:

```text
Error sending add rule data request (Rule exists)
There was an error in line 10 of /etc/audit/audit.rules
```

or on Ubuntu line 11, inspect `/etc/audit/rules.d/` for duplicate role-owned package watch files.

A known refactor leftover is:

```text
/etc/audit/rules.d/logbeat-package.rules
```

It may contain the same rules as:

```text
/etc/audit/rules.d/journal-forwarder-package.rules
```

The role should remove known legacy role-owned files before loading audit rules. If another duplicate filename is discovered, add it to `journal_forwarder_legacy_audit_rules_files` only if it is clearly role-owned.

### Audit package removal

Do not assume uninstall should always remove auditd in every environment. Current default is `journal_forwarder_audit_remove: true`, but production baselines may manage auditd independently. For those hosts, set:

```yaml
journal_forwarder_audit_remove: false
```

### Fluent Bit repository setup

Use the official Fluent Bit package repository at `https://packages.fluentbit.io/`.

Ubuntu uses an explicit keyring and apt source list. RHEL-family systems use a yum repository. Keep repository removal in uninstall aligned with install behavior.

### Public examples

Use `example.com` in public docs and examples. Do not commit non-public FQDNs, credentials, inventories, or real Graylog hostnames.

## Verification workflow for changes

At minimum, verify syntax from a harness outside the role checkout:

```bash
ANSIBLE_ROLES_PATH=/path/to/harness/roles \
ansible-playbook -i inventory.ini test.yml --syntax-check

ANSIBLE_ROLES_PATH=/path/to/harness/roles \
ansible-playbook -i inventory.ini test-uninstall.yml --syntax-check
```

For real validation, use disposable lab VMs and test the full lifecycle:

1. Fresh install.
2. Idempotent rerun.
3. If touching audit rules, create/reproduce stale legacy files and verify install cleanup.
4. Verify `augenrules --load` succeeds.
5. Uninstall.
6. Verify role-owned files, packages, repositories, and state are removed according to variables.
7. Reinstall from uninstalled state.
8. Final idempotent rerun.

Representative tested matrix for recent changes:

- Rocky Linux 8.10
- Rocky Linux 9.7
- Rocky Linux 10.1
- Ubuntu 24.04
- Ubuntu 26.04

Record exact commands and recap output in PRs.

## Maintainer checklist before PR

- Read the files before editing them.
- Keep all code, docs, variable names, and commit messages in English.
- Keep variable names under the `journal_forwarder_*`, `graylog_gelf_*`, or `graylog_fluent_bit_*` namespaces.
- Do not reintroduce Linux Fluent Bit logic into Windows roles.
- Run syntax checks.
- For behavior changes, run real OS validation where possible.
- Verify idempotency.
- Verify uninstall when touching install paths.
- Update `README.md`, `CHANGELOG.md`, and this `AGENT.md` when behavior changes.
