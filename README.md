# Infrastructure baseline: Ansible roles

Ansible roles I use to build and rebuild my own servers: a few VPS at different providers and
Proxmox guests at home, all on Debian 13. This repository is an extract of my private
infrastructure repository. Host names, addresses and secrets are left out; the roles are the
ones that run in production.

Comments and task names are in French, my working language for this project.

## Design rules

- **No inbound port open on the Internet.** Public traffic enters through Cloudflare Tunnel.
  Machine-to-machine traffic goes through Tailscale.
- **Every host is rebuilt from Git.** Servers are not backed up; data is. A host that breaks is
  reinstalled and reconfigured by `playbooks/site.yml`.
- **Idempotent or not done.** A second run of a role must report `changed=0`.
  `playbooks/check.yml` runs the whole site in check mode to detect drift.
- **No secret in the repository.** Secrets are encrypted with SOPS and age in the private
  repository and passed as variables.

## Roles

| Role | What it does |
|---|---|
| `base` | Hostname, time zone, locales, chrony, security-only unattended upgrades with a reboot window per host, capped journald, shell prompt and dynamic motd. |
| `hardening` | SSH with keys only and an allowed group, `deploy` and console-only `breakglass` users, locked root password, nftables with a default-drop input policy, hardened sysctl, AppArmor, unused services masked, Lynis audit with the report kept on the host. |
| `tailscale` | Installs from the official repository, joins the tailnet with a tagged key, manages subnet routes and exit node, restores the identity of a rebuilt node. |
| `unbound` | Recursive resolver with DNSSEC and no public forwarder, cache sized from the host RAM, static `resolv.conf` protected against DHCP and cloud-init, resolution test at the end of the role. |
| `docker` | Docker CE from the official repository, published ports bound to `127.0.0.1` by default, log limits, and a check that fails if a container publishes on all interfaces. |
| `cloudflared` | Cloudflare Tunnel connector as a replica, dedicated system user, optional hardened systemd unit, started only when its origins are reachable. |
| `caddy` | Caddy in a container as the local reverse proxy behind the tunnel. |
| `zabbix_agent` | Zabbix agent 2 in active mode (no inbound port) with auto-registration by host metadata. |
| `backup` | Daily restic backup to an append-only repository, SQL dumps taken first, a metric exposed to monitoring. |

## Checks

Read-only playbooks in `tests/`. Each one fails when it finds a gap.

- `tests/security-check.yml`: effective sshd configuration, nftables input chain, listening
  ports, Lynis score.
- `tests/no-public-ports.yml`: no socket listens on all interfaces.
- `tests/docker-published-ports.yml`: no container publishes on `0.0.0.0` or `[::]`.
- `tests/dns-check.yml`: resolution through Unbound and DNSSEC validation.

## Usage

```sh
# Dry run first, always
ansible-playbook -i inventories/example/hosts.yml playbooks/site.yml --check --diff

# Apply
ansible-playbook -i inventories/example/hosts.yml playbooks/site.yml

# Drift: must report changed=0
ansible-playbook -i inventories/example/hosts.yml playbooks/check.yml --check --diff
```

`inventories/example/hosts.yml` uses documentation addresses. Replace them with your own.

## Lint

```sh
yamllint .
ansible-lint
```

Both run in CI on every push (`.github/workflows/lint.yml`).
