# healthchecks

Ansible role to deploy private hosted [healthchecks.io](https://healthchecks.io/) — open source http push monitoring.

Installs the app from source, runs it under uwsgi (with background workers), and manages it via systemd. Supports SQLite (default) and PostgreSQL.

This role is made for Bare Metal/VM or LXC deployments for having low depenencies and transaprent IP Stack (IPv6 aswell). 

## Requirements

- Debian bookworm/trixie or Ubuntu jammy/noble
- `community.general` collection
- For PostgreSQL: pre-installed PostgreSQL (e.g. via `geerlingguy.postgresql`) and `community.postgresql` collection

## Role Variables

```yaml
# Source
healthchecks_version: "master"          # git branch/tag
healthchecks_repo: "https://github.com/healthchecks/healthchecks.git"
healthchecks_install_dir: "/opt/healthchecks"

# Database — "sqlite" or "postgres"
healthchecks_db: "sqlite"

# PostgreSQL (only when healthchecks_db: postgres)
healthchecks_db_name: "hc"
healthchecks_db_user: "hc"
healthchecks_db_password: ""            # use ansible-vault
healthchecks_db_host: "localhost"
healthchecks_db_port: 5432

# Application (required)
healthchecks_secret_key: ""             # use ansible-vault

# Application (site config)
healthchecks_allowed_hosts: "{{ inventory_hostname }}"
healthchecks_site_root: "https://{{ inventory_hostname }}"
healthchecks_site_name: "Healthchecks"
healthchecks_default_from_email: "healthchecks@{{ inventory_hostname }}"
healthchecks_ping_endpoint: "https://{{ inventory_hostname }}/ping/"
healthchecks_ping_email_domain: "{{ inventory_hostname }}"
healthchecks_registration_open: false

# Email — defaults to local postfix/relay on port 25
healthchecks_email_host: "localhost"
healthchecks_email_port: 25
healthchecks_email_use_tls: false
healthchecks_email_host_user: ""
healthchecks_email_host_password: ""
```

## Example Playbook

```yaml
- hosts: hc.example.com
  become: true
  roles:
    - role: healthchecks
      vars:
        healthchecks_secret_key: "{{ vault_hc_secret_key }}"
        healthchecks_allowed_hosts: "hc.example.com"
        healthchecks_site_root: "https://hc.example.com"
        healthchecks_default_from_email: "healthchecks@example.com"
```

## PostgreSQL Example

```yaml
- hosts: hc.example.com
  become: true
  roles:
    - role: geerlingguy.postgresql
      vars:
        postgresql_databases:
          - name: hc
        postgresql_users:
          - name: hc
            password: "{{ vault_hc_db_password }}"
            db: hc
            priv: ALL
    - role: healthchecks
      vars:
        healthchecks_db: postgres
        healthchecks_db_password: "{{ vault_hc_db_password }}"
        healthchecks_secret_key: "{{ vault_hc_secret_key }}"
```

## Reverse Proxy

The app listens on port 8000. Put nginx/caddy in front of it.

Example nginx snippet:
```nginx
location / {
    proxy_pass http://{{ hc_container_ip }}:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## License

MIT
