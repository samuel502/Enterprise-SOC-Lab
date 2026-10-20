# DNS Infrastructure (dnsmasq)

## Overview

This document describes the local DNS infrastructure built to support the phishing simulation. It gives the campaign a realistic domain name instead of a raw IP address, and provides the record types required for the later email authentication work.

The resolver runs on the Kali attacker VM and serves only the isolated lab network.

## Why a Local DNS Server Is Required

The lookalike domain used in this campaign is not registered on the public internet, so public resolvers return NXDOMAIN for it. Two problems follow:

1. The phishing link `http://northwlndtraders.com/invoice/NW-4471` would not resolve.
2. SPF, DKIM and DMARC are published as DNS **TXT records**. Without a resolver under our control there is nothing to query, and email authentication cannot be demonstrated at all.

A local resolver solves both. It is also the mechanism real phishing infrastructure uses — the only difference is that a real attacker registers the domain with a registrar rather than hand-writing the entry.

## Domain Design

| Domain | Role |
|---|---|
| `northwlndtraders.com` | Attacker-owned lookalike. Differs from the legitimate domain by a single character (`i` → `l`). |
| `northwindtraders.com` | Simulated legitimate supplier domain. Deliberately not authorised for Kali's IP, so spoofed mail fails SPF. |
| `contoso.com` | Victim domain. Resolves to the lab server so mail and the web lure stay on-network. |
| `mail.northwlndtraders.com` | MX target for the lab mail server. |

The single-character difference is intentional. It demonstrates that a victim cannot visually distinguish the two domains, and that header analysis is the only reliable way to tell them apart.

## Architecture

| Component | Host | Function |
|---|---|---|
| dnsmasq | Kali Linux | Local authoritative answers + upstream forwarding |
| Resolver | Windows 11 | Configured to query Kali for all lookups |

![dnsmasq configuration](../screenshots/dnsmasq-config.png)

## Build Steps

All commands run on the Kali attacker VM unless marked otherwise.

### 1. Recon before touching anything

Confirm the interface, address, whether anything already holds port 53, and that the web server is up.

```bash
ip -4 a
sudo ss -ulpn | grep ':53'
systemctl is-active apache2
```

Expected: `eth0` at `192.168.23.130`, no output from `ss` (port 53 free), `apache2` `active`.

### 2. Install

```bash
sudo apt update
sudo apt install -y dnsmasq dnsutils
```

dnsmasq installs as a **static unit and is not started automatically** on Kali. It has to be enabled explicitly.

### 3. Confirm the include directory is read

```bash
grep -n "conf-dir" /etc/dnsmasq.conf
```

The active line is `conf-dir=/etc/dnsmasq.d/,*.conf`. If it is missing:

```bash
echo 'conf-dir=/etc/dnsmasq.d/,*.conf' | sudo tee -a /etc/dnsmasq.conf
```

### 4. Write the lab configuration

```bash
sudo nano /etc/dnsmasq.d/lab.conf
```

```conf
# --- bind to the host-only interface and loopback only ---
interface=eth0
interface=lo
bind-interfaces
listen-address=192.168.23.130
listen-address=127.0.0.1

# --- deterministic behaviour ---
no-hosts
no-resolv
server=1.1.1.1
server=8.8.8.8

# --- lab domains resolve to Kali ---
address=/northwlndtraders.com/192.168.23.130
address=/northwindtraders.com/192.168.23.130
address=/mail.northwlndtraders.com/192.168.23.130
address=/contoso.com/192.168.23.130
address=/mail.contoso.com/192.168.23.130

# --- MX records ---
mx-host=northwlndtraders.com,mail.northwlndtraders.com,10
mx-host=contoso.com,mail.contoso.com,10

# --- SPF and DMARC records ---
txt-record=northwlndtraders.com,"v=spf1 ip4:192.168.23.130 -all"
txt-record=_dmarc.northwlndtraders.com,"v=DMARC1; p=none; rua=mailto:dmarc@northwlndtraders.com"

# the "real" supplier domain deliberately does NOT authorise Kali's IP
txt-record=northwindtraders.com,"v=spf1 ip4:203.0.113.10 -all"
txt-record=_dmarc.northwindtraders.com,"v=DMARC1; p=quarantine"

# --- DKIM public key, added during the mail phase ---
# value taken verbatim from /etc/opendkim/keys/northwlndtraders.com/mail.txt
txt-record=mail._domainkey.northwlndtraders.com,"v=DKIM1; h=sha256; k=rsa; p=<BASE64_PUBLIC_KEY>"
```

### 5. Start the resolver

```bash
sudo systemctl enable --now dnsmasq
sudo systemctl status dnsmasq --no-pager
```

If it fails to start, `journalctl -u dnsmasq -n 50 --no-pager` — in practice it is always a port 53 clash.

### 6. Verify resolution from Kali

```bash
dig @192.168.23.130 northwlndtraders.com +short
dig @192.168.23.130 northwlndtraders.com TXT +short
dig @192.168.23.130 northwindtraders.com TXT +short
dig @192.168.23.130 google.com +short
```

The first three must return the lab records and the last must return a public address, proving authoritative answers and upstream forwarding both work.

### 7. Point the victim endpoint at the resolver

On the Windows 11 VM, in an elevated command prompt:

```cmd
netsh interface ipv4 show interfaces
netsh interface ipv4 set dnsservers name="Ethernet0" static 192.168.23.130 primary
ipconfig /flushdns
```

`show interfaces` returns the interface index and exact name — substitute the real name if it is not `Ethernet0`. Windows caches negative answers, so the flush is required.

Confirm from the Windows side:

```cmd
nslookup northwlndtraders.com
nslookup google.com
```

### 8. Make the resolver answer on loopback

Added after the mail phase began. Postfix resolves SPF and DKIM records through the system resolver, so dnsmasq has to be reachable on loopback as well as on the lab interface.

```bash
echo "listen-address=127.0.0.1" | sudo tee -a /etc/dnsmasq.d/lab.conf
sudo systemctl restart dnsmasq
dig @127.0.0.1 northwlndtraders.com +short
```

### 9. Point Kali's own resolver at dnsmasq

```bash
nmcli -t -f NAME,DEVICE con show --active
```

Then, substituting the connection name returned above:

```bash
sudo nmcli con mod "Wired connection 1" ipv4.dns "127.0.0.1"
sudo nmcli con mod "Wired connection 1" ipv4.ignore-auto-dns yes
sudo nmcli con up "Wired connection 1"
```

`/etc/resolv.conf` is NetworkManager-managed, so editing it directly gets overwritten — the change has to go through nmcli.

Confirm:

```bash
cat /etc/resolv.conf
dig northwlndtraders.com +short
dig google.com +short
```

`nameserver 127.0.0.1`, the lab domain resolving, and `google.com` still working. If the last one breaks, forwarding died with it.

## Files

| File | Location |
|---|---|
| dnsmasq lab configuration | `/etc/dnsmasq.d/lab.conf` |
| dnsmasq main config | `/etc/dnsmasq.conf` |

## Related

- [Mail Server Setup](Mail-Server-Setup.md) — depends on the `contoso.com` and DKIM records defined here
