# The Plan: Building a Production-Ready Home Lab

This document outlines the step-by-step process for building a robust Docker Swarm cluster using Dokploy for orchestration. The plan covers everything from initial hardware setup to final cluster configuration.

!!! info "Prerequisites"
    Before starting, ensure you have:

    - 3-7 mini PCs or servers for the cluster
    - USB drives for OS installation
    - Network access and static IP addresses planned
    - Basic Linux administration knowledge

## 1. Install and Configure Mini PCs

### Operating System Installation

- Download and flash Ubuntu Server to USB drive (fix spelling: Ubuntu, not "Unbuntu")
**Installation Configuration:**
- **Minimal install**: Reduces attack surface and resource usage
- **OpenSSH enabled**: Essential for remote management
- **Static IP addresses**: Required for stable cluster communication

!!! tip "Installation Best Practices"
    - Use LVM for flexible storage management
    - Enable automatic security updates during installation
    - Set strong passwords or configure SSH key authentication
    - Document MAC addresses for DHCP reservations if needed

### Node Configuration

- **Cluster Topology:**
    - **metal0** `192.168.1.190` - Primary manager node
    - **metal1** `192.168.1.191` - Worker node
    - **metal2** `192.168.1.192` - Worker node
    - **metal3** `192.168.1.193` - Secondary manager node
    - **metal4** `192.168.1.194` - Worker node
    - **metal5** `192.168.1.195` - Secondary manager node
    - **metal6** `192.168.1.196` - Worker node

!!! note "Manager Node Strategy"
    Docker Swarm requires an odd number of manager nodes for quorum. With 7 nodes total, having 2 managers provides redundancy while maintaining cluster stability.
### Post-Installation Security Hardening

**Essential Security Setup:**
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install security tools
sudo apt install fail2ban ufw htop curl wget git -y

# Configure firewall
sudo ufw allow ssh
sudo ufw allow from 192.168.1.0/24  # Allow local network
sudo ufw enable

# Configure fail2ban for SSH protection
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

!!! warning "Security Considerations"
    - **Change default SSH port**: Consider moving SSH to a non-standard port
    - **Disable root login**: Ensure root SSH access is disabled
    - **Key-based authentication**: Disable password authentication once SSH keys are configured
    - **Regular updates**: Enable automatic security updates

**Additional Hardening Steps:**
```bash
# Disable root login (edit /etc/ssh/sshd_config)
sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Restart SSH service
sudo systemctl restart ssh

# Set up automatic updates
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```
## 2. Set Up Tailscale VPN

Tailscale provides secure, encrypted connectivity between nodes and enables remote access to the cluster.

### Installation Process

**Install Tailscale on each node:**
```bash
# Download and install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Connect to your Tailscale network with SSH enabled
sudo tailscale up --ssh --accept-routes
```

!!! info "Why Tailscale?"
    - **Zero-config VPN**: Automatic mesh networking between devices
    - **SSH replacement**: Built-in SSH access through the VPN
    - **MagicDNS**: Automatic hostname resolution
    - **ACL support**: Fine-grained access control policies

### Administrative Configuration

**From Tailscale Admin Console:**

1. **Approve all devices** - Authorise each node to join the network
2. **Enable MagicDNS** - Allows hostname-based connectivity
3. **Rename nodes** to match physical hostnames:
    - metal0
    - metal1
    - metal2
    - metal3
    - metal4
    - metal5
    - metal6
4. **Configure ACLs** (optional) - Restrict access between nodes if needed

### Connectivity Testing

**Verify network connectivity:**
```bash
# Test basic connectivity
ping metal0.tailnet-name.ts.net

# Test SSH connectivity
ssh metal1.tailnet-name.ts.net

# Verify all nodes can reach each other
for i in {0..6}; do
    echo "Testing metal$i..."
    ping -c 1 metal$i.tailnet-name.ts.net
done
```

!!! tip "Tailscale Best Practices"
    - **Use SSH keys**: Configure SSH key authentication for passwordless access
    - **Enable Tailscale SSH**: Simplifies access management and provides audit logs
    - **Monitor connections**: Regularly check the admin console for unauthorised devices
## 3. Install Docker Engine

Docker Engine is the foundation of the container orchestration platform. Install it on all nodes before setting up the Swarm cluster.

### Installation Process

**Install Docker on each node:**
```bash
# Download and install Docker using the official script
curl -fsSL https://get.docker.com | sh

# Add current user to docker group (avoid using sudo for docker commands)
sudo usermod -aG docker $USER

# Apply group membership (or logout/login)
newgrp docker

# Enable Docker service to start on boot
sudo systemctl enable docker
sudo systemctl start docker
```

### Post-Installation Configuration

**Verify Docker installation:**
```bash
# Check Docker version
docker --version

# Test Docker functionality
docker run hello-world

# Check Docker service status
sudo systemctl status docker
```

**Configure Docker daemon (optional but recommended):**
```bash
# Create Docker daemon configuration
sudo mkdir -p /etc/docker

# Configure logging and storage driver
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

# Restart Docker to apply configuration
sudo systemctl restart docker
```

!!! warning "Docker Security"
    - **Group membership**: Adding users to the docker group grants root-equivalent access
    - **Log rotation**: Configure log rotation to prevent disk space issues
    - **Resource limits**: Consider setting memory and CPU limits for containers
    - **Registry security**: Use trusted container registries and scan images for vulnerabilities
## 4. Install Dokploy on the Manager Node

Dokploy is a modern deployment platform that provides a GitOps workflow for Docker Swarm. It includes all necessary components for a complete orchestration solution.

### Installation Process

**Install Dokploy on the primary manager node (metal0):**
```bash
# Download and run the Dokploy installation script
curl -sSL https://dokploy.com/install.sh | sh
```

!!! info "What Dokploy Installs"
    The installation script automatically sets up:

    - **Dokploy**: Web-based deployment platform
    - **Traefik**: Reverse proxy and load balancer
    - **PostgreSQL**: Database for Dokploy metadata
    - **Redis**: Caching and session storage
    - **Docker Swarm**: Initialises the cluster (if not already done)

### Post-Installation Verification

**Check installation status:**
```bash
# Verify Dokploy services are running
docker service ls

# Check Dokploy logs
docker service logs dokploy_dokploy

# Verify Traefik is accessible
curl -I http://localhost:8080
```

**Expected services after installation:**
- `dokploy_dokploy` - Main Dokploy application
- `dokploy_traefik` - Reverse proxy
- `dokploy_postgres` - Database
- `dokploy_redis` - Cache

!!! warning "Installation Notes"
    - **Firewall**: Ensure port 3000 (Dokploy) and 8080 (Traefik) are accessible
    - **Resources**: Dokploy requires at least 2GB RAM and 10GB disk space
    - **Backup**: The PostgreSQL database contains all deployment configurations
    - **SSL**: Consider configuring SSL certificates for production use

## 5. Access Dokploy Web UI

Once Dokploy is installed, you can access the web interface to begin configuration and deployment management.

### Accessing the Interface

**Via Tailscale (recommended for security):**
```
http://metal0.tailnet-name.ts.net:3000
```

**Via local network (if firewall allows):**
```
http://192.168.1.190:3000
```

!!! tip "Access Methods"
    - **Tailscale**: Most secure, works from anywhere
    - **Local network**: Faster, but limited to LAN access
    - **Cloudflare Tunnel**: For secure external access (advanced setup)

### Initial Setup Configuration

**Complete the initial setup wizard:**

1. **Create Admin User**
    - Set a strong password
    - Use a valid email address for notifications
    - Enable two-factor authentication if available

2. **Configure Domain Settings**
    - Set default domain suffix (e.g., `cooked.beer`)
    - Configure DNS settings for automatic subdomain creation
    - Set up SSL certificate management

3. **Docker Registry Configuration**
    - Configure Docker Hub credentials (if using private images)
    - Set up private registry access (optional)
    - Configure image pull policies

4. **Cluster Settings**
    - Verify Docker Swarm status
    - Configure default networks
    - Set resource limits and quotas

!!! warning "Security Setup"
    - **Strong passwords**: Use a password manager for admin credentials
    - **HTTPS**: Configure SSL certificates for production use
    - **Access control**: Limit access to trusted networks or VPN
    - **Backup credentials**: Store admin credentials securely

## 6. Add Cluster Nodes

Expand the Docker Swarm cluster by adding worker nodes through the Dokploy interface.

### Adding Worker Nodes

**From Dokploy UI:**
1. Navigate to **Cluster > Add Node**
2. Copy the Docker Swarm join command provided
3. The command will look similar to:
   ```bash
   docker swarm join --token SWMTKN-1-xxxxx 192.168.1.190:2377
   ```

**Execute on each worker node:**
```bash
# SSH into each worker node
ssh metal1.tailnet-name.ts.net

# Run the join command (replace with actual token)
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.190:2377

# Verify the node joined successfully
docker node ls
```

### Adding Manager Nodes (Optional)

**For high availability, add additional manager nodes:**
```bash
# Generate manager join token on existing manager
docker swarm join-token manager

# Use the manager token on metal5
ssh metal5.tailnet-name.ts.net
docker swarm join --token SWMTKN-1-manager-token 192.168.1.190:2377
```

### Cluster Verification

**Verify cluster status:**
```bash
# Check all nodes from any manager
docker node ls

# Verify services can be deployed across nodes
docker service create --replicas 3 --name test-service nginx
docker service ps test-service

# Clean up test service
docker service rm test-service
```

!!! success "Cluster Health Checks"
    - **Node status**: All nodes should show "Ready" status
    - **Manager quorum**: Ensure odd number of managers (1, 3, or 5)
    - **Network connectivity**: Verify overlay networks work between nodes
    - **Resource availability**: Check CPU, memory, and disk space on all nodes

## 7. Post-Deployment Configuration

### Essential Next Steps

**Security Hardening:**

- Configure SSL certificates for Dokploy and Traefik
- Set up proper firewall rules between nodes
- Enable audit logging for Docker Swarm
- Configure backup strategies for Dokploy database

**Monitoring Setup:**

- Deploy monitoring stack (Prometheus, Grafana)
- Configure log aggregation (Loki, Fluentd)
- Set up alerting for cluster health
- Monitor resource usage across nodes

**Backup Strategy:**

- Backup Dokploy PostgreSQL database
- Document cluster configuration
- Create disaster recovery procedures
- Test restore procedures regularly

!!! tip "Production Readiness"
    Before deploying production workloads:

    - Test cluster failover scenarios
    - Verify backup and restore procedures
    - Configure monitoring and alerting
    - Document operational procedures
    - Train team members on cluster management

## 8. Troubleshooting Common Issues

### Docker Swarm Issues

**Node fails to join cluster:**
```bash
# Check firewall ports (2377, 7946, 4789)
sudo ufw status

# Verify Docker daemon is running
sudo systemctl status docker

# Check network connectivity
ping 192.168.1.190

# Reset and rejoin if necessary
docker swarm leave --force
docker swarm join --token <new-token> 192.168.1.190:2377
```

**Service deployment failures:**
```bash
# Check service status
docker service ps <service-name> --no-trunc

# View service logs
docker service logs <service-name>

# Check node constraints
docker node inspect <node-name>
```

### Dokploy Issues

**Web UI not accessible:**
```bash
# Check Dokploy service status
docker service ls | grep dokploy

# Verify port binding
sudo netstat -tlnp | grep :3000

# Check Dokploy logs
docker service logs dokploy_dokploy
```

**Database connection issues:**
```bash
# Check PostgreSQL service
docker service ps dokploy_postgres

# Verify database connectivity
docker exec -it $(docker ps -q -f name=dokploy_postgres) psql -U postgres -d dokploy
```

### Network Connectivity Issues

**Tailscale connectivity problems:**
```bash
# Check Tailscale status
sudo tailscale status

# Restart Tailscale service
sudo systemctl restart tailscaled

# Re-authenticate if necessary
sudo tailscale up --ssh --accept-routes
```

!!! danger "Emergency Procedures"
    If the cluster becomes unresponsive:

    1. **Document the issue**: Capture logs and error messages
    2. **Check node health**: Verify all nodes are accessible
    3. **Restart services**: Try restarting problematic services first
    4. **Cluster recovery**: Use `docker swarm init --force-new-cluster` as last resort
    5. **Restore from backup**: Have backup restoration procedures ready

## 9. Maintenance and Updates

### Regular Maintenance Schedule

**Weekly Tasks:**

- Check cluster health and node status
- Review service logs for errors
- Monitor resource usage
- Verify backup completion

**Monthly Tasks:**

- Update Docker Engine on all nodes
- Update Dokploy to latest version
- Review and rotate secrets
- Test disaster recovery procedures

**Quarterly Tasks:**

- Update Ubuntu Server on all nodes
- Review security configurations
- Capacity planning assessment
- Documentation updates

### Update Procedures

**Updating Docker Engine:**
```bash
# Update on each node (one at a time)
sudo apt update
sudo apt upgrade docker-ce docker-ce-cli containerd.io

# Restart Docker service
sudo systemctl restart docker

# Verify cluster health
docker node ls
```

**Updating Dokploy:**
```bash
# Check current version
docker service inspect dokploy_dokploy --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'

# Update using Dokploy's update mechanism
curl -sSL https://dokploy.com/install.sh | sh
```

!!! warning "Update Best Practices"
    - **Test updates**: Always test in a development environment first
    - **Rolling updates**: Update nodes one at a time to maintain availability
    - **Backup first**: Create backups before major updates
    - **Monitor closely**: Watch for issues during and after updates
    - **Rollback plan**: Have a rollback strategy ready

## 10. Resources and Documentation

### Essential Documentation

- **[Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)**: Official Docker Swarm guide
- **[Dokploy Documentation](https://dokploy.com/docs)**: Dokploy setup and configuration
- **[Tailscale Documentation](https://tailscale.com/kb/)**: VPN setup and troubleshooting
- **[Ubuntu Server Guide](https://ubuntu.com/server/docs)**: Operating system administration

### Useful Commands Reference

**Docker Swarm Management:**
```bash
# Cluster status
docker node ls
docker service ls
docker stack ls

# Service management
docker service create --name <name> <image>
docker service scale <service>=<replicas>
docker service update <service>

# Troubleshooting
docker service ps <service> --no-trunc
docker service logs <service>
docker node inspect <node>
```

**System Monitoring:**
```bash
# Resource usage
htop
df -h
free -h

# Network connectivity
ping <host>
netstat -tlnp
ss -tlnp
```

!!! success "Congratulations!"
    You now have a production-ready Docker Swarm cluster with Dokploy orchestration. This setup provides a solid foundation for deploying and managing containerised applications in your home lab environment.