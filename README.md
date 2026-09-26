# soc-siem-lab

A hands-on **SOC / SIEM home lab** for practicing threat detection and incident response.

## What this lab does

Simulates a small enterprise security environment:
- **Wazuh SIEM** (manager + indexer + dashboard) collecting logs and generating alerts
- **Windows Active Directory** (domain controller with Sysmon)
- **Vulnerable Linux web app** (OWASP Juice Shop / DVWA)
- **Honeypot** (Cowrie / OpenCanary)
- **Kali Linux** running controlled attack scenarios (recon, brute force, web attacks)

## Architecture

See [docs/architecture.md](docs/architecture.md) for the network diagram and IP plan.

## Timeline

7-day sprint. Progress tracked via GitHub milestones and issues.

- **Day 1** — Environment + Wazuh Manager deployment
- **Day 2** — Wazuh agents + vulnerable app
- **Day 3** — Windows Server + Active Directory + Sysmon
- **Day 4** — Honeypot + dashboards configuration
- **Day 5** — Web/network attack scenarios
- **Day 6** — Windows/AD attack scenarios + rule tuning
- **Day 7** — Final testing + documentation + report

## Documentation

- Architecture: [docs/architecture.md](docs/architecture.md)
- Day-by-day journal: [docs/journal/](docs/journal/)
- Component-specific setup: [wazuh/](wazuh/)

## Author

Maria — 4th-year Computer Networks & Cybersecurity engineering student, EMSI Tanger.
