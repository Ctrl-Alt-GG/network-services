# Copilot Instructions — Network Services

## Architecture Overview

Ansible project deploying DDI (DNS, DHCP, NTP) + baseline hardening to **Rocky Linux (EL)** hosts. Main playbooks:

- **ddi.yaml** — deploys all network services to `ddi-01`/`ddi-02` (roles: `base` → `users` → `dns` → `ntp`; DHCP via inventory group)
- **sshjump.yaml** — deploys only `base` + `users` to the SSH jump host
- **librenms.yaml** — deploys LibreNMS monitoring (roles: `base` → `users` → `librenms`) with Nginx, PHP-FPM, MariaDB, SNMP
- **lancache.yaml** — deploys LANcache content caching proxy (roles: `base` → `users` → `docker` → `lancache`) with Docker, monolithic cache + SNI proxy
- **mediamtx.yaml** — deploys MediaMTX streaming/restreaming server (roles: `base` → `users` → `docker` → `mediamtx`) for multi-platform streaming to Twitch/YouTube/Kick
- **ansible-setup.yaml** — one-time bootstrap (interactive `vars_prompt`) to create an `ansible` user with SSH keys and passwordless sudo on a new machine

Roles apply in dependency order: `base` (hardening, firewalld, SELinux, fail2ban, sysctl, auditd) → `users` (creates users from `group_vars/all.yaml` list with remote SSH key URLs) → `docker` (for container workloads) → service roles.

## Variable Hierarchy

Follow the existing layering strictly — don't flatten or duplicate variables across levels:

| Scope | File | Contents |
|-------|------|----------|
| All hosts | `group_vars/all.yaml` | `ansible_become`, `users` list (name + `ssh_key_url`) |
| DNS group | `group_vars/dns.yaml` | `lancache` config, `upstreams` list |
| DHCP group | `group_vars/dhcp.yaml` | `interfaces`, `subnets` (CIDR), `dns_servers`, `ntp_servers` |
| NTP group | `group_vars/ntp.yaml` | `ntp_upstreams` (with `nts` flag), `ntp_allow` CIDRs |
| LibreNMS group | `group_vars/librenms.yaml` | `librenms_domain`, `librenms_db_password`, `librenms_snmp_community`, admin credentials |
| LANcache group | `group_vars/lancache.yaml` | `lancache_cache_dir`, `lancache_cache_disk_size`, `lancache_upstream_dns` |
| MediaMTX group | `group_vars/mediamtx.yaml` | `mediamtx_ingest` (path, stream_key), `mediamtx_restream_destinations`, `mediamtx_fallback` |
| Per-host | `host_vars/<host>.yaml` | `ansible_host`, `ansible_user`, `ansible_ssh_private_key_file` |
| Role defaults | `roles/<role>/defaults/main.yaml` | Lowest-priority defaults (e.g., `timezone`, `cache_domains_repo`) |

## Conventions & Patterns

- **YAML style**: Use `.yaml` extension (not `.yml`). Start files with `---`, end with `...`.
- **FQCN required**: Always use fully qualified collection names (`ansible.builtin.package`, `ansible.posix.firewalld`, `community.general.timezone`). No exceptions.
- **Templates**: All templates start with `# {{ansible_managed}}`. Use Jinja2 loops/conditionals; avoid complex logic — keep it in tasks via `set_fact` when needed (see DNS `cache_domains_list`).
- **Handlers**: Named as `Restart <service>` or `Reload <service>`. Each role owns its handlers; `Restart auditd` appears in multiple roles (base, dns, dhcp, ntp) since audit rules are per-role.
- **Config validation**: Templates that support it use `validate:` parameter (e.g., `unbound-checkconf %s`, `visudo -cf %s`, `sshd -t -f %s`).
- **Firewall pattern**: Each service role enables its own firewalld service (`ansible.posix.firewalld` with `service:`, `permanent: true`, `immediate: true`). No rich rules or rate limiting.
- **Security hardening**: Every role adds auditd rules via `blockinfile` to `/etc/audit/rules.d/<service>.rules` and notifies `Restart auditd`. Service roles apply systemd sandboxing where possible (see DHCP's `security.conf` override).
- **SELinux**: Enforcing mode assumed on bare-metal/VM hosts. All SELinux tasks and `restorecon` handlers are guarded with `when: ansible_selinux.status == 'enabled'` so playbooks run safely on LXC containers (or any host without SELinux).

## Key Implementation Details

- **DHCP subnet calculation**: `kea-dhcp4.conf.j2` auto-derives pool ranges from CIDR using `ansible.utils.ipaddr` filters — `nthhost(1)` for pool start, `last_usable` for router, `ipmath(-1)` for pool end. Requires `ansible.utils` collection.
- **LANcache (DNS)**: Optional feature toggled by `lancache.enabled` in `group_vars/dns.yaml`. When enabled, clones `uklans/cache-domains` repo, processes domain lists, and renders Unbound local-zone redirects via `lancache-redirect.conf.j2`.
- **NTP NTS support**: `chrony.conf.j2` conditionally appends `nts` flag per upstream based on `src.nts` boolean in `ntp_upstreams`.
- **Users role**: Fetches SSH public keys from remote URLs (`ssh_key_url`). The `users` list in `group_vars/all.yaml` applies to all hosts.
- **LibreNMS**: Full-stack monitoring deployment (Nginx + PHP-FPM + MariaDB + SNMP). Uses Remi repo for PHP on Rocky 9/10. APP_KEY is auto-generated on first run and preserved across subsequent deploys. Scheduler uses systemd timer + cron from upstream `dist/` files. Requires `community.mysql` collection.
- **LANcache**: Docker-based content caching proxy using `lancachenet/monolithic` (HTTP cache) and `lancachenet/sniproxy` (HTTPS passthrough). Deployed via Docker Compose. Works in tandem with the DNS role's `lancache.enabled` setting which redirects game CDN domains to the cache IP. Requires `community.docker` collection.
- **MediaMTX**: Docker-based RTMP/RTSP/HLS streaming server for multi-platform restreaming. Accepts a single authenticated ingest stream and forwards to Twitch, YouTube, Kick simultaneously. Supports fallback video playback when no live stream is active. Requires `community.docker` collection.
- **Docker role**: Shared role for Docker CE installation used by `lancache` and `mediamtx`. Installs Docker CE, docker-compose-plugin, configures daemon.json, and adds audit rules.

## Running Playbooks

```bash
# Install required collections first
ansible-galaxy collection install -r requirements.yaml

# Deploy DDI services
ansible-playbook ddi.yaml

# Deploy SSH jump host
ansible-playbook sshjump.yaml

# Deploy LibreNMS monitoring
ansible-playbook librenms.yaml

# Deploy LANcache content caching
ansible-playbook lancache.yaml

# Deploy MediaMTX streaming server
ansible-playbook mediamtx.yaml

# Bootstrap a new machine (interactive prompts)
ansible-playbook ansible-setup.yaml
```

## Adding a New Role

1. Create `roles/<name>/{tasks,handlers,templates,meta,defaults}/main.yaml`
2. Set `meta/main.yaml` with `min_ansible_version: "2.14"`, platform `EL`, and list required collections
3. Follow the security pattern: firewalld service rule, auditd rules in `/etc/audit/rules.d/<name>.rules`, SELinux `restorecon` handler
4. Add the role to the appropriate playbook and create matching `group_vars/<group>.yaml` if new inventory group is needed
