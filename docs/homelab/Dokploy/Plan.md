# The Plan

## 1. Install and configure mini PCs

- Download and flash Unbuntu server to USB drive
- Install with:
    - Minimal install
    - OpenSSH enabled
    - Static IP
- Hostnames:
    - metal0
    - metal1
    - metal2
    - metal3
- Post install basics:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install fail2ban ufw -y
sudo ufw allow ssh
sudo ufw enable
```
## 2. Set up Tailscale
- Install Tailscale on each node:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
```
- From tailscale admin:
    - Approve all devices
    - Enable MagicDNS
    - Rename nodes to match hostnames:
        - metal0
        - metal1
        - metal2
        - metal3
- test connectivity:
```bash
ping metal0.tailnet-name.ts.net
```
## 3. Install docker engine
- Install Docker on each node:
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```
## 4. Install dokploy on the master node

- On the master node (metal0):
```bash
curl -sSL https://dokploy.com/install.sh | sh
```
- This installs:
    - Dokploy
    - Traefik reverse proxy
    - PostgreSQL & Redis
    - Docker (if not already)

## 6. Access Dokploy WEb UI
- Via Tailscale:
http://metal0.tailnet-name.ts.net:8080

- Intitial setup:
    - Create admin user
    - Set default domain suffix 
    - Configure Docker registry

## 7. Add Cluster Nodes
- From Dokploy UI > Cluster > Add Node:
   - Copy Docker swarm join command
   - SSH into each node and run command