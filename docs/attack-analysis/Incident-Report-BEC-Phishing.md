# Incident Report — Credential Phishing (BEC Simulation)

| Field | Detail |
|---|---|
| Report ID | IR-2026-001 |
| Date of investigation | 26 October 2026 |
| Analyst | Samuel Nadel |
| Environment | Enterprise Mini-SOC Lab (isolated, authorized, test data only) |
| Affected host | WIN11-Client (192.168.23.129) |
| Attacker host | Kali Linux (192.168.23.130) |
| Detection source | Suricata → Wazuh (custom rule, signature ID 1000001) |
| Detection time | 2026-10-26 12:12:08 (local lab time) |
| Classification | True positive — credential submission over unencrypted HTTP |
| Severity | Suricata signature severity 1 (high); Wazuh rule level 3 (low — see Recommendations) |
| Status | Contained in lab; recommendations documented |

## Executive Summary

A simulated Business Email Compromise (BEC) phishing attack was executed end to end
against a monitored Windows endpoint and successfully detected by the SOC lab.

The attacker, operating from a Kali Linux host, sent a fraudulent supplier invoice email
to a user on the Windows endpoint. The email impersonated "Northwind Traders — Accounts
Receivable" and linked to a lookalike domain (`northwlndtraders.com`) hosting a cloned
Microsoft 365 login page. The user opened the email, followed the link, submitted
credentials, and was redirected to a fake invoice portal.

A custom Suricata rule detected the credential submission in transit — an HTTP POST
containing a password field over an unencrypted connection — and the alert was forwarded
to Wazuh, where it was investigated. The detection confirms the lab can identify
credential theft at the network layer, a gap that earlier endpoint-only monitoring
(Sysmon) could not cover.

## Timeline

All times are local lab time on 26 October 2026. The detection time is authoritative
(from the alert); the preceding steps are approximate operator actions leading up to it.

| Time | Event | Evidence source |
|---|---|---|
| ~12:10 | Phishing email sent from Kali to the victim mailbox | Kali Postfix / mail logs |
| ~12:11 | User opens the email and clicks the link to the lookalike domain | Victim action |
| 12:12:08 | User submits credentials on the fake login page (HTTP POST to `/login.php`) | Wazuh alert (sig 1000001) |
| 12:12:08 | Server responds with a 302 redirect to the fake invoice portal | Same alert (`http.redirect`) |

## Detection and Evidence

The detection fired on the credential submission and was recorded in Wazuh with the
following key fields.

| Field | Value | Meaning |
|---|---|---|
| `agent.name` / `agent.ip` | WIN11-Client / 192.168.23.129 | The victim endpoint |
| `data.dest_ip` : `data.dest_port` | 192.168.23.130 : 80 | Attacker server, over HTTP (unencrypted) |
| `data.app_proto` | http | Application-layer protocol inspected |
| `data.direction` | to_server | Data flowing from victim to attacker |
| `data.http.http_method` | POST | A form was submitted (the credentials) |
| `data.http.url` | /login.php | Attacker's server-side credential handler |
| `data.http.http_refer` | http://northwlndtraders.com/phishing.html | The fake login page the user came from |
| `data.http.hostname` | northwlndtraders.com | Lookalike domain (one character off the real one) |
| `data.http.status` | 302 | Server redirected the browser after capture |
| `data.http.redirect` | /invoice/NW-4471/portal.html | Redirect to the fake invoice portal |
| `data.http.http_user_agent` | Edge on Windows 10/11 | The victim's browser |
| `data.alert.signature` | SOC LAB - Possible credential submission over unencrypted HTTP | The custom rule that fired |
| `data.alert.signature_id` | 1000001 | Suricata signature ID |
| `data.alert.category` | Potential Corporate Privacy Violation | Suricata classification |

**Interpretation.** A single alert reconstructs the core of the attack: a user on
WIN11-Client submitted a login form over plain HTTP to an attacker-controlled server at a
lookalike domain, and was immediately redirected to a fake invoice page. The fake login
form (`phishing.html`) posts the credentials to `login.php`, a server-side handler on the
attacker's host that records them to a file and returns the 302 redirect to the fake
invoice portal. Because the traffic was unencrypted HTTP, the credentials were exposed in
transit — the basis of the detection.

Note on scope: the subsequent viewing of the invoice page is visible only as the 302
redirect target in this alert. It did not generate a separate alert, because the custom
rule triggers only on a credential submission (a POST containing a password field), not
on an ordinary page load.

Screenshots: `alert-document-details-1.png`, `alert-document-details-2.png`.

## Indicators of Compromise (IOCs)

| Type | Indicator | Note |
|---|---|---|
| Domain | `northwlndtraders.com` | Attacker-owned lookalike of `northwindtraders.com` |
| IP address | `192.168.23.130` | Attacker web/mail server (lab-internal) |
| URL path | `/phishing.html` | Cloned Microsoft 365 login page |
| URL path | `/login.php` | Server-side credential capture handler |
| URL path | `/invoice/NW-4471/portal.html` | Fake invoice portal (post-capture redirect) |
| Sender | `accounts@northwlndtraders.com` | Spoofed "Accounts Receivable" sender |
| Email subject | `Invoice NW-4471 - Payment Due 24 Oct` | Lure subject line |
| Signature ID | Suricata `1000001` | Detection that fired |

## Email Authentication Analysis (SPF, DKIM, DMARC)

The SOC did not have email-gateway telemetry (see Limitations), so the email artifacts
were analyzed separately on the mail infrastructure. Two delivery variants were tested
with an identical body and lookalike display name; only the sending domain differed.

| | Direct spoof of real domain | Attacker-owned lookalike |
|---|---|---|
| Sending domain | northwindtraders.com (not owned) | northwlndtraders.com (owned) |
| SPF | fail | pass |
| DKIM | none | pass |
| DMARC | fail (p=quarantine) | pass |

**Why this matters.** The direct spoof of the genuine domain fails authentication and
would be quarantined, because the attacker does not control that domain's DNS or signing
keys. The lookalike domain passes all three checks — not because it is trustworthy, but
because the attacker owns it and can publish valid SPF, DKIM, and DMARC records for it.
The key lesson: **SPF/DKIM/DMARC "pass" proves an email genuinely came from the sending
domain, not that the domain is legitimate.** Lookalike domains defeat authentication-based
filtering, which is why user-facing detection and header/domain scrutiny remain necessary.
The attack in this incident used the lookalike domain — the variant that passes every
check.

Full analysis: [SPF / DKIM / DMARC Analysis](SPF-DKIM-DMARC-Analysis.md).

## MITRE ATT&CK Mapping

| Tactic | Technique | Observed in this incident |
|---|---|---|
| Resource Development | T1583.001 Acquire Infrastructure: Domains | Registered/served the lookalike `northwlndtraders.com` |
| Initial Access | T1566.002 Phishing: Spearphishing Link | Invoice email with a link to the fake login page |
| Credential Access | T1056 Input Capture (web portal) | Cloned Microsoft 365 page captured submitted credentials |

## Impact Assessment

In this simulation the credentials were test values and no real account was compromised.
In a production equivalent, the impact would be the exposure of valid user credentials to
an attacker, enabling account takeover, mailbox access, and follow-on fraud (for example,
redirecting supplier payments). The detection occurred at the point of credential
submission, which in a real environment would allow a fast response before the
credentials are abused.

## Response and Recommendations

Immediate response actions (as they would apply in a production SOC):

1. **Isolate the host.** Contain WIN11-Client from the network to prevent any follow-on
   attacker activity, without powering it off (to preserve evidence).
2. **Force a password reset** for the affected user and invalidate active sessions, on the
   assumption the submitted credentials are compromised.
3. **Block the indicators.** Block the lookalike domain `northwlndtraders.com` and the
   attacker IP at the web proxy and firewall.
4. **Check for other affected hosts.** Search the SIEM for any other endpoints that
   contacted the same domain or IP, or received the same email. (Only one host exists in
   this lab; this is the production next step.)

Detection and hardening recommendations:

5. **Raise the alert severity in Wazuh.** The Suricata alert currently maps to Wazuh rule
   level 3 (low), which risks being overlooked. A custom local rule should raise this
   specific signature to a high level (e.g., 10–12) so it stands out, without raising all
   Suricata alerts and creating noise.
6. **Add email telemetry to the SIEM.** Ingesting mail-gateway or mail-server logs would
   bring sender, headers, and authentication verdicts into the SIEM, so phishing can be
   investigated in one place rather than on the mail host.

## Response Actions Performed (Lab Demonstration)

To validate the response playbook, the following actions were carried out in the lab.
Actions requiring infrastructure not present in the lab are noted as documented-only.

| # | Action | Status |
|---|---|---|
| 1 | Isolate the host | Documented only — trivial with a single VM; EDR isolation in production |
| 2 | Force password reset | Documented only — no identity system in lab yet; testable after Active Directory is added |
| 3 | Block the indicators | Performed |
| 4 | Check for other affected hosts | Performed |

**#3 — Blocking the attacker (performed).** A Windows Defender Firewall outbound rule was
created to block all traffic to the attacker host `192.168.23.130`:

```
New-NetFirewallRule -DisplayName "SOC-LAB Block Attacker 192.168.23.130" -Direction Outbound -RemoteAddress 192.168.23.130 -Action Block
```

After applying the rule, the phishing page was no longer reachable, confirming
containment. Note: because the attacker's DNS resolver (dnsmasq) and web server run on the
same host (`192.168.23.130`), blocking that IP also blocked name resolution, so the
browser reported a DNS resolution failure rather than a connection timeout. In a
production network, where DNS and the web server are separate systems, blocking the web
server IP would instead produce a connection timeout. The rule was removed after the
demonstration.

**#4 — Scoping the incident (performed).** To determine whether any other host was
affected, the SIEM was searched across all agents for the indicators of compromise — the
attacker IP (`data.dest_ip: 192.168.23.130`) and the lookalike domain
(`data.http.hostname: northwlndtraders.com`). Searching the indicator, rather than the
alert, finds any host that contacted the attacker, not only those that triggered a
detection. Only `WIN11-Client` (192.168.23.129) appeared, confirming no other host was
affected. The alert's Wazuh rule level was confirmed as 3 (low), supporting
recommendation #5.

Screenshots: `response-firewall-block.png`, `response-page-blocked.png`, `response-siem-scope.png`.

## Detection Engineering Notes

This incident is also a record of building and validating the detection itself.

- **Gap identified.** Endpoint monitoring (Sysmon) recorded the network connection to the
  attacker server but did not identify that credentials were submitted, because Sysmon
  does not inspect packet contents.
- **Control added.** Suricata was integrated with Wazuh and a custom rule (signature ID
  1000001) was written to flag HTTP POST requests containing a password field —
  i.e., credential submission over unencrypted HTTP.
- **Validation caveat.** During testing, Suricata reported "Engine started" but silently
  failed to log, because it lacked permission to write to its log files when not run with
  administrator privileges. Running Suricata elevated resolved it. Lesson: a sensor
  reporting "started" is not proof it is working; log output must be verified at the
  source (confirming `eve.json` was being written) before trusting the pipeline.

## Limitations and Next Steps

- **No email telemetry in the SIEM.** The mail server is not a SIEM data source in this
  lab, so email artifacts were analyzed separately. Production SOCs receive this from an
  email security platform.
- **Single host.** The lab contains one endpoint, so "check for other affected hosts"
  could not be exercised, only documented.
- **Sensor not resilient.** Suricata runs interactively and must be restarted manually
  after a reboot or VM resume. Configuring it to run as a service with health monitoring
  is a planned improvement.
- **Plaintext HTTP only.** The detection relies on unencrypted traffic. Over HTTPS the
  credentials would not be visible to Suricata; detection would then depend on endpoint
  or email-layer controls.

## Appendix — Detection Rule

The Suricata signature that produced this alert:

```
alert http any any -> any any (msg:"SOC LAB - Possible credential submission over unencrypted HTTP"; flow:established,to_server; http.method; content:"POST"; http.request_body; content:"password="; nocase; classtype:policy-violation; sid:1000001; rev:1;)
```

The rule matches any HTTP POST whose request body contains the literal `password=`
pattern. It does not require the destination IP to be known in advance, so it generalizes
to any unencrypted credential submission, not only this attacker.

---

*This exercise was conducted in an isolated, authorized lab environment using test
credentials, for defensive security education.*
