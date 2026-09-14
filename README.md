# dotfiles

Personal post-install configs and reference notes for a privacy-hardened Linux (Fedora KDE) and Windows setup: encrypted DNS, authenticated time sync, and reduced telemetry.

Everything here is a drop-in file or a reference document, not an installer — nothing runs itself. Copy what you want to the destinations below.

## What's included

- **`common/`** — cross-platform references: a partial `dnscrypt-proxy.toml` (overrides only) and a list of privacy-focused DNS/NTP servers.
- **`linux/`** — Fedora KDE post-install notes, plus `networking/` drop-ins for DNS and transport tuning:
  - `encrypted-dns/` and `proxy-dns/` are two **alternative** encrypted-DNS stacks — deploy one, not both. Under `encrypted-dns/`, systemd-resolved speaks DNS-over-TLS directly to public resolvers; under `proxy-dns/`, it forwards to a local `dnscrypt-proxy` instead.
  - `90-disable-nm-dns.conf` takes NetworkManager out of the DNS path.
  - `99-net-transport-optimization.conf` is a heavily-annotated sysctl drop-in (BBR/fq, ECN, MTU blackhole recovery, buffer ceilings).
  - `nts/` configures chrony for NTS-only (authenticated), non-smearing time sync.
  - Each DNS variant ships a `time-sync-rescue.md` runbook for the cold-boot DNS/clock deadlock (see Notes).
- **`windows/`** — Group Policy privacy hardening checklist, `winget` app lists, a UTC-RTC registry tweak, a disk-partitioning script, and misc hardware/app config notes.

## Installation

There's no single install script; apply the pieces you want by hand.

**Linux drop-ins** (pick one of `encrypted-dns/` or `proxy-dns/` for the DNS line):

```bash
sudo cp linux/networking/90-disable-nm-dns.conf /etc/NetworkManager/conf.d/
sudo cp linux/networking/99-net-transport-optimization.conf /etc/sysctl.d/
sudo cp linux/networking/encrypted-dns/90-dns-strict-policy.conf /etc/systemd/resolved.conf.d/
# or: sudo cp linux/networking/proxy-dns/90-dns-bridge-policy.conf /etc/systemd/resolved.conf.d/

sudo cp linux/networking/nts/chrony.conf /etc/chrony.conf
sudo cp linux/networking/nts/chronyd /etc/sysconfig/chronyd

sudo systemctl restart NetworkManager systemd-resolved chronyd
sudo sysctl -p /etc/sysctl.d/99-net-transport-optimization.conf
sudo dracut --force   # the initramfs carries a copy of the sysctl file
```

If you're using `proxy-dns/`, also merge `common/dnscrypt-proxy-reference.txt` into `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` (it's overrides only, not a complete config).

**Windows:**

```bat
:: Run as Administrator; edit the disk number inside the script first
windows\setup-partitions.bat

:: Import the UTC RTC tweak
reg import windows\enable-utc.reg
```

Then run the lines in `windows/apps.txt` (or `apps-for-others.txt`) through `winget` one at a time, and work through `windows/windows-group-policy-settings.md` manually in `gpedit.msc` — Group Policy isn't scriptable from a plain file.

## Requirements

- Linux side: systemd-resolved and NetworkManager (Fedora KDE defaults); `dnscrypt-proxy` only if using the `proxy-dns/` variant; chrony 4.6+ for NTS.
- Windows side: `winget`; Pro/Enterprise/Education edition for `gpedit.msc`.

## Notes

- `dns=none` (in `90-disable-nm-dns.conf`) means nothing writes `/etc/resolv.conf` — point it at the stub yourself (symlink to `/run/systemd/resolve/stub-resolv.conf`) or name resolution stops at the next reboot.
- Both DNS stacks depend on a correct clock for TLS, and time sync depends on DNS — a circular dependency covered by the `time-sync-rescue.md` runbooks and by `chronyd -s`, which floors the clock at boot from the drift file.
- `tuned` applies its profile after `systemd-sysctl` at boot and can override values from `99-net-transport-optimization.conf`; re-check after a full reboot if something regresses.

## Licensing

`windows/windows-group-policy-settings.md` is a derivative of the [Privacy Guides Group Policy Settings guide](https://www.privacyguides.org/en/os/windows/group-policies/), licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) per the attribution header in that file. Everything else in this repo has no separate license file.
