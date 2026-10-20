# Mail Server Setup

## Overview

This document describes the local mail infrastructure built to deliver the phishing
email realistically, and the authentication controls configured to inspect it.

Everything runs on the Kali attacker VM inside the isolated lab network. No public mail
relay is involved, so the DNS records that govern SPF, DKIM and DMARC stay fully under
our control.

## Why a Local Mail Server

Sending through a public provider (Gmail, for example) means borrowing that provider's
domain and DNS records. You cannot change their SPF or DKIM configuration, so there is
nothing to analyse.

A local server allows:

- Full control of SPF, DKIM and DMARC records
- Delivery to a mailbox we own
- Repeatable, inspectable results
- No dependency on internet mail routing

## Architecture

| Component | Role | Port |
|---|---|---|
| Postfix | Mail transfer agent (send and receive) | 25 |
| Dovecot | IMAP server for the victim mailbox | 143 |
| OpenDKIM | DKIM signing and verification | 8891 |
| OpenDMARC | SPF and DMARC verification | 8893 |
| dnsmasq | DNS, including SPF and DKIM TXT records | 53 |
| Thunderbird (Windows) | Victim mail client | - |

## DNS Records Required

Added to `/etc/dnsmasq.d/lab.conf`:

```
address=/contoso.com/192.168.23.130
mx-host=contoso.com,mail.contoso.com,10
txt-record=mail._domainkey.northwlndtraders.com,"v=DKIM1; h=sha256; k=rsa; ","p=..."
```

The MX record directs mail for the victim domain at the lab server. The DKIM TXT record
publishes the public half of the signing key.

dnsmasq was also configured to answer on loopback (`listen-address=127.0.0.1`) so the
local Postfix could resolve lab domains. Kali's own resolver was pointed at dnsmasq via
NetworkManager:

```
sudo nmcli con mod "Wired connection 1" ipv4.dns "127.0.0.1"
sudo nmcli con mod "Wired connection 1" ipv4.ignore-auto-dns yes
sudo nmcli con up "Wired connection 1"
```

This matters for a reason beyond convenience: SPF and DKIM verification work by looking
up TXT records. If the mail server cannot resolve lab domains, every authentication
check fails for the wrong reason.

## Postfix Configuration

```
myhostname = mail.northwlndtraders.com
mydomain = northwlndtraders.com
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, northwlndtraders.com, contoso.com, kali, localhost.localdomain, localhost
home_mailbox = Maildir/
mynetworks = 127.0.0.0/8, 192.168.23.0/24
virtual_alias_maps = lmdb:/etc/postfix/virtual
smtpd_milters = inet:localhost:8891, inet:localhost:8893
non_smtpd_milters = inet:localhost:8891, inet:localhost:8893
```

Key points:

- `mydestination` includes both the server's own domain and the victim domain, so mail
  to either is delivered locally rather than relayed
- `virtual_alias_maps` maps the full victim address onto a local system user
- `smtpd_milters` chains OpenDKIM before OpenDMARC, so DMARC can use the DKIM result
- `lmdb` is used instead of `hash` to avoid Berkeley DB deprecation warnings

### Virtual Alias

`/etc/postfix/virtual`:

```
sarah.mitchell@contoso.com    sarah@contoso.com
```

A domain-qualified alias cannot go in `/etc/aliases` — that file only accepts local
names and emits `name must be local and have no domain`.

## Victim Mailbox

```
sudo useradd -m -s /bin/bash sarah
sudo passwd sarah
```

## Dovecot Configuration

Dovecot 2.4 changed several setting names. `mail_location` no longer exists; it is split
into `mail_driver` and `mail_path`.

```
mail_driver = maildir
mail_path = %{home}/Maildir
mail_inbox_path = %{home}/Maildir
auth_allow_cleartext = yes
ssl = no
```

The defaults are mbox at `~/mail`, which does not match Postfix's `Maildir/` delivery.
Without these changes the mailbox appears empty.

`auth_allow_cleartext = yes` is required because there is no TLS certificate, and
`disable_plaintext_auth` was replaced in 2.4 by the inverted `auth_allow_cleartext`.

## Authentication Verification (OpenDMARC)

```
Socket inet:8893@localhost
AuthservID mail.northwlndtraders.com
RejectFailures false
SPFSelfValidate true
```

`RejectFailures false` reports verdicts rather than blocking, so failed mail is still
delivered and can be inspected. `SPFSelfValidate` makes OpenDMARC perform the SPF check
itself, producing both SPF and DMARC results.

## DKIM Signing (OpenDKIM)

```
Mode            sv
Socket          inet:8891@localhost
Domain          northwlndtraders.com
Selector        mail
KeyFile         /etc/opendkim/keys/northwlndtraders.com/mail.private
InternalHosts   192.168.23.0/24, 127.0.0.1, ::1
LogWhy          yes
```

Key generated with:

```
sudo opendkim-genkey -b 2048 -d northwlndtraders.com -s mail -D /etc/opendkim/keys/northwlndtraders.com
```

## Issues Encountered

### Berkeley DB deprecation

Postfix emitted `hash/btree support is deprecated` on every message. Switched
`virtual_alias_maps` to `lmdb`.

### Config not applying

An initial `postconf -e` batch silently did not take, leaving `mydestination` at its
debconf default. Caught by verifying with `postconf -n` before testing rather than
assuming.

### Mail bounced instead of delivering

The virtual alias rewrote to a bare username, which Postfix appended `$myorigin` to,
producing an address outside `mydestination`. Postfix then tried to relay it and failed
with `Host or domain name not found`. Fixed by adding the server's own domain to
`mydestination` and rewriting the alias to a fully-qualified local address.

### Kali could not resolve lab domains

`/etc/resolv.conf` pointed at the VMware host gateway, which has no knowledge of the lab
domains. Postfix therefore could not resolve `northwlndtraders.com` at all. Fixed by
pointing the resolver at dnsmasq.

### Dovecot defaulted to mbox

Dovecot 2.4's default is mbox at `~/mail`, while Postfix writes Maildir at `~/Maildir`.
The mailbox appeared empty until both were aligned.

### opendkim returned 421 to all mail

`/etc/default/opendkim` is a legacy file that the systemd unit ignores, so the `SOCKET`
line had no effect. opendkim was listening on a Unix socket while Postfix was pointed at
a TCP port, so every connection failed. Fixed by setting `Socket` in `/etc/opendkim.conf`,
which is the file the unit actually reads.

### opendkim accepted mail but did not sign

The journal showed `external host [192.168.23.130] attempted to send as
northwlndtraders.com`. The default trust list covers loopback only, so the sender was
treated as external and skipped. Fixed with an `InternalHosts` entry covering the lab
subnet.

## Verification

```bash
dig @127.0.0.1 mail._domainkey.northwlndtraders.com TXT +short
dig @127.0.0.1 northwlndtraders.com TXT +short
sudo doveadm auth test sarah Password123
```

Signature validity confirmed with:

```bash
sudo opendkim-testmsg < messagefile
```

## Files

| File | Location |
|---|---|
| Postfix main config | `/etc/postfix/main.cf` |
| Virtual aliases | `/etc/postfix/virtual` |
| Dovecot mail config | `/etc/dovecot/conf.d/10-mail.conf` |
| OpenDKIM config | `/etc/opendkim.conf` |
| DKIM keys | `/etc/opendkim/keys/northwlndtraders.com/` |
| OpenDMARC config | `/etc/opendmarc.conf` |
