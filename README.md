# Ansible Role: tailscale

Install [Tailscale](https://tailscale.com/) on Debian, enable `tailscaled`, and
idempotently join the tailnet from an auth key. Optionally run the node as a
**subnet router** — advertise one or more LAN CIDRs to the tailnet and enable IP
forwarding, so roaming clients reach the whole LAN through this one box without
every host joining the tailnet — and/or as an **exit node**, so clients can route
all their internet traffic through it (full tunnel).

The role:

- Adds the official Tailscale apt repo (signing key under `/etc/apt/keyrings`
  via the modern `signed-by` convention) and installs `tailscale`
  (version-pinnable).
- Enables and starts `tailscaled`.
- When `tailscale_advertise_routes` is set or `tailscale_advertise_exit_node` is
  true, enables IPv4/IPv6 forwarding — persisted to a dedicated drop-in **and**
  re-applied after `network-online.target` by a small systemd unit, so it
  survives a reboot even in an unprivileged LXC (where `systemd-sysctl` runs too
  early and the value is otherwise reset to 0 on boot).
- Runs `tailscale up` from a (vaulted) auth key **only when the node is not
  already up** — idempotent on re-runs.

## Requirements

- **Debian** (bookworm/trixie). `gather_facts` must be on in the consuming play
  — the apt key/repo URLs are keyed off `ansible_distribution_release`.
- The `ansible.posix` collection (for the forwarding sysctls).
- A **Tailscale auth key**, supplied via the caller's vault. A *tagged,
  reusable / pre-authorized* key lets the node come up unattended and ACL'd. The
  role does not mint or store keys.
- For subnet routing inside an **unprivileged LXC**: the container needs
  `/dev/net/tun`. Pass it through at the hypervisor (e.g. the `proxmox-lxc`
  role's `proxmox_lxc_devices`).

> **Check mode:** the apt, keyring and sysctl tasks preview cleanly. Reading
> `tailscale status` and running `tailscale up` are skipped under `--check`
> (the binary may not be installed yet), but the argument-spec validation and
> the asserts still run — so `--check` is a meaningful dry-run of the variable
> surface.

## Role Variables

Defined in `defaults/main.yml`; `meta/argument_specs.yml` is the authoritative,
complete surface and is validated at role start.

| Variable | Default | Description |
|---|---|---|
| `tailscale_authkey` | `""` | Auth key for non-interactive `tailscale up`. **Supply via vault.** Required before a not-yet-up node will join. |
| `tailscale_advertise_routes` | `""` | CIDR(s) to advertise (subnet-router mode), e.g. `192.168.178.0/24`. When set, IP forwarding is enabled. |
| `tailscale_advertise_exit_node` | `false` | Offer this node as an exit node (full-tunnel egress for clients). When true, IP forwarding is enabled. Approve once in the admin console. |
| `tailscale_forwarding_sysctl_file` | `/etc/sysctl.d/99-tailscale-forwarding.conf` | Drop-in where subnet-router/exit-node forwarding is persisted; re-applied at boot by a systemd unit so it survives a reboot in an unprivileged LXC. |
| `tailscale_accept_dns` | `false` | Accept tailnet DNS (MagicDNS). False on a subnet router. |
| `tailscale_hostname` | `""` | Node name in the tailnet; empty uses the system hostname. |
| `tailscale_up_extra_args` | `""` | Extra whitespace-separated args appended to `tailscale up`. |
| `tailscale_package` | `tailscale` | Debian package name. |
| `tailscale_version` | `""` | Optional pinned apt version; empty tracks the repo's current. |
| `tailscale_service` | `tailscaled` | systemd service name. |
| `tailscale_apt_keyring_path` | `/etc/apt/keyrings/tailscale.noarmor.gpg` | Signing-key path. |
| `tailscale_apt_key_url` | _(computed)_ | Signing-key URL for the Debian release. |
| `tailscale_apt_list_path` | `/etc/apt/sources.list.d/tailscale.list` | apt source list path. |
| `tailscale_apt_repo` | _(computed)_ | Full `deb [signed-by=...] ...` source line. |

## Idempotency & changing config later

`tailscale up` runs only when `tailscale status` reports a backend state other
than `Running`, so re-applying the role to a joined node makes no changes. To
**change** advertised routes or DNS on a node that is already up, the gate means
the role won't re-run `up` — apply the change on the host directly, e.g.:

```sh
tailscale set --advertise-routes=192.168.178.0/24,10.0.0.0/24
tailscale set --advertise-exit-node
tailscale set --accept-dns=false
```

(or `tailscale down` first, then re-run the role).

## Example Playbook

```yaml
- name: Tailscale subnet router
  hosts: tailscale_router      # Debian host/LXC with /dev/net/tun
  gather_facts: true
  become: true
  roles:
    - role: tailscale
      vars:
        tailscale_authkey: "{{ vault_tailscale_authkey }}"
        tailscale_advertise_routes: "192.168.178.0/24"
        tailscale_advertise_exit_node: true   # optional: full-tunnel egress
        tailscale_accept_dns: false
```

After the first run, **approve the advertised route and/or exit node once** in
the Tailscale admin console (or pre-authorize with an auto-approver ACL tag).
Exit nodes are then selected per-client ("use exit node").

## License

MIT
