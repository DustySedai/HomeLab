# Networking

## Network Overview

The server is connected to the local network through Wi-Fi.

| Component     | Configuration     |
| ------------- | ----------------- |
| Interface     | `wlp2s0`          |
| Connection    | Wi-Fi             |
| LAN           | `<LAN_SUBNET>`    |
| Server IP     | `<SERVER_LAN_IP>` |
| Gateway       | `<GATEWAY_IP>`    |
| Configuration | Static IP         |
| Remote access | SSH / Tailscale   |

The server uses a static address because DHCP reservation cannot be configured on the current router.

### Current routing

```text
default via <GATEWAY_IP> dev wlp2s0
<LAN_SUBNET> dev wlp2s0 src <SERVER_LAN_IP>
```

Docker networks are also present as local routes.

---

## Network Configuration

The server uses `ifupdown` for the network configuration.

Main configuration:

```text
/etc/network/interfaces
```

The `wlp2s0` interface is configured with a static IPv4 address.

The actual address is intentionally omitted from the public documentation.

`systemd-networkd` and NetworkManager are not used for the server network configuration.

---

## LAN Access

Services exposed to the local network:

| Service     | Address           |   Port |
| ----------- | ----------------- | -----: |
| SSH         | `<SERVER_LAN_IP>` |   `22` |
| Nginx       | `<SERVER_LAN_IP>` | `8080` |
| Uptime Kuma | `<SERVER_LAN_IP>` | `3001` |
| Immich      | `<SERVER_LAN_IP>` | `2283` |

LAN access is restricted to the local subnet.

The exact subnet and server address are intentionally omitted from the public documentation.

---

## Tailscale

Tailscale is used for remote access without exposing Immich directly through the router.

The server has a Tailscale address assigned by the Tailscale network.

Immich is accessible remotely through the server's Tailscale address:

```text
http://<TAILSCALE_SERVER_IP>:2283
```

The firewall allows TCP/2283 from the Tailscale CGNAT range:

```text
100.64.0.0/10
```

Individual device addresses are not documented publicly.

Tailscale is used as a private remote-access path; it is not configured as an Exit Node.

---

## Docker Networking

Docker uses several isolated networks.

Current relevant networks:

```text
bridge
homelab_internal
immich_default
uptime-kuma_default
host
none
```

### homelab_internal

A shared Docker bridge network was created for communication between homelab services:

```text
homelab_internal
```

The bridge's dynamically assigned IP range and container IP addresses are intentionally omitted from the public documentation.

Services that need internal communication can use Docker's internal DNS instead of the server's LAN address.

For example, Uptime Kuma monitors Immich using:

```text
http://immich_server:2283
```

This keeps service-to-service communication inside Docker.

### Immich network

Immich also maintains its own Docker network:

```text
immich_default
```

### Uptime Kuma network

Uptime Kuma maintains its own Docker network:

```text
uptime-kuma_default
```

Uptime Kuma is connected to both its original network and `homelab_internal`.

---

## Network Access Model

Current intended access paths:

```text
LAN
 ├──→ SSH        :22
 ├──→ Nginx      :8080
 ├──→ Uptime Kuma:3001
 └──→ Immich     :2283

Tailscale
 └──→ Immich     :2283

Docker
 └── Uptime Kuma → Immich :2283
```

Internet access to these services is not allowed by the host firewall.

---

## Service Access

| Service     | LAN     | Tailscale | Docker internal      |
| ----------- | ------- | --------- | -------------------- |
| SSH         | `:22`   | —         | Kuma → server        |
| Nginx       | `:8080` | —         | —                    |
| Uptime Kuma | `:3001` | —         | —                    |
| Immich      | `:2283` | `:2283`   | `immich_server:2283` |

Exact IP addresses are intentionally excluded from this public documentation.

---

## Notes

* The server uses a static LAN address.
* The router is not configured with a DHCP reservation for the server.
* Docker service-to-service communication should preferably use Docker DNS/service names rather than fixed container IP addresses.
* `homelab_internal` is intended for communication between homelab services that require it.
* Firewall rules should not depend unnecessarily on dynamically assigned Docker bridge IP addresses.
* LAN access provides network access to the service, while application-level authentication controls access to application data.
* Private network addresses, Tailscale device addresses and Docker container addresses are intentionally omitted from the public repository.

