# VPS-dotfiles

Hardened Bash automation for setting up and verifying baseline security controls on an Ubuntu VPS, with guarded SSH-access cutover tooling.

## Repository contents

- `secure-vps.sh`  
  Main hardening script with:
  - inventory mode (`--check`)
  - verification mode (`--verify`)
  - guided apply mode (`--apply`)
  - optional SSH auth hardening (`--harden-ssh`)
  - optional Fail2ban skip (`--no-fail2ban`)

- `scripts/restrict-ssh.sh`  
  Applies an nftables guard that limits inbound SSH (`tcp/22`) to:
  - loopback (for SSH-over-Cloudflare Tunnel path)
  - `tailscale0` (for Tailscale SSH path)

  Includes timed rollback safety (`at` job), manual rollback, and confirm flow.

## Prerequisites

- Ubuntu host (the main script exits on non-Ubuntu systems)
- Root privileges (`sudo`)
- `systemd` environment
- Network/provider console or rescue access before applying remote-access changes

`secure-vps.sh` installs required packages when needed (`ufw`, `fail2ban`).  
`scripts/restrict-ssh.sh` requires `nft`, `at`, and an active `atd` service.

## Safe usage workflow

Run from the repository root:

```bash
sudo bash ./secure-vps.sh --check
sudo bash ./secure-vps.sh --verify
sudo bash ./secure-vps.sh --apply
```

If your shell session is over SSH, preserve `SSH_CONNECTION` when checking/verifying:

```bash
sudo env SSH_CONNECTION="${SSH_CONNECTION:-}" bash ./secure-vps.sh --check
sudo env SSH_CONNECTION="${SSH_CONNECTION:-}" bash ./secure-vps.sh --verify
```

SSH-auth hardening is opt-in during apply:

```bash
sudo bash ./secure-vps.sh --apply --harden-ssh
```

Restrict SSH to loopback + Tailscale only (run this from an already-approved access path):

```bash
sudo ./scripts/restrict-ssh.sh apply
sudo ./scripts/restrict-ssh.sh status
sudo ./scripts/restrict-ssh.sh confirm
# or, if needed:
sudo ./scripts/restrict-ssh.sh rollback
```

## Important warnings

- Keep your current SSH session open while applying firewall or SSH changes.
- Always test a second login before closing your original session.
- `secure-vps.sh --apply` is interactive and intentionally conservative:
  - no full package upgrade
  - no reboot
  - no SSH port change
  - no provider-level firewall changes
- `restrict-ssh.sh apply` can terminate direct public-IP SSH access immediately.
- Use only one host firewall manager at a time long-term; verify policy behavior before retiring another firewall layer.

## Customization notes

- Adjust SSH-interface expectations for `restrict-ssh.sh` with:
  - `TAILSCALE_IF` (defaults to `tailscale0`)
- Review prompted port rules carefully in `secure-vps.sh --apply`; this script stores allowed inbound ports at:
  - `/var/lib/vps-hardening/allowed_ports`
