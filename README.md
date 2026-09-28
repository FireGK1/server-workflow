# Home Server Workflow

A privacy-conscious overview of a self-hosted Ubuntu server. This repository is documentation, not a backup: it intentionally excludes hostnames, IP addresses, paths, domains, credentials, compose files, logs, and service configuration.

## Platform

| Component | Version |
| --- | --- |
| Ubuntu Server | 24.04 LTS |
| systemd | 255 |
| Docker Engine | 29 |
| Docker Compose | v2.40 |
| Git | 2.43 |

## Service groups

- Media library and request management
- Download automation behind a VPN gateway
- Photo management with a database, cache, and machine-learning worker
- Local DNS filtering
- Monitoring, dashboard, analytics, and scheduled network checks

## Deployment workflow

```text
compose definition + environment/secrets outside Git
                    |
                    v
              Docker Compose
                    |
        +-----------+-----------+
        v                       v
  application containers   named/bind-mounted data
        |
        v
  health checks + restart policies
```

Each service group has its own Compose project. Secrets are injected at deployment time and are never committed. Services use restart policies; dependent services wait for health checks where that matters. Internal networks link only the containers that need to communicate, while management interfaces are bound only to intended network interfaces.

## Security practices represented

- Keep secrets and runtime environment files outside version control.
- Use least-exposure port bindings instead of publishing every container publicly.
- Separate network-sensitive workloads from other service groups.
- Use health checks and restart policies for routine recovery.
- Prefer read-only mounts and `no-new-privileges` where a service supports them.

## What is intentionally absent

No real configuration, credentials, API keys, VPN settings, storage locations, hostnames, IP addresses, domain names, user IDs, logs, or private media metadata are published here.
