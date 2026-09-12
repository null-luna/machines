# dotfiles

Modular post-install configs, system hardening, and cross-platform reference notes for Linux (Fedora KDE) and Windows. The focus is a privacy-hardened, fail-closed network stack: encrypted DNS end to end, authenticated (NTS) time synchronization, and telemetry reduction.

Files are drop-ins and reference material, not an installer. Nothing here runs itself — copy what you want to the paths listed under [Deployment](#deployment).

## Repository structure

```
common/    Cross-platform references (dnscrypt-proxy overrides, DNS/NTP server list)
linux/     Fedora KDE post-install: networking drop-ins, hardware notes
windows/   Windows post-install: GPO privacy settings, winget app lists, disk setup
```

### common/

- `dnscrypt-proxy-reference.txt` — Partial `dnscrypt-proxy.toml` (overrides from defaults only; not a standalone config). Selects DoH/DoH3 resolvers requiring DNSSEC, no-log, and no-filter policies, with commented listen addresses for both Linux and Windows.
- `privacy-dns-ntp-reference.md` — Public DNS resolvers with their encrypted-transport endpoints and ECS behavior, plus NTP servers split into NTS-capable non-smearing sources and leap-smearing sources. Covers Cloudflare, Google, and Quad9; the remaining resolvers referenced in `dnscrypt-proxy-reference.txt` (NextDNS, ControlD, AdGuard) are not yet documented here.

### linux/

- `minimal-postinstall.txt` — Fedora Everything (netinstall) package-selection notes for a minimal KDE base.
- `config-notes.md` — Hardware-specific notes: Jellyfin via Podman, Energy Efficient Ethernet disable, OBS encoder settings for both a discrete-AMD desktop and an Intel-iGPU laptop.

#### linux/networking/

Two alternative encrypted-DNS stacks are provided — deploy **one**, not both:

- `encrypted-dns/` — systemd-resolved speaks DNS-over-TLS **directly** to public resolvers (strict mode, fail-closed, no plaintext fallback). Twelve endpoints across Cloudflare, Quad9-unfiltered, and Google, IPv4 and IPv6, each as an `addr#SNI` tuple.
- `proxy-dns/` — systemd-resolved acts as a thin bridge to a **local dnscrypt-proxy** instance on `127.0.0.1:53`, which handles DoH/HTTP3, load balancing, and upstream DNSSEC (pair with `common/dnscrypt-proxy-reference.txt`).

Each variant ships a matching `time-sync-rescue.md` runbook for the cold-boot deadlock where a wrong clock breaks TLS validation before time can sync. The two runbooks differ: under `proxy-dns/` a bad clock breaks DNS resolution itself, so the resolver must be revived before chrony can even resolve its sources.

Shared pieces:

- `90-disable-nm-dns.conf` — NetworkManager drop-in that removes NM from the DNS path entirely (`dns=none` plus `systemd-resolved=false`, closing both the `resolv.conf` and D-Bus routes). NM still learns DHCP/RA nameservers but never applies them.
- `99-net-transport-optimization.conf` — sysctl drop-in: BBR + fq, MTU blackhole probing, `tcp_notsent_lowat`, ECN, keepalive tightening, and raised buffer ceilings for QUIC/HTTP-3 workloads. Heavily annotated with the observed Fedora 44 baseline for every value it changes, plus a list of knobs deliberately left alone.
- `nts/` — chrony configured for NTS-only, non-smearing time sync (`chrony.conf`, `authselectmode require`, five independent operators) plus the matching `/etc/sysconfig/chronyd` flags file (`-s -F 2`) that defends against the DNS↔time deadlock.

### windows/

- `windows-group-policy-settings.md` — Group Policy privacy hardening checklist (telemetry, Cloud Content, Recall/Copilot, OneDrive, Search, etc.). **Separately licensed — see below.**
- `apps.txt` / `apps-for-others.txt` — winget one-liners: a personal app set, and a broader-compatibility set (all VC++ redists, VLC, JDK) for machines set up for other people.
- `enable-utc.reg` — Store the RTC in UTC (`RealTimeIsUniversal`) for dual-boot clock sanity.
- `setup-partitions.bat` — **Destructive.** Wipes the hardcoded target disk and lays down a GPT layout (2 GiB ESP, MSR, 192 GiB Windows, 1 GiB Recovery). Read it and verify the disk number before running.
- `config-notes.md` — PotPlayer/LAV Filters setup, NTP server change, OBS encoder settings.

## Deployment

| File | Destination |
| --- | --- |
| `linux/networking/90-disable-nm-dns.conf` | `/etc/NetworkManager/conf.d/` |
| `linux/networking/99-net-transport-optimization.conf` | `/etc/sysctl.d/` |
| `linux/networking/encrypted-dns/90-dns-strict-policy.conf` | `/etc/systemd/resolved.conf.d/` |
| `linux/networking/proxy-dns/90-dns-bridge-policy.conf` | `/etc/systemd/resolved.conf.d/` |
| `linux/networking/nts/chrony.conf` | `/etc/chrony.conf` |
| `linux/networking/nts/chronyd` | `/etc/sysconfig/chronyd` |
| `common/dnscrypt-proxy-reference.txt` | merge into `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` |

Two things are easy to miss:

- `dns=none` means **nothing writes `/etc/resolv.conf`.** Point it at the stub yourself — symlink to `/run/systemd/resolve/stub-resolv.conf` — or name resolution stops at the first reboot.
- The initramfs carries a copy of the sysctl file. Run `sudo dracut --force` after editing it, and re-verify values after a **full** reboot: tuned applies its profile after systemd-sysctl and can override the file.

## Design notes

The whole stack is built around one circular dependency:

> Encrypted DNS needs valid TLS → TLS needs a correct clock → correcting the clock needs NTS → NTS needs DNS.

The defenses are layered and intentionally non-overlapping:

- `chronyd -s` floors the system clock at boot to the driftfile's mtime, putting certificates inside their validity window before any name is resolved.
- `nocerttimecheck 1` covers chrony's own NTS-KE TLS sessions — and *only* those, never the resolver's.
- `DNSSEC=no` in both resolved variants: local validation depends on accurate time, so signature-window failures are avoided by delegating validation upstream (to the DoT providers, or to dnscrypt-proxy's `require_dnssec`).
- `ntsdumpdir` persists NTS cookies across restarts to skip the key exchange, though cookies still cannot resolve hostnames.
- The `time-sync-rescue.md` runbooks are the manual break-glass, bootstrapping time from a plain-HTTP `Date` header so the fix path never depends on TLS.

Fail-closed behavior comes from three settings acting together in either DNS variant: an empty `FallbackDNS=` (no compiled-in plaintext escape hatch), `Domains=~.` (every query on every link is routed upstream), and NetworkManager being fully out of the DNS path. Remove any one and the stack silently degrades to plaintext instead of failing.

## Licensing

**`windows/windows-group-policy-settings.md`** is a derivative of the [Privacy Guides Group Policy Settings guide](https://www.privacyguides.org/en/os/windows/group-policies/) and is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), per the attribution header inside the file. If you redistribute or adapt that file, CC BY-SA 4.0 terms apply to it (attribution + share-alike).
