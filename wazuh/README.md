# Wazuh Manager — Single-node Deployment

## Prerequisites (Ubuntu Server 24.04)

```bash
sudo apt update && sudo apt install -y docker.io git
sudo usermod -aG docker $USER
newgrp docker
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

Install Docker Compose v2 binary:

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

## Deploy

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.12.0
cd wazuh-docker/single-node
docker compose -f generate-indexer-certs.yml run --rm generator
docker compose up -d
```

## Verify

```bash
docker ps
```

Expected: 3 containers Up (indexer, manager, dashboard).

Access dashboard: https://<vm-ip> (default credentials: admin / SecretPassword).

## Notes

- Minimum 8GB RAM allocated to the VM.
- Minimum 40GB disk (60GB recommended with headroom).
- Stack uses ~3GB of Docker images.
