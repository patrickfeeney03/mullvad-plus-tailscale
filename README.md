# Use Tailscale with Mullvad on Linux

Use Tailscale and the Mullvad desktop app at the same time on Linux. This
configuration marks Tailnet traffic so Mullvad permits the kernel to route it
through `tailscale0`.

Based on [TheOrangeOne's guide][orangeone] and
[Mullvad's advanced Linux split-tunnelling documentation][mullvad-docs].
This variant adds IPv6 support, interface-restricted inbound rules, and a
dedicated systemd service.

> **Using NordVPN instead?** See the [NordVPN version](nordvpn/README.md).

**Tested on 2026-07-28 with:**

- Ubuntu 26.04
- Mullvad 2026.3
- Tailscale 1.98.9

## What it installs

- [`mullvad-tailscale.nft`](mullvad-tailscale.nft) contains the firewall
  marks.
- [`tailscale-mullvad.service`](tailscale-mullvad.service) loads and removes
  that nftables table without taking ownership of the rest of the firewall.

The dedicated service does not enable the generic `nftables.service` and does
not flush Mullvad's or Tailscale's dynamically managed firewall tables.

> **Important:** Traffic addressed to the Tailnet bypasses Mullvad's tunnel
> routing so it can enter `tailscale0`. Tailscale still encrypts this traffic.
> Traffic not addressed to the Tailnet remains governed by Mullvad. Incoming
> connections also remain subject to your Tailscale access controls.

## Prerequisites

- Linux with systemd
- Mullvad VPN desktop app
- Tailscale
- nftables (`nft`)
- The standard Tailscale interface name, `tailscale0`

Confirm the required commands, services, and interface exist:

```bash
command -v mullvad tailscale nft
systemctl status mullvad-daemon tailscaled --no-pager
ip link show tailscale0
```

## Install

Clone the repository:

```bash
git clone https://github.com/patrickfeeney03/mullvad-plus-tailscale.git
cd mullvad-plus-tailscale
```

Check both configuration files before installing them:

```bash
sudo nft --check --file mullvad-tailscale.nft
systemd-analyze verify "$PWD/tailscale-mullvad.service"
```

No output means the checks succeeded.

Install and enable the compatibility service:

```bash
sudo install -Dm0644 mullvad-tailscale.nft \
  /etc/nftables.d/mullvad-tailscale.nft
sudo install -Dm0644 tailscale-mullvad.service \
  /etc/systemd/system/tailscale-mullvad.service
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-mullvad.service
```

If `command -v nft` reports a path other than `/usr/sbin/nft`, update the
service file to use the reported absolute path before installing it.

Do not enable the generic `nftables.service` solely for this setup.

## Verify

Check that the compatibility service is enabled and active:

```bash
systemctl is-enabled tailscale-mullvad.service
systemctl is-active tailscale-mullvad.service
```

The commands should report `enabled` and `active`.

Confirm that Mullvad remains connected:

```bash
mullvad status
curl https://am.i.mullvad.net/connected
```

Find an online Tailnet device and test it using its Tailscale IP or MagicDNS
name:

```bash
tailscale status
tailscale ping 100.x.y.z
ping -c 3 100.x.y.z
```

Inspect the compatibility rules and their packet counters:

```bash
sudo nft list table inet mullvad_tailscale
```

The relevant IPv4 or IPv6 counters should increase as Tailnet traffic passes.

## Address ranges and inbound access

- `100.64.0.0/10` is Tailscale's IPv4 shared-address range. It covers
  `100.64.0.0` through `100.127.255.255`.
- `fd7a:115c:a1e0::/48` is Tailscale's IPv6 range.

The output rules allow this computer to initiate Tailnet connections. The
input rules allow other Tailnet devices to initiate connections to this
computer. Incoming exceptions are restricted to packets that actually arrive
through `tailscale0`.

## Update

Pull the latest version, validate it, reinstall both files, and reload:

```bash
git pull --ff-only
sudo nft --check --file mullvad-tailscale.nft
systemd-analyze verify "$PWD/tailscale-mullvad.service"
sudo install -Dm0644 mullvad-tailscale.nft \
  /etc/nftables.d/mullvad-tailscale.nft
sudo install -Dm0644 tailscale-mullvad.service \
  /etc/systemd/system/tailscale-mullvad.service
sudo systemctl daemon-reload
sudo systemctl reload tailscale-mullvad.service
```

## Troubleshooting

Inspect the service and recent logs:

```bash
systemctl status tailscale-mullvad.service --no-pager
journalctl -u tailscale-mullvad.service -b --no-pager
journalctl -u tailscaled -b --no-pager
```

Confirm the expected interfaces and rules exist:

```bash
ip -brief address show tailscale0
ip -brief address show wg0-mullvad
sudo nft list table inet mullvad_tailscale
```

If a future Mullvad update changes its split-tunnelling implementation, verify
these values against Mullvad's current documentation:

```text
Connection-tracking mark: 0x00000f41
Routing/meta mark:         0x6d6f6c65
```

## Disable or remove

Temporarily disable and restore the compatibility rule:

```bash
sudo systemctl stop tailscale-mullvad.service
sudo systemctl start tailscale-mullvad.service
```

Remove it completely:

```bash
sudo systemctl disable --now tailscale-mullvad.service
sudo rm /etc/systemd/system/tailscale-mullvad.service
sudo rm /etc/nftables.d/mullvad-tailscale.nft
sudo systemctl daemon-reload
```

Stopping the service deletes only the `inet mullvad_tailscale` table.

## Acknowledgements

- [Accessing Tailscale whilst using Mullvad][orangeone] by TheOrangeOne
- [Mullvad: Split tunneling with Linux (advanced)][mullvad-docs]

## License

MIT — see [LICENSE](LICENSE).

[orangeone]: https://theorangeone.net/posts/tailscale-mullvad/
[mullvad-docs]: https://mullvad.net/en/help/split-tunneling-with-linux-advanced
