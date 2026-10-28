# Fixing Hostname (mDNS) Resolution for a Linux VM in VirtualBox

A fresh-install guide for making a Linux VM pingable by hostname (e.g. `myvm.local`) from the Windows/macOS host — resilient to DHCP address changes, with nothing hardcoded.

## The problem

You installed a Linux VM in VirtualBox with a **bridged adapter**, and `ping myvm.local` from the host fails. The VM gets its IP via DHCP (so the IP can change), and you don't want to hardcode it anywhere.

## Why this happens

- `.local` is the **mDNS** (multicast DNS) domain. Names like `myvm.local` are resolved via multicast UDP on port **5353**, *not* by your router's DNS.
- The VM must run an mDNS daemon (**avahi-daemon**) that publishes `<hostname>.local` → the VM's *current* IP, and re-publishes whenever DHCP changes the address.
- The VM's firewall must allow inbound UDP 5353.
- For the VM to resolve *its own* `.local` name, the glibc **nss-mdns** NSS module must be installed and listed in `/etc/nsswitch.conf`.
- The system hostname must be a **short** name (e.g. `myvm`), **not** `myvm.local` — a hostname ending in `.local` collides with the mDNS TLD and makes avahi publish the loopback address (`127.0.0.1` / `::1`) instead of the real IP.

Windows 10/11 and macOS resolve `.local` names via mDNS natively — no host-side install needed, once the VM side is correct.

## Prerequisites

1. **VirtualBox network = Bridged Adapter.** With NAT, the VM sits on a separate subnet and mDNS multicast cannot reach the host. Bridged puts the VM on your LAN so it gets a LAN IP and multicasts reach all hosts.
2. **Host network profile allows multicast (Windows):** if pings still fail after this guide, set the host's network adapter profile to **Private** (Public profiles can block mDNS multicast).
3. **No DHCP IP hardcoded anywhere** — we keep resolution fully dynamic.

## General procedure (any distro)

1. Set a short hostname: `sudo hostnamectl set-hostname myvm`
2. Install the mDNS stack: `avahi-daemon`, `avahi-tools` (verification), and `nss-mdns` (the NSS resolver).
3. Enable + start avahi: `sudo systemctl enable --now avahi-daemon`
4. Open the firewall for mDNS (UDP 5353).
5. Add `mdns` to the `hosts:` line in `/etc/nsswitch.conf`.
6. Restart avahi: `sudo systemctl restart avahi-daemon`
7. Verify (see [Verification](#verification)).

## Distro commands

### CentOS Stream / RHEL / Rocky / AlmaLinux (dnf)

`nss-mdns` lives in **EPEL** — enable EPEL first if not already.

```bash
# EPEL (CentOS Stream / Rocky / AlmaLinux)
sudo dnf install -y epel-release
# RHEL:
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E %rhel).noarch.rpm

sudo dnf install -y avahi-daemon avahi-tools nss-mdns
sudo systemctl enable --now avahi-daemon

# firewall (firewalld)
sudo firewall-cmd --add-service=mdns --permanent
sudo firewall-cmd --reload
```

### Fedora (dnf)

```bash
sudo dnf install -y avahi-daemon avahi-tools nss-mdns
sudo systemctl enable --now avahi-daemon
sudo firewall-cmd --add-service=mdns --permanent && sudo firewall-cmd --reload
```

### Debian (apt)

```bash
sudo apt update
sudo apt install -y avahi-daemon avahi-utils libnss-mdns
sudo systemctl enable --now avahi-daemon
# Debian's libnss-mdns postinst usually edits nsswitch.conf automatically — verify step 5.
# No firewall enabled by default on Debian server; if using nftables/ufw, see "Firewall backends".
```

### Ubuntu (apt)

```bash
sudo apt update
sudo apt install -y avahi-daemon avahi-utils libnss-mdns
sudo systemctl enable --now avahi-daemon
# ufw is often active on Ubuntu:
sudo ufw allow 5353/udp
```

### openSUSE Tumbleweed / Leap (zypper)

```bash
sudo zypper install -y avahi avahi-tools nss-mdns
sudo systemctl enable --now avahi-daemon
# firewalld is the default on modern openSUSE:
sudo firewall-cmd --add-service=mdns --permanent && sudo firewall-cmd --reload
```

### Arch Linux (pacman)

```bash
sudo pacman -S avahi nss-mdns avahi-utils
sudo systemctl enable --now avahi-daemon
# Arch usually has no firewall by default; if using ufw/firewalld/nftables, allow 5353/udp.
```

### Alpine Linux (apk) — partial support

```bash
sudo apk add avahi avahi-tools
sudo rc-update add avahi-daemon
sudo rc-service avahi-daemon start
# Allow 5353/udp in /etc/awall or your firewall.
```

> **Caveat:** Alpine uses **musl libc**, which does not support glibc NSS modules. `nss-mdns` does **not** work for *local* `.local` resolution on Alpine. avahi-daemon still **publishes** the VM's name so *other* hosts (Windows/macOS/other Linux) can resolve `myvm.local` — but on Alpine itself, resolve the VM via `/etc/hosts` or `mdns-scan`, not nss-mdns.

## Firewall backends (UDP 5353)

| Backend | Command |
|---|---|
| firewalld (RHEL/Fedora/openSUSE) | `sudo firewall-cmd --add-service=mdns --permanent && sudo firewall-cmd --reload` |
| ufw (Ubuntu/Debian) | `sudo ufw allow 5353/udp` |
| nftables | `sudo nft add rule inet filter input udp dport 5353 accept` |
| iptables | `sudo iptables -A INPUT -p udp --dport 5353 -j ACCEPT` |
| (no firewall) | nothing needed |

## nsswitch.conf

Edit the `hosts:` line to include `mdns`. The universally safe, verified form (append at the end — does not break normal DNS):

```
hosts:      files dns myhostname mdns
```

> Some guides suggest `files mdns_minimal [NOTFOUND=return] dns myhostname` for slightly faster `.local` lookups. Only use that if you're certain the bare hostname is resolvable via `files` (e.g. a `127.0.1.1 myvm` line, the Debian convention) — otherwise it can prevent non-`.local` names from resolving. The append form above is the safe default and was verified working (both `myvm.local` → real IP *and* `google.com` → resolved).

## /etc/hosts

Keep it free of the VM's real DHCP address. The defaults are fine:

```
127.0.0.1   localhost localhost.localdomain
::1         localhost localhost6
```

Optional (Debian-style) for a stable `hostname -f` without hardcoding the real IP:

```
127.0.1.1   myvm
```

**Do not** add `192.168.x.x myvm.local myvm` — that line goes stale the moment DHCP hands out a different address.

## Verification

On the VM:

```bash
hostname                       # → myvm  (short, no .local)
avahi-resolve -n myvm.local    # → <current-vm-ipv4>
getent ahostsv4 myvm.local     # → <current-vm-ipv4>
getent hosts google.com        # sanity: normal DNS still works
```

From the Windows host:

```
ping myvm.local                # resolves to the VM's current address (IPv6 link-local first)
ping -4 myvm.local             # force IPv4 → <current-vm-ipv4>
```

From a macOS host:

```bash
dscacheutil -q host -a name myvm.local    # → ip_address
ping myvm.local                            # macOS ping has no -4; resolve via dscacheutil / dns-sd
```

From another Linux host (with nss-mdns installed):

```bash
getent hosts myvm.local        # → <current-vm-ipv4>
ping myvm.local
```

## Making it DHCP-proof

Everything above is **dynamic**: avahi re-publishes the current interface address whenever DHCP changes it (it watches netlink `RTM_NEWADDR`/`RTM_DELADDR` events). No file references a fixed IP, so the hostname keeps resolving across lease renewals — on both the VM itself and the host.

For **zero** drift, add a **DHCP reservation** on your router for the VM's MAC address (recommended for a bridged VM you SSH into — keeps IP-based tooling and host keys stable). This is optional; name resolution keeps working without it.

## Troubleshooting

| Symptom | Check |
|---|---|
| `ping myvm.local` fails from host | Is the VirtualBox adapter **Bridged** (not NAT)? Is avahi running (`systemctl status avahi-daemon`)? Is UDP 5353 open on the VM firewall? |
| `avahi-resolve -n myvm.local` returns `127.0.0.1` / `::1` | Hostname collides with `.local`. Set a short hostname: `sudo hostnamectl set-hostname myvm`, then `sudo systemctl restart avahi-daemon`. |
| VM resolves `myvm.local` to a stale IP | The real IP is hardcoded in `/etc/hosts` — remove that line; rely on `mdns` in nsswitch. |
| `getent hosts google.com` fails after nsswitch edit | You used the `mdns_minimal [NOTFOUND=return]` form without a `127.0.1.1` hosts entry. Revert to `files dns myhostname mdns`. |
| Name works on the VM but not from the Windows host | Windows network profile may be **Public** (blocks multicast). Set to **Private**, or allow mDNS through Windows Firewall. |
| Alpine: `.local` doesn't resolve *on* Alpine | Expected — musl has no glibc NSS. avahi publishes for other hosts; use `/etc/hosts` or `mdns-scan` on Alpine itself. |