# Home Server Workflow

A detailed, privacy-conscious map of a self-hosted Ubuntu server. It documents the architecture and operational patterns—not a deployable copy. All hostnames, IP addresses, paths, domains, credentials, API keys, VPN settings, media metadata, and configuration values are intentionally omitted.

## Platform

| Component | Version |
| --- | --- |
| Ubuntu Server | 24.04 LTS |
| systemd | 255 |
| Docker Engine | 29 |
| Docker Compose | v2.40 |
| Git | 2.43 |

## Architecture

```text
Ubuntu Server
├── systemd
│   └── Starts Docker and host-level services after boot
└── Docker Compose projects
    ├── Media
    │   ├── hls-select
    │   │   └── Refreshes a curated streaming playlist for the media stack
    │   ├── Jellyfin
    │   │   └── Serves the media library and waits for hls-select to start
    │   ├── Seerr
    │   │   └── Accepts media requests and connects users to the library
    │   ├── Radarr
    │   │   └── Manages movie discovery and library automation
    │   ├── Sonarr
    │   │   └── Manages TV discovery and library automation
    │   ├── Lidarr
    │   │   └── Manages music discovery and library automation
    │   ├── Bazarr
    │   │   └── Manages subtitle acquisition for the library
    │   ├── Prowlarr
    │   │   └── Centralizes indexer configuration for the automation tools
    │   └── Uptime Kuma
    │       └── Monitors service availability
    ├── Downloads
    │   ├── Gluetun
    │   │   └── VPN gateway and firewall boundary
    │   ├── qBittorrent
    │   │   └── Shares Gluetun's network namespace; no independent network path
    │   └── vpn-guardian
    │       └── Watches VPN health and stops qBittorrent when the gateway is unhealthy
    ├── Photos
    │   ├── Immich Server
    │   │   └── Web/API service for the photo library
    │   ├── Immich Machine Learning
    │   │   └── Performs photo-analysis jobs with a persistent model cache
    │   ├── Valkey
    │   │   └── Cache and job-queue dependency
    │   └── PostgreSQL
    │       └── Persistent application database with health checks
    ├── Infrastructure
    │   ├── Homarr
    │   │   └── Dashboard; reads Docker state through a read-only socket mount
    │   ├── Jellystat
    │   │   └── Media-usage analytics
    │   ├── Jellystat PostgreSQL
    │   │   └── Dedicated analytics database
    │   └── Speedtest Tracker
    │       └── Runs scheduled network measurements and retains historical results
    └── DNS
        └── Pi-hole
            └── Provides local DNS filtering and forwards upstream queries
```

## Service responsibilities

| Area | Containers | Workflow |
| --- | --- | --- |
| Media serving | Jellyfin, hls-select | The playlist helper starts first; Jellyfin then serves the locally managed media library and streaming playlist. |
| Media requests | Seerr | Provides the request layer between users and library-management services. |
| Library automation | Radarr, Sonarr, Lidarr, Bazarr, Prowlarr | Each service owns one media category or shared indexer configuration. They communicate with the download-control network only where required. |
| Download boundary | Gluetun, qBittorrent, vpn-guardian | qBittorrent uses the VPN gateway's network namespace. The guardian watches the gateway and stops qBittorrent if VPN health is lost. |
| Photo library | Immich Server, Immich Machine Learning, Valkey, PostgreSQL | The server coordinates the app; Valkey supports queued work; PostgreSQL stores persistent metadata; the ML worker processes analysis jobs. |
| Operations | Uptime Kuma, Homarr, Jellystat, Speedtest Tracker | Monitoring, dashboard access, media analytics, and scheduled network measurements. |
| DNS | Pi-hole | Local DNS filtering with a dedicated container and persistent application data. |

## Compose project workflow

Each stack is an independent Docker Compose project:

```text
compose file + environment/secrets outside Git
                    |
                    v
              docker compose up -d
                    |
        +-----------+-----------+
        v                       v
  application containers   persistent volumes / bind mounts
        |                       |
        +----------+------------+
                   v
       health checks, dependencies, restart policies
```

- Secrets are supplied at deployment time from files outside the repository.
- Persistent data is mounted outside the Compose definitions so container replacement does not erase application state.
- Services that have hard dependencies use Compose health checks or start ordering.
- Every long-running container has a restart policy for normal recovery after a failure or reboot.
- Compose networks are scoped per stack; only explicitly connected services share a network.

## Security and reliability patterns

### Network separation

The downloads stack has a dedicated control network. qBittorrent does not receive its own network namespace: it runs inside the VPN gateway's namespace, so its traffic cannot bypass the gateway without breaking the container's network connectivity. The guardian adds a second runtime check by stopping the downloader if gateway health fails.

### Least exposure

Management interfaces are bound only to intended interfaces. Containers do not automatically become public merely because they run on the server. Internal service-to-service traffic stays on Docker networks where possible.

### Least privilege

Where supported, containers use `no-new-privileges`. The dashboard gets read-only access to Docker state. Data mounts are limited to the paths each service needs.

### Persistent state

Databases, application configuration, media, and model caches are persistent. Databases include health checks before dependent application services start. This keeps routine container recreation separate from durable user data.

### Monitoring and recovery

Uptime Kuma monitors service availability. Health checks gate selected dependencies. Restart policies handle ordinary process or host-reboot recovery; no custom orchestration layer is needed for this scale.

## Deliberately excluded

This is a portfolio overview, not operational documentation. It does **not** publish:

- Hostnames, public or private IP addresses, DNS names, or ports
- Credentials, tokens, API keys, VPN provider details, or secret-file locations
- Storage paths, user/group identifiers, device details, or media contents
- Compose files, environment files, logs, backups, or database data
- Firewall rules, external access routes, or network topology values
