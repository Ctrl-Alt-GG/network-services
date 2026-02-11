# Copilot Instructions — Network Services

## Architecture Overview

Ansible project deploying DDI (DNS, DHCP, NTP) + baseline hardening to **Rocky Linux (EL)** hosts. Two main playbooks:

- **ddi.yaml** — deploys all network services to `ddi-01`/`ddi-02` (roles: `base` → `users` → `dns` → `ntp`; DHCP via inventory group)
- **sshjump.yaml** — deploys only `base` + `users` to the SSH jump host
- **ansible-setup.yaml** — one-time bootstrap (interactive `vars_prompt`) to create an `ansible` user with SSH keys and passwordless sudo on a new machine

Roles apply in dependency order: `base` (hardening, firewalld, SELinux, fail2ban, sysctl, auditd) → `users` (creates users from `group_vars/all.yaml` list with remote SSH key URLs) → service roles.

## Variable Hierarchy

Follow the existing layering strictly — don't flatten or duplicate variables across levels:

| Scope | File | Contents |
|-------|------|----------|
| All hosts | `group_vars/all.yaml` | `ansible_become`, `users` list (name + `ssh_key_url`) |
| DNS group | `group_vars/dns.yaml` | `lancache` config, `upstreams` list |
| DHCP group | `group_vars/dhcp.yaml` | `interfaces`, `subnets` (CIDR), `dns_servers`, `ntp_servers` |
| NTP group | `group_vars/ntp.yaml` | `ntp_upstreams` (with `nts` flag), `ntp_allow` CIDRs |
| Per-host | `host_vars/<host>.yaml` | `ansible_host`, `ansible_user`, `ansible_ssh_private_key_file` |
| Role defaults | `roles/<role>/defaults/main.yaml` | Lowest-priority defaults (e.g., `timezone`, `cache_domains_repo`) |

## Conventions & Patterns

- **YAML style**: Use `.yaml` extension (not `.yml`). Start files with `---`, end with `...`.
- **FQCN required**: Always use fully qualified collection names (`ansible.builtin.package`, `ansible.posix.firewalld`, `community.general.timezone`). The only exception is the `users` role which uses short-form for some modules.
- **Templates**: All templates start with `# {{ansible_managed}}`. Use Jinja2 loops/conditionals; avoid complex logic — keep it in tasks via `set_fact` when needed (see DNS `cache_domains_list`).
- **Handlers**: Named as `Restart <service>` or `Reload <service>`. Each role owns its handlers; `Restart auditd` appears in multiple roles (base, dns, dhcp, ntp) since audit rules are per-role.
- **Config validation**: Templates that support it use `validate:` parameter (e.g., `unbound-checkconf %s`, `visudo -cf %s`, `sshd -t -f %s`).
- **Firewall pattern**: Each service role opens its own firewalld port with rate-limiting rich rules (e.g., `limit value="1000/s"` for DNS, `"100/m"` for DHCP, `"10/s"` for NTP).
- **Security hardening**: Every role adds auditd rules via `blockinfile` to `/etc/audit/rules.d/<service>.rules` and notifies `Restart auditd`. Service roles apply systemd sandboxing where possible (see DHCP's `security.conf` override).
- **SELinux**: Enforcing mode assumed. Roles install SELinux tooling and use `restorecon` handlers after file operations.

## Key Implementation Details

- **DHCP subnet calculation**: `kea-dhcp4.conf.j2` auto-derives pool ranges from CIDR using `ansible.utils.ipaddr` filters — `nthhost(1)` for pool start, `last_usable` for router, `ipmath(-1)` for pool end. Requires `ansible.utils` collection.
- **LANcache (DNS)**: Optional feature toggled by `lancache.enabled` in `group_vars/dns.yaml`. When enabled, clones `uklans/cache-domains` repo, processes domain lists, and renders Unbound local-zone redirects via `lancache-redirect.conf.j2`.
- **NTP NTS support**: `chrony.conf.j2` conditionally appends `nts` flag per upstream based on `src.nts` boolean in `ntp_upstreams`.
- **Users role**: Fetches SSH public keys from remote URLs (`ssh_key_url`). The `users` list in `group_vars/all.yaml` applies to all hosts.

## Running Playbooks

```bash
# Install required collections first
ansible-galaxy collection install -r requirements.yaml

# Deploy DDI services
ansible-playbook ddi.yaml

# Deploy SSH jump host
ansible-playbook sshjump.yaml

# Bootstrap a new machine (interactive prompts)
ansible-playbook ansible-setup.yaml
```

## Adding a New Role

1. Create `roles/<name>/{tasks,handlers,templates,meta,defaults}/main.yaml`
2. Set `meta/main.yaml` with `min_ansible_version: "2.14"`, platform `EL`, and list required collections
3. Follow the security pattern: firewalld rule with rate limiting, auditd rules in `/etc/audit/rules.d/<name>.rules`, SELinux `restorecon` handler
4. Add the role to the appropriate playbook and create matching `group_vars/<group>.yaml` if new inventory group is needed
