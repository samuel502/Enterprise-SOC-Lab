# Lab Environment Configuration

## Overview
Three-VM architecture for BEC (Business Email Compromise) attack simulation and detection engineering in a SOC environment.

## Virtual Machine Specifications

### Windows 11 VM (Victim)
- **Hardware:**
  - RAM: 4 GB
  - CPU: 4 cores
  - Storage: 64 GB
  - Network: NAT network
- **Software:**
  - Windows 11 Pro
  - Microsoft 365
  - Outlook
  - Sysmon (v15+)
  - PowerShell 5.1+
- **Purpose:** Phishing target, credential harvesting victim

### Kali Linux VM (Attacker)
- **Hardware:**
  - RAM: 2 GB
  - CPU: 4 cores
  - Storage: 60 GB
  - Network: NAT network
- **Software:**
  - Kali Linux 2024.x
  - Metasploit Framework
  - Social Engineering Toolkit (SET)
  - Python 3.x
  - Apache2
  - Wireshark
  - tcpdump
- **Purpose:** Attack simulation, credential capture

### Ubuntu Server 22.04 LTS (Analysis)
- **Hardware:**
  - RAM: 6 GB
  - CPU: 4 cores
  - Storage: 100 GB
  - Network: NAT network
- **Software:**
  - Wazuh SIEM (v4.x)
  - Elasticsearch
  - Kibana
  - Filebeat
- **Purpose:** Log collection, alert monitoring, investigation

## Security Considerations
- **Snapshots**: No support - VMware Workstation Player does not provide snapshot capabilities
  - **Backup Strategy**: Manual VM folder copies before major changes
  - **Process**: Shut down VM completely, copy entire VM folder to backup location
- **Network Monitoring**: Yes - Wireshark and tcpdump available on Kali VM for traffic analysis
- **Credential Usage**: Confirmed - using only test/fake credentials
- **Traffic Encryption**: No - Phishing page uses HTTP (not HTTPS) for form submission
- **VM Isolation**: All VMs on NAT network with controlled internet access
