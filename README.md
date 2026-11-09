# Enterprise Mini-SOC Lab

A hands-on cybersecurity lab designed to simulate attacks, investigate security events, and develop detection capabilities using Wazuh SIEM, Sysmon, and Suricata.

The project focuses on practical SOC operations, including endpoint monitoring, network traffic analysis, threat detection, and security alert investigation.

## Featured: Incident Report

**[Incident Report: Credential Phishing (BEC Simulation)](docs/attack-analysis/Incident-Report-BEC-Phishing.md)** (IR-2026-001)

A simulated BEC attack, run end to end and investigated in Wazuh. A forged supplier invoice email from the lookalike domain `northwlndtraders.com` led the user to a cloned Microsoft 365 login page, and a custom Suricata rule caught the credential submission over unencrypted HTTP. The report covers the timeline, IOCs, MITRE ATT&CK mapping, response actions performed in the lab (blocking the attacker IP and scoping the SIEM by IOC), and the Wazuh severity tuning that came out of the investigation.

## Lab Architecture

| Component | Purpose |
|---|---|
| Windows 11 | Monitored endpoint / simulated victim |
| Kali Linux | Controlled attack simulation |
| dnsmasq | Attacker-controlled DNS: lookalike domain and forged SPF/DMARC records |
| Postfix + Dovecot | Attacker-controlled SMTP/IMAP mail infrastructure with DKIM signing |
| Ubuntu Server | Wazuh SIEM, event collection, and analysis |
| Sysmon | Windows process and network telemetry |
| Suricata | Network inspection and custom detection rules |
| VMware | Virtualized lab environment |

All attack simulations are performed in a controlled, authorized lab using test data.

## Key Achievements

- Deployed a Wazuh SIEM environment with a connected Windows endpoint.
- Integrated Sysmon for process-creation and network-connection monitoring.
- Enabled Wazuh archive indexing for investigation of events that do not generate alerts.
- Implemented a controlled phishing and credential-submission simulation.
- Identified an initial detection gap: network connections were recorded, but credential submissions were not identified.
- Integrated Suricata with Wazuh and developed a custom rule detecting possible credential submissions over unencrypted HTTP.
- Successfully generated and investigated the resulting security alert in Wazuh Threat Hunting.
- Identified that Wazuh mapped the alert to a low default level (3) and wrote a custom Wazuh rule (`100100`) raising that one signature to level 12.
- Wrote a formal incident report with timeline, IOCs, ATT&CK mapping and response actions performed in the lab.
- Stood up attacker-owned DNS and mail infrastructure, enabling a realistic email-delivered BEC scenario rather than a bare HTML page.
- Reproduced both email impersonation techniques — a lookalike domain and a direct domain spoof — and captured the resulting SPF, DKIM and DMARC verdicts for each.

## Detection Engineering Highlight

**Scenario:** A simulated user submits dummy credentials through an unencrypted HTTP login form.

**Detection:** A custom Suricata rule identifies an HTTP POST request containing the tested password-field pattern.

**Result:** The detection was forwarded to Wazuh and appeared as a security alert.

**Severity tuning:** Suricata rated the alert severity 1, its highest category, but Wazuh mapped it to its built-in Suricata rule `86601` at the default level 3 (low). That level was identified as too low for a credential-exposure alert, since it risks being overlooked in triage. A custom Wazuh rule (`100100`) was written as a child of `86601` that matches only signature `1000001` and raises it to level 12, so every other Suricata alert keeps its default severity. Retesting confirmed the alert now fires at level 12 under rule `100100`.

| | Before | After |
|---|---|---|
| Suricata signature | `1000001` | `1000001` |
| Wazuh rule | `86601` (built-in) | `100100` (custom) |
| Wazuh alert level | `3` | `12` |

This exercise demonstrates the progression from basic telemetry collection to application-layer detection, SIEM investigation and severity tuning. Rule and validation details are in the [incident report](docs/attack-analysis/Incident-Report-BEC-Phishing.md#detection-engineering-notes).

## Email Authentication Lab

A BEC scenario was built on top of the phishing page: a forged invoice email from a supplier, sent from attacker-controlled mail infrastructure, landing in a victim mailbox with full header visibility.

Two delivery variants were tested against the same payload:

| Variant | Sender address | SPF | DKIM | DMARC |
|---|---|---|---|---|
| Lookalike domain (attacker-owned) | `accounts@northwlndtraders.com` | pass | pass | pass |
| Direct spoof (unowned domain) | `accounts@northwindtraders.com` | fail | none | fail (p=quarantine) |

The lookalike variant passes everything because the attacker owns the domain, its DNS, and its DKIM key. The spoofed variant fails because the attacker does not own the domain and holds no signing key — the classic reason a legitimate-looking invoice lands in quarantine.

Full write-up, headers and screenshots: [BEC / Phishing Simulation](docs/attack-scenarios/BEC-Simulation.md).

## Documentation

### Environment Setup
- [Sysmon Configuration](docs/environment/Sysmon-Configuration.md)
- [Suricata Integration](docs/environment/Suricata-Integration.md)
- [Wazuh Server Deployment](docs/environment/Wazuh-Server-Deployment.md)
- [Wazuh Agent Deployment](docs/environment/Wazuh-Agent-Deployment.md)
- [DNS Setup](docs/environment/DNS-Setup.md)
- [Mail Server Setup](docs/environment/Mail-Server-Setup.md)
- [VM Configuration](docs/environment/VM-Configuration.md)
- [VM Troubleshooting](docs/environment/VM-Troubleshooting.md)

### Attack Simulations and Investigations
- [Incident Report: Credential Phishing (BEC Simulation)](docs/attack-analysis/Incident-Report-BEC-Phishing.md)
- [BEC / Phishing Simulation](docs/attack-scenarios/BEC-Simulation.md)
- [SPF / DKIM / DMARC Analysis](docs/attack-analysis/SPF-DKIM-DMARC-Analysis.md)
- [HTTP Credential Exposure Detection](docs/environment/Credential-Exposure-Detection.md)

## Project Status

**Completed:** SIEM deployment, endpoint telemetry collection, network inspection, controlled phishing simulation, custom HTTP credential detection, Wazuh alert verification, Wazuh severity tuning (custom rule `100100`, level 12), attacker-controlled DNS and mail infrastructure, a two-variant BEC simulation with full email authentication analysis, a detection baseline for the email-delivered attack, and a formal incident report with timeline, IOCs and ATT&CK mapping.

**In progress / Planned:**
- Reduce false positives and tune alert prioritization beyond the credential-submission rule.
- Configure automatic Suricata background monitoring.
- Bring mail server logs into the SIEM so the email stage can be investigated alongside the network alert.
- Expand behavioral detection with PowerShell, persistence, and simulated command-and-control scenarios.
- Explore Active Directory monitoring and AI-assisted alert triage.

## Disclaimer

This project is intended exclusively for cybersecurity education, authorized security testing, and defensive research. All attack simulations are conducted in a controlled lab environment using test credentials.
