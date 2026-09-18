# irewall Documentation (nftables)

This document details the firewall configuration for the homelab server, managed via `nftables`. The configuration is designed with a restrictive default policy but **optimized to coexist with Docker**, avoiding interference with the rules and tables dynamically created by the Docker daemon.

## Base Policies (Default)

The `inet filter` table is configured with the following default policies:

*   **INPUT:** `drop` (All unexplicitly authorized incoming traffic is discarded).
*   **FORWARD:** `drop` (Routing between networks is not allowed unless declared).
*   **OUTPUT:** `accept` (The server has unrestricted outbound access).

*Note on Docker:* Instead of using `flush ruleset` when reloading the firewall, we use `destroy table inet filter`. This exclusively clears our custom rules without breaking Docker's internal routing.

---

## Known Interfaces and Subnets

*   **`wlp2s0`**: Main network interface (Wi-Fi / LAN). Connected to the `192.168.100.0/24` subnet.
*   **`br-ed510586f505`**: Docker bridge network interface. Hosts services like Uptime Kuma (`172.18.0.2`).
*   **`lo`**: Local loopback interface (internal system traffic).

---

## Inbound Rules (INPUT Chain)

These rules control the traffic directed specifically to the host operating system.

| Port/Protocol | Service | Allowed Source | Justification |
| :--- | :--- | :--- | :--- |
| **22 (TCP)** | SSH | LAN (`192.168.100.0/24`) | Administrative access from any device on the local network. |
| **22 (TCP)** | SSH | Uptime Kuma (`172.18.0.2`) | Allows Uptime Kuma to authenticate via SSH on the host to monitor resources or execute scripts. |
| **8080 (TCP)** | Nginx | LAN (`192.168.100.0/24`) | Local access to the web server or reverse proxy exposed on this port. |
| **3001 (TCP)** | Uptime Kuma | LAN (`192.168.100.0/24`) | Access to the monitoring dashboard web interface from the local network. |
| **ICMP (Echo)** | Ping | LAN (`192.168.100.0/24`) | Network diagnostics to verify if the server is online from other local devices. |

*Additional protections:* Packets with an invalid state are explicitly dropped (`ct state invalid drop`), and established/related connections are permitted.

---

## Outbound Rules (OUTPUT Chain)

Currently, the default policy for outbound traffic is set to `accept`, meaning the server can freely initiate outbound connections. This section is reserved for future use in case strict egress filtering needs to be implemented.

| Port/Protocol | Service | Allowed Destination | Justification |
| :--- | :--- | :--- | :--- |
| *(Reserved)* | *TBD* | *TBD* | *(Placeholder for future explicit outbound rules)* |


## Traffic Forwarding (FORWARD Chain)

This section manages the traffic passing through the host server towards Docker containers (communication between the physical network interface and virtual interfaces).

| Route | Protocol | Destination Port | Action | Description |
| :--- | :--- | :--- | :--- | :--- |
| **wlp2s0 → br-ed5105...** | TCP | `3001` | `accept` | Allows requests from the physical network to reach the Uptime Kuma container. |
| **br-ed5105... → wlp2s0** | TCP | Source: `3001` | `accept` | Allows *return* traffic from Uptime Kuma back to the LAN, restricted strictly to already established connections. |
