# SPF, DKIM and DMARC Analysis

## Overview

This document analyses how the two phishing delivery variants fare against email
authentication. Both carry an identical message body and an identical link; only the
sending domain differs.

The purpose is to demonstrate what authentication actually catches, and what it does not.

## Method

Two messages were sent from the attacker host to the victim mailbox using `swaks`, and
the resulting `Authentication-Results` headers examined.

| | Variant A | Variant B |
|---|---|---|
| Display name | Northwind Traders - Accounts Receivable | Northwind Traders - Accounts Receivable |
| Envelope sender | accounts@northwindtraders.com | accounts@northwlndtraders.com |
| Header From | accounts@northwindtraders.com | accounts@northwlndtraders.com |
| Domain owned by attacker | no | yes |
| Body | identical | identical |

The only visible difference to a recipient is a single character: `northwind` versus
`northwlnd`.

## Controls Under Test

### SPF

SPF authorises which hosts may send mail for a domain. It is checked against the
**envelope** sender, and the result is published as a DNS TXT record.

| Domain | SPF record |
|---|---|
| northwlndtraders.com | `v=spf1 ip4:192.168.23.130 -all` |
| northwindtraders.com | `v=spf1 ip4:203.0.113.10 -all` |

The second record deliberately authorises a different IP, simulating a domain the
attacker does not control.

### DKIM

DKIM cryptographically signs the message so the recipient can verify it was not altered
and that it originated from a domain holding the private key.

A 2048-bit key was generated for `northwlndtraders.com` with selector `mail`, and the
public half published in DNS.

### DMARC

DMARC ties SPF and DKIM together with the visible `From:` header, requiring them to
**align**. It also publishes a handling policy.

| Domain | DMARC policy |
|---|---|
| northwlndtraders.com | `p=none` |
| northwindtraders.com | `p=quarantine` |

## Results

### Variant A - spoofed domain (not owned)

```
Return-Path: 
Authentication-Results: mail.northwlndtraders.com; spf=fail smtp.mailfrom=northwindtraders.com
Authentication-Results: mail.northwlndtraders.com; dmarc=fail (p=quarantine dis=none) header.from=northwindtraders.com
DKIM-Signature: (absent)
```

- **SPF fails** — the sending IP is not authorised by the domain's record
- **DKIM absent** — the attacker holds no signing key for this domain
- **DMARC fails** — no aligned pass, so the published policy applies

With `p=quarantine`, a real receiver would quarantine this message.

### Variant B - lookalike domain (owned)

```
Return-Path: 
Authentication-Results: mail.northwlndtraders.com; spf=pass smtp.mailfrom=northwlndtraders.com
Authentication-Results: mail.northwlndtraders.com; dmarc=pass (p=none dis=none) header.from=northwlndtraders.com
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=northwlndtraders.com; s=mail; ...
```

- **SPF passes** — the sending IP is authorised
- **DKIM passes** — signature valid against the published key
- **DMARC passes** — aligned pass, policy satisfied

Signature validity was independently confirmed:

```
sudo opendkim-testmsg < messagefile
```

### Summary

| Check | Variant A | Variant B |
|---|---|---|
| SPF | fail | pass |
| DKIM | none | pass |
| DMARC | fail (p=quarantine) | pass (p=none) |

## Analysis

The critical finding is that **authentication tells you nothing about whether the domain
is genuine**.

Variant B passes every check. Every control reports success. Yet the domain is a
one-character typo registered by the attacker. Authentication proves the message really
came from `northwlndtraders.com` — it does not prove `northwlndtraders.com` is who it
claims to be.

Variant A fails every check, because the attacker is claiming a domain whose DNS they do
not control. This is the case authentication was designed to catch.

So the two outcomes are:

- **Spoof an unowned domain** — caught, but likely filtered
- **Register a lookalike** — invisible to authentication, but passes every filter

The second is the more effective attack, which is why it is common in practice.

### Display name masking

Both variants use the identical display name. In a mail client's list view, only the
display name is shown, so the messages are indistinguishable until opened. The
distinguishing character is buried in the address field.

## Detection Considerations

### What authentication gives you

- A reliable verdict on whether the envelope domain is authorised
- Tamper detection via DKIM
- Policy enforcement for domains that publish DMARC

### What it does not give you

- Any judgement on whether a lookalike domain is legitimate
- Protection for a domain whose owner publishes no DMARC record
- Anything at all if the receiver does not check

### Recommended controls

- Publish DMARC at `p=quarantine` or `p=reject` on domains you own
- Monitor DMARC aggregate reports for unauthorised senders
- Consider lookalike-domain registration for high-value brand names
- Do not treat a passing authentication result as proof of legitimacy

## Limitations

- Both variants were sent from the same host, so SPF is evaluated against a single IP
- DMARC aggregate reporting was not configured, so no reports were generated
- The `dkim=` verdict does not appear in `Authentication-Results` because OpenDKIM signs
  and verifies in the same pass; the signature was confirmed valid out of band
- No TLS was configured, so transport encryption was not evaluated
