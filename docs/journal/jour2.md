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

## Task 2 - Deploy OWASP Juice Shop as vulnerable target

### Method

Deployed OWASP Juice Shop as a Docker container on the same Ubuntu VM that hosts the Wazuh Manager and Agent. This co-location is intentional: the Wazuh Agent already installed (Task 1) will be able to monitor Juice Shop logs for the Day 3 attack scenarios.

### Commands

```bash
docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop
docker ps
```

### Verification

- 4 containers running on the VM: wazuh.manager, wazuh.indexer, wazuh.dashboard, juice-shop
- Juice Shop accessible from Windows host at http://192.168.199.129:3000

### Manual attack validation - SQL Injection (OWASP A03:2021)

To confirm Juice Shop is indeed vulnerable and ready as an attack target, performed a manual SQL injection on the login form:

- Email field: `' OR 1=1--`
- Password field: `test`
- Result: authenticated as admin@juice-sh.op without knowing the actual admin password

Two Juice Shop challenges auto-triggered:
1. Login Admin - proof of successful SQL injection (authentication bypass)
2. Error Handling - triggered by malformed input, exposing raw error to the client

### Explanation

Juice Shop backend concatenates user input into the SQL query. The payload closes the email string prematurely with `'`, forces the WHERE clause to always evaluate true with `OR 1=1`, and comments out the password check with `--`.

Real-world mitigation: use prepared statements / parameterized queries instead of string concatenation. All modern ORMs (Sequelize, Django ORM, Hibernate) do this by default.

### Screenshot

- docs/screenshots/jour2-juice-shop-sqli-admin.png - proof of admin login via SQL injection, both challenges solved


## Task 3 - Active Directory Domain Setup (lab.local)

### Method

Installed AD DS role on Windows Server 2025 Datacenter, then promoted to first Domain Controller of new forest lab.local.

### Environment challenges

1. First AD install interrupted by power loss during initial promotion attempt, leaving CBS corruption (error 0x800f0983).
2. Insufficient VM disk space (20GB) - expanded to 60GB via VMware + LVM resize (removed Recovery partition first).
3. Insufficient VM RAM (4GB) caused freeze at 30% - increased to 6GB.
4. Windows Defender real-time protection slowed installation - disabled with Set-MpPreference -DisableRealtimeMonitoring $true.
5. Windows Update service interfering - temporarily stopped wuauserv.
6. Install-WindowsFeature freezing due to RPC issues - bypassed using direct DISM: DISM /Online /Enable-Feature /FeatureName:DirectoryServices-DomainController /Source:wim:D:\sources\install.wim:4 /LimitAccess /All

### Recovery commands used

```powershell
DISM /Online /Cleanup-Image /CheckHealth
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
Set-MpPreference -DisableRealtimeMonitoring $true
Stop-Service wuauserv -Force
Set-Service wuauserv -StartupType Disabled
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns -SafeModeAdministratorPassword (ConvertTo-SecureString "SocLabDSRM2026!" -AsPlainText -Force) -Force
```

### Final domain configuration

- Forest: lab.local
- Mode: Windows2025Domain (first new functional level since 2016)
- NetBIOS: LAB
- DC: WIN-1GVQTED008D.lab.local (192.168.199.131)
- DNS Server: integrated on DC

### Users created

- alice (Alice Martin) - password Wazuh2026!
- bob (Bob Dubois) - password Wazuh2026!
- admin.test (Admin Test) - password Wazuh2026!

Weak intentionally-guessable password to enable Hydra brute-force testing at Day 3.

### Group created

- IT-Admins (Global, Security) with members: alice, admin.test
- Rationale: high-privilege group targeted by kerberoasting scenarios.

### DNS validation

nslookup lab.local returns 192.168.199.131 - domain resolution operational.

### Screenshots

- docs/screenshots/jour2-ad-forest-lab-local-created.png
- docs/screenshots/jour2-ad-users-created.png
- docs/screenshots/jour2-ad-complete-verification.png

### Real-world learnings

- Always take a VM snapshot before AD DS promotion in production.
- DCs need at least 6-8 GB RAM to avoid swap-related freezes.
- Microsoft recommends AV exclusions on DCs (NTDS.dit, SYSVOL, GPT paths).
- Windows Server 2025 introduces first new domain functional level since 2016 (improved Kerberos PKINIT, stronger NTDS encryption).

