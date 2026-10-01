# ansible-role-keycloak

Installs and configures [Keycloak](https://www.keycloak.org/) (Quarkus distribution) from the
official release tarball, running as a systemd service under a dedicated non-root user.

Supports RHEL/CentOS/Rocky/Alma (8/9) and Debian/Ubuntu.

## What it does

- Installs a JDK (OpenJDK 17 by default)
- Creates a dedicated `keycloak` system user/group
- Downloads and extracts the requested Keycloak release to `/opt/keycloak-<version>`,
  symlinked as `/opt/keycloak`
- Renders `conf/keycloak.conf` (database, hostname, HTTP/HTTPS, logging)
- Runs `kc.sh build` to produce an optimized server image (optional)
- Installs and manages a hardened systemd unit
- Optionally opens firewalld ports

## Requirements

- Ansible >= 2.14
- Target host with internet access to `github.com` (or point `keycloak_download_url` at an
  internal mirror)
- A pre-provisioned database (PostgreSQL/MySQL/MariaDB/MSSQL/Oracle) if not using the
  bundled dev database

## Role Variables

See `defaults/main.yml` for the full list with defaults. Key ones:

| Variable | Default | Description |
|---|---|---|
| `keycloak_version` | `26.0.7` | Keycloak release to install |
| `keycloak_start_mode` | `start` | `start` (production) or `start-dev` |
| `keycloak_db` | `dev-file` | `dev-file`, `postgres`, `mysql`, `mariadb`, `mssql`, `oracle` |
| `keycloak_db_url_host` / `_port` / `_database` / `_username` / `_password` | — | DB connection settings, used when `keycloak_db != dev-file` |
| `keycloak_hostname` | `ansible_fqdn` | Hostname Keycloak advertises to clients |
| `keycloak_http_port` / `keycloak_https_port` | `8080` / `8443` | Listener ports |
| `keycloak_https_enabled` | `false` | Enable TLS in Keycloak itself (else terminate TLS at a proxy) |
| `keycloak_admin_user` / `keycloak_admin_password` | `admin` / `changeme` | Bootstrap admin account, only applied on first start |
| `keycloak_java_opts` | `-Xms512m -Xmx1024m` | JVM heap flags |
| `keycloak_extra_conf_lines` | `[]` | Raw lines appended to `keycloak.conf` |
| `keycloak_extra_environment` | `{}` | Extra env vars for the systemd unit |

**Always override `keycloak_admin_password` and `keycloak_db_password`** via Ansible Vault or
`group_vars` — never leave the defaults in place for a real deployment.

## Example Playbook

```yaml
- hosts: keycloak_servers
  become: true
  roles:
    - role: ansible-role-keycloak
      vars:
        keycloak_version: "26.0.7"
        keycloak_start_mode: start
        keycloak_hostname: sso.example.com
        keycloak_db: postgres
        keycloak_db_url_host: db.example.internal
        keycloak_db_url_database: keycloak
        keycloak_db_username: keycloak
        keycloak_db_password: "{{ vault_keycloak_db_password }}"
        keycloak_admin_user: admin
        keycloak_admin_password: "{{ vault_keycloak_admin_password }}"
        keycloak_proxy_headers: xforwarded
        keycloak_https_enabled: false   # TLS terminated at a front proxy
```

## Behind a reverse proxy

Set `keycloak_proxy_headers: xforwarded` (or `forwarded`) and keep `keycloak_https_enabled:
false`; terminate TLS at your load balancer/reverse proxy and forward to
`keycloak_http_port` over the internal network.

## License

MIT
