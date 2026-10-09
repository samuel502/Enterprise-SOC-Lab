# Wazuh Server Deployment

## Overview
Wazuh SIEM server deployed on Ubuntu 22.04 LTS for centralized log collection and analysis.

## Server Configuration
- **OS**: Ubuntu Server 22.04 LTS
- **RAM**: 6 GB
- **CPU**: 4 cores
- **Storage**: 100 GB
- **Network**: NAT network (IP: 192.168.x.x/24)

## Wazuh Components Installed
- Wazuh Manager (v4.13.1)
- Wazuh Dashboard (v4.13.1)
- Wazuh Indexer
- Wazuh API

## Installation Method
- Ubuntu package repository installation

## Services Status
### Running Services:
- wazuh-modulesd
- wazuh-monitor
- wazuh-logcollector
- wazuh-remoted
- wazuh-syscheckd
- wazuh-analysisd
- wazuh-execd
- wazuh-db
- wazuh-authd
- wazuh-api

### Services Not Running:
- wazuh-maild (mail notifications not required)
- wazuh-agentlessd (not used in this setup)
- wazuh-integratord (not configured)
- wazuh-csyslogd (not configured)

## Troubleshooting
- **Issue**: Ubuntu VM freezing during operation
- **Resolution**: 
  - Installed open-vm-tools-desktop
  - Ran `sudo apt update` (package lists only)
  - Rebooted system
  - Freezing occurred after `apt update` but resolved after reboot

## Access Information
- **API URL**: http://<IP>:55000
- **Dashboard**: Not installed (Kibana installation caused dependency issues)
