# Firewall

The server uses **nftables** as its host-based firewall.

The firewall follows a default-deny approach for incoming and forwarded traffic. Outgoing traffic is allowed.

## Firewall Policy

| Chain     | Default policy |
| --------- | -------------- |
| `input`   | DROP           |
| `forward` | DROP           |
| `output`  | ACCEPT         |

Established and related connections are allowed so that return traffic for permitted connections can flow normally.

Invalid connection states are dropped.

---

## Input Rules

The `input` chain controls traffic destined directly for the server.

### Allowed traffic

| Source                  | Protocol | Port / Type  | Purpose                    |
| ----------------------- | -------- | ------------ | -------------------------- |
| Local LAN               | TCP      | `22`         | SSH                        |
| Local LAN               | ICMP     | Echo request | Network diagnostics        |
| Local LAN               | TCP      | `8080`       | Nginx                      |
| Local LAN               | TCP      | `3001`       | Uptime Kuma                |
| Local LAN               | TCP      | `2283`       | Immich                     |
| Docker service networks | TCP      | `22`         | Uptime Kuma → server SSH   |
| Established/related     | —        | —            | Return traffic             |
| Loopback                | —        | —            | Local system communication |

The exact LAN addresses and Docker container addresses are intentionally omitted from the public documentation.

---

## Forwarding Rules

The `forward` chain controls traffic passing through the server, including traffic forwarded to Docker containers.

### LAN → Services

The local network is allowed to reach:

| Destination     |   Port | Service     |
| --------------- | -----: | ----------- |
| Server / Docker | `3001` | Uptime Kuma |
| Server / Docker | `2283` | Immich      |

The source is restricted to the local LAN subnet.

### Tailscale → Immich

Tailscale clients are allowed to reach Immich:

```text
Tailscale (100.64.0.0/10)
        │
        └──→ TCP 2283
              Immich
```

No equivalent forwarding rule exists for Uptime Kuma or the other LAN services through Tailscale.

### Return traffic

Established and related traffic is allowed back through the appropriate interface.

This allows responses to permitted LAN and Tailscale connections without creating broad inbound rules.

---

## Current Access Matrix

| Source          | SSH `22` | Nginx `8080` | Kuma `3001` | Immich `2283` |
| --------------- | -------: | -----------: | ----------: | ------------: |
| Local LAN       |        ✅ |            ✅ |           ✅ |             ✅ |
| Tailscale       |        ❌ |            ❌ |           ❌ |             ✅ |
| Internet        |        ❌ |            ❌ |           ❌ |             ❌ |
| Docker internal |  Limited |            ❌ |    Internal |      Internal |

`Docker internal` access is controlled primarily by Docker networking and container configuration.

---

## Docker Integration

Docker manages its own nftables rules for container networking and published ports.

The custom firewall therefore does **not** replace Docker's networking rules.

The custom `inet filter` table provides the host-level policy that controls which LAN and Tailscale traffic is allowed to reach the published services.

The Docker-managed tables should not be manually modified unless there is a specific reason to do so.

### Shared Docker network

Services that require internal communication can use:

```text
homelab_internal
```

For example:

```text
Uptime Kuma → http://immich_server:2283
```

This communication remains inside Docker and does not require exposing Immich through the LAN interface.

---

## Security Model

The current firewall follows these principles:

* Default deny for inbound traffic.
* Default deny for forwarded traffic.
* Only required service ports are exposed to the LAN.
* Remote access is limited to Immich through Tailscale.
* Tailscale access is restricted to the Tailscale CGNAT range `100.64.0.0/10`.
* Return traffic is allowed through connection tracking.
* Docker networking remains managed by Docker.
* Firewall rules avoid depending on dynamically assigned Docker container IPs whenever possible.
* Application authentication remains responsible for controlling access to application data.

---

## Exposed Ports

The currently relevant TCP ports are:

|   Port | Service     | Exposure                  |
| -----: | ----------- | ------------------------- |
|   `22` | SSH         | LAN / internal monitoring |
| `8080` | Nginx       | LAN                       |
| `3001` | Uptime Kuma | LAN                       |
| `2283` | Immich      | LAN / Tailscale           |

No Internet-facing inbound rule is intentionally configured for these services.

---

## Configuration

Main firewall configuration:

```text
/etc/nftables.conf
```

The nftables service is enabled so the firewall configuration is loaded automatically during system startup.

Private LAN addresses, Docker bridge addresses and individual Tailscale device addresses are intentionally excluded from this public documentation.

