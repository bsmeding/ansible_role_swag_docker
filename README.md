# Ansible Role: SWAG Docker

This role sets up the [SWAG](https://docs.linuxserver.io/images/docker-swag) (Secure Web Application Gateway) container using Ansible. SWAG is a full-fledged Nginx web server and reverse proxy with LetsEncrypt/ZeroSSL built-in, php8, fail2ban, and more.

## Requirements

- Docker must be installed on the target host (see Ansible role [bsmeding.docker](https://galaxy.ansible.com/ui/standalone/roles/bsmeding/docker/)).
- `community.docker` Ansible collection.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

### Container Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `swag__name` | `swag` | Name of the container. |
| `swag__image` | `lscr.io/linuxserver/swag` | Docker image to use. |
| `swag__pull_image` | `true` | Pull the image before creating the container. |
| `swag__port_http` | `80` | Host port to map to container port 80. |
| `swag__port_https` | `443` | Host port to map to container port 443. |
| `swag__network` | `proxy` | Name of the Docker network for SWAG. |
| `swag__network_cidr` | `172.16.81.0/26` | CIDR for the SWAG network. |
| `swag__network_mode` | `default` | Docker network mode (`default`, `bridge`, `host`, etc.). |
| `swag__home` | `/opt/{{ swag__name }}` | Host directory for persistent data. |

### SWAG / LetsEncrypt Configuration

**Note:** By default, this role configures SWAG in **Staging** mode with **No Validation** (no SSL/ACME) to prevent accidental rate-limiting during testing. You **must** set `swag__email` and `swag__validation` (and set `swag__staging: false`) for production use.

| Variable | Default | Description |
|----------|---------|-------------|
| `swag__url` | `example.com` | The top-level domain for your server. |
| `swag__subdomains` | `www` | Subdomains to include in the cert. |
| `swag__validation` | `''` (Empty) | Validation method (`http`, `dns`, `duckdns`). Leave empty to disable ACME. |
| `swag__email` | `''` (Empty) | Email address for LetsEncrypt registration. Leave empty to disable ACME. |
| `swag__staging` | `true` | Use LetsEncrypt staging server. Set to `false` for production. |
| `swag__only_subdomains` | `false` | If true, the root domain will not be included in the cert. |
| `swag__certprovider` | `omit` | Certificate provider (`letsencrypt` or `zerossl`). |
| `swag__dnsplugin` | `omit` | DNS plugin for DNS validation (e.g., `cloudflare`). |
| `swag__docker_mods` | `linuxserver/mods:swag-cloudflare-real-ip` | Docker mods to apply (e.g., for Cloudflare real IP). |

### Cloudflare Configuration (Optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `swag__cloudflare_api_token` | `omit` | API Token for Cloudflare DNS validation. |
| `swag__cloudflare_zone_id` | `omit` | Cloudflare Zone ID. |
| `swag__cloudflare_account_id` | `omit` | Cloudflare Account ID. |

### Reverse Proxy Configuration

This role can automatically create Nginx proxy configuration files for subdomains.

`swag__proxy_confs_subdomain` is a list of dictionaries:

```yaml
swag__proxy_confs_subdomain:
  - server_name: "plex"
    # Additional template variables if needed
```

This will create `plex.subdomain.conf` in the `proxy-confs` directory using the template.

### Advanced Configuration

- `swag__default_env`: Default environment variables (PUID/PGID, TZ, etc.).
- `swag__env`: Custom environment variables to merge/override.
- `docker_uid`: UID for the user inside the container (default: 1040).
- `docker_gid`: GID for the group inside the container (default: 1001).

## Example Playbook

```yaml
- hosts: servers
  roles:
    - role: bsmeding.swag_docker
      vars:
        swag__url: "mydomain.com"
        swag__validation: "http"
        swag__email: "admin@mydomain.com"
        swag__staging: false
        swag__proxy_confs_subdomain:
          - server_name: "whoami"
```

## License

MIT

## Author Information

This role was created by Bart Smeding.
