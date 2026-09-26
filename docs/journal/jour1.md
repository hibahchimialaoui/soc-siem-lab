# Jour 1 — Environment & Wazuh Manager

**Date:** 2026-09-26
**Duration:** ~4h
**Objective:** Prepare isolated VMware network + VMs, deploy Wazuh Manager via Docker, verify dashboard access.

## What was done

- Created 3 VMs in VMware: Kali, Ubuntu Server 24.04, Windows Server 2025
- Installed Docker + Docker Compose v2 on the Ubuntu VM (future Wazuh Manager)
- Set up SSH remote access from Windows PowerShell (workaround for VMware clipboard issue on text-mode Ubuntu Server)
- Extended VM disk from 10GB to 60GB using LVM resize (needed for Wazuh images)
- Deployed Wazuh single-node stack: Manager + Indexer + Dashboard
- Verified dashboard access at https://192.168.199.129

## Key commands

### Docker + prerequisites (on Ubuntu VM)

```bash
sudo apt update
sudo apt install -y docker.io git
sudo usermod -aG docker $USER
newgrp docker
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### Docker Compose v2 (official binary)

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

### Disk extension (after resize in VMware to 60GB)

```bash
sudo apt install -y cloud-guest-utils
sudo growpart /dev/sda 3
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
df -h
```

### Wazuh deployment

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.12.0
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
docker ps
```

### SSH setup for remote work

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
ip a
```

From Windows host: `ssh hiba@192.168.199.129`

## Problems & solutions

- **docker-compose: command not found** on Ubuntu 24.04. docker-compose v1 is deprecated, docker-compose-plugin package not in default repos. Solution: installed Compose v2 as an official binary in /usr/local/lib/docker/cli-plugins/. Command becomes `docker compose` (with space).
- **VMware clipboard not working** in text-mode Ubuntu Server console. Solution: installed openssh-server, connected from Windows PowerShell via SSH.
- **no space left on device** during `docker compose pull`. Initial 10GB disk insufficient (~6-8GB extracted images). Solution: extended disk to 60GB in VMware, then resized partition + LV + filesystem.

## Screenshots

- docs/screenshots/jour1-dashboard.png — Wazuh dashboard operational after deployment

## Result

Wazuh Manager operational, dashboard accessible from Windows host at https://192.168.199.129. Ready for Day 2 (agent installation + vulnerable app deployment).
