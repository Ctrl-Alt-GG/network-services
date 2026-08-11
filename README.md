# Network Services

Ansible playbooks for deploying DDI (DNS, DHCP, NTP), Bracket, MediaMTX streaming, and baseline hardening on Rocky Linux 9/10 hosts.

## Prerequisites

- Ansible >= 2.14 on the control node
- Rocky Linux 9 or 10 target hosts with SSH access
- A user with sudo (or root) on each target

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yaml
```

## Quick Start

1. Edit the inventory and add hosts to groups in `inventory.yaml`
1. Create `host_vars/<hostname>.yaml` per host:

```yaml
---
ansible_host: 10.0.0.1
ansible_user: your_ssh_user
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
...
```

1. Edit `group_vars/` for environment-specific values
1. Run the matching playbook:

```bash
ansible-playbook ddi.yaml        # DNS + DHCP + NTP
ansible-playbook sshjump.yaml    # SSH jump host (base + users only)
ansible-playbook bracket.yaml    # Bracket + nginx reverse proxy
ansible-playbook mediamtx.yaml   # MediaMTX + nginx reverse proxy
```

## Inventory Layout

```yaml
all:
  children:
    dns:      # Unbound DNS resolvers
    dhcp:     # Kea DHCP servers
    ntp:      # Chrony NTP servers
    bracket:  # Bracket application host
    mediamtx: # MediaMTX streaming server
    other:    # SSH jump hosts, etc.
```

  ## Bracket Deployment Notes

  The Bracket role deploys Docker Compose based on the official Bracket Docker docs:

  - Frontend container on loopback `127.0.0.1:3000`
  - Backend container on loopback `127.0.0.1:8400`
  - Postgres for persistent state

  The role also configures a reusable reverse proxy role that provides:

  - Nginx TLS termination for `bracket_domain`
  - `/api/` proxy to backend (`127.0.0.1:8400`)
  - `/` proxy to frontend (`127.0.0.1:3000`)
  - TLS termination using pre-provisioned certificate files

  Deterministic per-host secrets are generated from machine identity:

  - Postgres password
  - JWT secret
  - Initial Bracket admin password

  Configure Bracket in `group_vars/bracket.yaml`.

## MediaMTX Deployment Notes

The MediaMTX role now deploys:

- MediaMTX via Docker Compose at `/opt/mediamtx`
- Reusable `reverse_proxy` role site config at `/etc/nginx/conf.d/mediamtx.conf`
- TLS-enabled proxy expecting pre-provisioned certificate files

Compose bindings:

- `1935/tcp` and `1936/tcp` exposed publicly (RTMP/RTMPS)
- `8888/tcp` and `9997/tcp` bound to loopback only

Nginx provides:

- HTTPS termination for `mediamtx_domain`
- `/api/` proxy to `127.0.0.1:9997` with source IP allow-list
- `/` proxy to `127.0.0.1:8888` with strict CORS controls

Certificate behavior:

- Roles do not request certificates automatically.
- If certificate files are missing, the role fails with guidance to provision `/etc/letsencrypt/live/<domain>/fullchain.pem` and `privkey.pem` manually.

## Variables

Primary vars live in `group_vars/mediamtx.yaml`:

- `mediamtx_domain`
- `mediamtx_certbot_email` (compatibility variable, not used for automatic issuance)
- `mediamtx_admin_password`
- `mediamtx_streamer_password`
- `mediamtx_ingest_password`
- `mediamtx_youtube_stream_key`
- `mediamtx_twitch_stream_key`
- `mediamtx_api_allow_ip`

Use `ansible-vault` for all passwords and stream keys.

## Bootstrap

For first-time machine setup, run:

```bash
ansible-playbook ansible-setup.yaml
```
