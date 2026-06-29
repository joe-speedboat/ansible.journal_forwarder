# Changelog

All notable changes to `joe-speedboat.journal_forwarder` are documented in this file.

## v1.0.1 - 2026-06-29

### Added

- Added Linux audit package-change forwarding with role-managed audit rules for RPM/DNF/YUM and DPKG/APT tooling.
- Added optional forwarding for selected Linux security and application log files, including auth, secure, nginx, Apache, and httpd log paths.
- Added OS-specific install, configure, service, and uninstall task paths for:
  - Ubuntu/Debian-family systems.
  - RHEL-family systems.
  - RHEL 8 compatible systems using command-module package handling where required.
- Added `tasks/uninstall.yml` so consumers can remove the role footprint through `include_role` with `tasks_from: uninstall.yml`.
- Added cleanup for role-owned legacy audit rule files left by earlier/refactored versions:
  - `/etc/audit/rules.d/package.rules`
  - `/etc/audit/rules.d/logbeat-package.rules`
- Added `journal_forwarder_legacy_audit_rules_files` to make known legacy role-owned audit rule cleanup explicit and reviewable.
- Added README installation guidance for Ansible Galaxy and direct GitHub checkout usage.

### Changed

- Split Linux Fluent Bit forwarding into this role as the Linux-only companion to the Windows Winlogbeat forwarding role.
- Reworked the role layout to use explicit OS-specific task files:
  - `00_validate.yml`
  - `10_install.yml`
  - `20_configure.yml`
  - `30_service.yml`
  - `uninstall.yml`
- Updated Fluent Bit configuration to send all records to Graylog through the GELF output.
- Standardized Graylog record metadata:
  - `journal_forwarder=true`
  - `log_type=journald`
  - `log_type=auditd`
  - `log_type=security_file`
- Standardized GELF `source` handling by adding `hostname ${HOSTNAME}` in Fluent Bit filters and using `Gelf_Host_Key hostname`.
- Updated README search examples and role documentation.
- Updated README Windows forwarding reference to point to `joe-speedboat.winlogbeat_forwarder`; this role remains Linux-only.

### Fixed

- Fixed `augenrules --load` failures caused by duplicate package-change watch rules from stale legacy role files.
- Fixed uninstall cleanup so current and legacy role-owned audit rule files are removed.
- Fixed uninstall cleanup for Fluent Bit repository/keyring/config/state artifacts.
- Fixed Ubuntu Fluent Bit repository setup to use the official Fluent Bit apt repository shell flow with an explicit keyring and source list.
- Fixed RHEL-family repository setup to use package-manager repositories rather than old shared setup paths.

### Operational notes

- By default, uninstall removes audit packages when `journal_forwarder_audit_remove: true`.
- If auditd is managed by another baseline role, set `journal_forwarder_audit_remove: false` before running uninstall.
- The role intentionally removes only known role-owned audit rule files; it does not remove arbitrary site audit policy such as `/etc/audit/rules.d/audit.rules`.

### Verified

Validated in a lab on:

- Rocky Linux 8.10
- Rocky Linux 9.7
- Rocky Linux 10.1
- Ubuntu 24.04
- Ubuntu 26.04

Validation covered:

- Fresh install.
- Idempotent rerun.
- Stale `logbeat-package.rules` cleanup before `augenrules --load`.
- Uninstall cleanup of current and legacy audit rule files.
- Uninstall cleanup of Fluent Bit config/state/repository/keyring artifacts.
- Reinstall from an uninstalled state.
- Final idempotent rerun.
