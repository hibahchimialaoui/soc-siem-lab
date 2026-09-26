# Jour 2 — Agents, Vulnerable App & Honeypot

**Date:** 2026-09-26
**Duration:** in progress
**Objective:** Install Wazuh agents on all monitored machines, deploy vulnerable app and honeypot as attack targets.

## Task 1 — Wazuh agent on Ubuntu Manager (self-monitoring)

### Method

Installed the Wazuh agent on the Ubuntu VM itself using the official APT repository. The agent monitors localhost (127.0.0.1 as Manager IP).

### Commands

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update

sudo WAZUH_MANAGER='127.0.0.1' WAZUH_AGENT_NAME='ubuntu-manager-selfmon' apt install -y wazuh-agent=4.12.0-1
sudo apt-mark hold wazuh-agent

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Problem: version mismatch

Latest agent from APT was v4.14.8 but Manager runs v4.12.0. Wazuh refuses this configuration:

### Solution

- Purge newer agent: `sudo apt purge wazuh-agent && sudo rm -rf /var/ossec`
- Install matching version: `apt install wazuh-agent=4.12.0-1`
- Hold package to prevent auto-upgrade: `sudo apt-mark hold wazuh-agent`

### Verification

Key log lines from `/var/ossec/logs/ossec.log` confirming success:

Dashboard confirmation: `https://192.168.199.129` -> Endpoints -> agent `ubuntu-manager-selfmon` visible as Active (v4.12.0, group `default`).

### Screenshot

- `docs/screenshots/jour2-agent-ubuntu-active.png`
