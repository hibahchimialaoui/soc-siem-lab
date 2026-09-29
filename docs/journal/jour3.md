
### Actions realisees

1. **Boot VM Kali** dans VMware, network adapter configure sur le VMnet isole (meme reseau que Ubuntu/Windows).
2. **IP verifiee** : `192.168.199.130/24` sur eth0 (DHCP).
3. **Connectivite validee** :
4. **Outils verifies** (nmap, hydra, sqlmap, wfuzz, gobuster : deja presents dans Kali standard).
5. **Installation crackmapexec** :
```bash
   sudo apt update && sudo apt install -y crackmapexec
```
   Note : crackmapexec 5.4.0-0kali7. Successeur `nxc` (NetExec) recommande a terme.

### Difficultes rencontrees

- **Honeypot OpenCanary non-persistant** : le test `nc 192.168.199.129 21` a repondu `Connection refused` — le daemon ne demarre pas automatiquement au boot de la VM Ubuntu. A corriger en creant un service systemd, ou a relancer manuellement avant chaque session (`sudo opencanaryd --start`). Sera adresse au demarrage de l''Attack #2.

### Etat final

Kali (192.168.199.130) pret pour lancer les 5 scenarios d''attaque des issues #21 a #25.
