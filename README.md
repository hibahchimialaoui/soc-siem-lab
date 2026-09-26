# soc-siem-lab

A hands-on **SOC / SIEM home lab** for practicing threat detection and incident response.

## What this lab does

Simulates a small enterprise security environment:
- **Wazuh SIEM** (manager + indexer + dashboard) collecting logs and generating alerts
- **Windows Active Directory** (domain controller with Sysmon)
- **Vulnerable Linux web app** (OWASP Juice Shop)
- **Honeypot** (OpenCanary)
- **Kali Linux** running controlled attack scenarios (recon, brute force, web attacks)

## Architecture

See [docs/architecture.md](docs/architecture.md) for the network diagram and IP plan.

## Timeline (3-day sprint)

Progress tracked via GitHub milestones and issues.

- **Day 1** — Environment + Wazuh Manager deployment
- **Day 2** — Agents (Ubuntu + Windows AD) + honeypot + vulnerable app
- **Day 3** — Attack scenarios + rule tuning + final documentation

## Documentation

- Architecture: [docs/architecture.md](docs/architecture.md)
- Day-by-day journal: [docs/journal/](docs/journal/)
- Component-specific setup: [wazuh/](wazuh/)

## Author

Hiba HACHIMI ALAOUI — 5th-year Computer Networks & Cybersecurity engineering student, EMSI Tanger.
