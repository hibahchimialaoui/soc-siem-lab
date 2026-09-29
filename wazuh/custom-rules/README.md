# Wazuh custom rules

Règles personnalisées ajoutées au manager Wazuh (container `single-node-wazuh.manager-1`).

## Fichier

### [`local_rules_opencanary.xml`](local_rules_opencanary.xml)
Règles qui matchent les événements JSON du honeypot OpenCanary ingérés par l''agent Wazuh sur la VM Ubuntu.

| Rule ID | Level | Description |
|---|---|---|
| 100200 | 10 | Activité générique sur honeypot (règle parent) |
| 100201 | **12** | SSH login attempt (creds capturés) |
| 100202 | **12** | HTTP request |
| 100203 | **12** | HTTP login attempt |
| 100204 | **12** | FTP login attempt |
| 100205 | **12** | Telnet login attempt |

## Installation

```bash
sudo docker cp local_rules_opencanary.xml single-node-wazuh.manager-1:/var/ossec/etc/rules/
sudo docker exec single-node-wazuh.manager-1 chown wazuh:wazuh /var/ossec/etc/rules/local_rules_opencanary.xml
sudo docker exec single-node-wazuh.manager-1 chmod 640 /var/ossec/etc/rules/local_rules_opencanary.xml
sudo docker exec single-node-wazuh.manager-1 /var/ossec/bin/wazuh-control restart
```

## Vérification

Dans le dashboard Wazuh ? **Threat Hunting** ? **Events**, filtre :
