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
ansible-playbook tacacs.yaml     # TACACS+ authentication server
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
    tacacs:   # TACACS+ authentication server
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

## TACACS+ Deployment Notes

The TACACS+ role installs EPEL's `tacacs` package on Rocky Linux 9 and authenticates
local Linux accounts through PAM. It supports legacy-compatible authentication for
Juniper EX3300, Juniper SRX340, and Arista DCS-7050SX devices. Configure it in
`group_vars/tacacs.yaml`:

- Set a unique shared secret of at least 32 characters. The supplied variable file is
  plaintext for initial setup; encrypt it with `ansible-vault` before production use.
- Every device receives its own key derived from that secret and the client name, so a
  compromised device does not expose the others. Renaming a client changes its key.
- Add each device management address to `tacacs_clients`. The role refuses to deploy
  without an explicit client allowlist.
- Set `tacacs_allowed_group` to the local Linux group allowed to log in to network
  devices. The default is `networkadmins`; the TACACS+ role creates this group.
- Set `tacacs_allowed_users` to the local Linux users that should be members of that
  group. These users must exist before the TACACS+ role runs, such as accounts created
  by the shared `users` role.
- Maintain a separate local emergency administrator on every network device. This role
  does not manage device configuration or fallback behavior.

TACACS+ uses a shared secret to obscure packet bodies; it does not provide TLS. Place
the server and device management interfaces on a trusted management network and apply
network ACLs that allow TCP/49 only between the listed devices and this server.

To read back the key to configure on a device, recompute the same derivation locally:

```bash
python3 -c 'import hashlib,sys; print(hashlib.sha256(f"{sys.argv[1]}:{sys.argv[2]}".encode()).hexdigest()[:32])' \
  "$TACACS_SHARED_SECRET" sw-01
```

EPEL does not currently publish the `tacacs` package for Rocky Linux 10. The role
therefore fails early on Rocky 10 rather than building the archived upstream source.
Add an approved maintained package source before extending the deployment to Rocky 10.

## Bootstrap

For first-time machine setup, run:

```bash
ansible-playbook ansible-setup.yaml
```
