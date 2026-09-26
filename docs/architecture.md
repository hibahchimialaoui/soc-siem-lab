# Architecture

## Network Plan

Isolated network in VMware (Host-only vmnet, not bridged to any real network).

Addressing plan: `192.168.199.0/24`

| Machine | OS | IP | Role |
|---|---|---|---|
| Kali Linux | Kali rolling | 192.168.199.10 | Attacker |
| Wazuh Manager | Ubuntu Server 24.04 | 192.168.199.129 | SIEM (manager + indexer + dashboard, Docker) |
| Windows Server | Windows Server 2025 | 192.168.199.40 | Active Directory + Sysmon (Day 3) |
| Linux Server | Ubuntu Server 24.04 | 192.168.199.30 | Vulnerable web app (Day 2) |
| Honeypot | Ubuntu Server 24.04 | 192.168.199.50 | Cowrie / OpenCanary (Day 4) |

## Component Choices

- **Wazuh (single-node, Docker)** — fastest to deploy, includes indexer (OpenSearch) and dashboard with built-in MITRE ATT&CK module.
- **Ubuntu 24.04 LTS** — long-term support, well-supported by Wazuh.
- **VMware Workstation Pro** — free for personal use, supports host-only networking for full isolation.

## Security Notes

- Lab is fully isolated from any production network.
- Default Wazuh credentials (admin / SecretPassword) accepted for lab given isolation. Would rotate in real deployment.
- All inter-component communication is TLS-encrypted (internal PKI generated at deployment).
