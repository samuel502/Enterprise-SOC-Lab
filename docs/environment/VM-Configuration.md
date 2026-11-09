# Lab Environment Configuration

## Overview
Three-VM architecture for BEC (Business Email Compromise) attack simulation and detection engineering in a SOC environment.

## Virtual Machine Specifications

### Windows 11 VM (Victim)
- **Hardware:**
  - RAM: 4 GB
  - CPU: 4 cores
  - Storage: 64 GB
  - Network: VMware NAT (192.168.23.0/24), `192.168.23.129`
- **Software:**
  - Windows 11 Pro
  - Thunderbird (victim mail client)
  - Sysmon (v15+)
  - Wazuh agent 4.14.8
  - Suricata 8.0.7 with Npcap 1.89
  - PowerShell 5.1+
- **Purpose:** Phishing target, credential harvesting victim

### Kali Linux VM (Attacker)
- **Hardware:**
  - RAM: 2 GB
  - CPU: 4 cores
  - Storage: 60 GB
  - Network: VMware NAT (192.168.23.0/24), `192.168.23.130`
- **Software:**
  - Kali Linux 2024.x
  - Python 3.x
  - Apache2 and PHP
  - dnsmasq (lookalike domain DNS)
  - Postfix, Dovecot, OpenDKIM and OpenDMARC (mail infrastructure)
  - Wireshark
  - tcpdump
- **Purpose:** Attack simulation, credential capture

### Ubuntu Server 22.04 LTS (Analysis)
- **Hardware:**
  - RAM: 6 GB
  - CPU: 4 cores
  - Storage: 100 GB
  - Network: VMware NAT (192.168.23.0/24), `192.168.23.134`
- **Software:**
  - Wazuh SIEM (v4.x): manager, indexer and dashboard (all-in-one install)
  - Filebeat
- **Purpose:** Log collection, alert monitoring, investigation

## Security Considerations
- **Snapshots**: No support - VMware Workstation Player does not provide snapshot capabilities
  - **Backup Strategy**: Manual VM folder copies before major changes
  - **Process**: Shut down VM completely, copy entire VM folder to backup location
- **Network Monitoring**: Yes - Wireshark and tcpdump available on Kali VM for traffic analysis
- **Credential Usage**: Confirmed - using only test/fake credentials
- **Traffic Encryption**: No - Phishing page uses HTTP (not HTTPS) for form submission
- **VM Isolation**: No - All VMs run on VMware NAT (192.168.23.0/24) and reach the internet through the host. This is fine for a detection lab using test credentials. A stricter host-only or isolated network would be the right choice when detonating real malware, so it cannot reach the internet.
- **Victim DNS**: The Windows VM's DNS points at the attacker's dnsmasq on Kali so the lookalike domain resolves. This is a lab stand-in for public registration of the domain; a real victim would resolve it through normal public DNS, not the attacker's server. See [DNS Setup](DNS-Setup.md).
