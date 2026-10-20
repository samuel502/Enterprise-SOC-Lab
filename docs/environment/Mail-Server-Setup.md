# Mail Server Setup

## Overview

This document describes the local mail infrastructure built to deliver the phishing email realistically, and the authentication controls configured to inspect it.

Everything runs on the Kali attacker VM inside the isolated lab network. No public mail relay is involved, so the DNS records that govern SPF, DKIM and DMARC stay fully under our control.

The guide is written to be followed from a standing start. Every command is explained, and every step says what you should see before you move on.

## Why a Local Mail Server

Sending through a public provider such as Gmail means borrowing that provider's domain and DNS records. You cannot change their SPF or DKIM configuration, so there is nothing to analyse.

A local server allows:

- Full control of SPF, DKIM and DMARC records
- Delivery to a mailbox we own
- Repeatable, inspectable results
- No dependency on internet mail routing

## What You Will Build

- A mail server that sends and receives mail for the lab domains
- A mailbox for the victim user
- A DKIM signing key, with its public half published in DNS
- Two filters that inspect incoming mail and stamp a verdict onto every message
- A mail client on the Windows VM, so the victim's inbox can be opened

## Three Concepts You Need First

These three are what the whole email phase is about. You only need the summary here; the analysis lives in the separate SPF/DKIM/DMARC document.

**SPF** — a DNS record listing which servers are allowed to send mail for a domain. When mail arrives, the receiving server compares the sending IP against that list. Pass means the IP is authorised; fail means it is not.

**DKIM** — a cryptographic signature added to the outgoing message by the sending server. The matching public key is published in DNS. The receiving server checks the signature, which proves the message was not altered in transit and that it came from someone holding the private key.

**DMARC** — a DNS record that ties the two together. It says "for my domain, SPF and DKIM must pass, and here is what to do when they don't" — do nothing (`p=none`), quarantine (`p=quarantine`), or reject (`p=reject`).

The point of the exercise: all three confirm that a domain is *authorised*. None of them confirm that a domain is *genuine*. A lookalike domain under the attacker's control passes all three.

## Architecture

| Component | Role | Port |
|---|---|---|
| Postfix | Mail transfer agent (send and receive) | 25 |
| Dovecot | IMAP server for the victim mailbox | 143 |
| OpenDKIM | DKIM signing and verification | 8891 |
| OpenDMARC | SPF and DMARC verification | 8893 |
| dnsmasq | DNS, including SPF and DKIM TXT records | 53 |
| Thunderbird (Windows) | Victim mail client | - |

## How to Read the Commands

- **Kali** means the Kali Linux virtual machine. **Windows** means the Windows 11 virtual machine. Run each command on the machine named in the heading.
- On Kali, open a terminal with `Ctrl + Alt + T`.
- A line starting with `sudo` runs the command as administrator. The first time you use it in a session, Kali asks for your password. **The password does not appear on screen as you type it** — that is normal.
- When a command spans several lines, the trailing `\` means "this command continues on the next line". You can paste the whole thing at once.
- Where a command produces output, the guide says what you should see. Compare before continuing.

## Build Steps

### Step 1 — Prerequisites

The DNS work must be finished first. Postfix resolves SPF and DKIM records through the system resolver, so if dnsmasq is not running and reachable on loopback, every authentication check fails for the wrong reason. Confirm:

```bash
dig @127.0.0.1 northwlndtraders.com +short
```

Expected: `192.168.23.130`. If this returns nothing, go back to the DNS document before continuing.

### Step 2 — Install the packages

```bash
sudo apt update
sudo apt install -y postfix dovecot-core dovecot-imapd
```

- **postfix** — the mail server
- **dovecot-core**, **dovecot-imapd** — the IMAP server that lets a mail client read the mailbox

Postfix opens a full-screen configuration wizard during install. It asks for a "General mail configuration type". Choose **No configuration** and press Enter.

**Why:** choosing one of the other presets writes a full config file for a normal internet mail server. We are building something custom, so it is cleaner to start near-empty and set every option deliberately in step 5. If you already installed Postfix and picked a preset, you can re-run the wizard with `sudo dpkg-reconfigure postfix`.

Now the filters:

```bash
sudo apt install -y opendkim opendkim-tools
```

- **opendkim** — the signing and verification filter
- **opendkim-tools** — contains `opendkim-genkey`, used to create the signing key

```bash
sudo apt install -y opendmarc
```

During install, OpenDMARC asks whether to configure a database with `dbconfig-common`. Answer **No**. The database is only needed for reporting; verdicts work without it.

### Step 3 — Create the victim mailbox user

```bash
sudo useradd -m -s /bin/bash sarah
```

Creates a Linux user called `sarah`. `-m` creates the home directory, `-s /bin/bash` gives her a normal shell.

```bash
sudo passwd sarah
```

Sets her password. Type it twice — again, nothing appears on screen while typing.

Postfix will deliver mail into `/home/sarah/Maildir/`, and Dovecot will serve that same directory over IMAP. The two must agree on the location, which is why step 9 matters.

### Step 4 — Map the victim email address onto that user

Mail arrives addressed to `sarah.mitchell@contoso.com`, but the Linux user is just `sarah`. Something has to translate one into the other.

```bash
sudo nano /etc/postfix/virtual
```

```conf
sarah.mitchell@contoso.com    sarah@contoso.com
```

Save with `Ctrl + O`, `Enter`, then `Ctrl + X`.

**Why not `/etc/aliases`?** That file only accepts plain local names. A domain-qualified alias like `sarah.mitchell@contoso.com` is rejected with the error `name must be local and have no domain`. The `virtual_alias_maps` file exists for exactly this case.

A plain text file is not enough — Postfix needs a lookup database built from it:

```bash
sudo postmap /etc/postfix/virtual
```

This reads the file and writes `/etc/postfix/virtual.lmdb` beside it. **Every time you edit `/etc/postfix/virtual`, you must re-run `postmap`**, or Postfix keeps using the old copy.

### Step 5 — Configure Postfix

Postfix is configured with `postconf -e`, which sets one option at a time in `/etc/postfix/main.cf` and keeps the file's formatting intact. Run these one by one:

```bash
sudo postconf -e "myhostname = mail.northwlndtraders.com"
sudo postconf -e "mydomain = northwlndtraders.com"
sudo postconf -e "myorigin = \$mydomain"
sudo postconf -e "inet_interfaces = all"
sudo postconf -e "mydestination = \$myhostname, northwlndtraders.com, contoso.com, kali, localhost.localdomain, localhost"
sudo postconf -e "home_mailbox = Maildir/"
sudo postconf -e "mynetworks = 127.0.0.0/8, 192.168.23.0/24"
sudo postconf -e "virtual_alias_maps = lmdb:/etc/postfix/virtual"
sudo postconf -e "smtpd_milters = inet:localhost:8891, inet:localhost:8893"
sudo postconf -e "non_smtpd_milters = inet:localhost:8891, inet:localhost:8893"
```

Note the `\$` in the third and fifth commands. `$mydomain` and `$myhostname` are Postfix variables, not shell variables. Inside double quotes the shell would expand them to nothing, so the backslash escapes them and passes the literal text through.

#### What each setting does

| Setting | What it does |
|---|---|
| `myhostname` | The name this server calls itself in mail headers. |
| `mydomain` | The domain this server is responsible for. |
| `myorigin` | What gets appended to bare usernames so they become complete addresses. |
| `inet_interfaces = all` | Listen for mail on every network card. |
| `mydestination` | Addresses matching these are delivered **locally**. This must include both the server's own domain and the victim domain, or Postfix will try to relay instead of deliver. |
| `home_mailbox = Maildir/` | Deliver into `Maildir/` inside each user's home directory, rather than the old single-file `mbox` format. |
| `mynetworks` | Which addresses are trusted to relay mail without authenticating. |
| `virtual_alias_maps` | The address translation file from step 4. `lmdb:` selects the database format. |
| `smtpd_milters` | Filters to run on mail arriving from the network. OpenDKIM is listed first so OpenDMARC can use its result. |
| `non_smtpd_milters` | The same filters, for mail injected locally rather than over the network. |

`lmdb` is used instead of the older `hash`. Postfix prints `hash/btree support is deprecated` on every message otherwise.

Confirm the settings actually landed:

```bash
sudo postconf -n
```

This prints every option that differs from the default. Read it and check that `mydestination` contains `contoso.com` and that `virtual_alias_maps` is set. **Do this before testing.** A `postconf -e` batch can appear to run cleanly and still not take effect, and the failure is silent.

Restart Postfix to apply:

```bash
sudo systemctl restart postfix
```

### Step 6 — Generate the DKIM signing key

This creates a public/private key pair. The private key stays on the server and signs outgoing mail; the public key is published in DNS so recipients can verify it.

```bash
sudo mkdir -p /etc/opendkim/keys/northwlndtraders.com
sudo opendkim-genkey -b 2048 -d northwlndtraders.com -s mail -D /etc/opendkim/keys/northwlndtraders.com
```

- `-b 2048` — key size, 2048 bits
- `-d` — the domain the key belongs to
- `-s mail` — the selector. This is a label for the key, and it becomes part of the DNS name
- `-D` — the directory to write into

Two files are created: `mail.private` (keep secret) and `mail.txt` (the public key, to be published).

```bash
sudo chown -R opendkim:opendkim /etc/opendkim/keys
sudo chmod 600 /etc/opendkim/keys/northwlndtraders.com/mail.private
```

The OpenDKIM service runs as its own unprivileged user and will not be able to read the key otherwise. The private key is restricted to that user only.

Look at the public key:

```bash
sudo cat /etc/opendkim/keys/northwlndtraders.com/mail.txt
```

It looks like `mail._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; p=MIIBIjANBg..." )`. The long `p=` value is the key itself.

### Step 7 — Publish the key and the victim domain in DNS

Add to `/etc/dnsmasq.d/lab.conf`:

```conf
address=/contoso.com/192.168.23.130
mx-host=contoso.com,mail.contoso.com,10

txt-record=mail._domainkey.northwlndtraders.com,"v=DKIM1; h=sha256; k=rsa; p=<PASTE_THE_p_VALUE>"
```

- The `address=` and `mx-host=` lines make the victim domain resolve to the lab server, so mail to `contoso.com` is delivered here instead of being sent to the internet.
- The `txt-record` publishes the public half of the key from step 6, under the name `<selector>._domainkey.<domain>`.

Copy the `p=` value out of `mail.txt` and paste it between the quotes. Remove the line breaks that appear in the file — the whole value has to be one continuous string.

Apply the change:

```bash
sudo systemctl restart dnsmasq
```

Check it resolves:

```bash
dig @127.0.0.1 mail._domainkey.northwlndtraders.com TXT +short
dig @127.0.0.1 contoso.com +short
```

Both must return something. If the first is empty, the record is malformed — check the quotes and that the `p=` value has no stray spaces.

### Step 8 — Configure OpenDKIM

```bash
sudo nano /etc/opendkim.conf
```

Add or set:

```conf
Mode            sv
Socket          inet:8891@localhost
Domain          northwlndtraders.com
Selector        mail
KeyFile         /etc/opendkim/keys/northwlndtraders.com/mail.private
InternalHosts   192.168.23.0/24, 127.0.0.1, ::1
LogWhy          yes
```

| Setting | What it does |
|---|---|
| `Mode sv` | Sign outgoing mail, verify incoming mail. |
| `Socket` | Listen on TCP port 8891 on loopback. This must match the port in the Postfix `milters` settings. |
| `Domain` / `Selector` / `KeyFile` | Which key to sign with. |
| `InternalHosts` | Which senders to trust. **Anything not listed is treated as external and skipped.** |
| `LogWhy` | Write the reason for each decision to the log. Invaluable when signing silently does not happen. |

**Note:** `/etc/default/opendkim` is a legacy file that the systemd service ignores. Setting the socket there has no effect. It must go in `/etc/opendkim.conf`, which is the file the service actually reads.

### Step 9 — Configure OpenDMARC

```bash
sudo nano /etc/opendmarc.conf
```

```conf
Socket inet:8893@localhost
AuthservID mail.northwlndtraders.com
RejectFailures false
SPFSelfValidate true
```

| Setting | What it does |
|---|---|
| `Socket` | Listen on TCP port 8893. Must match the Postfix `milters` setting. |
| `AuthservID` | The name stamped into the `Authentication-Results` header, identifying who did the checking. |
| `RejectFailures false` | Report a verdict instead of blocking. Failed mail still gets delivered, so it can be inspected. |
| `SPFSelfValidate true` | Do the SPF check internally, producing both an SPF and a DMARC result. |

### Step 10 — Configure Dovecot

Dovecot 2.4 renamed several settings. The old `mail_location` no longer exists and has been split in two.

In `/etc/dovecot/conf.d/10-mail.conf`:

```conf
mail_driver = maildir
mail_path = %{home}/Maildir
mail_inbox_path = %{home}/Maildir
```

In the auth settings (`/etc/dovecot/conf.d/10-auth.conf`):

```conf
auth_allow_cleartext = yes
```

And in the SSL settings:

```conf
ssl = no
```

**Why:** Dovecot's default is the old `mbox` format at `~/mail`, while Postfix is delivering `Maildir` at `~/Maildir`. If the two disagree, the mailbox appears permanently empty even though mail was delivered successfully.

`auth_allow_cleartext = yes` is needed because there is no TLS certificate in the lab. In 2.4 the old `disable_plaintext_auth` was replaced by this inverted setting.

### Step 11 — Start everything

```bash
sudo systemctl enable --now opendkim opendmarc dovecot
sudo systemctl restart postfix
```

`enable` makes each service start at boot; `--now` starts it immediately. Postfix is restarted last so it picks up the milter sockets, which now exist.

Check all four:

```bash
systemctl is-active postfix dovecot opendkim opendmarc
```

All four must print `active`. Check the sockets are listening:

```bash
sudo ss -lntp | grep -E '8891|8893'
```

You should see both ports. If either is missing, the corresponding service failed to start — read `journalctl -u opendkim` or `journalctl -u opendmarc`.

### Step 12 — Test local delivery

Prove mail gets delivered and that authentication headers are produced, before involving a mail client.

```bash
sudo apt install -y swaks
swaks --server 127.0.0.1 \
      --to sarah.mitchell@contoso.com \
      --from accounts@northwlndtraders.com \
      --h-From 'Northwind Traders Accounts <accounts@northwlndtraders.com>' \
      --h-Subject 'Invoice NW-4471 overdue'
```

`swaks` is a scriptable mail sender. Note the two separate address flags: `--from` sets the envelope sender (the `Return-Path`) and `--h-From` sets the visible `From` header. They must be given separately — `--from` does not parse a display name, and putting one there produces a mangled envelope address.

Look at the log:

```bash
journalctl -u postfix -n 40 --no-pager
```

Expected: a `status=sent` line naming the recipient. Kali has no `rsyslog`, so `/var/log/mail.log` does not exist — `journalctl` is the way to read the mail log here.

Check the mail landed:

```bash
sudo ls /home/sarah/Maildir/new/
```

`sudo` is required because the home directory is mode 700. There should be one file. Read it and look for the headers:

```bash
sudo cat /home/sarah/Maildir/new/*
```

Expected: an `Authentication-Results` header reading `spf=pass` and `dmarc=pass`, and a `DKIM-Signature` header. The signature proves OpenDKIM signed it; the verdicts prove OpenDMARC checked it.

Check the signature is cryptographically valid:

```bash
sudo opendkim-testmsg < /home/sarah/Maildir/new/<filename>
```

No output means the signature verifies correctly.

### Step 13 — Connect the victim mail client

On the **Windows** VM. The VM has no internet access, so fetch the installer from Kali first.

On Kali:

```bash
sudo cp ~/Downloads/thunderbird*.exe /var/www/html/
```

On Windows, browse to `http://192.168.23.130/thunderbird-<version>.exe` and install it.

Then in Thunderbird, choose **Manual configuration** and enter:

| Field | Value |
|---|---|
| Protocol | IMAP |
| Hostname | `192.168.23.130` |
| Port | `143` |
| Connection security | None |
| Authentication | Normal password |
| Username | `sarah` |

Accept the certificate warning when it appears. The inbox should now show the test message.

## Issues Encountered

Each of these was hit and fixed during the build. They are the parts worth reading.

### Berkeley DB deprecation

Postfix emitted `hash/btree support is deprecated` on every message. Switched `virtual_alias_maps` to `lmdb`.

### Config not applying

An initial `postconf -e` batch silently did not take, leaving `mydestination` at its debconf default. Caught by verifying with `postconf -n` before testing rather than assuming.

### Mail bounced instead of delivering

The virtual alias rewrote to a bare username, which Postfix appended `$myorigin` to, producing an address outside `mydestination`. Postfix then tried to relay it and failed with `Host or domain name not found`. Fixed by adding the server's own domain to `mydestination` and rewriting the alias to a fully-qualified local address.

### Kali could not resolve lab domains

`/etc/resolv.conf` pointed at the VMware host gateway, which has no knowledge of the lab domains. Postfix therefore could not resolve `northwlndtraders.com` at all. Fixed by pointing the resolver at dnsmasq.

### Dovecot defaulted to mbox

Dovecot 2.4's default is mbox at `~/mail`, while Postfix writes Maildir at `~/Maildir`. The mailbox appeared empty until both were aligned.

### opendkim returned 421 to all mail

`/etc/default/opendkim` is a legacy file that the systemd unit ignores, so the `SOCKET` line had no effect. opendkim was listening on a Unix socket while Postfix was pointed at a TCP port, so every connection failed. Fixed by setting `Socket` in `/etc/opendkim.conf`, which is the file the unit actually reads.

### opendkim accepted mail but did not sign

The journal showed `external host [192.168.23.130] attempted to send as northwlndtraders.com`. The default trust list covers loopback only, so the sender was treated as external and skipped. Fixed with an `InternalHosts` entry covering the lab subnet.

## Verification

```bash
dig @127.0.0.1 mail._domainkey.northwlndtraders.com TXT +short
dig @127.0.0.1 northwlndtraders.com TXT +short
sudo doveadm auth test sarah Password123
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

## Related

- [DNS Setup](DNS-Setup.md) — the resolver and TXT records this build depends on
- [SPF / DKIM / DMARC Analysis](../attack-analysis/SPF-DKIM-DMARC-Analysis.md)
- [BEC / Phishing Simulation](../attack-scenarios/BEC-Simulation.md)
