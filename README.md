# Edge Monitoring Stack (Basic)

Lightweight monitoring agent for Docker hosts without PostgreSQL. Collects metrics and logs and forwards to central server.

## Components

- **Prometheus Agent**: Scrapes metrics and forwards via remote write (0 local retention)
- **Promtail**: Collects container logs and forwards to central Loki
- **cAdvisor**: Collects Docker container metrics
- **Node Exporter**: Collects host system metrics

## Requirements

- Docker 20.10+
- Docker Compose 2.0+
- Access to central monitoring server (VPC or internet)
- Minimal resources: 512MB RAM, 1 vCPU

## Deployment

### Option 1: Portainer GitOps (Recommended)

1. Create this repository on GitHub
2. In Portainer on edge host, go to **Stacks** → **Add stack**
3. Select **Git Repository**
4. Configure:
   - **Name**: `monitoring-edge`
   - **Repository URL**: `https://github.com/YOUR-USERNAME/monitoring-edge-basic`
   - **Branch**: `main`
5. Add environment variables (see below)
6. Enable **GitOps** for auto-updates
7. Deploy

### Option 2: Manual Deployment

```bash
git clone https://github.com/YOUR-USERNAME/monitoring-edge-basic.git
cd monitoring-edge-basic

# Create .env file
cp .env.example .env
nano .env

# Deploy
docker-compose up -d

# Check status
docker-compose ps
```

## Environment Variables

### For DigitalOcean VPC Setup (Recommended)

```env
# Unique hostname for this edge host
HOSTNAME=edge-host-1

# Central server VPC private IPs
CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write
CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push

# No authentication needed for VPC
```

### For Remote VPS (HTTPS + Basic Auth)

```env
# Unique hostname for this edge host
HOSTNAME=remote-vps-1

# Central server public URLs
CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push

# Basic authentication credentials
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD=your-password
```

## Verification

### Check Container Status

```bash
# All containers should be running
docker ps | grep monitoring

# Expected containers:
# - monitoring-prometheus
# - monitoring-promtail
# - monitoring-cadvisor
# - monitoring-node-exporter
```

### Check Logs

```bash
# Prometheus - should show successful remote writes
docker logs monitoring-prometheus | grep "remote write"
# Expected: "msg"="Remote write succeeded"

# Promtail - should show successful pushes
docker logs monitoring-promtail | grep "Successfully sent"

# cAdvisor - should be collecting metrics
docker logs monitoring-cadvisor

# Node Exporter - should be exporting metrics
docker logs monitoring-node-exporter
```

### Verify in Grafana

1. Open central Grafana: `https://monitoring.example.com`
2. Go to **Explore** → **Prometheus**
3. Run query:
   ```promql
   up{host="edge-host-1"}
   ```
4. Should see metrics from this host

5. Go to **Explore** → **Loki**
6. Run query:
   ```logql
   {host="edge-host-1"}
   ```
7. Should see logs from this host

## Metrics Collected

### From cAdvisor (per container)
- CPU usage
- Memory usage
- Network I/O
- Disk I/O
- Filesystem usage

### From Node Exporter (host)
- CPU utilization
- Memory usage
- Disk space
- Network statistics
- System load
- Process count

### Collection Frequency
- cAdvisor: Every 10 seconds
- Node Exporter: Every 15 seconds
- Prometheus agent: Every 15 seconds

## Logs Collected

Promtail collects logs from:
- All Docker containers on the host
- Available at `/var/lib/docker/containers/`

Logs are streamed real-time to central Loki with < 1s delay.

## Resource Usage

Typical resource consumption:
- **CPU**: < 5% average
- **Memory**: ~200-300MB total
- **Network**: ~100-500KB/s (depends on number of containers)
- **Disk**: Minimal (no local storage)

## Troubleshooting

### Prometheus Remote Write Failing

```bash
# Check logs
docker logs monitoring-prometheus | grep "error"

# Common issues:
# 1. Wrong CENTRAL_PROMETHEUS_URL
# 2. Network connectivity issues
# 3. Authentication failure (wrong credentials)

# Test connectivity
# For VPC:
ping 10.116.0.2
curl http://10.116.0.2:9090/-/healthy

# For HTTPS:
curl -u remote:password https://monitoring.example.com/prometheus/api/v1/write
```

### Promtail Not Sending Logs

```bash
# Check logs
docker logs monitoring-promtail | grep "error"

# Common issues:
# 1. Docker socket permissions
# 2. Wrong CENTRAL_LOKI_URL
# 3. Authentication failure

# Check Docker socket access
docker exec monitoring-promtail ls -l /var/run/docker.sock
```

### cAdvisor Issues

```bash
# Check if cAdvisor is running
docker ps | grep cadvisor

# If container keeps restarting:
# Check if cgroups are enabled (see below)

# Check cAdvisor logs
docker logs monitoring-cadvisor
```

### Enable cgroups (If needed)

On some systems (especially Raspberry Pi), cgroups need to be enabled:

```bash
# Check if enabled
cat /proc/cmdline | grep cgroup

# If missing, edit grub config
sudo nano /etc/default/grub

# Add to GRUB_CMDLINE_LINUX:
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1

# Update grub
sudo update-grub
sudo reboot
```

## Network Configuration

### DigitalOcean VPC

Ensure edge host is in same VPC as central server:

1. DigitalOcean console → Networking → VPC
2. Check edge host is member of VPC
3. Use VPC private IP for central server connections
4. No authentication needed

See [../docs/VPC-SETUP.md](../docs/VPC-SETUP.md) for details.

### Remote VPS (HTTPS)

Ensure:
1. Central server domain resolves correctly
2. SSL certificate is valid
3. Basic Auth credentials are correct
4. Firewall allows outbound HTTPS (443)

## Updating

### With GitOps (Automatic)

1. Push changes to GitHub
2. Wait for Portainer to auto-update (5 min interval)
3. Verify changes: `docker ps`

### Manual Update

```bash
cd monitoring-edge-basic
git pull
docker-compose pull
docker-compose up -d
```

## Security

1. **Minimal privileges**: Containers run with least privileges needed
2. **Read-only mounts**: Log directories mounted read-only
3. **No exposed ports**: All ports internal to Docker network
4. **Secure transport**: HTTPS for remote VPS, private network for VPC

## Migration from Single-Host Stack

If migrating from old single-host monitoring:

1. Deploy this edge stack
2. Verify data flowing to central Grafana
3. Stop old Grafana/Prometheus containers
4. Remove old volumes (after backup):
   ```bash
   docker volume rm old-grafana-data
   docker volume rm old-prometheus-data
   ```

## Maintenance

### Check Disk Usage

```bash
# Edge stack should use minimal disk (no data stored)
du -sh /var/lib/docker/volumes/monitoring-edge_*

# Only promtail-data should exist (positions file, few KB)
```

### Monitor Resource Usage

```bash
# Real-time stats
docker stats

# Edge stack should use < 300MB RAM total
```

### Log Rotation

Promtail doesn't store logs locally. Docker handles log rotation:

```bash
# Check Docker logging driver
docker info | grep "Logging Driver"

# Configure in /etc/docker/daemon.json:
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Restart Docker
sudo systemctl restart docker
```

## Advanced Configuration

### Custom Scrape Targets

Edit `prometheus/prometheus.yml` to add custom targets:

```yaml
scrape_configs:
  - job_name: 'custom-app'
    static_configs:
      - targets: ['app:8080']
        labels:
          host: '${HOSTNAME}'
          app: 'my-app'
```

### Filter Logs

Edit `promtail/promtail-config.yaml` to drop noisy logs:

```yaml
pipeline_stages:
  - match:
      selector: '{job="docker"}'
      stages:
        - drop:
            expression: ".*healthcheck.*"
        - drop:
            expression: ".*debug.*"
```

### Adjust Scrape Intervals

In `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 30s  # Reduce load (from 15s)
```

## Getting Help

- Check [../docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md)
- Review logs: `docker-compose logs`
- GitHub Issues: [Report problems here]
- Prometheus docs: https://prometheus.io/docs/

## License

MIT License
