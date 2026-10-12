
# Enterprise SOC Lab

A hands-on cybersecurity lab designed to simulate attacks, investigate security events, and develop detection capabilities using Wazuh SIEM, Sysmon, and Suricata.

The project focuses on practical SOC operations, including endpoint monitoring, network traffic analysis, threat detection, and security alert investigation.

## Lab Architecture

| Component | Purpose |
|---|---|
| Windows 11 | Monitored endpoint / simulated victim |
| Kali Linux | Controlled attack simulation |
| Ubuntu Server | Wazuh SIEM, event collection, and analysis |
| Sysmon | Windows process and network telemetry |
| Suricata | Network inspection and custom detection rules |
| VMware | Virtualized lab environment |

All attack simulations are performed in an isolated, authorized lab using test data.

## Key Achievements

- Deployed a Wazuh SIEM environment with a connected Windows endpoint.
- Integrated Sysmon for process-creation and network-connection monitoring.
- Enabled Wazuh archive indexing for investigation of events that do not generate alerts.
- Implemented a controlled phishing and credential-submission simulation.
- Identified an initial detection gap: network connections were recorded, but credential submissions were not identified.
- Integrated Suricata with Wazuh and developed a custom rule detecting possible credential submissions over unencrypted HTTP.
- Successfully generated and investigated the resulting security alert in Wazuh Threat Hunting.

## Detection Engineering Highlight

**Scenario:** A simulated user submits dummy credentials through an unencrypted HTTP login form.

**Detection:** A custom Suricata rule identifies an HTTP POST request containing the tested password-field pattern.

**Result:** The detection was forwarded to Wazuh and appeared as a security alert.

- Suricata signature: `1000001`
- Wazuh rule: `86601`
- Current Wazuh alert level: `3`

This exercise demonstrates the progression from basic telemetry collection to application-layer detection and SIEM investigation.

## Documentation

### Environment Setup
- [Sysmon Configuration](docs/environment/Sysmon-Configuration.md)
- [Suricata Integration](docs/environment/Suricata-Integration.md)
- [Wazuh Server Deployment](docs/environment/Wazuh-Server-Deployment.md)
- [Wazuh Agent Deployment](docs/environment/Wazuh-Agent-Deployment.md)

### Attack Simulations and Investigations
- [BEC / Phishing Simulation](docs/attack-scenarios/BEC-Simulation.md)
- [HTTP Credential Exposure Detection](docs/attack-analysis/HTTP-Credential-Exposure-Detection.md)

## Project Status

**Completed:** SIEM deployment, endpoint telemetry collection, network inspection, controlled phishing simulation, custom HTTP credential detection, and Wazuh alert verification.

**In progress / Planned:**
- Improve alert prioritization and reduce false positives.
- Configure automatic Suricata background monitoring.
- Expand behavioral detection with PowerShell, persistence, and simulated command-and-control scenarios.
- Explore Active Directory monitoring and AI-assisted alert triage.

## Disclaimer

This project is intended exclusively for cybersecurity education, authorized security testing, and defensive research. All attack simulations are conducted in a controlled lab environment using test credentials.
