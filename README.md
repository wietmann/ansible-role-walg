Ansible Role: WAL-G
===================

Ansible role to setup [WAL-G](https://github.com/wal-g/wal-g) PostgreSQL backup tool.

This role does not handle PostgreSQL installation. You may want to use this role in combination with another role to install PostgreSQL. See **Example Playbook** section for primer of variables substitution from [geerlingguy.postgresql](https://galaxy.ansible.com/geerlingguy/postgresql). PostgreSQL does not need to be installed on target host as long as you disable all PostgreSQL configuration tasks (see **Role Variables** section). 

WAL-G have changed release binaries name scheme since 1.0, which is for now unsupported by this role. You can force installation by setting `walg_binary_name` and `walg_download_url` to tarball URL and exact file name.

Requirements
------------

Python, Ansible >= 2.9

Role Variables
--------------

|Variable|Deafule value|Description|
|-|-|-|
|`walg_version`|`0.2.19`|WAL-G version to download|
|`walg_binary_dir`|`/usr/local/bin`|Path to directory containing WAL-G binary|
|`walg_conf`|-|Dictionary containing WAL-G configuration parameters. See [WAL-G documentation](https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md) for available parameters|
|`walg_configure_logrotate`|`True`|Whether to configure WAL-G logs rotation. Logrotate should be installed|
|`walg_logs_dir`|`/var/log/wal-g`|Path to logs directory|
|`walg_user`|`postgres`|WAL-G owner (should be same with Postgres process owner)|
|`walg_group`|`postgres`|WAL-G group (should be same with Postgres process group)|
|`walg_pg_data_dir`|distro-specific|Path to PostgreSQL PGDATA directory|
|`walg_configure_pg_settings`|`True`|Wheteher to apply settings to PostreSQL global configuration (need restarting)|
|`walg_wal_archive_timeout`|`3600`|Value of `wal_archive_timeout` PostgreSQL configuration option|
|`walg_pg_config_path`|distro-specific|Path to directory containing PostgreSQL|
|`walg_pg_service`|distro-specific|PostgreSQL service name in distro init system|
|`walg_pg_global_config_options`|-|List of dictionaries containing global PostgreSQL configuration options|
|`walg_pg_global_config_options[*].option`|-|Configuration option name|
|`walg_pg_global_config_options[*].value`|-|Configuration option value|
|`walg_configure_crontab`|`True`|Whether to configure Cron jobs. Cron should be installed|
|`walg_backup_cmd`|`{{ walg_binary_dir }}/{{ walg_binary_name }} backup-push {{ walg_pg_data_dir }} >> {{ walg_logs_dir }}/backup-push.log 2>&1`|Cron backup command|
|`walg_prune_cmd`|`{{ walg_binary_dir }}/{{ walg_binary_name }} delete retain FULL 12 >> {{ walg_logs_dir }}/delete.log 2>&1`|Cron prune command|
|`walg_backup_cron_minute`|`30`|Cron backup job minutes field|
|`walg_backup_cron_hour`|`0`|Cron backup job hour field|
|`walg_backup_cron_day`|`*`|Cron backup job day field|
|`walg_backup_cron_month`|`*`|Cron backup job month field|
|`walg_backup_cron_weekday`|`*`|Cron backup job weekday field|
|`walg_prune_cron_minute`|`0`|Cron prune job minutes field|
|`walg_prune_cron_hour`|`6`|Cron prune job hour field|
|`walg_prune_cron_day`|`*`|Cron prune job day field|
|`walg_prune_cron_month`|`*`|Cron prune job month field|
|`walg_prune_cron_weekday`|`*`|Cron prune job weekday field|
|`walg_cron_jobs`|`x`|List of dictionaries describing custom Cron jobs to configure (overrides `walg_*_cron_*` values)|
|`walg_cron_jobs[*].name`|-|Cron job name|
|`walg_cron_jobs[*].minute`|`*`|Cron job minutes field|
|`walg_cron_jobs[*].day`|`*`|Cron job day field|
|`walg_cron_jobs[*].month`|`*`|Cron job month field|
|`walg_cron_jobs[*].weekday`|`*`|Cron job weekday field|
|`walg_cron_jobs[*].job`|`*`|Cron job command|

Dependencies
------------

None.

Supported platforms
-------------------

* RHEL 7, 8

**NB1:** RHEL 7 official repo contains PostgreSQL 9.2, which is _quite_ old. I don't ever know if WAL-G supports it, at least Molecule playbook failed to restart Potgres due to unsupported `wal_level` value. Upstream PG from PGDG repo works fine.

Example Playbook
----------------

Using S3 compatible storage, PGP encryption and [geerlingguy.postgresql](https://galaxy.ansible.com/geerlingguy/postgresql) to setup PostgreSQL itself:

```yaml
---
- name: Setup PostgreSQL and WAL-G
  hosts: all
  become: yes

  vars:
    walg_user: "{{ postgresql_user }}"
    walg_group: "{{ postgresql_group }}"
    walg_pg_data_dir: "{{ postgresql_data_dir }}"
    walg_pg_config_path: "{{ postgresql_config_path }}"
    walg_pg_service: "{{ postgresql_daemon }}"
    walg_conf:
      PGDATA: "{{ walg_pg_data_dir }}"
      PGHOST: "/var/run/postgresql"
      WALG_S3_PREFIX: "s3://some_bucket"
      AWS_ACCESS_KEY_ID: "VerySecretAccessKey"
      AWS_SECRET_ACCESS_KEY: "VerySecretSecret"
      WALG_COMPRESSION_METHOD: "brotli"
      WALG_DELTA_MAX_STEPS: "6"
      WALG_PGP_KEY_PATH: "{{ walg_postgres_home_dir }}/walg.pgp.public.key"

  roles:
    - role: geerlingguy.postgresql
    - role: wietmann.walg

```

Testing
-------

This role uses [Molecule](https://molecule.readthedocs.io/en/stable/) for testing. Molecule is run against Docker containers with enabled systemd support (see `default` scenario). Docker Engine, `docker` library and `molecule-docker` driver Python packages should be installed on controller host.

[geerlingguy.postgresql](https://galaxy.ansible.com/geerlingguy/postgresql) role is used to install PostgreSQL.

There are two scenarios:
* `default` - uses PostgreSQL from distro repository
* `pgdg` - uses PostgreSQL from PGDG repository

Preferred way to test role for compatibility with different versions of Ansible is to use [Tox](https://tox.wiki/en/stable/). See `tox.ini` for the full list of supported Ansible versions.

Run tests for all environments:
```
tox
```

List available environments:
```
tox -l
```

Run tests only for specific environment (e.g. Ansible 2.9):
```
tox -e py3-ansible29
```

Run only specific Molecule subcommand (e.g. `molecule verify`):

```
# All test environments
tox -- molecule verify
# Specific test environments
tox -e py3-ansible29 -- molecule verify
```

License
-------

SPDX:MIT  
See the full text in [LICENSE](LICENSE) file.

Author Information
------------------

This role was created in 2021 by Dmitry Danilov.
