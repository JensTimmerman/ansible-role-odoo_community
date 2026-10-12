# Changelog

All notable changes to this role are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed

- Relicensed the role to **GPL-3.0-or-later** (the previous permissive
  license no longer applies): replaced `LICENSE` with the full GNU GPL v3
  text, set `galaxy_info.license` to `GPL-3.0-or-later` in `meta/main.yml`,
  and updated the README license section (noting that the role installs Odoo
  Community, which remains under its own upstream LGPLv3 terms).

## [1.0.0] - 2026-10-11

### Added

- Generic `odoo` role extracted from the in-repo Odoo 19 role: PostgreSQL
  provisioning (dedicated non-superuser role + database + unaccent,
  loopback-only listener), the Odoo Community runtime (Python virtualenv,
  unprivileged hardened systemd service, templated `odoo.conf`), optional
  Peppol e-invoicing (`account_peppol`) and optional daily DB + filestore
  backups via systemd timer.
- RL9-first variable layout (`vars/RedHat.yml`) with an optional Debian /
  Ubuntu path (`vars/Debian.yml`), including PGDG repository bootstrap and
  `wkhtmltopdf` installation on both families.
- Configurable PostgreSQL auth: loopback `trust` by default, `scram-sha-256`
  when `odoo_db_password` is set.
- Role argument specs (`meta/argument_specs.yml`), molecule default scenario,
  `tests/` smoke playbook, yamllint / ansible-lint configuration and a GitHub
  Actions CI workflow.
- README with required variables, quick start and operator runbook.

[1.0.0]: https://github.com/JensTimmerman/ansible-role-odoo_community/releases/tag/v1.0.0
