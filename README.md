# ansible-role-mariadb

Ansible role that installs and hardens MariaDB on RHEL 8, 9 and 10 (and compatible: AlmaLinux, Rocky, Oracle Linux), using only the packages in the distribution's AppStream repository.

Features:

- MariaDB 10.11 from AppStream: the `mariadb:10.11` module stream on EL8 and EL9, the stock packages on EL10. MariaDB.org repositories are never used.
- Hardening based on the DISA MariaDB Enterprise 10.x STIG: audit logging, password complexity, account lockout, idle session timeouts, no `LOAD DATA LOCAL`, no symbolic links, a confined `secure_file_priv`, restrictive file modes, and removal of anonymous users, remote root and the test database.
- TLS with a supplied certificate or a self-signed CA and server certificate the role creates. TLS 1.2+ only, and plain-text TCP is refused.
- Data-at-rest encryption with the `file_key_management` plugin, covering InnoDB tables, redo log, temporary files, binary logs and Aria tables. The key file can itself be encrypted, and keys can be rotated.
- Databases and users, with TLS required for remote accounts.
- SELinux labeling for relocated directories and ports, a fapolicyd trust file for any non-RPM files, and FIPS checks.
- Molecule tests with podman on EL8, EL9 and EL10, plus a key rotation scenario.

phpMyAdmin and Apache live in their own roles: [ansible-role-phpmyadmin](https://github.com/jmaughmer/ansible-role-phpmyadmin) and [ansible-role-apache](https://github.com/jmaughmer/ansible-role-apache).

## Requirements

- Ansible core 2.15 or newer. RHEL 8 targets need ansible-core 2.16 or older on the controller, because 2.17+ no longer supports EL8's Python 3.6, which is where the dnf bindings live.
- Collections from `requirements.yml`: `ansible.posix`, `community.general`, `community.mysql` and `community.crypto`.
- Managed hosts need the BaseOS and AppStream repositories (or a Satellite/Katello mirror of them).

```bash
ansible-galaxy collection install -r requirements.yml
```

## Quick start

```yaml
- hosts: db_servers
  become: true
  roles:
    - role: ansible-role-mariadb
      vars:
        mariadb_bind_address: 0.0.0.0
        mariadb_firewalld_sources: [10.0.0.0/24]
        mariadb_encryption_enabled: true
        mariadb_databases:
          - name: kea
        mariadb_users:
          - name: kea
            host: "10.0.0.%"
            password: "{{ vault_kea_db_password }}"
            priv: "kea.*:ALL"
```

`root@localhost` logs in with `unix_socket` only, so it has no password and only the OS root user can use it. The role connects that way for every SQL task. Use `sudo mariadb` for administration.

## Package source

| Release | Source | Default version |
|---|---|---|
| EL8 | AppStream module `mariadb:10.11` | 10.11 |
| EL9 | AppStream module `mariadb:10.11` | 10.11 |
| EL10 | AppStream (non-modular) | 10.11 |

Set `mariadb_module_stream` to pick another stream on EL8/EL9 (10.5 is the minimum the role supports). The role refuses to switch the stream of a host that already has one enabled: that is a major upgrade, so dump the databases, run `dnf module switch-to mariadb:<stream>`, then `mariadb-upgrade`.

## STIG hardening

`mariadb_stig_enabled: true` (the default) turns on the settings below. Each has its own variable, so you can relax one without turning the others off.

| Setting | Variable | Default |
|---|---|---|
| `server_audit` plugin, `FORCE_PLUS_PERMANENT`, logs CONNECT, DDL and DCL | `mariadb_audit_*` | `/var/log/mariadb/server_audit.log`, 30 x 100 MB |
| `simple_password_check` | `mariadb_password_minimal_*` | 15 chars, 1 digit, 1 upper and 1 lower case, 1 special |
| Lock account after failed logins | `mariadb_max_password_errors` | 3 (`FLUSH PRIVILEGES` unlocks) |
| Password lifetime | `mariadb_default_password_lifetime` | 0 (the STIG asks for 60 on interactive accounts) |
| Idle session timeout | `mariadb_wait_timeout` | 600 s |
| `local_infile` | `mariadb_local_infile` | off |
| `symbolic-links` | `mariadb_symbolic_links` | off |
| `secure_file_priv` | `mariadb_secure_file_priv` | `/var/lib/mysql-files` |
| TLS 1.2+, `require_secure_transport` | `mariadb_tls_*` | on |
| Data directory / log directory modes | `mariadb_datadir_mode`, `mariadb_log_dir_mode` | `0750` |
| Remove anonymous users, remote root, `test` | `mariadb_remove_*` | on |

Passwords you set through `mariadb_users` have to meet the complexity rules, or MariaDB rejects them.

The STIG also has organisational items no role can settle for you (backups, who reviews the audit log, account reviews). Send the audit log to your SIEM with `mariadb_audit_output_type: syslog`, or collect the file.

## TLS

```yaml
# Self-signed (default): a private CA plus a server certificate, RSA 3072 / SHA-256.
mariadb_tls_source: selfsigned
mariadb_tls_selfsigned_subject_alt_names:
  - DNS:db01.corp.example.com
  - IP:10.0.0.20

# Supplied: PEM content (from Vault) or files on the controller.
mariadb_tls_source: provided
mariadb_tls_cert_content: "{{ vault_db_cert }}"
mariadb_tls_key_content: "{{ vault_db_key }}"
mariadb_tls_ca_src: files/corp-ca.pem
```

Files go to `/etc/pki/mariadb` (`server.crt`, `server.key`, `ca.crt`). The self-signed server certificate is re-issued when it is within `mariadb_tls_selfsigned_renew_days` (30) of expiry. Clients verify the server with `ca.crt`:

```bash
mariadb -h db01 -u kea -p --ssl --ssl-ca=/etc/pki/mariadb/ca.crt --ssl-verify-server-cert
```

With `require_secure_transport` on, every TCP connection must use TLS; local socket connections are not affected. Accounts in `mariadb_users` whose host is not `localhost` get `REQUIRE SSL` unless the entry sets `tls_requires` (use `X509` to demand client certificates).

## Data-at-rest encryption

```yaml
mariadb_encryption_enabled: true
```

That loads `file_key_management`, encrypts InnoDB tables and redo log, temporary files and tables, binary logs and Aria tables, and generates a 256-bit key on the host. The key file is encrypted with a separate file key (`mariadb_encryption_keyfile_encrypted: true`), so `/etc/mysql/encryption` holds `keyfile.enc` and `keyfile.key`, both `root:mysql 0440`. This keeps keys out of backups of the data directory; anyone with root on the host can still read them, which is the limit of file-based key management.

To supply keys yourself, typically from Ansible Vault:

```yaml
mariadb_encryption_keys:
  - { id: 1, key: "{{ vault_mariadb_key_1 }}" }   # openssl rand -hex 32
  - { id: 2, key: "{{ vault_mariadb_key_2 }}" }
```

### Rotating keys

`file_key_management` has no key versions, so rotation means adding a key with a new id and moving tables to it.

- Generated keys: run once with `-e mariadb_encryption_rotate=true`. The role adds the next key id, makes it the default for new tables and restarts MariaDB.
- Supplied keys: add a new entry to `mariadb_encryption_keys`. The highest id becomes active, or set `mariadb_encryption_key_id`.

After the restart the role finds InnoDB tables whose tablespace still uses an older key (from `information_schema.INNODB_TABLESPACES_ENCRYPTION`) and runs `ALTER TABLE ... ENCRYPTED=YES ENCRYPTION_KEY_ID=<new>` on each. That rebuilds the table, so plan rotations for a quiet period on large databases, or set `mariadb_encryption_reencrypt: false` and do it yourself. Schemas in `mariadb_encryption_reencrypt_exclude_schemas` (`mysql` and the virtual schemas) are skipped.

Keep old keys. Key 1 always protects the system tablespace, redo log and binary logs (MariaDB uses key 2, when it exists, for temporary files), and older backups still need the key they were written with. The role warns if supplied keys drop an id the current key file has.

## SELinux

With SELinux enforcing, the role labels a non-default port as `mysqld_port_t`, relocated data and `secure_file_priv` directories as `mysqld_db_t`, a relocated log directory as `mysqld_log_t`, and the key directory as `mysqld_etc_t`, then runs `restorecon`. Extra rules go in `mariadb_selinux_fcontexts`, and booleans in `mariadb_selinux_booleans`.

## fapolicyd

MariaDB, its plugins and everything else the role installs come from RPMs, which fapolicyd trusts. If you add files outside RPMs (a custom plugin, a backup script), list them in `mariadb_fapolicyd_trusted_files`. The role writes their path, size and SHA-256 to `/etc/fapolicyd/trust.d/mariadb` and refreshes fapolicyd.

## FIPS

The role works on hosts in FIPS mode: TLS follows the system crypto policy (leave `mariadb_tls_cipher` empty), self-signed certificates use RSA 3072 and SHA-256, and encryption uses AES with 256-bit keys.

- `mariadb_fips_required: true` fails the run when the host is not in FIPS mode. The role does not turn FIPS on, since that needs `fips-mode-setup --enable` (EL8/EL9) or a FIPS install (EL10), and a reboot.
- `mariadb_crypto_policy` sets the system crypto policy (`FIPS`, `FIPS:OSPP`...).
- On FIPS hosts the role insists on 64 hex character (256-bit) encryption keys.
- Don't use the `ed25519` authentication plugin on FIPS hosts; it does not use the validated OpenSSL module.

## Firewall

When firewalld is running and `mariadb_bind_address` is not loopback, the role opens `mariadb_port` in `mariadb_firewalld_zone`, or only for `mariadb_firewalld_sources` through rich rules.

## Variables

See [`defaults/main.yml`](defaults/main.yml) for every variable with comments. The main ones:

| Variable | Default | Purpose |
|---|---|---|
| `mariadb_module_stream` | `""` (10.11) | AppStream module stream on EL8/EL9 |
| `mariadb_bind_address` / `mariadb_port` | `127.0.0.1` / `3306` | Listener |
| `mariadb_datadir` | `/var/lib/mysql` | Data directory |
| `mariadb_databases` / `mariadb_users` | `[]` | Databases and accounts |
| `mariadb_mysqld_options` | `{}` | Extra `[mysqld]` settings |
| `mariadb_stig_enabled` | `true` | STIG settings |
| `mariadb_tls_enabled` / `mariadb_tls_source` | `true` / `selfsigned` | TLS |
| `mariadb_encryption_enabled` | `false` | Data-at-rest encryption |
| `mariadb_encryption_rotate` | `false` | One-shot key rotation |
| `mariadb_fips_required` | `false` | Fail unless FIPS is on |

Tags: `mariadb_install`, `mariadb_tls`, `mariadb_encryption`, `mariadb_config`, `mariadb_security`, `mariadb_selinux`, `mariadb_fapolicyd`, `mariadb_fips`, `mariadb_firewall`, `mariadb_service`, `mariadb_secure`, `mariadb_databases`.

## Testing

Tests run locally with podman; there is no CI. On the AlmaLinux 10 WSL distribution (or any EL10 host):

```bash
scripts/test.sh            # lint, default (EL9 + EL10), rotate, el8
scripts/test.sh lint default
```

The script creates two virtualenvs under `.cache/test`: one with current ansible-core, and one with ansible-core 2.16 for the EL8 scenario. The repository directory must be named `ansible-role-mariadb` so molecule can find the role. If rootless systemd containers don't work on your WSL setup, run it with `sudo`.

Containers can't exercise SELinux, FIPS or firewalld, so those paths are covered by lint and need a test on a real host.

## License

MIT
