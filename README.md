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

### Tailscale Real IP Configuration (Optional)

When accessing SWAG through Tailscale, nginx may see all requests as coming from `127.0.0.1` instead of the real Tailscale client IP. This causes ACL (access control list) rules to fail. Enable Tailscale real IP support to extract the real client IP from Tailscale traffic.

| Variable | Default | Description |
|----------|---------|-------------|
| `swag__enable_tailscale_real_ip` | `false` | Enable Tailscale real IP extraction. When enabled, automatically includes Tailscale real IP configuration in all generated proxy configs. |
| `swag__tailscale_network_cidr` | `100.0.0.0/8` | Tailscale network CIDR range. Change only if using a custom Tailscale network range. |

**Note:** When `swag__enable_tailscale_real_ip` is set to `true`:
- **Template-generated configs**: All proxy configs generated from `swag__proxy_confs_subdomain` using the `template_subdomain.conf.j2` template will automatically include the Tailscale real IP configuration. No manual intervention needed.
- **Static/manual config files**: Proxy config files that are manually created or copied from `files/nginx/proxy-confs/` (not generated from the template) must be manually updated to include the Tailscale real IP configuration. Add the following lines to each server block, right after the `server_name` directive:
  ```nginx
  server_name your-domain.com;
  
  # Tailscale Real IP Configuration
  include /config/nginx/tailscale-real-ip.conf;
  
  # ... rest of config
  ```

**Important:** The Tailscale real IP configuration trusts both the Tailscale network (`100.0.0.0/8`) and localhost (`127.0.0.1`) to extract the real client IP from the `X-Forwarded-For` header. However, if Tailscale's docker mod doesn't set the `X-Forwarded-For` header, you may still see `127.0.0.1` in logs. In that case, you may need to:

1. Check if traffic is actually coming from a Tailscale IP (`100.x.x.x`) by examining nginx access logs with `$remote_addr`
2. Ensure your ACL rules include `allow 127.0.0.1/32;` if Tailscale forwards traffic via localhost
3. Consider using Tailscale's `TAILSCALE_SERVE_MODE` and `TAILSCALE_SERVE_PORT` environment variables to configure how Tailscale forwards traffic

This allows nginx to extract the real client IP from the `X-Forwarded-For` header for Tailscale traffic, enabling ACL rules to work correctly.

### Tailscale Device Persistence

To prevent Tailscale from creating a new device entry every time the SWAG container is redeployed:

1. **Use a Reusable Auth Key**: Create a reusable auth key in the Tailscale admin console (Settings → Keys → Generate auth key → check "Reusable"). Ephemeral (one-time) keys will cause a new device to be created on each container restart.

2. **Set Consistent Hostname**: Ensure `TAILSCALE_HOSTNAME` is set to a consistent value (e.g., `proxy`) in `swag__env`:
   ```yaml
   swag__env:
     TAILSCALE_HOSTNAME: proxy
   ```

3. **Persist Tailscale State**: The role automatically creates `{{ swag__home }}/tailscale` directory and mounts it to `/var/lib/tailscale` in the container. Ensure this volume is included in `swag__directory_volumes`:
   ```yaml
   swag__directory_volumes:
     - "{{ swag__home }}/config:/config"
     - "{{ swag__home }}/tailscale:/var/lib/tailscale"
   ```

The Tailscale state directory (`{{ swag__home }}/tailscale`) is automatically created by the role to ensure state persistence across container redeployments.

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
        swag__enable_tailscale_real_ip: true  # Enable Tailscale real IP support
        swag__proxy_confs_subdomain:
          - server_name: "whoami"
            listen: 443
            acl:
              - 'allow 100.0.0.0/8'  # Tailscale network
              - 'deny all'
```

## License

MIT

## Author Information

This role was created by Bart Smeding.
