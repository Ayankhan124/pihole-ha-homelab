# Monitoring and Observability

## 1. Overview

Monitoring and observability are used to provide visibility into the health, performance, and availability of the Pi-hole High Availability infrastructure.

A dedicated monitoring VM, `monitor01`, hosts the monitoring stack.

The monitoring architecture includes:

* Prometheus
* Grafana
* Alertmanager
* Node Exporter
* Pi-hole Exporter
* Blackbox Exporter
* Telegram notifications

The monitoring system observes both the infrastructure and the DNS service.

---

## 2. Monitoring Server

The monitoring server is:

```text id="3v5v8k"
Hostname: monitor01
IP:       172.29.144.5
OS:       Debian 13
```

The monitoring components run on this dedicated VM rather than directly on the Pi-hole nodes.

This provides separation between:

```text id="4j3xq7"
DNS Infrastructure
        │
        │ metrics / health data
        ▼
Monitoring Infrastructure
```

This separation is useful because a failure on a Pi-hole node should not also take down the monitoring system responsible for detecting that failure.

---

## 3. Monitoring Architecture

The overall monitoring architecture is:

```text id="y1qvkn"
                    ┌──────────────────┐
                    │    pihole01      │
                    │  172.29.144.3    │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │                  │
                    ▼                  ▼
              Node Exporter      Pi-hole Exporter
                    │                  │
                    └────────┬─────────┘
                             │
                             │ metrics
                             ▼
                       ┌─────────────┐
                       │ Prometheus  │
                       └──────┬──────┘
                              │
                              ▼
                         ┌─────────┐
                         │ Grafana │
                         └─────────┘


                    ┌──────────────────┐
                    │    pihole02      │
                    │  172.29.144.2    │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │                  │
                    ▼                  ▼
              Node Exporter      Pi-hole Exporter
                    │                  │
                    └────────┬─────────┘
                             │
                             ▼
                        Prometheus


                    ┌──────────────────┐
                    │  Blackbox        │
                    │  Exporter        │
                    └────────┬─────────┘
                             │
                    HTTP / ICMP / DNS
                             │
                             ▼
                        Prometheus
```

Alerting is handled through:

```text id="2m7w5b"
Prometheus
    │
    ▼
Alertmanager
    │
    ▼
Telegram
```

---

## 4. Prometheus

Prometheus is the central metrics collection and monitoring system.

It periodically scrapes metrics from configured targets.

The basic flow is:

```text id="f0wt3m"
Exporter
   │
   │ metrics
   ▼
Prometheus
   │
   ├── Store time-series data
   │
   └── Evaluate alert rules
```

Prometheus collects metrics from the Pi-hole nodes and monitoring infrastructure.

---

## 5. Prometheus Targets

The monitoring system includes several types of targets.

### Node Exporter

Node Exporter provides operating-system and hardware-related metrics.

Examples include:

* CPU usage
* Memory usage
* Disk usage
* Network traffic
* System load
* Filesystem information
* Uptime

The general architecture is:

```text id="k6a3d4"
Pi-hole VM
    │
    ▼
Node Exporter
    │
    ▼
Prometheus
```

---

## 6. Pi-hole Exporter

Pi-hole Exporter exposes Pi-hole-specific metrics to Prometheus.

These metrics provide visibility into the DNS filtering service.

Examples include:

* DNS queries
* Blocked queries
* Client activity
* Query statistics
* Pi-hole operational metrics

The architecture is:

```text id="pmc4ga"
Pi-hole
   │
   ▼
Pi-hole Exporter
   │
   ▼
Prometheus
```

This complements Node Exporter because system metrics alone cannot explain the health of the DNS service.

---

## 7. Blackbox Exporter

Blackbox Exporter is used for active endpoint probing.

Instead of only asking a server whether it is running, Blackbox Exporter tests whether a service is actually reachable and responding.

It can be used for checks such as:

* ICMP
* HTTP
* DNS
* TCP

The general flow is:

```text id="h7awqq"
Prometheus
    │
    ▼
Blackbox Exporter
    │
    ▼
Target
    │
    └── Response
```

This provides an additional layer of monitoring beyond exporter-based metrics.

---

## 8. Why Multiple Monitoring Methods Are Used

Different exporters answer different questions.

| Monitoring Method | Main Question                                       |
| ----------------- | --------------------------------------------------- |
| Node Exporter     | Is the operating system healthy?                    |
| Pi-hole Exporter  | Is Pi-hole functioning correctly?                   |
| Blackbox Exporter | Can an external probe actually reach the service?   |
| Prometheus        | What is the historical state of the infrastructure? |
| Grafana           | What does the data look like?                       |
| Alertmanager      | What requires attention?                            |

This creates layered observability.

---

## 9. Grafana

Grafana provides dashboards for visualizing Prometheus metrics.

The architecture is:

```text id="qgsp6x"
                    Prometheus
                         │
                         │ PromQL
                         ▼
                      Grafana
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
             CPU       Memory      DNS
             │          │          │
             └──────────┴──────────┘
```

Dashboards can be used to visualize:

* CPU utilization
* Memory usage
* Disk usage
* Network traffic
* DNS query rates
* Blocked queries
* Pi-hole health
* Node availability
* Service response times

---

## 10. Prometheus and Grafana Relationship

Prometheus is responsible for collecting and storing metrics.

Grafana is responsible for visualizing them.

```text id="5d34ri"
Node / Service
      │
      ▼
   Exporter
      │
      ▼
 Prometheus
      │
      ▼
   Grafana
```

Grafana does not replace Prometheus.

It uses Prometheus as a data source.

---

## 11. Alertmanager

Prometheus can detect conditions that match configured alert rules.

Alertmanager handles those alerts.

The architecture is:

```text id="wq5s11"
Prometheus
    │
    │ Alert
    ▼
Alertmanager
    │
    │ Routing
    ▼
Notification channel
```

Alertmanager can group, route, suppress, and manage alerts before they are delivered to a notification system.

---

## 12. Telegram Notifications

Telegram is used as a notification channel for important alerts.

The notification flow is:

```text id="b5e1jx"
Metric
  │
  ▼
Prometheus
  │
  │ Alert rule triggered
  ▼
Alertmanager
  │
  ▼
Telegram
  │
  ▼
Administrator
```

Sensitive Telegram credentials must never be committed to the public GitHub repository.

For documentation, use placeholders such as:

```text id="x29c5d0"
TELEGRAM_BOT_TOKEN=<redacted>
TELEGRAM_CHAT_ID=<redacted>
```

---

## 13. Example Alert Categories

The monitoring system can be used to detect conditions such as:

### Node availability

```text id="a4r6tj"
Node unreachable
```

### High CPU usage

```text id="49aj5d"
CPU usage > configured threshold
```

### High memory usage

```text id="5xj5x9"
Memory usage > configured threshold
```

### Disk usage

```text id="4d7zkm"
Filesystem usage > configured threshold
```

### DNS service failure

```text id="hlb2c8"
Pi-hole DNS unavailable
```

### Unbound failure

```text id="y1f6kd"
Unbound unavailable
```

### HA VIP failure

```text id="n08h2e"
Expected VIP unavailable
```

### Blackbox probe failure

```text id="ih5dsy"
Endpoint probe failed
```

The actual thresholds should match the alert rules deployed on the monitoring server.

---

## 14. Monitoring DNS Availability

One of the most important monitoring goals is verifying that DNS is actually available.

A DNS probe can test the HA endpoint:

```text id="1e48nf"
172.29.144.4
```

The conceptual flow is:

```text id="jcw4xy"
Blackbox Exporter
       │
       │ DNS probe
       ▼
172.29.144.4
       │
       ▼
Active Pi-hole
       │
       ▼
DNS response
```

This tests the actual client-facing DNS endpoint rather than only checking whether the Pi-hole VM is online.

---

## 15. Monitoring the Individual Nodes

Both Pi-hole nodes should be monitored individually.

```text id="74w5um"
pihole01
 ├── Node Exporter
 └── Pi-hole Exporter

pihole02
 ├── Node Exporter
 └── Pi-hole Exporter
```

This makes it possible to determine which node is unhealthy even when the HA DNS service remains operational.

---

## 16. Monitoring the HA VIP

The virtual IP:

```text id="c07hqp"
172.29.144.4
```

is monitored separately from the individual nodes.

This distinction is important.

For example:

```text id="q7s0my"
pihole01 → UP
pihole02 → UP
VIP      → DOWN
```

Both nodes could be healthy while the HA layer itself is broken.

Conversely:

```text id="u6jv6r"
pihole01 → DOWN
pihole02 → UP
VIP      → UP
```

This indicates that HA failover is functioning as intended.

---

## 17. Observability Layers

The project uses several observability layers:

```text id="j8z6b5"
Layer 1
Operating System
      │
      ▼
Node Exporter

Layer 2
Pi-hole
      │
      ▼
Pi-hole Exporter

Layer 3
Network / Service Availability
      │
      ▼
Blackbox Exporter

Layer 4
Metrics Storage
      │
      ▼
Prometheus

Layer 5
Visualization
      │
      ▼
Grafana

Layer 6
Alerting
      │
      ▼
Alertmanager

Layer 7
Notification
      │
      ▼
Telegram
```

This allows failures to be investigated from multiple perspectives.

---

## 18. Troubleshooting with Monitoring

Monitoring should help answer:

> What failed?

rather than simply:

> Something failed.

For example, if DNS stops working:

### Check Grafana

Look for:

* Node availability
* CPU/memory changes
* DNS query rate
* Exporter status
* Blackbox probe status

### Check Prometheus

Determine whether the relevant target is being scraped.

### Check Alertmanager

Determine whether an alert was generated and routed.

### Check the Pi-hole node

```bash id="2e3xan"
pihole status
```

### Check Unbound

```bash id="kl1s3v"
sudo systemctl status unbound
```

### Check Keepalived

```bash id="nd5y0s"
sudo systemctl status keepalived
```

This provides a structured troubleshooting path.

---

## 19. Monitoring Failure Scenarios

### Scenario 1 — pihole01 fails

```text id="s1u6a5"
pihole01
   X

pihole02
   │
   └── VIP
```

Expected monitoring behavior:

```text id="n6j2f1"
Node Exporter → DOWN
Pi-hole Exporter → DOWN
Blackbox node check → FAILED
VIP DNS check → SHOULD REMAIN UP
```

This demonstrates whether the HA system successfully protected DNS availability.

---

### Scenario 2 — Both Pi-hole nodes fail

Expected:

```text id="1v8t8h"
pihole01 → DOWN
pihole02 → DOWN
VIP DNS → DOWN
```

This should generate a high-priority alert.

---

### Scenario 3 — Pi-hole service fails

The VM may remain reachable:

```text id="x3c0x1"
VM → UP
Node Exporter → UP
Pi-hole → DOWN
DNS probe → FAILED
```

This demonstrates why multiple monitoring layers are necessary.

---

### Scenario 4 — Monitoring server fails

If `monitor01` fails:

```text id="1y3j8r"
monitor01
    X
```

DNS service may continue operating.

However, observability and alerting will be unavailable until monitoring is restored.

This is an important distinction:

```text id="f7zv4h"
Monitoring failure ≠ DNS failure
```

---

## 20. Prometheus Health

Prometheus itself should also be monitored.

Important checks include:

* Prometheus service status
* Target availability
* Scrape failures
* Storage availability
* Alert evaluation

The service can be checked using the operating system's service manager.

Example:

```bash id="3uh1qb"
systemctl status prometheus
```

Logs can be inspected using:

```bash id="n31aq2"
journalctl -u prometheus
```

---

## 21. Exporter Health

Exporters should be checked when metrics disappear.

For example:

```text id="z1e8a8"
Prometheus
    │
    ▼
Exporter
    X
```

A node can still be operational even if its exporter has failed.

This is why exporter availability should be monitored independently.

---

## 22. Monitoring Checklist

The monitoring stack should verify:

```text id="7ixq4g"
[ ] monitor01 is reachable
[ ] Prometheus is running
[ ] Grafana is running
[ ] Alertmanager is running
[ ] Node Exporter is running
[ ] Pi-hole Exporter is running
[ ] Blackbox Exporter is running
[ ] pihole01 is monitored
[ ] pihole02 is monitored
[ ] HA VIP is monitored
[ ] DNS availability is monitored
[ ] Alerts are generated
[ ] Alerts reach Telegram
```

---

## 23. Security Considerations

Monitoring systems can expose significant infrastructure information.

Important security practices include:

* Protect Grafana authentication.
* Restrict Prometheus access.
* Protect Alertmanager.
* Do not expose exporters unnecessarily to the Internet.
* Protect Telegram bot credentials.
* Do not commit secrets to GitHub.
* Use network-level access controls where appropriate.
* Keep monitoring software updated.

Exporter endpoints should generally be accessible only to the monitoring infrastructure.

---

## 24. Monitoring Architecture Summary

The complete monitoring architecture is:

```text id="p7v6ac"
                    ┌─────────────────┐
                    │    pihole01     │
                    │   172.29.144.3  │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
              Node Exporter     Pi-hole Exporter
                    │                 │
                    └────────┬────────┘
                             │
                             ▼
                       ┌───────────┐
                       │Prometheus │
                       └─────┬─────┘
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
             Grafana                 Alertmanager
                │                         │
                │                         ▼
                │                      Telegram
                │
                ▼
             Dashboards


                    ┌─────────────────┐
                    │    pihole02     │
                    │   172.29.144.2  │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
              Node Exporter     Pi-hole Exporter
                    │                 │
                    └────────┬────────┘
                             │
                             ▼
                        Prometheus


                    ┌─────────────────┐
                    │ Blackbox        │
                    │ Exporter        │
                    └────────┬────────┘
                             │
                     DNS / HTTP / ICMP
                             │
                             ▼
                         Prometheus
```

---

## 25. Summary

The monitoring system provides visibility into both the infrastructure and the services running on it.

The main data flow is:

```text id="1c4rj8"
Systems
   │
   ▼
Exporters
   │
   ▼
Prometheus
   │
   ├──────────────► Grafana
   │
   └──────────────► Alertmanager
                          │
                          ▼
                       Telegram
```

The combination of infrastructure metrics, Pi-hole-specific metrics, active service probing, dashboards, and notifications allows the homelab to detect and investigate failures rather than discovering them only when DNS stops working.

This monitoring architecture completes the project's observability layer.
