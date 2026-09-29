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


## Task 4 - Sysmon + Wazuh Agent on Windows Server DC

### Method

Installed Sysmon (SwiftOnSecurity config) for detailed Windows event capture, then deployed Wazuh Windows agent v4.12.0 pointing to Manager at 192.168.199.129. Added Sysmon Operational channel to agent config for forwarding to Manager.

### Commands used

```powershell
# 1. Download Sysmon + SwiftOnSecurity config
mkdir C:\Setup
cd C:\Setup
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Setup\Sysmon.zip"
Expand-Archive -Path "C:\Setup\Sysmon.zip" -DestinationPath "C:\Setup\Sysmon" -Force
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Setup\Sysmon\sysmonconfig-export.xml"

# 2. Install Sysmon
cd C:\Setup\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml

# 3. Download and install Wazuh agent v4.12.0
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi" -OutFile "C:\Setup\wazuh-agent-4.12.0-1.msi"
Start-Process msiexec.exe -Wait -ArgumentList '/i C:\Setup\wazuh-agent-4.12.0-1.msi /q WAZUH_MANAGER="192.168.199.129" WAZUH_AGENT_NAME="windows-dc-lab"'

# 4. Add Sysmon channel to Wazuh config, then start service
Start-Service Wazuh
```

### Verification

- 4 log channels monitored: Application, Security, System, Microsoft-Windows-Sysmon/Operational
- Additional modules enabled: SCA (CIS Windows Server 2025 policy), Syscollector, FIM
- Dashboard shows 2 active agents (Ubuntu + Windows), 1187 alerts in first 24h
- Level 15+ critical alert triggered by PowerShell process creation on DC (Sysmon Event ID 1)

### Pipeline validation

End-to-end flow confirmed:
Sysmon (Event ID 1: process creation) -> Windows Event Log -> Wazuh Agent -> Wazuh Manager -> Indexer -> Dashboard.

The critical alert (level 15) triggered by legitimate SSH-launched PowerShell demonstrates the SIEM works. In production, a rule tuning would distinguish admin activity from LOLBin attacks using Event ID 4104 (ScriptBlock content) and parent process context.

### Screenshots

- docs/screenshots/jour2-wazuh-2-agents-active-alerts.png - both agents active, alerts summary
- docs/screenshots/jour2-sysmon-critical-alert-powershell.png - drill-down of critical alert showing Sysmon data

### Learnings

- Version pinning between agent and manager is critical (both v4.12.0). Wazuh refuses newer agents with clear error - design choice preventing format incompatibility.
- Sysmon SwiftOnSecurity config is the de-facto standard - balanced between coverage and log volume.
- CIS Windows Server 2025 policy in SCA gives automated compliance evaluation - strong reporting bonus.
- Alert tuning is necessary: even legitimate admin activity (PowerShell via SSH) triggers critical alerts on a DC. In production, exceptions would be defined per role/user.


---

## Task 5 — Honeypot OpenCanary + intégration Wazuh (issue #13)

### Objectif
Déployer un honeypot multi-services sur la VM Ubuntu (192.168.199.129), l'intégrer à Wazuh, et écrire des règles custom pour transformer chaque interaction avec un service leurre en **alerte de niveau 12** dans le dashboard.

### Actions réalisées

**a) Installation OpenCanary 0.9.10 dans un venv Python**
```bash
mkdir ~/honeypot && cd ~/honeypot
python3 -m venv env
source env/bin/activate
pip install opencanary scapy pcapy-ng
opencanaryd --copyconfig    # crée /etc/opencanaryd/opencanary.conf
```

**b) Activation des services sur ports non-conflictuels**
Modification de `/etc/opencanaryd/opencanary.conf` (voir [`honeypot/opencanary.conf`](../../honeypot/opencanary.conf)) : FTP:21, Telnet:23, SSH:**2222** (le vrai OpenSSH tourne sur 22), HTTP:**8888** (Wazuh occupe 443, Juice Shop 3000), MySQL:3306, VNC:5000, Redis:6379.

**c) Démarrage du daemon**
```bash
sudo env "PATH=$PATH" opencanaryd --start
sudo ss -tlnp | grep -E ":(21|23|2222|3306|5000|6379|8888)"
# → 7 lignes LISTEN twistd
```

**d) Test de génération d'alerte locale**
```bash
ssh -o StrictHostKeyChecking=no admin@127.0.0.1 -p 2222
# password: hacker2026
# → événement logtype 4002 (SSH login) capturé dans /var/tmp/opencanary.log
```
JSON capturé : `{"logtype": 4002, "logdata": {"USERNAME": "admin", "PASSWORD": "hacker2026"}, "src_host": "127.0.0.1", "dst_port": 2222, ...}`

**e) Intégration Wazuh agent**
Ajout d'un bloc `<localfile>` dans `/var/ossec/etc/ossec.conf` sur la VM Ubuntu pour surveiller `/var/tmp/opencanary.log` en `log_format: json`, puis restart de l'agent.

**f) Création de règles custom sur le manager Wazuh**
Wazuh ne fournit pas de règles OpenCanary par défaut. Fichier [`wazuh/custom-rules/local_rules_opencanary.xml`](../../wazuh/custom-rules/local_rules_opencanary.xml) créé avec 6 règles :
- **100200** (level 10) : règle parent, matche tout événement `node_id: opencanary-1`
- **100201** (level 12) : SSH login attempt (creds capturés)
- **100202/203** (level 12) : HTTP request / login
- **100204** (level 12) : FTP login attempt
- **100205** (level 12) : Telnet login attempt

Installation via `docker cp` dans le container `single-node-wazuh.manager-1`, puis `wazuh-control restart`.

### Résultats

✅ Alerte **niveau 12** ("OpenCanary: SSH login attempt on honeypot (credentials captured)") visible dans le dashboard Wazuh → **Threat Hunting** → **Events** avec filtre `rule.id : 100201`.

Le pipeline complet fonctionne :
```
Attaquant --> OpenCanary (port 2222) --> /var/tmp/opencanary.log
                                       --> Wazuh Agent
                                       --> Wazuh Manager (Docker)
                                       --> Rule 100201 (level 12)
                                       --> Dashboard Wazuh
```

Preuve : ![Alerte OpenCanary niveau 12](../screenshots/jour2-honeypot-opencanary-ssh-alert.png)

### Difficultés rencontrées

1. **Conflit de ports HTTP** : `http.port: 8080` par défaut entrait en collision avec `httpproxy.port: 8080` (même si httpproxy désactivé). Résolu en passant HTTP sur 8888.
2. **Wazuh manager dans Docker** : `/var/ossec/etc/rules/` n'existe pas sur l'hôte Ubuntu — les règles doivent être copiées **dans le container** via `docker cp`, puis le manager restart via `docker exec /var/ossec/bin/wazuh-control restart`.
3. **Absence de règles OpenCanary natives** : Wazuh ne fournit rien pour ce format JSON — écriture de 6 règles custom (parent + 5 filles par service).

### Valeur pédagogique

Ce composant montre concrètement la **détection par leurre** — un des rares mécanismes SIEM à taux de faux positifs proche de zéro : toute interaction avec un service inexistant est par définition suspecte.



