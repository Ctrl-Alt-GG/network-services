# Network Services

Ansible playbooks for deploying DDI (DNS, DHCP, NTP), LANcache, LibreNMS monitoring, MediaMTX streaming, and baseline hardening on Rocky Linux 9/10 hosts.

## Prerequisites

- Ansible >= 2.14 on the control node
- Rocky Linux 9 or 10 target hosts with SSH access
- A user with sudo (or root) on each target

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yaml
```

## Quick Start

1. **Edit the inventory** — add your hosts under the appropriate groups in `inventory.yaml`
1. **Create host_vars** — for each host, create `host_vars/<hostname>.yaml`:

```yaml
---
ansible_host: 10.0.0.1
ansible_user: your_ssh_user
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
...
```

1. **Customise group_vars** — edit the files under `group_vars/` (see below)
1. **Run a playbook**:

```bash
ansible-playbook ddi.yaml        # DNS + DHCP + NTP
ansible-playbook sshjump.yaml    # SSH jump host (base + users only)
ansible-playbook librenms.yaml   # LibreNMS monitoring stack
ansible-playbook lancache.yaml   # LANcache content caching proxy
ansible-playbook mediamtx.yaml   # MediaMTX streaming/restreaming
```

### Bootstrapping a new machine

If the target hasn't been configured for Ansible yet, run the interactive bootstrap playbook first. It creates an `ansible` user with SSH keys and passwordless sudo:

```bash
ansible-playbook ansible-setup.yaml
```

## Inventory Layout

```yaml
all:
  children:
    dns:      # Unbound DNS resolvers
    dhcp:     # Kea DHCP servers
    ntp:      # Chrony NTP servers
    librenms: # LibreNMS monitoring
    lancache: # LANcache content caching
    mediamtx: # MediaMTX streaming server
    other:    # SSH jump hosts, etc.
```

A host can belong to multiple groups (e.g., `ddi-01` is in `dns`, `dhcp`, and `ntp`). The `ddi.yaml` playbook targets each group separately with the matching role.

## Configuration Reference

### All hosts — `group_vars/all.yaml`

| Variable | Description |
| -------- | ----------- |
| `users` | List of users to create on every host. Each entry has `name` and `ssh_key_url` (URL to their public SSH key). |

### Base role — `roles/base/defaults/main.yaml`

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `timezone` | `Europe/Budapest` | System timezone |

The base role applies to all hosts: SSH hardening, firewalld, SELinux enforcing, fail2ban, sysctl network protections, auditd, and common CLI tools.

### DNS — `group_vars/dns.yaml`

| Variable | Description |
| -------- | ----------- |
| `upstreams` | List of upstream DNS forwarder IPs |
| `lancache.enabled` | Set `true` to enable LANcache domain redirects |
| `lancache.ipaddrs` | LANcache server IPs (used when enabled) |
| `private_zones` | List of private DNS zones (see below) |

**Private zones** let you host internal DNS records. Each zone supports:

```yaml
private_zones:
  - name: home.lab                              # Zone domain (required)
    records:                                     # A, AAAA, CNAME, MX, TXT, SRV, etc.
      - "server.home.lab.  IN A 192.168.1.10"
      - "db.home.lab.      IN A 192.168.1.11"
    reverse_zone: 1.168.192.in-addr.arpa        # Optional: reverse DNS zone
    ptr_records:                                  # Optional: PTR records
      - "10.1.168.192.in-addr.arpa.  IN PTR server.home.lab."
    transparent: false                            # false (default) = NXDOMAIN for undefined names
                                                  # true = fall through to upstream forwarders
```

### DHCP — `group_vars/dhcp.yaml`

| Variable | Description |
| -------- | ----------- |
| `interfaces` | Network interfaces Kea should listen on (e.g., `["eno1"]`) |
| `subnets` | List of CIDR ranges to serve (e.g., `["192.168.128.0/24"]`). Pool ranges, router, and gateway are **auto-calculated** from CIDR. |
| `dns_servers` | DNS servers pushed to clients |
| `ntp_servers` | NTP servers pushed to clients |

Pool calculation from CIDR: first usable → pool start, last usable → router, last usable minus one → pool end.

### NTP — `group_vars/ntp.yaml`

| Variable | Description |
| -------- | ----------- |
| `ntp_upstreams` | List of upstream time servers. Each has `host`, `type` (default `server`), and `nts` (`true` to enable NTS authentication). |
| `ntp_allow` | CIDRs allowed to query this NTP server (e.g., `["192.168.0.0/16"]`) |

### LibreNMS — `group_vars/librenms.yaml`

| Variable | Description |
| -------- | ----------- |
| `librenms_domain` | FQDN for the Nginx vhost (e.g., `librenms.example.com`) |
| `librenms_db_password` | MariaDB password — **change this** |
| `librenms_snmp_community` | SNMP community string — **change this** |
| `librenms_admin_user` | Initial web admin username |
| `librenms_admin_password` | Initial web admin password — **change this** |

Additional tunables are in `roles/librenms/defaults/main.yaml` (PHP version, install dir, DB host/name/user, PHP-FPM pool sizing).

### LANcache — `group_vars/lancache.yaml`

| Variable | Description |
| -------- | ----------- |
| `lancache_cache_dir` | Path to the cache data directory (default: `/srv/lancache/data`) |
| `lancache_logs_dir` | Path to the cache logs directory (default: `/srv/lancache/logs`) |
| `lancache_cache_disk_size` | Maximum cache size (default: `500g`) |
| `lancache_upstream_dns` | DNS server the cache uses to resolve origins (default: `1.1.1.1`) |

Additional tunables in `roles/lancache/defaults/main.yaml` (container images, `cache_max_age`).

**Two-part setup** — LANcache requires both:

1. The **lancache role** on the cache host (this section) — runs the Docker-based caching proxy
2. The **DNS role** with `lancache.enabled: true` in `group_vars/dns.yaml` — redirects game CDN domains to the cache host's IP via `lancache.ipaddrs`

```yaml
# group_vars/dns.yaml — point DNS at your LANcache host
lancache:
  enabled: true
  ipaddrs:
    - 192.168.1.50   # IP of your lancache host
```

### MediaMTX — `group_vars/mediamtx.yaml`

| Variable | Description |
| -------- | ----------- |
| `mediamtx_ingest.path` | Stream path name (default: `live`) |
| `mediamtx_ingest.stream_key` | Authentication key for publishers — **change this** |
| `mediamtx_restream_destinations` | Platform configs (twitch, youtube, kick) with `enabled`, `url`, `stream_key` |
| `mediamtx_fallback.enabled` | Play fallback video when no live stream (default: `false`) |
| `mediamtx_fallback.video_path` | Path to fallback video inside container |
| `mediamtx_fallback.host_path` | Host directory containing the fallback video |

Additional tunables in `roles/mediamtx/defaults/main.yaml` (ports, recording, HLS, API).

**Usage**: Configure your streaming software (OBS, etc.) to publish to `rtmp://<server>:1935/live` with the stream key. The stream is automatically forwarded to all enabled platforms.

## What Each Role Does

| Role | Services | Hardening |
| ---- | -------- | --------- |
| **base** | SSH, firewalld, fail2ban | SELinux enforcing, sysctl network protection, auditd rules, password-auth disabled |
| **users** | Creates users with SSH keys from remote URLs | Users added to `wheel` group |
| **dns** | Unbound (forwarding + private zones + optional LANcache) | DNSSEC validation, `private-address` rebind protection, firewalld, auditd |
| **dhcp** | Kea DHCPv4 + control agent | Systemd sandboxing (`NoNewPrivileges`, `ProtectSystem=strict`), firewalld, auditd |
| **ntp** | Chrony (with optional NTS) | Firewalld, auditd |
| **librenms** | Nginx, PHP-FPM, MariaDB, SNMP, LibreNMS | SELinux booleans + file contexts, firewalld, auditd |
| **docker** | Docker CE, docker-compose-plugin | Audit rules for Docker binaries and config |
| **lancache** | lancachenet/monolithic + sniproxy (via docker role) | SELinux container contexts, firewalld, auditd |
| **mediamtx** | MediaMTX streaming server (via docker role) | SELinux container contexts, firewalld, auditd |

## Project Structure

```text
├── ansible.cfg              # Inventory path, SSH pipelining
├── inventory.yaml           # Host groups
├── requirements.yaml        # Ansible Galaxy collections
├── ddi.yaml                 # Playbook: DNS + DHCP + NTP
├── sshjump.yaml             # Playbook: SSH jump host
├── librenms.yaml            # Playbook: LibreNMS
├── lancache.yaml            # Playbook: LANcache
├── mediamtx.yaml            # Playbook: MediaMTX streaming
├── ansible-setup.yaml       # Playbook: bootstrap new machine
├── group_vars/
│   ├── all.yaml             # Users (all hosts)
│   ├── dns.yaml             # Upstreams, LANcache, private zones
│   ├── dhcp.yaml            # Interfaces, subnets, DNS/NTP servers
│   ├── ntp.yaml             # Upstream time servers, allowed CIDRs
│   ├── librenms.yaml        # Domain, DB/SNMP/admin credentials
│   ├── lancache.yaml        # Cache dir, size, upstream DNS
│   └── mediamtx.yaml        # Ingest config, restream destinations, fallback
├── host_vars/
│   └── <hostname>.yaml      # Per-host connection details
└── roles/
    ├── base/                # OS hardening
    ├── users/               # User creation
    ├── dns/                 # Unbound
    ├── dhcp/                # Kea DHCP
    ├── ntp/                 # Chrony
    ├── librenms/            # LibreNMS stack
    ├── docker/              # Docker CE (shared)
    ├── lancache/            # LANcache Docker stack
    └── mediamtx/            # MediaMTX streaming
```

## Tips

- **Dry run**: `ansible-playbook ddi.yaml --check --diff` to preview changes without applying
- **Single host**: `ansible-playbook ddi.yaml --limit ddi-01` to target one host
- **Debug DNS**: Set `verbosity: 2` in the Unbound template, or use `unbound-control log_queries on` on the host
- **Debug DHCP**: Leases are in `/var/lib/kea/kea-leases4.csv`, logs in `/var/log/kea/`
- **Secrets**: Consider using `ansible-vault encrypt_string` for passwords in `group_vars/librenms.yaml`
