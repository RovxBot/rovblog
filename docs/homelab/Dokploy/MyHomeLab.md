# My Home Lab, Rebuilt: A Practical, Battle-Tested Guide

I recently tore down and rebuilt my home lab on Docker Swarm. This post walks through the architecture, the why behind key choices, and the gritty fixes that made everything fast, reliable, and maintainable. If you’re wrangling media, smart-home gear, and photos—all while keeping stateful data safe on NAS—this is for you.

!!! note "What You'll Learn"
    This guide covers a production-ready home lab setup using Docker Swarm with real-world solutions for common problems like NFS storage, network discovery, and database performance issues.

## Topology at a Glance
- **Cluster**: Docker Swarm across metal0 … metal6 (7-node cluster for high availability)
- **Orchestrator UX**: [Dokploy](https://dokploy.com/) to deploy stacks from a GitHub repo (compose-as-code approach)
- **Ingress**: Traefik on the overlay network `dokploy-network` routing public hostnames
- **Tunnels**: Cloudflare Tunnel for select services (e.g., Home Assistant front door)
- **Storage Architecture**:
    - **Rockstor** at 192.168.1.184 for persistent appdata (NFS v4.1)
    - **Synology NAS#2** at 192.168.1.154 (“Plex Media” share, NFS v3) for media & Immich uploads
    - **Synology NAS#1** at 192.168.1.153 (“emby_media” share, NFS v3) for legacy media
- **Networking Strategy** (split-brain by design):
    - **Host networking** where discovery/UPnP matters (Jellyfin, Home Assistant)
    - **Overlay networking** for everything else, with an edge sidecar pattern to bridge to host-net apps

!!! tip "Why Docker Swarm?"
    Docker Swarm provides native clustering with built-in load balancing, service discovery, and rolling updates. Unlike Kubernetes, it's lightweight and perfect for home labs where simplicity matters more than enterprise features.

## Architecture Overview

### Why This Setup Works

This home lab architecture is designed around three core principles:

1. **Separation of Concerns**: Storage, compute, and networking are handled by specialised components
2. **Hybrid Networking**: Use the right network mode for each application's requirements
3. **Data Locality**: Keep performance-critical data local while sharing configuration via NFS

### Hardware Foundation

The cluster consists of seven nodes (metal0-metal6) running Docker Swarm:
- **metal0**: Primary node with fastest storage (SSD) for databases
- **metal1-metal6**: Worker nodes with varying capabilities
- **Network**: Gigabit Ethernet with plans for 10GbE upgrade
- **Storage**: Mix of local SSDs for databases and NFS for shared data
## Core Design Patterns That Pay Off

### 1) NFS Done Right: Match the Server and the Path

Network File System (NFS) is the backbone of this setup, providing shared storage across the cluster. The key insight is that different NAS systems require different NFS versions and mounting approaches.

I use NFS v4.1 for Rockstor appdata and NFS v3 for Synology shares. On Synology, the export includes spaces (`:/volume1/Plex Media`), so you must pass the device exactly as exported and quote correctly in Compose.

!!! warning "NFS Version Compatibility"
    Mixing NFS versions incorrectly will cause mount failures. Always check your NAS export settings and match the version exactly. Synology typically uses NFS v3, while modern Linux NAS solutions like Rockstor prefer v4.1.

Example volume definitions:

```yaml
# Rockstor appdata (NFS v4.1)
driver_opts:
  type: nfs
  o: addr=192.168.1.184,nfsvers=4.1,rw
  device: :/export/appdata/<app>/config

# Synology shares (NFS v3)
driver_opts:
  type: nfs
  o: addr=192.168.1.154,nfsvers=3,rw
  device: ":/volume1/Plex Media"
```
#### Quick Sanity Checks I Use

Before deploying any application that relies on NFS, always test the mount manually:

```bash
# Create a probe volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.154,nfsvers=3,rw \
  --opt device=":/volume1/Plex Media" \
  plex_media_probe

# Write a marker to prove the mount
docker run --rm -v plex_media_probe:/t alpine sh -lc \
 'mkdir -p "/t/images"; echo ok > "/t/images/.probe"; ls -lah "/t/images"'
```

!!! tip "NFS Troubleshooting"
    If the probe fails, check these common issues:

    - **Export path**: Verify the exact path on your NAS (including spaces)
    - **NFS version**: Match your NAS server's supported version
    - **Permissions**: Ensure the NFS export allows your Docker host's IP
    - **Firewall**: Check that NFS ports (2049, 111) are open

If this fails, fix the export or the nfsvers before deploying the app.

!!! info "Edge Sidecar Pattern Benefits"
    This architectural pattern provides several advantages:

    - **Network isolation**: Keep most services on secure overlay networks
    - **Service discovery**: Host networking apps can still use multicast/broadcast
    - **Clean routing**: External access through Traefik with proper SSL termination
    - **Flexibility**: Easy to add authentication middleware or rate limiting
## 2) “Edge sidecar” to bridge host-net apps to Traefik
Some apps (Home Assistant, Jellyfin) benefit from network_mode: host for multicast/discovery or DLNA. But I still want pretty hostnames via Traefik. The trick is a tiny nginx sidecar on the overlay that proxies to the app’s LAN IP:port.
- App (host net) binds to the node (e.g., metal4:8123).
- ha-edge (overlay) exposes / → proxies to 192.168.1.194:8123.
- Traefik routes assistant.cooked.beer to the sidecar.
This keeps all the “smart home” discovery local while giving me a clean public entry (via Cloudflare Tunnel or Traefik).

### 3) Keep Hot, Lock-Sensitive DBs Local (The Jellyfin Fix)

SQLite databases and NFS don't mix well—you'll eventually encounter "database is locked" errors and poor performance. This is because SQLite relies on file locking mechanisms that don't work reliably over network filesystems.

!!! warning "SQLite + NFS = Problems"
    Never run SQLite databases directly on NFS shares. The file locking mechanisms SQLite relies on don't work properly over network filesystems, leading to corruption and performance issues.
## Storage Architecture Deep Dive

### Understanding NFS in a Home Lab Context

Network File System (NFS) is crucial for sharing configuration and data across cluster nodes, but it requires careful planning:

**NFS Version Selection:**
- **NFS v3**: Stateless protocol, better for simple file sharing (Synology default)
- **NFS v4.1**: Stateful with better security and performance features (modern Linux NAS)

**Performance Considerations:**
- **Network bandwidth**: Ensure adequate network capacity for concurrent access
- **Latency sensitivity**: Keep databases local, share configuration via NFS
- **Caching**: Enable client-side caching where appropriate

**Mount Options Explained:**
```bash
# Common NFS mount options
rw          # Read-write access
soft        # Fail gracefully on timeout (use with caution)
hard        # Retry indefinitely (safer for critical data)
intr        # Allow interruption of NFS calls
timeo=600   # Timeout in deciseconds (60 seconds)
retrans=2   # Number of retries before giving up
```

!!! warning "NFS Performance Impact"
    While NFS is convenient for shared configuration, it introduces network latency. For performance-critical applications like databases, always use local storage with NFS only for configuration and logs.

The solution is to overlay just the database path onto a local volume while keeping the rest of the configuration on NFS:
```yaml
volumes:
  - jellyfin_config_v2:/config          # NFS (settings/plugins/logs)
  - jellyfin_cache_v2:/cache            # NFS (artwork cache)
  - jellyfin_db_local:/config/data      # LOCAL (DB files only)
  - type: tmpfs                         # Transcode scratch = RAM
    target: /transcode
    tmpfs:
      size: 8g
```
I also pinned Jellyfin to metal0 (which has the fastest disk) via Swarm constraint/label.

**Result**: Fast library scans, no more database locks, and reliable performance.

!!! tip "Database Performance Strategy"
    For any application with a local database (SQLite, LevelDB, etc.):

    1. **Keep databases local** - Use node-local storage for DB files
    2. **Pin to specific nodes** - Use Swarm constraints to ensure consistency
    3. **Separate concerns** - Keep config/cache on NFS, databases local
    4. **Use fast storage** - SSDs make a huge difference for database performance

## The Applications

### Jellyfin (Media Server)

Jellyfin is the heart of the media setup, serving movies, TV shows, and music to devices throughout the home.

**Why host networking?** DLNA/UPnP discovery protocols work best with direct network access. These protocols rely on multicast and broadcast packets that don't traverse Docker's overlay networks well.

**Storage Strategy:**
- **Config/Cache**: Rockstor NFS v4.1 for settings, plugins, and artwork cache
- **Database**: Local SSD storage at `/var/lib/jellyfin-db` for performance
- **Media Sources**:
    - `:/volume1/Plex Media` → mounted at `/media/plex` (NFS v3)
    - `:/volume1/emby_media` → mounted at `/media/emby` (NFS v3)

**Performance Optimisations:**
- **Transcoding**: 8GB tmpfs at `/transcode` for fast temporary file operations
- **Node pinning**: Constrained to metal0 (fastest storage) for consistent performance

**Access Methods:**
- **Internal**: Direct access on `http://192.168.1.190:8096`
- **External**: Tailscale Funnel (e.g., `https://metal0.…ts.net`)
- **No Traefik**: Direct exposure avoids overlay network complexity for DLNA

!!! note "Why Not Traefik for Jellyfin?"
    While Traefik works great for web apps, media servers benefit from direct network access for device discovery and DLNA streaming. The edge sidecar pattern adds unnecessary complexity here.

### Radarr, Sonarr, Prowlarr, SABnzbd, Jellyseerr

- Pattern: Straight overlay services behind Traefik.
- Downloads / media paths: Keep consistent with Jellyfin to avoid post-processing shenanigans.
- NFS: Synology shares use nfsvers=3, Rockstor appdata uses 4.1.
- Common gotcha: Mount errors like “invalid argument” or “no such file or directory” almost always mean the export path or NFS version doesn’t match reality.
Example Traefik labels:
```yaml
deploy:
  labels:
    - traefik.enable=true
    - traefik.http.routers.radarr.rule=Host(`radarr.cooked.beer`)
    - traefik.http.routers.radarr.entrypoints=web
    - traefik.http.services.radarr.loadbalancer.server.port=7878
    - traefik.docker.network=dokploy-network
    - traefik.http.routers.radarr.middlewares=authelia@docker
```

!!! tip "*arr Stack Configuration Tips"
    - **Consistent paths**: Use identical mount points across all *arr services and Jellyfin
    - **Authentication**: Implement Authelia or similar for secure external access
    - **Resource limits**: Set appropriate CPU and memory limits to prevent resource starvation
    - **Health checks**: Configure proper health checks for reliable service management
### Vaultwarden (Password Manager)

Vaultwarden is a lightweight, self-hosted Bitwarden server implementation that provides enterprise-grade password management for the home lab.

**Architecture**: Simple overlay service behind Traefik with authentication middleware.

**Storage**: Configuration persisted to Rockstor (`/export/appdata/vaultwarden`) for backup and persistence across node failures.

!!! warning "Security Best Practices"
    - **Never commit secrets**: Use Dokploy's environment variables or secrets management
    - **Enable 2FA**: Configure TOTP for admin access
    - **Regular backups**: Password databases are critical—ensure they're included in your backup strategy
    - **Admin token rotation**: Regularly rotate admin tokens and API keys

### Immich (self-hosted photos)
- Services: immich-server, immich-machine-learning, redis/valkey, and postgres.

- Where “uploads” live: Synology NAS#2 under :/volume1/Plex Media/images (NFS v3) mounted at /usr/src/app/upload.
Immich expects a structure with thumbs, library, encoded-video, backups, etc.
I verified folder health by dropping a marker file .immich inside critical dirs (especially encoded-video).

- DB: Immich’s Postgres pinned to metal2 (label immichdb=1) with its data on the node’s local disk (/srv/appdata/immich/postgres). The Immich team warns against network shares for the DB—respect that.
!!! tip "Immich Performance Optimisation"
    **Machine Learning Performance:**

    - **GPU acceleration**: Consider adding GPU support for faster facial recognition
    - **Model caching**: Store ML models on fast storage for quicker startup
    - **Memory allocation**: Ensure adequate RAM for the ML container (4GB minimum)

    **Storage Performance:**

    - **Thumbnail generation**: Use local SSD storage for thumbnail cache
    - **Video transcoding**: Ensure adequate CPU or hardware acceleration
    - **Database tuning**: Optimise PostgreSQL settings for photo metadata workloads

- Model cache: Rockstor NFS (/export/appdata/immich/model-cache) for machine-learning models.

- Networking gotcha: immich-server must see redis in the same network. If you see getaddrinfo ENOTFOUND redis, put them on the same overlay or set REDIS_HOST env to the service DNS.

!!! warning "Immich Storage Health"
    Immich requires specific directory structures for proper operation. Always verify that required directories exist and are writable:

    ```bash
    # Verify Immich directory structure
    docker run --rm -v immich_uploads:/data alpine sh -c "
    mkdir -p /data/{thumbs,library,encoded-video,backups,profile}
    touch /data/encoded-video/.immich
    ls -la /data/
    "
    ```
If you see:
```bash
### Home Assistant Deep Dive

Home Assistant is the central hub for smart home automation, requiring careful network configuration to work properly with IoT devices.

**Why Host Networking is Essential:**
- **mDNS Discovery**: Many smart devices use multicast DNS for discovery
- **SSDP Protocol**: UPnP devices rely on Simple Service Discovery Protocol
- **Direct Device Communication**: Some integrations require direct IP communication
- **Performance**: Eliminates network translation overhead for real-time automation

**Network Requirements:**
```bash
# Essential ports for Home Assistant
Port 8123/tcp  # Web interface
Port 5353/udp  # mDNS (multicast DNS)
Port 1900/udp  # SSDP (UPnP discovery)
Port 21063/tcp # HomeKit (if enabled)
```

**Integration Examples:**
- **Philips Hue**: Requires mDNS for bridge discovery
- **Sonos**: Uses SSDP for speaker discovery
- **Chromecast**: Relies on multicast for device detection
- **MQTT**: Direct TCP communication with IoT devices
Failed to read (/data/encoded-video/.immich): ENOENT ...
```
create the directory and a .immich marker so Immich's health checks pass.

### Home Assistant
Goal: Discovery on the LAN and a pretty hostname with Cloudflare/Traefik.
- HA runs host-net on metal4, binding to 8123 locally for SSDP/mDNS.
- Edge sidecar (ha-edge) is an nginx container on the overlay that proxies to 192.168.1.194:8123.
- Traefik routes assistant.cooked.beer → ha-edge.
- Firewall (UFW on the node):
    - 8123/tcp from LAN, 5353/udp (mDNS), 1900/udp (SSDP).
Minimal, robust HA config (/config/configuration.yaml):

```yaml
homeassistant:
  name: Cooked Home
  time_zone: Australia/Adelaide
  unit_system: metric
  currency: AUD

# Essential services/UI
default_config:

# Reverse proxy hygiene
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 192.168.1.0/24
    - 10.0.0.0/8
    - 172.16.0.0/12
```
!!! warning "Home Assistant Configuration Gotchas"
    - `external_url` / `internal_url` are optional; only use them if you have specific requirements
    - Misplacing them or adding unsupported `http:` keys will break the UI completely
    - Always test configuration changes in a development environment first

The working sidecar (key bits):
```yaml
ha-edge:
  image: nginx:alpine
  networks: [dokploy-network]
  environment:
    - HA_UPSTREAM=192.168.1.194:8123
  volumes:
    - ha-nginx-template:/etc/nginx/templates:ro
  deploy:
    labels:
      - traefik.enable=true
      - traefik.docker.network=dokploy-network
      - traefik.http.routers.ha.rule=Host(`assistant.cooked.beer`)
      - traefik.http.routers.ha.entrypoints=web
      - traefik.http.services.ha.loadbalancer.server.port=8080
```
I generate the nginx config from a tiny init container so the environment variable substitution is bullet-proof (no shell quoting surprises).

!!! tip "Nginx Template Pattern"
    Using nginx templates with environment variable substitution is more reliable than shell-based config generation. The nginx:alpine image supports this natively via the `/etc/nginx/templates` directory.

### Backups with Duplicacy CLI + Swarm Cron
I didn’t want to pay for Duplicacy Web, so I wired Duplicacy CLI to run on a schedule using swarm-cronjob.
- Target: Backblaze B2 bucket (S3-compatible endpoint).
!!! note "Backup Strategy Deep Dive"
    **Why Duplicacy?**

    - **Deduplication**: Saves significant storage space with incremental backups
    - **Encryption**: Client-side encryption ensures data privacy
    - **Multiple backends**: Supports various cloud storage providers
    - **Reliability**: Lock-free design prevents corruption from interrupted backups

    **Alternative backup solutions to consider:**

    - **Restic**: Similar features with different CLI interface
    - **Borg**: Excellent for local backups with deduplication
    - **Rclone**: Great for simple sync operations to cloud storage
- Repo: The Immich uploads volume (read-only).
- Schedule: Every 3 days at midnight.
- Retention: Keep the latest 2 snapshots.
- State: Duplicacy .duplicacy preferences live on Rockstor so jobs can run stateless anywhere.
Key idea: A one-time init service (scaled to 0) and two cron-driven services (backup & prune). The cron magic is all labels; no external scheduler needed.

Example backup label:
```bash
- swarm.cronjob.command=duplicacy -log -pref-dir /config/.duplicacy -repository /data backup -stats -threads 4
```
Prune:
```bash
- swarm.cronjob.command=duplicacy -log -pref-dir /config/.duplicacy -repository /data prune -a -keep 0:2
```
B2 env (store as secrets, not plaintext):
```ini
B2_ACCOUNT_ID=...
B2_ACCOUNT_KEY=...
DUPLICACY_PASSWORD=...                # if you choose to encrypt
B2_BUCKET=cooked-photos
B2_PREFIX=immich
```
### Network Security Deep Dive

**UFW Configuration Example:**
```bash
# Basic UFW setup on each node
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH access (change port as needed)
sudo ufw allow from 192.168.1.0/24 to any port 22

# Docker Swarm cluster communication
sudo ufw allow from 192.168.1.190/32 to any port 2377  # Manager
sudo ufw allow from 192.168.1.0/24 to any port 7946   # Node communication
sudo ufw allow from 192.168.1.0/24 to any port 4789   # Overlay network

# Home Assistant specific
sudo ufw allow from 192.168.1.0/24 to any port 8123   # Web interface
sudo ufw allow from 192.168.1.0/24 to any port 5353   # mDNS
sudo ufw allow from 192.168.1.0/24 to any port 1900   # SSDP

sudo ufw enable
```

!!! danger "Firewall Safety"
    Always test firewall rules carefully. Consider keeping a console session open when applying new rules in case you lock yourself out via SSH.

### Authentication Strategy

**Authelia Integration**: For services that don't have built-in authentication, Authelia provides:
- Single sign-on across all services
- Two-factor authentication support
- Fine-grained access control
- Integration with external identity providers

**Example Authelia middleware configuration:**
```yaml
# In your Traefik-enabled service
deploy:
  labels:
    - traefik.http.routers.service.middlewares=authelia@docker
```

## Dokploy + Swarm: The Workflow

Dokploy provides a GitOps-style deployment workflow for Docker Swarm, though it's still in beta and has room for improvement. Despite its current limitations, the core workflow is solid and production-ready.

!!! info "Dokploy Beta Status"
    Dokploy is currently in beta, so expect some rough edges. However, the core GitOps workflow is stable enough for production home lab use. The development team is actively improving the platform.

**Deployment Process:**
- **GitOps-ish approach**: Each application has a `compose.yaml` in a Git repository
- **Automated deployment**: Dokploy pulls changes and runs `docker stack deploy`
- **Service naming**: Stacks appear as `apps-<name>-<suffix>_<service>` in Docker

**Essential Debugging Commands:**
```bash
# Check service status and placement
docker service ps <service> --no-trunc

# View service logs
docker service logs <service>

# Inspect service configuration
docker service inspect <service> --format '{{.Spec.TaskTemplate.ContainerSpec.Mounts}}'

# Test volume mounts
docker run --rm -v <volume>:/t alpine ls -lah /t

# Get detailed error information
docker service ps <service> --no-trunc --format "table {{.ID}}\t{{.Name}}\t{{.Image}}\t{{.Node}}\t{{.DesiredState}}\t{{.CurrentState}}\t{{.Error}}"
```

!!! tip "Debugging Pro Tips"
    - Always use `--no-trunc` on error messages—they contain the exact details about what went wrong
    - Test volume mounts independently before deploying full stacks
    - Check service placement with `docker service ps` to ensure proper node constraints

## Security and Hygiene
- Firewall on nodes: Open only what you need (HA’s 8123, mDNS/SSDP from LAN).
- Secrets: Keep admin tokens, DB creds, and B2 keys out of git. Dokploy env/secrets work well.
- TLS strategy: Internal HTTP is fine for LAN; external paths go through Traefik + Tunnel with TLS terminated at the edge.

## Lessons Learned (so you don’t have to)
1. NFS versions matter. Synology’s NFS v3 exports behave differently from NFS v4/v4.1. Match the server.
2. Spaces in export paths are real paths. Quote the device string exactly as exported.
3. SQLite over NFS is fragile. Put the hot .db files on local ext4 and leave the bulk on NFS.
4. Discovery wants host-net. Don’t fight Home Assistant or DLNA. Bridge to Traefik with a sidecar instead.
5. Cron in Swarm is easy. swarm-cronjob + containerized CLIs (Duplicacy) is clean and portable.
6. Read errors literally. Most “mount failed” messages tell you the problem (wrong vers, wrong device, or permission).

## Advanced Topics and Future Improvements

### Monitoring and Observability

While not covered in detail here, a production home lab benefits from proper monitoring:

!!! tip "Monitoring Stack Recommendations"
    - **Prometheus + Grafana**: For metrics collection and visualisation
    - **Loki**: For centralised log aggregation
    - **Uptime Kuma**: For simple service availability monitoring
    - **Node Exporter**: For hardware metrics from each cluster node

### Disaster Recovery Planning

**Backup Strategy Expansion:**
- **Configuration backups**: Regular exports of Dokploy configurations
- **Database dumps**: Automated PostgreSQL and other database backups
- **Infrastructure as Code**: Document all manual configurations for reproducibility

## Performance Tuning and Optimisation

### Docker Swarm Optimisation

**Node Resource Management:**
```bash
# Check node resource usage
docker node ls
docker system df
docker system events

# Monitor service resource consumption
docker stats $(docker ps --format "table {{.Names}}" | grep -v NAMES)
```

**Service Placement Strategies:**
- **Database services**: Pin to nodes with fastest storage (SSDs)
- **CPU-intensive services**: Distribute across nodes with adequate CPU
- **Memory-heavy services**: Consider memory constraints when placing services

### Network Performance

**Overlay Network Tuning:**
```bash
# Check overlay network health
docker network ls
docker network inspect dokploy-network

# Monitor network traffic
sudo iftop -i docker_gwbridge
```

**Bandwidth Considerations:**
- **Media streaming**: Ensure adequate bandwidth for multiple concurrent streams
- **Backup operations**: Schedule during off-peak hours to avoid congestion
- **NFS traffic**: Monitor NFS performance during peak usage

!!! tip "Performance Monitoring"
    Implement monitoring early to identify bottlenecks before they impact user experience. Tools like Prometheus, Grafana, and Node Exporter provide excellent visibility into system performance.

## Maintenance and Updates

### Regular Maintenance Tasks

**Weekly:**
- Check service health and logs
- Verify backup completion
- Monitor storage usage

**Monthly:**
- Update container images
- Review security logs
- Test disaster recovery procedures

**Quarterly:**
- Update host operating systems
- Review and rotate secrets
- Capacity planning assessment

!!! note "Update Strategy"
    Always test updates in a development environment first. Use Docker Swarm's rolling update capabilities to minimise downtime during production updates.

### Storage Performance Optimisation

**Tiered Storage Strategy:**
- **Tier 1 (SSD)**: Databases, application caches, frequently accessed data
- **Tier 2 (NFS/HDD)**: Configuration files, logs, infrequently accessed media
- **Tier 3 (Cloud)**: Backups, archival data

**NFS Performance Tuning:**
```bash
# Optimal NFS mount options for performance
mount -t nfs -o rsize=1048576,wsize=1048576,hard,intr,timeo=600 \
  192.168.1.184:/export/appdata /mnt/appdata
```

### Network Performance Optimisation

**Bandwidth Planning:**
- **4K streaming**: 25-50 Mbps per stream
- **Photo uploads**: Burst bandwidth for mobile device sync
- **Backup operations**: Schedule during off-peak hours

**Quality of Service (QoS):**
- Prioritise real-time traffic (Home Assistant, media streaming)
- Limit backup bandwidth during peak hours
- Monitor network utilisation patterns

!!! note "Scaling Considerations"
    As your home lab grows, consider:

    - **Resource allocation**: Monitor CPU and memory usage across nodes
    - **Storage growth**: Plan for data growth, especially with photo/video services
    - **Network bandwidth**: Ensure adequate bandwidth for backup and sync operations
    - **Power consumption**: Balance performance with energy efficiency

## Conclusion

This home lab setup provides a robust foundation for self-hosted services while maintaining simplicity and reliability. The key principles—proper storage separation, network pattern consistency, and security hygiene—apply regardless of your specific hardware or service choices.

The combination of Docker Swarm's simplicity with Dokploy's GitOps workflow creates a maintainable platform that can evolve with your needs. While there are rough edges (especially with Dokploy being in beta), the core architecture is solid and production-ready for home use.

**Next Steps:**
- Implement comprehensive monitoring
- Expand backup coverage to all critical services
- Document disaster recovery procedures
- Consider adding development/staging environments for testing changes

## Troubleshooting Quick Reference

### Common Issues and Solutions

**NFS Mount Failures:**
```bash
# Test NFS connectivity
showmount -e 192.168.1.154

# Manual mount test
sudo mount -t nfs -o nfsvers=3 192.168.1.154:/volume1/Plex\ Media /mnt/test
```

**Docker Swarm Service Issues:**
```bash
# Check service status across all nodes
docker service ps --no-trunc <service_name>

# View service configuration
docker service inspect <service_name>

# Check node availability
docker node ls
```

**Traefik Routing Problems:**
```bash
# Check Traefik dashboard
curl -H "Host: traefik.cooked.beer" http://localhost:8080/dashboard/

# Verify service discovery
docker service ls --filter label=traefik.enable=true
```

!!! info "Getting Help"
    When asking for help in forums or Discord:

    - Include the exact error message (use `--no-trunc`)
    - Share your Docker Compose configuration (sanitised)
    - Mention your NFS server type and version
    - Include relevant log snippets from `docker service logs`

## Resources and References

- **[Dokploy Documentation](https://dokploy.com/docs)**: Official documentation for the deployment platform
- **[Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)**: Comprehensive guide to Docker Swarm
- **[Traefik Documentation](https://doc.traefik.io/traefik/)**: Reverse proxy and load balancer configuration
- **[Duplicacy Documentation](https://duplicacy.com/guide/)**: Backup software configuration and best practices
- **[Home Assistant Documentation](https://www.home-assistant.io/docs/)**: Smart home automation platform

!!! success "Final Thoughts"
    Building a reliable home lab is an iterative process. Start simple, document everything, and gradually add complexity as you understand each component. The patterns described here have been battle-tested in a production home environment and should serve as a solid foundation for your own setup.

    **Key Success Factors:**

    - **Start small**: Begin with essential services and expand gradually
    - **Document everything**: Your future self will thank you for detailed notes
    - **Test thoroughly**: Always verify changes in a safe environment first
    - **Monitor actively**: Implement observability from day one
    - **Plan for failure**: Design with redundancy and recovery in mind