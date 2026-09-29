# Honeypot — OpenCanary

## Composant
[OpenCanary](https://github.com/thinkst/opencanary) 0.9.10 — honeypot Python open-source de Thinkst, capable de simuler plusieurs services réseau vulnérables (SSH, FTP, HTTP, MySQL, Telnet, VNC, Redis) et de logger toute tentative d''interaction.

## Rôle dans le lab
Détecter les tentatives de connexion sur des services **leurres** exposés sur des ports non utilisés par les vrais services de la VM Ubuntu (192.168.199.129). Tout contact avec ces ports est **par définition suspect** (aucun trafic légitime).

## Ports activés

| Service | Port honeypot | Port du vrai service | Remarque |
|---|---|---|---|
| FTP | 21 | (aucun) | Bannière fake |
| Telnet | 23 | (aucun) | Honeycreds admin/admin1 |
| SSH | **2222** | 22 (vrai OpenSSH) | Décalé pour éviter conflit |
| MySQL | 3306 | (aucun) | Banner 5.5.43 |
| VNC | 5000 | (aucun) | Simulé |
| Redis | 6379 | (aucun) | Simulé |
| HTTP | **8888** | 443 (Wazuh) / 3000 (Juice Shop) | Décalé pour éviter conflits |

## Config
Voir [`opencanary.conf`](opencanary.conf) — copie exacte de `/etc/opencanaryd/opencanary.conf` sur la VM.

## Log
`/var/tmp/opencanary.log` — format JSON, une ligne par événement (`logtype` identifie le type : 4002 = SSH login, 2001 = HTTP GET, etc.).

## Intégration Wazuh
L''agent Wazuh de la VM Ubuntu surveille ce log via `<localfile>` (voir `/var/ossec/etc/ossec.conf`). Les règles custom qui matchent les événements OpenCanary sont dans [`../wazuh/custom-rules/local_rules_opencanary.xml`](../wazuh/custom-rules/local_rules_opencanary.xml).

## Démarrer / arrêter

```bash
cd ~/honeypot
source env/bin/activate
sudo env "PATH=$PATH" opencanaryd --start
sudo env "PATH=$PATH" opencanaryd --stop
```

## Tester

```bash
# Depuis la VM elle-même (ou depuis Kali sur 192.168.199.129)
ssh -p 2222 admin@192.168.199.129    # ? alerte 100201 (level 12)
curl http://192.168.199.129:8888/    # ? alerte 100202 (level 12)
```
