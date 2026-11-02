# Use Tailscale with NordVPN on Linux

Use Tailscale and the NordVPN Linux app at the same time with NordVPN's
firewall enabled. This configuration re-marks Tailscale's packets with
NordVPN's own firewall mark so NordVPN's firewall accepts them.

The NordVPN version of the [Mullvad setup](../README.md) in this repository.

**Written on 2026-11-02 for:**

- Omarchy (Arch Linux), nftables 1.1.7
- NordVPN Linux app (NordLynx), firewall mark `0xe1f1`
- Tailscale

## Why NordVPN blocks Tailscale

With its firewall enabled, NordVPN's `inet nordvpn` table drops all traffic
except loopback, the `nordlynx` tunnel, and packets that carry NordVPN's
firewall mark (`0xe1f1`). That blocks Tailscale twice:

1. **tailscaled's own traffic** (coordination server, DERP relays, STUN,
   direct WireGuard to peers) is addressed to public IPs. Tailscale tags it
   with mark `0x80000` so it leaves through the physical interface instead
   of looping through `tailscale0`. NordVPN's firewall drops it there.
2. **Tailnet traffic** entering or leaving `tailscale0` matches none of
   NordVPN's accept rules either.

Allowlisting `100.64.0.0/10` or individual Tailscale IPs does not fix
problem 1, because those packets are not addressed to `100.x` addresses.

## What it does

- [`nordvpn-tailscale.nft`](nordvpn-tailscale.nft) loads a separate
  `inet nordvpn_tailscale` table that:
  - re-marks packets carrying Tailscale's `0x80000` mark with `0xe1f1`, so
    NordVPN's firewall accepts them and NordVPN's routing sends them outside
    the tunnel;
  - sets connection mark `0xe1f1` on traffic leaving or arriving through
    `tailscale0`.
- [`tailscale-nordvpn.service`](tailscale-nordvpn.service) loads and removes
  that table without touching NordVPN's, Tailscale's or ufw's tables.

The output chain runs at priority `-160`, after conntrack (`-200`) and before
NordVPN's output chain (`mangle`, `-150`). The input chain runs at `-10`,
before NordVPN's input chain (`filter`, `0`).

> **Important:** tailscaled's own traffic leaves through your normal
> connection, not NordVPN. Tailscale's servers and your peers see your real
> public IP. Tailnet traffic is still encrypted by Tailscale. All other
> traffic remains governed by NordVPN.

## Limitations

- **No Tailscale exit nodes.** Re-marking removes Tailscale's routing-loop
  protection. With an exit node active, tailscaled's packets would be routed
  back into `tailscale0`.
- **Keep Tailscale DNS limited to MagicDNS names.** If the tailnet is set to
  override local DNS or use global nameservers, tailscaled may forward DNS
  queries outside NordVPN.
- **Unsupported by NordVPN.** `0xe1f1` is NordVPN's internal mark, not a
  documented split-tunnelling interface. A NordVPN update can break this.
- **Kill Switch untested.** Behaviour while NordVPN is disconnected with the
  Kill Switch enabled has not been verified.

## Prerequisites

- Linux with systemd
- NordVPN Linux app with its firewall enabled
- Tailscale
- nftables (`nft`)
- The standard Tailscale interface name, `tailscale0`

Confirm NordVPN's firewall mark is still `0xe1f1`:

```bash
nordvpn settings | grep 'Firewall Mark'
```

If it differs, update every `0x0000e1f1` in `nordvpn-tailscale.nft`.

## Install

Clone the repository and enter this directory:

```bash
git clone https://github.com/patrickfeeney03/mullvad-plus-tailscale.git
cd mullvad-plus-tailscale/nordvpn
```

If the Mullvad version is installed, remove it first (see
[Disable or remove](../README.md#disable-or-remove)).

Check both configuration files:

```bash
sudo nft --check --file nordvpn-tailscale.nft
systemd-analyze verify "$PWD/tailscale-nordvpn.service"
```

Install and enable the compatibility service:

```bash
sudo install -Dm0644 nordvpn-tailscale.nft \
  /etc/nftables.d/nordvpn-tailscale.nft
sudo install -Dm0644 tailscale-nordvpn.service \
  /etc/systemd/system/tailscale-nordvpn.service
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-nordvpn.service
```

If `command -v nft` reports a path other than `/usr/bin/nft`, update the
service file before installing it.

## Verify

```bash
systemctl is-active tailscale-nordvpn.service
nordvpn settings | grep Firewall
nordvpn status
tailscale status
tailscale ping 100.x.y.z
sudo nft list table inet nordvpn_tailscale
```

The counters should increase as Tailscale traffic passes. Reconnect NordVPN
or switch servers once and check that the table and Tailscale still work.

## Disable or remove

```bash
sudo systemctl disable --now tailscale-nordvpn.service
sudo rm /etc/systemd/system/tailscale-nordvpn.service
sudo rm /etc/nftables.d/nordvpn-tailscale.nft
sudo systemctl daemon-reload
```

Stopping the service deletes only the `inet nordvpn_tailscale` table.

## License

MIT — see [LICENSE](../LICENSE).
