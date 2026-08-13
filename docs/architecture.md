# Pi-hole HA Homelab Architecture

## 1. Overview

This project is a high-availability DNS infrastructure built as a personal homelab.

The goal is to provide a reliable DNS service using two Pi-hole servers with Unbound as the recursive DNS resolver and Keepalived for high availability.

The infrastructure also includes a dedicated monitoring server running Prometheus, Grafana, Alertmanager, and Blackbox Exporter.

All virtual machines are hosted using Oracle VirtualBox.

---

## 2. Network Topology

The homelab uses the `172.29.144.0/27` network.

Clients do not directly depend on either Pi-hole server. Instead, DNS clients use the Keepalived virtual IP address:

```text
172.29.144.4
```

This allows the DNS service to remain available if one of the Pi-hole nodes becomes unavailable.

### Network Diagram

```text
                         LAN
                          │
                          │
                    DNS Requests
                          │
                          ▼
                 ┌─────────────────┐
                 │   HA DNS VIP    │
                 │   172.29.144.4  │
                 │   Keepalived    │
                 └────────┬────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
                ▼                   ▼
        ┌──────────────┐     ┌──────────────┐
        │   pihole01   │     │   pihole02   │
        │   PRIMARY    │     │  SECONDARY   │
        │ 172.29.144.3 │     │ 172.29.144.2 │
        └──────┬───────┘     └──────┬───────┘
               │                    │
               ▼                    ▼
          ┌─────────┐          ┌─────────┐
          │ Unbound │          │ Unbound │
          └─────────┘          └─────────┘


                    Monitoring Network
                          │
                          ▼
                  ┌──────────────┐
                  │   monitor01  │
                  │ 172.29.144.5 │
                  └──────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Prometheus       Grafana      Alertmanager
          │
          ▼
    Blackbox Exporter
```

---

## 3. Infrastructure Components

| Host        | IP Address     | Role                 | Operating System |
| ----------- | -------------- | -------------------- | ---------------- |
| `pihole01`  | `172.29.144.3` | Primary DNS server   | Debian 13        |
| `pihole02`  | `172.29.144.2` | Secondary DNS server | Debian 13        |
| `monitor01` | `172.29.144.5` | Monitoring server    | Debian 13        |
| HA VIP      | `172.29.144.4` | DNS virtual IP       | Keepalived       |

---

## 4. DNS Layer

Each Pi-hole node provides DNS filtering and uses a local Unbound instance as its recursive DNS resolver.

### DNS flow

```text
Client
  │
  │ DNS query
  ▼
172.29.144.4
  │
  │ Keepalived
  ▼
Active Pi-hole node
  │
  │ Pi-hole filtering
  ▼
Unbound
  │
  │ Recursive DNS resolution
  ▼
Authoritative DNS infrastructure
```

The client only needs to know the HA virtual IP rather than the individual Pi-hole server addresses.

---

## 5. High Availability

Keepalived provides the high-availability layer.

The virtual IP is:

```text
172.29.144.4
```

The two Pi-hole nodes have their own static addresses:

```text
pihole01 → 172.29.144.3
pihole02 → 172.29.144.2
```

The VIP is assigned to the active Keepalived node.

If the active node fails the configured health/failover mechanism allows the VIP to move to the other node.

This prevents clients from losing DNS service simply because one Pi-hole server becomes unavailable.

---

## 6. Monitoring Architecture

The monitoring server is:

```text
monitor01
172.29.144.5
```

It hosts:

* Prometheus
* Grafana
* Alertmanager
* Blackbox Exporter

The Pi-hole nodes also expose monitoring metrics using:

* Node Exporter
* Pi-hole Exporter

Blackbox monitoring is used for external service checks including:

* HTTP checks
* ICMP checks
* DNS checks

Alerts are delivered through Telegram.

---

## 7. Monitoring Flow

```text
                    ┌─────────────────┐
                    │    monitor01    │
                    │  172.29.144.5   │
                    └────────┬────────┘
                             │
                       Prometheus
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        Node Exporter   Pi-hole Exporter  Blackbox
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                         Prometheus
                             │
                             ▼
                          Grafana
                             │
                             ▼
                       Alertmanager
                             │
                             ▼
                          Telegram
```

---

## 8. Software Stack

| Component         | Purpose                                 |
| ----------------- | --------------------------------------- |
| Debian 13         | Operating system                        |
| Oracle VirtualBox | Virtualization                          |
| Pi-hole           | DNS filtering and local DNS service     |
| Unbound           | Recursive DNS resolver                  |
| Keepalived        | High-availability virtual IP            |
| Node Exporter     | System metrics                          |
| Pi-hole Exporter  | Pi-hole metrics                         |
| Prometheus        | Metrics collection and alert evaluation |
| Grafana           | Metrics visualization                   |
| Blackbox Exporter | HTTP, ICMP and DNS monitoring           |
| Alertmanager      | Alert handling and routing              |
| Telegram          | Alert notifications                     |

---

## 9. Design Goals

The architecture was designed around several goals:

### High Availability

DNS should continue working when one Pi-hole node becomes unavailable.

### Privacy

Unbound performs recursive DNS resolution locally rather than relying directly on a third-party DNS resolver.

### Observability

The infrastructure should be measurable and visible through metrics, dashboards, health checks, and alerts.

### Recoverability

The project includes backup and synchronization mechanisms to reduce the impact of configuration or node failures.

### Learning

The homelab is also designed as a practical learning environment for:

* Linux administration
* DNS
* Networking
* High availability
* Infrastructure monitoring
* Troubleshooting
* Automation
* System reliability

---

## 10. Project Architecture Summary

The resulting architecture separates the infrastructure into three major layers:

```text
┌─────────────────────────────────────┐
│             DNS CLIENTS             │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│          HIGH AVAILABILITY           │
│             Keepalived               │
│          VIP: 172.29.144.4           │
└──────────────────┬──────────────────┘
                   │
          ┌────────┴────────┐
          ▼                 ▼
     ┌─────────┐       ┌─────────┐
     │pihole01 │       │pihole02 │
     │  .3     │       │  .2     │
     └────┬────┘       └────┬────┘
          │                 │
          ▼                 ▼
     ┌─────────┐       ┌─────────┐
     │ Unbound │       │ Unbound │
     └─────────┘       └─────────┘

                   +

        ┌─────────────────────┐
        │      monitor01      │
        │      172.29.144.5   │
        ├─────────────────────┤
        │ Prometheus          │
        │ Grafana             │
        │ Alertmanager        │
        │ Blackbox Exporter   │
        └─────────────────────┘
```

This architecture provides a DNS service with redundancy, recursive resolution, monitoring, alerting, and operational visibility.
