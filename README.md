# ansible-role-odoo_community

> **Disclaimer:** This role is an independent, unofficial, community-maintained
> project. It is **not affiliated with, endorsed by, or sponsored by Odoo S.A.**
> or any of its subsidiaries. "Odoo" is a registered trademark of Odoo S.A. The
> role installs **Odoo Community**, the open-source edition released under the
> GNU LGPLv3 (or later); it does not install, configure, or license Odoo
> Enterprise.

Generic Ansible role that installs and configures an
[Odoo](https://www.odoo.com/) Community server on RHEL/Rocky Linux 9
(or Debian/Ubuntu) without Docker:

- **PostgreSQL** (PGDG) with a dedicated non-superuser role + database and the
  `unaccent` extension, listening on loopback only.
- **Odoo runtime**: source checkout from `odoo/odoo`, a Python virtualenv, an
  unprivileged system user and a hardened systemd service.
- **Peppol e-invoicing** (optional): installs `account_peppol` +
  `account_edi_ubl_cii` and applies the accounting defaults (self-billing sale
  journal, purchase journal, `web.base.url`, polling interval).
- **Backups** (optional): a daily database + filestore dump on a systemd timer
  with local retention.

The role is site-agnostic: no inventory, domain, vault variable name or
environment coupling is baked in. Every secret is supplied by the caller.

## Requirements

- Ansible `>= 2.11` (`argument_specs` support).
- A target host on the EL family (RHEL/Rocky/AlmaLinux 9) or Debian/Ubuntu.
- The PostgreSQL role the role delegates to, installed as a dependency, e.g.:

  ```yaml
  # requirements.yml
  roles:
    - name: geerlingguy.postgresql
      src: geerlingguy.postgresql
  ```

  and point the role at it with `odoo_postgresql_role_name`. The default is
  `ansible-role-postgresql` (a thin rename of the same role).
- `become: true` (the role installs packages, creates users and manages
  systemd units).
- The collections `ansible.builtin`, `ansible.posix` and
  `community.postgresql`.

## Role Variables

The full, validated contract lives in `meta/argument_specs.yml`; defaults are
in `defaults/main.yml`. The most important ones:

| Variable | Default | Description |
| --- | --- | --- |
| `odoo_version` | `"19.0"` | Odoo major version line. |
| `odoo_repo_url` | `https://github.com/odoo/odoo` | Source remote. |
| `odoo_repo_version` | `"{{ odoo_version }}"` | Git ref checked out. |
| `odoo_base_dir` | `/opt/odoo` | Runtime base dir (home + source + venv). |
| `odoo_user` / `odoo_group` | `odoo` / `odoo` | Unprivileged owner. |
| `odoo_http_port` / `odoo_gevent_port` | `8069` / `8072` | Odoo listeners. |
| `odoo_bind_interface` | `127.0.0.1` | Interface the listeners bind to. |
| `odoo_workers` / `odoo_max_cron_threads` | `2` / `1` | Worker pool sizing. |
| `odoo_proxy_mode` / `odoo_list_db` | `true` / `false` | `odoo.conf` flags. |
| `odoo_base_url` | `""` | Published `web.base.url`; empty leaves Odoo's default. |
| `odoo_db_name` / `odoo_db_user` | `odoo` / `odoo` | Database + role. |
| `odoo_db_host` / `odoo_db_port` | `localhost` / `5432` | Loopback DB. |
| `odoo_db_password` | `""` | Empty = loopback trust; set for scram-sha-256. |
| `odoo_postgresql_version` | `"16"` | PGDG major version. |
| `odoo_python_version` | `"3.12"` | Virtualenv interpreter. |
| `odoo_config_admin_passwd` | `""` | **Required** when the runtime is managed. |
| `odoo_manage_postgresql` | `true` | Toggle the PostgreSQL phase. |
| `odoo_manage_runtime` | `true` | Toggle the runtime phase. |
| `odoo_manage_peppol` | `true` | Toggle the Peppol phase. |
| `odoo_manage_backups` | `true` | Toggle the backup phase. |
| `odoo_run_verify` | `true` | Run read-only verification tasks. |

### Required variables

- `odoo_config_admin_passwd` — the Odoo master password (`admin_passwd`).
  Supply it from a vaulted source; the role never ships a plaintext value and
  fails loudly when it is empty (only when `odoo_manage_runtime` is enabled).

### Secrets

- `odoo_config_admin_passwd` (**required**, supply from a vault).
- `odoo_db_password` (optional; only if you deliberately move off loopback
  trust auth).

No secret is stored in this repository.

## Quick start

```yaml
# playbook.yml
- name: "Provision Odoo"
  hosts: "odoo"
  become: true
  vars:
    # Supply the master password from a vaulted source, e.g. ansible-vault
    # encrypted group_vars, an external secret store, or --extra-vars. Never
    # commit a plaintext value.
    odoo_config_admin_passwd: "{{ vaulted_odoo_admin_passwd }}"
    odoo_base_url: "https://odoo.example.com"
  roles:
    - role: "jenstimmerman.odoo_community"
```

```yaml
# requirements.yml
roles:
  - name: geerlingguy.postgresql
  - name: jenstimmerman.odoo_community
    src: https://github.com/JensTimmerman/ansible-role-odoo_community
```

```console
$ ansible-galaxy role install -r requirements.yml
$ ansible-playbook playbook.yml
```

Reverse-proxy and firewall configuration are intentionally out of scope;
point your proxy at `http://127.0.0.1:{{ odoo_http_port }}`.

## Example Playbook (contract only)

`tests/test.yml` runs the role with the mutating phases disabled so it is safe
on a CI runner:

```console
$ ansible-playbook -i tests/inventory tests/test.yml --syntax-check
$ ansible-playbook -i tests/inventory tests/test.yml
```

## Molecule

A container-free `default` scenario (the delegated driver running on
`localhost`) validates the variable contract and the read-only checks:

```console
$ molecule test
```

Converge and verify disable all `odoo_manage_*` phases, so the scenario does
not install packages or start services.

## Operator runbook

- **Peppol registration**: registering the company as a Peppol participant
  requires KYC and is an operator action (Accounting → Configuration →
  Settings → Peppol Electronic Invoicing, using the correct EAS scheme and
  identifier for your jurisdiction). The role only installs the modules and
  creates the journals.
- **Offsite copies**: local dumps live under `odoo_backup_dir`. Copying them
  off-site is manual. Set `odoo_offsite_target` and print the reminder with:

  ```console
  $ ansible-playbook playbook.yml --tags odoo-offsite-reminder
  ```

## Publishing to Ansible Galaxy

This is a **role** (not a collection), so metadata lives in `meta/main.yml`
under `galaxy_info` — there is no `galaxy.yml`.

Owner actions (require credentials not available in the build environment):

1. The GitHub repository `JensTimmerman/ansible-role-odoo_community` carries
   `main` (bootstrap) plus the `v1.0.0` tag.
2. Import it into Galaxy:
   `ansible-galaxy role import jenstimmerman ansible-role-odoo_community`
   (or via the Galaxy web UI). Galaxy reads `meta/main.yml` for
   `role_name: odoo_community` and creates `jenstimmerman.odoo_community`.

## License

This role is licensed under the **GNU General Public License v3.0 or later**
(`GPL-3.0-or-later`) — see [LICENSE](LICENSE).

The role installs [Odoo Community](https://www.odoo.com/), which is a separate
work distributed by Odoo S.A. under the **GNU LGPLv3 (or later)**. The two
licenses are independent and compatible: this role's own source is
GPL-3.0-or-later, while Odoo Community remains under its own upstream LGPLv3
terms. This project is not affiliated with, endorsed by, or sponsored by
Odoo S.A.
