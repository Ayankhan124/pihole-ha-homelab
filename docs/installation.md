# Installation and Initial Setup

## 1. Overview

This document describes the initial installation and preparation of the Pi-hole High Availability homelab.

The infrastructure runs as virtual machines using Oracle VirtualBox and Debian 13.

The installation is divided into several stages:

1. Create the virtual machines
2. Install Debian 13
3. Configure networking
4. Install Pi-hole
5. Install Unbound
6. Configure Keepalived
7. Configure monitoring
8. Configure backup and synchronization

---

## 2. Virtualization

The virtual machines are hosted using **Oracle VirtualBox**.

The main infrastructure consists of:

| VM          | Purpose                      |
| ----------- | ---------------------------- |
| `pihole01`  | Primary Pi-hole + Unbound    |
| `pihole02`  | Secondary Pi-hole + Unbound  |
| `monitor01` | Monitoring and observability |

All systems use Debian 13.

---

## 3. VM Network Configuration

The Pi-hole servers use static IP addresses.

### pihole01

```text
Hostname: pihole01
IP:       172.29.144.3
Role:     Primary Pi-hole
```

### pihole02

```text
Hostname: pihole02
IP:       172.29.144.2
Role:     Secondary Pi-hole
```

### monitor01

```text
Hostname: monitor01
IP:       172.29.144.5
Role:     Monitoring server
```

### High Availability VIP

Keepalived provides the virtual IP used by DNS clients:

```text
VIP: 172.29.144.4
```

The VIP moves between `pihole01` and `pihole02` depending on the Keepalived state.

---

## 4. Debian Installation

Debian 13 was installed on each virtual machine.

After installation, the basic system was updated:

```bash
sudo apt update
sudo apt upgrade
```

Useful base utilities were installed as required:

```bash
sudo apt install curl wget git vim nano htop
```

The hostname of each machine was configured according to its role.

For example:

```bash
sudo hostnamectl set-hostname pihole01
```

The equivalent hostname was configured on the other machines.

---

## 5. Network Configuration

The Pi-hole nodes use static IPv4 addresses because DNS infrastructure should have predictable addresses.

The network configuration was configured through Debian's networking system.

Example structure:

```text
auto eth0
iface eth0 inet static
    address 172.29.144.3
    gateway <gateway>
```

The exact gateway and interface configuration should be adjusted according to the local network.

After making network changes, connectivity was verified using tools such as:

```bash
ip addr
ip route
ping <gateway>
ping 1.1.1.1
```

DNS connectivity was also tested after the DNS infrastructure was installed.

---

## 6. Pi-hole Installation

Pi-hole was installed on both DNS nodes.

The installation was performed using the official Pi-hole installation method.

After installation, the following components were verified:

```text
Pi-hole Core
Pi-hole Web
Pi-hole FTL
```

The Pi-hole nodes were then configured with their respective static IP addresses.

The DNS service was tested locally before moving to the high-availability configuration.

---

## 7. Unbound Installation

Unbound was installed on both Pi-hole servers.

Unbound acts as the local recursive DNS resolver.

The architecture is:

```text
Client
   │
   ▼
Pi-hole
   │
   ▼
Unbound
   │
   ▼
DNS root / authoritative servers
```

Unbound was configured to listen locally on:

```text
127.0.0.1:5335
```

This keeps the recursive resolver local to the Pi-hole node.

The configuration was validated using:

```bash
sudo unbound-checkconf
```

The service was then checked:

```bash
sudo systemctl status unbound
```

A successful configuration should report that the Unbound service is active and that the configuration contains no validation errors.

---

## 8. Pi-hole and Unbound Integration

Pi-hole was configured to forward DNS requests to the local Unbound resolver.

The resulting DNS path is:

```text
Client
   │
   ▼
Pi-hole
   │
   │ DNS filtering
   ▼
Unbound :5335
   │
   │ Recursive resolution
   ▼
Internet DNS infrastructure
```

This allows Pi-hole to provide filtering while Unbound performs recursive DNS resolution.

---

## 9. Keepalived Installation

Keepalived was installed on both Pi-hole nodes.

Its purpose is to provide a shared virtual IP address.

The virtual IP is:

```text
172.29.144.4
```

The two nodes therefore have:

```text
pihole01 → 172.29.144.3
pihole02 → 172.29.144.2
VIP      → 172.29.144.4
```

Clients use the VIP instead of directly depending on one Pi-hole server.

---

## 10. High Availability Concept

Under normal operation, one Pi-hole node owns the virtual IP.

For example:

```text
172.29.144.4
      │
      ▼
pihole01
172.29.144.3
```

If the active node becomes unavailable, Keepalived can move the VIP to the other node:

```text
172.29.144.4
      │
      ▼
pihole02
172.29.144.2
```

This provides redundancy for the DNS service.

---

## 11. Initial Verification

After installing the core components, the following checks were performed.

### Check network interfaces

```bash
ip addr
```

### Check routing

```bash
ip route
```

### Check Pi-hole

```bash
pihole status
```

### Check Unbound

```bash
sudo systemctl status unbound
```

### Validate Unbound configuration

```bash
sudo unbound-checkconf
```

### Check Keepalived

```bash
sudo systemctl status keepalived
```

### Test DNS resolution

DNS resolution can be tested with:

```bash
dig example.com @127.0.0.1
```

The DNS service was also tested through the HA virtual IP:

```bash
dig example.com @172.29.144.4
```

---

## 12. Installation Result

At the end of the initial installation stage, the infrastructure contains:

```text
pihole01
 ├── Debian 13
 ├── Pi-hole
 ├── Unbound
 └── Keepalived

pihole02
 ├── Debian 13
 ├── Pi-hole
 ├── Unbound
 └── Keepalived

monitor01
 └── Monitoring infrastructure
```

The DNS service is accessed through:

```text
172.29.144.4
```

The detailed configuration of each individual component is documented separately.

See:

* [Pi-hole](pihole.md)
* [Unbound](unbound.md)
* [Keepalived](keepalived.md)
* [Monitoring](monitoring.md)
* [Backup](backup.md)
