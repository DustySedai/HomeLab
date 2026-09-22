# Networking

## Network interface

The server uses the wireless interface:

```text
wlp2s0
```

Wireless adapter:

```text
RTL8188CE 802.11b/g/n WiFi Adapter
```

MAC address:

```text
9c:b7:0d:3a:dd:fe
```

The server is connected to the local network through Wi-Fi.

---

## Initial DHCP configuration

Initially, `wlp2s0` received its network configuration through DHCP.

The server was assigned:

```text
IP address: 192.168.100.35/24
Gateway:    192.168.100.1
```

The DHCP-assigned address was not ideal for a homelab server because services such as Pi-hole and monitoring applications benefit from a predictable address.

Router access was not available, so a DHCP reservation could not be configured.

A static IP was therefore configured directly on the server.

---

## Static IP configuration

The selected static address is:

```text
192.168.100.250/24
```

The gateway is:

```text
192.168.100.1
```

The configuration is stored in:

```text
/etc/network/interfaces
```

Relevant configuration:

```text
allow-hotplug wlp2s0
iface wlp2s0 inet static
    address 192.168.100.250
    netmask 255.255.255.0
    gateway 192.168.100.1
    wpa-ssid [configured locally]
    wpa-psk [configured locally]
```

The Wi-Fi credentials are intentionally not included in the documentation.

---

## DHCP interference

After changing the interface to static configuration, the server continued to receive the previous DHCP address:

```text
192.168.100.35
```

The interface temporarily contained both addresses:

```text
192.168.100.250/24
192.168.100.35/24
```

The routing table also continued to use the DHCP configuration:

```text
default via 192.168.100.1 dev wlp2s0 proto dhcp src 192.168.100.35
```

### Diagnosis

The system was using `ifupdown` to manage the interface:

```text
ifup@wlp2s0.service
        ↓
/usr/sbin/ifup --allow=hotplug wlp2s0
```

The journal showed that `ifup` was starting `dhcpcd`:

```text
dhcpcd-10.1.0 starting
wlp2s0: soliciting a DHCP lease
wlp2s0: offered 192.168.100.35 from 192.168.100.1
wlp2s0: leased 192.168.100.35
wlp2s0: adding default route via 192.168.100.1
```

This explained why DHCP remained active despite the static `ifupdown` configuration.

---

## Preventing DHCP on the static interface

The DHCP client was instructed not to manage `wlp2s0`.

File:

```text
/etc/dhcpcd.conf
```

Configuration added near the beginning of the file:

```text
denyinterfaces wlp2s0
```

This prevents `dhcpcd` from managing the wireless interface.

---

## Removing the existing DHCP lease

Because the DHCP client had already configured the interface, changing the configuration alone did not immediately remove the existing address and routes.

The active `dhcpcd` instance was stopped with:

```bash
sudo dhcpcd -x wlp2s0
```

The DHCP address was then removed:

```bash
sudo ip addr del 192.168.100.35/24 dev wlp2s0
```

The static network route was restored:

```bash
sudo ip route add 192.168.100.0/24 dev wlp2s0 src 192.168.100.250
```

The default route was restored:

```bash
sudo ip route add default via 192.168.100.1 dev wlp2s0
```

---

## Verification

Local gateway connectivity was verified:

```bash
ping -c 4 192.168.100.1
```

Internet connectivity was verified using a public IP:

```bash
ping -c 4 1.1.1.1
```

DNS resolution and external connectivity were verified with:

```bash
ping -c 4 debian.org
```

All tests succeeded with 0% packet loss.

---

## Reboot verification

The server was rebooted to verify that the configuration was persistent.

After reboot:

```bash
ip addr show wlp2s0
```

reported only:

```text
inet 192.168.100.250/24
```

The previous DHCP address `192.168.100.35` was no longer present.

The routing table contained:

```text
default via 192.168.100.1 dev wlp2s0 onlink
192.168.100.0/24 dev wlp2s0 proto kernel scope link src 192.168.100.250
```

Connectivity was successfully verified again:

```text
192.168.100.1  → reachable
1.1.1.1        → reachable
debian.org     → reachable
```

### Final network state

```text
Interface:    wlp2s0
IPv4:         192.168.100.250/24
Gateway:      192.168.100.1
Configuration: Static
DHCP:         Disabled for wlp2s0
```

The static IP configuration survives reboot and is ready to be used by homelab services such as Pi-hole and monitoring applications.

