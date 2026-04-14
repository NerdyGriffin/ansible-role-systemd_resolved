# Ansible Role: `nerdygriffin.systemd_resolved`

Configure `systemd-resolved` DNS settings via a drop-in file under `/etc/systemd/resolved.conf.d/`, with VLAN-aware DC ordering derived from Ansible inventory.

## What it does

Renders `/etc/systemd/resolved.conf.d/domain-member.conf` (name configurable) containing `DNS=`, `FallbackDNS=`, and `Domains=` keys, then restarts `systemd-resolved` via a handler when the file changes.

The `DNS=` list is built automatically from inventory data in the pattern used by the Windows-side `DnsClientData.psd1` / `Set-Dns.ps1`:

```
[host's VLAN gateway, *same-VLAN DCs, *other-VLAN DCs]
```

This takes advantage of split-DNS when the local router forwards AD-zone queries to DCs (e.g., UniFi "Forward Domain" policy), while leaving DCs as a direct fallback.

Hosts where `systemd-resolved` is not the active resolver (e.g., Rocky 8 hosts still using `/etc/resolv.conf` statically) are **skipped automatically**.

## Requirements

- Ansible 2.14+
- Linux target with `systemd-resolved` (Ubuntu 22.04+, Debian 11+, EL 8/9)
- The `ipaddr` filter (part of `ansible.netcommon` or ships with Ansible core depending on version)

## Role Variables

All defaults live in `defaults/main.yml`. Defaults are sensible for most domain-joined hosts.

| Variable | Default | Purpose |
|----------|---------|---------|
| `systemd_resolved_manage` | `true` | Master on/off switch. Set `false` on a host to skip entirely. |
| `systemd_resolved_force_apply_on_inactive` | `false` | When true, write the drop-in even if `systemd-resolved` is stopped. |
| `systemd_resolved_dropin_name` | `"domain-member.conf"` | File name under `/etc/systemd/resolved.conf.d/`. |
| `systemd_resolved_dropin_dir` | `/etc/systemd/resolved.conf.d` | Drop-in directory. |
| `systemd_resolved_dropin_mode` | `"0644"` | File mode. |
| `systemd_resolved_search_domains` | `[]` | `Domains=` override list. Empty = use `hostvars.network.search_domains` if defined, else `[ansible_domain]`. |
| `systemd_resolved_fallback_dns` | `[1.1.1.1, ...]` | `FallbackDNS=` list. |
| `systemd_resolved_dns_override` | `[]` | If non-empty, skip VLAN derivation and use this list verbatim as `DNS=`. |
| `systemd_resolved_dc_group` | `domain_controllers` | Inventory group providing the DC list. |

## Inventory expectations

For the VLAN-aware DNS derivation to work, inventory should have:

1. A `networks.vlans` list in `group_vars/all/main.yml` (or equivalent), with each entry containing at least `cidr` and `gateway`:
   ```yaml
   networks:
     vlans:
       - id: 20
         name: lan
         cidr: 10.9.20.0/24
         gateway: 10.9.20.1
   ```
2. A `domain_controllers` group (name configurable via `systemd_resolved_dc_group`) listing each DC as an inventory host with `ansible_host` set to its IP.

If the host's `ansible_host` doesn't match any VLAN's CIDR, the role falls back to "all DCs in inventory order" for the `DNS=` list (no gateway, no partitioning).

## Example Playbook

```yaml
- hosts: linux
  gather_facts: true
  become: true
  roles:
    - nerdygriffin.systemd_resolved
```

### Single-host cutover

```bash
ansible-playbook configure_systemd_resolved.yml -l term-8
```

### Force apply on a host where the service is temporarily stopped

```bash
ansible-playbook configure_systemd_resolved.yml -l hl15-00 \
  -e systemd_resolved_force_apply_on_inactive=true
```

### Ad-hoc DNS override (skip VLAN logic)

```bash
ansible-playbook configure_systemd_resolved.yml -l term-8 \
  -e '{"systemd_resolved_dns_override":["10.9.20.1","10.9.20.10","10.9.20.16"]}'
```

## Verification

After a run, check the rendered file and resolver state:

```bash
cat /etc/systemd/resolved.conf.d/domain-member.conf
resolvectl status | head -30
```

## License

MIT

## Author

[NerdyGriffin](https://github.com/NerdyGriffin)
