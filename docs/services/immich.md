# Immich

## Overview

Immich is deployed on the homelab server using Docker Compose.

The deployment consists of multiple containers managed as a single Docker Compose application.

All Immich containers are currently healthy.

---

## Deployment

Immich is managed through Docker Compose.

The application data is stored under:

```text
/srv/immich
```

The deployment uses the Docker network:

```text
immich_default
```

Immich is also connected to the shared homelab network:

```text
homelab_internal
```

This allows other homelab services to communicate with Immich internally through Docker DNS.

---

## Network Access

Immich listens on TCP port:

```text
2283
```

### LAN

Immich is available from the local network through the server's LAN address:

```text
http://<SERVER_LAN_IP>:2283
```

Access is restricted by the host firewall to the local LAN.

### Tailscale

Immich is also available remotely through the server's Tailscale address:

```text
http://<TAILSCALE_SERVER_IP>:2283
```

The firewall allows access to TCP/2283 from the Tailscale CGNAT range:

```text
100.64.0.0/10
```

Immich is not intentionally exposed directly to the public Internet.

---

## Internal Docker Access

Services connected to `homelab_internal` can access Immich using its Docker service name:

```text
http://immich_server:2283
```

Uptime Kuma uses this address for monitoring Immich.

This keeps monitoring traffic inside the Docker network instead of routing it through the LAN interface.

---

## Docker Containers

The Immich deployment consists of the containers provided by the official Docker Compose configuration.

The exact container names and IP addresses are not considered stable configuration and are therefore not documented here.

Container health can be checked with:

```bash
sudo docker compose ps
```

---

## Monitoring

Uptime Kuma monitors Immich through the shared Docker network.

Monitor target:

```text
http://immich_server:2283
```

This avoids depending on the server's LAN or Tailscale address for internal monitoring.

---

## Storage

Immich data is located under:

```text
/srv/immich
```

The `/srv` filesystem is the primary storage location for the homelab services.

Immich's persistent data must be preserved when updating or recreating the containers.

Before making significant changes to the deployment, verify the status of the `/srv` filesystem and the relevant Docker volumes/bind mounts.

---

## Authentication and Data Access

Network access to Immich does not grant access to the stored photo library.

There are two separate layers of access:

```text
Network access
      ↓
Immich application
      ↓
User authentication / permissions
      ↓
Photos and other application data
```

The firewall controls who can reach the Immich service.

Immich itself controls which authenticated users can access application data.

---

## Security Considerations

* Immich is not intentionally exposed directly to the Internet.
* LAN access is restricted by nftables.
* Remote access is provided through Tailscale.
* Tailscale access is restricted to `100.64.0.0/10`.
* Internal monitoring uses Docker networking.
* Docker container IP addresses should not be treated as permanent identifiers.
* Immich application credentials and database credentials must not be stored in the public repository.
* Sensitive configuration files and secrets must remain outside Git.

---

## Useful Addresses

| Purpose                  | Address                             |
| ------------------------ | ----------------------------------- |
| LAN access               | `http://<SERVER_LAN_IP>:2283`       |
| Tailscale access         | `http://<TAILSCALE_SERVER_IP>:2283` |
| Internal Docker access   | `http://immich_server:2283`         |
| Docker Compose directory | `<IMMICH_COMPOSE_DIRECTORY>`        |
| Persistent data          | `/srv/immich`                       |

Private IP addresses and credentials are intentionally omitted from the public documentation.
