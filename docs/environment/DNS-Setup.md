# DNS Infrastructure (dnsmasq)

## Overview

This document describes the local DNS infrastructure built to support the phishing
simulation. It gives the campaign a realistic domain name instead of a raw IP address,
and provides the record types required for the later email authentication work.

The resolver runs on the Kali attacker VM and serves only the isolated lab network.

## Why a Local DNS Server Is Required

The lookalike domain used in this campaign is not registered on the public internet,
so public resolvers return NXDOMAIN for it. Two problems follow:

1. The phishing link `http://northwlndtraders.com/invoice/NW-4471` would not resolve.
2. SPF, DKIM and DMARC are published as DNS **TXT records**. Without a resolver under
   our control there is nothing to query, and email authentication cannot be
   demonstrated at all.

A local resolver solves both. It is also the mechanism real phishing infrastructure
uses — the only difference is that a real attacker registers the domain with a
registrar rather than hand-writing the entry.

## Domain Design

| Domain | Role |
|---|---|
| `northwlndtraders.com` | Attacker-owned lookalike. Differs from the legitimate domain by a single character (`i` → `l`). |
| `northwindtraders.com` | Simulated legitimate supplier domain. Deliberately not authorised for Kali's IP, so spoofed mail fails SPF. |
| `mail.northwlndtraders.com` | MX target for the lab mail server. |

The single-character difference is intentional. It demonstrates that a victim cannot
visually distinguish the two domains, and that header analysis is the only reliable
way to tell them apart.

## Architecture

| Component | Host | Function |
|---|---|---|
| dnsmasq | Kali Linux | Local authoritative answers + upstream forwarding |
| Resolver | Windows 11 | Configured to query Kali for all lookups |

![dnsmasq configuration](../screenshots/dnsmasq-config.png)

## Configuration

**File:** `/etc/dnsmasq.d/lab.conf`

```conf
# --- bind only to the host-only interface ---
interface=eth0
bind-interfaces
listen-address=192.168.23.130

# --- deterministic behaviour ---
no-hosts
no-resolv
server=1.1.1.1
server=8.8.8.8

# --- lab domains resolve to Kali ---
address=/northwlndtraders.com/192.168.23.130
address=/northwindtraders.com/192.168.23.130
address=/mail.northwlndtraders.com/192.168.23.130
mx-host=northwlndtraders.com,mail.northwlndtraders.com,10

# --- records for the SPF/DKIM/DMARC phase ---
txt-record=northwlndtraders.com,"v=spf1 ip4:192.168.23.130 -all"
txt-record=_dmarc.northwlndtraders.com,"v=DMARC1; p=none; rua=mailto:dmarc@northwlndtraders.com"

# the "real" supplier domain deliberately does NOT authorise Kali's IP
txt-record=northwindtraders.com,"v=spf1 ip4:203.0.113.10 -all"
txt-record=_dmarc.northwindtraders.com,"v=DMARC1; p=quarantine"
