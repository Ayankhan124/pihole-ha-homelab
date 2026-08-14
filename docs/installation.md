# 🛠️ Installation and Initial Setup

This document outlines the step-by-step installation and preparation of the Pi-hole High Availability homelab. 

The infrastructure is hosted as virtual machines using Oracle VirtualBox, running Debian 13.

---

## 1. Virtualization & OS Provisioning

Provision three virtual machines in Oracle VirtualBox. Install a fresh instance of Debian 13 on each node.

| VM Name     | Static IP      | Role                         |
| ----------- | -------------: | ---------------------------- |
| `pihole01`  | `172.29.144.3` | Primary Pi-hole + Unbound    |
| `pihole02`  | `172.29.144.2` | Secondary Pi-hole + Unbound  |
| `monitor01` | `172.29.144.5` | Monitoring and observability |
| **HA VIP**  | **`172.29.144.4`** | **Client-facing DNS IP** |

Once Debian 13 is installed, log into each VM and update the base system:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget git vim nano htop -y
```

Set the hostname for each machine corresponding to its role:
```bash
sudo hostnamectl set-hostname pihole01 # Repeat for pihole02 and monitor01
```

---

## 2. Network Configuration

DNS infrastructure requires highly predictable addressing. You must assign static IPv4 addresses to your nodes before installing any services.

Edit your Debian network interfaces file (typically `/etc/network/interfaces`):

```text
auto eth0
iface eth0 inet static
    address 172.29.144.3
    gateway <your_network_gateway>
```
*(Adjust the IP and gateway for `pihole02` and `monitor01` accordingly).*

**Verify Connectivity:**
```bash
ip addr
ip route
ping 1.1.1.1
```

---

## 3. Core DNS Installation (Pi-hole & Unbound)

Perform these steps on **both** `pihole01` and `pihole02`.

### A. Install Unbound
Unbound will act as the local recursive DNS resolver, querying root servers directly rather than relying on third-party upstream providers like Cloudflare.

```bash
sudo apt install unbound -y
```
Configure Unbound to listen locally on port `5335` to avoid conflicting with Pi-hole on port `53`:
```text
# Local resolution port
127.0.0.1:5335
```

### B. Install Pi-hole
Install Pi-hole using the official automated script:

```bash
curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash
```
During the setup wizard:
1. Confirm the static IP address.
2. Set your custom upstream DNS server to point to the local Unbound instance (`127.0.0.1#5335`).

---

## 4. High Availability (Keepalived)

To prevent a single node failure from taking down your network, we use Keepalived to share a Virtual IP (VIP). Clients will query this VIP (`172.29.144.4`) instead of the individual nodes. 

Install Keepalived on **both** `pihole01` and `pihole02`:

```bash
sudo apt install keepalived -y
```

*(See the dedicated [Keepalived Configuration](keepalived.md) document for the specific VRRP setup).*

---

## 5. Initial Verification

Before moving on to the monitoring or synchronization setup, validate that the core infrastructure is healthy.

**1. Validate Services:**
```bash
pihole status
sudo systemctl status unbound
sudo systemctl status keepalived
```

**2. Validate Configurations:**
```bash
sudo unbound-checkconf
```

**3. Test DNS Resolution:**
```bash
# Test local Unbound resolution
dig example.com @127.0.0.1 -p 5335

# Test HA VIP resolution
dig example.com @172.29.144.4
```

---

## ➡️ Next Steps

With the base installation complete, proceed to configure the specific components:

* [Pi-hole Configuration](pihole.md)
* [Unbound Configuration](unbound.md)
* [Keepalived Configuration](keepalived.md)
* [Backup & Synchronization](backup.md)
* [Monitoring & Observability](monitoring.md)