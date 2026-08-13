# Troubleshooting Guide

## 1. Overview

This document contains troubleshooting procedures and lessons learned while building and operating the Pi-hole High Availability homelab.

The troubleshooting approach is based on isolating failures layer by layer rather than changing multiple components at the same time.

The main layers are:

```text
Network
   │
   ▼
Pi-hole
   │
   ▼
Unbound
   │
   ▼
Keepalived
   │
   ▼
Monitoring
```

When troubleshooting DNS, each layer should be tested independently.

---

# 2. General Troubleshooting Method

When a problem occurs, follow this order:

```text
1. Check network connectivity
        ↓
2. Check the affected service
        ↓
3. Check service logs
        ↓
4. Validate configuration
        ↓
5. Test the service locally
        ↓
6. Test the service remotely
        ↓
7. Test the HA VIP
        ↓
8. Check monitoring
```

Avoid making several configuration changes at once.

A single change followed by a test makes it easier to identify the actual cause.

---

# 3. Network Troubleshooting

## 3.1 Check Network Interfaces

Run:

```bash
ip addr
```

Verify that the expected IP address is assigned.

For example:

```text
pihole01 → 172.29.144.3
pihole02 → 172.29.144.2
```

---

## 3.2 Check Routing

Run:

```bash
ip route
```

Verify that the default route exists and points to the correct gateway.

---

## 3.3 Test Gateway Connectivity

```bash
ping <gateway>
```

If the gateway cannot be reached, DNS troubleshooting should stop here until the network problem is resolved.

---

## 3.4 Test Internet Connectivity Without DNS

Use an IP address:

```bash
ping 1.1.1.1
```

If this works but DNS queries fail, the problem is likely related to DNS rather than general Internet connectivity.

---

# 4. Pi-hole Troubleshooting

## 4.1 Check Pi-hole Status

Run:

```bash
pihole status
```

This provides a quick indication of whether the Pi-hole DNS service is operational.

---

## 4.2 Check Pi-hole FTL

Pi-hole FTL is the core DNS engine.

Check:

```bash
systemctl status pihole-FTL
```

View recent logs:

```bash
journalctl -u pihole-FTL --since "10 minutes ago"
```

Follow logs in real time:

```bash
journalctl -u pihole-FTL -f
```

---

## 4.3 Test Pi-hole Locally

Run:

```bash
dig example.com @127.0.0.1
```

If this works, Pi-hole is responding to DNS requests locally.

---

## 4.4 Test Pi-hole From Another Host

From another machine:

```bash
dig example.com @172.29.144.3
```

or:

```bash
dig example.com @172.29.144.2
```

This determines whether the DNS service is accessible over the network.

---

# 5. Unbound Troubleshooting

## 5.1 Check Service Status

```bash
sudo systemctl status unbound
```

If the service is inactive or failed, inspect the logs:

```bash
sudo journalctl -u unbound --since "10 minutes ago"
```

---

## 5.2 Validate the Configuration

Before restarting Unbound after configuration changes:

```bash
sudo unbound-checkconf
```

This is one of the most important troubleshooting commands in the project.

---

# 6. Unbound Root Hints Problem

During development, Unbound configuration validation reported a missing root hints file.

The general error occurs when the configuration references a file that does not exist.

Check the configured path:

```bash
grep -R "root-hints" /etc/unbound/
```

Then verify that the referenced file exists:

```bash
ls -l <root-hints-file>
```

If the file is missing, correct the path or restore the required root hints file.

After correcting the problem:

```bash
sudo unbound-checkconf
```

Only restart Unbound after the configuration passes validation.

---

# 7. Duplicate DNSSEC Trust Anchor Problem

Another configuration problem encountered during the project was a duplicate:

```text
auto-trust-anchor-file
```

definition.

This can occur when the same setting is configured in multiple Unbound configuration files.

Search for all definitions:

```bash
grep -R "auto-trust-anchor-file" /etc/unbound/
```

Review the results and remove the unintended duplicate configuration.

Then run:

```bash
sudo unbound-checkconf
```

A successful validation should produce no configuration errors.

---

# 8. Unbound Port Troubleshooting

The project uses:

```text
127.0.0.1:5335
```

for the local Unbound listener.

Check whether the port is listening:

```bash
sudo ss -lntup | grep 5335
```

Expected behavior is a listener associated with:

```text
127.0.0.1:5335
```

---

## 8.1 Test Unbound Directly

```bash
dig example.com @127.0.0.1 -p 5335
```

If this fails, Pi-hole is not the first thing to troubleshoot.

Focus on Unbound.

---

# 9. DNS Troubleshooting Method

A useful layered DNS test is:

### Test 1 — Unbound

```bash
dig example.com @127.0.0.1 -p 5335
```

### Test 2 — Pi-hole

```bash
dig example.com @127.0.0.1
```

### Test 3 — Individual Pi-hole node

```bash
dig example.com @172.29.144.3
```

### Test 4 — Other Pi-hole node

```bash
dig example.com @172.29.144.2
```

### Test 5 — HA VIP

```bash
dig example.com @172.29.144.4
```

This creates the following troubleshooting tree:

```text
Unbound
   │
   ├── FAIL → Troubleshoot Unbound
   │
   └── PASS
        │
        ▼
      Pi-hole
        │
        ├── FAIL → Troubleshoot Pi-hole
        │
        └── PASS
             │
             ▼
       Individual Node
             │
             ▼
          HA VIP
             │
             ├── FAIL → Troubleshoot Keepalived
             │
             └── PASS → DNS stack operational
```

---

# 10. Keepalived Troubleshooting

## 10.1 Check Service

```bash
sudo systemctl status keepalived
```

---

## 10.2 Check Logs

```bash
sudo journalctl -u keepalived --since "10 minutes ago"
```

For real-time monitoring:

```bash
sudo journalctl -u keepalived -f
```

---

## 10.3 Validate Configuration

Before restarting Keepalived:

```bash
sudo keepalived -t
```

Correct any reported configuration errors before restarting the service.

---

# 11. VIP Troubleshooting

The HA VIP is:

```text
172.29.144.4
```

Check which node currently owns it:

```bash
ip addr
```

The active node should normally show:

```text
172.29.144.4
```

The standby node should normally not show the VIP.

---

# 12. VIP Does Not Fail Over

If the active Pi-hole fails but the VIP does not move:

### Step 1 — Check Keepalived

```bash
systemctl status keepalived
```

### Step 2 — Check logs

```bash
journalctl -u keepalived
```

### Step 3 — Check network interface

```bash
ip addr
```

### Step 4 — Check VRRP configuration

Verify:

* Interface
* VRID
* Priority
* State
* Virtual IP
* Health checks

### Step 5 — Check health-check commands

If a service health check is configured, execute the health-check command manually and verify its exit status.

---

# 13. Both Nodes Appear to Own the VIP

If both nodes appear to own:

```text
172.29.144.4
```

investigate immediately.

Possible causes include:

* VRRP communication failure
* Incorrect interface
* Incorrect VRID
* Network isolation
* Firewall rules
* Incorrect Keepalived configuration

Check:

```bash
journalctl -u keepalived
```

and:

```bash
ip addr
```

The two nodes should normally coordinate VIP ownership.

---

# 14. Pi-hole Works but VIP Does Not

This is an important diagnostic scenario.

Suppose:

```text
pihole01 → DNS works
pihole02 → DNS works
VIP      → DNS fails
```

This strongly suggests that the individual DNS services are healthy and the problem is likely in the HA/network layer.

Check:

```bash
systemctl status keepalived
```

Then:

```bash
ip addr
```

Finally:

```bash
dig example.com @172.29.144.4
```

---

# 15. One Pi-hole Node Fails

Expected behavior:

```text
pihole01
   X

pihole02
   │
   └── 172.29.144.4
```

Verify:

```bash
ip addr
```

on `pihole02`.

Then test:

```bash
dig example.com @172.29.144.4
```

If DNS continues to work, the HA layer is functioning correctly.

---

# 16. DNS Fails After Failover

If the VIP moves successfully but DNS does not work:

Test the surviving Pi-hole directly:

```bash
dig example.com @172.29.144.2
```

or:

```bash
dig example.com @172.29.144.3
```

Then test its local Unbound:

```bash
dig example.com @127.0.0.1 -p 5335
```

Possible failure points:

```text
VIP
 │
 ▼
Pi-hole
 │
 ▼
Unbound
 │
 ▼
Recursive DNS
```

---

# 17. Gravity / Blocklist Troubleshooting

Pi-hole Gravity can be updated with:

```bash
pihole -g
```

If an update fails, review:

* DNS connectivity
* Blocklist URL availability
* Download errors
* Database errors
* Invalid list formats

The update should not be considered successful simply because the command completed.

Review the output for failed sources.

---

# 18. Blocklist Problems

An overly aggressive blocklist can cause legitimate websites or services to stop working.

If a domain fails to resolve:

```bash
dig example.com @172.29.144.4
```

Determine whether Pi-hole is blocking it.

If necessary, inspect Pi-hole's query and blocking information.

The correct troubleshooting process is:

```text
Domain fails
    │
    ▼
Check DNS response
    │
    ▼
Determine whether Pi-hole blocked it
    │
    ├── YES → Review blocklist / allowlist
    │
    └── NO → Continue DNS troubleshooting
```

---

# 19. Monitoring Troubleshooting

The monitoring stack itself can fail independently of the DNS infrastructure.

The monitoring VM is:

```text
monitor01
172.29.144.5
```

---

## 19.1 Prometheus

Check:

```bash
systemctl status prometheus
```

View logs:

```bash
journalctl -u prometheus
```

If metrics disappear, check whether the relevant targets are still reachable.

---

## 19.2 Node Exporter

If system metrics disappear:

```bash
systemctl status node-exporter
```

Then check:

```bash
journalctl -u node-exporter
```

The exact service name may vary depending on the installation method.

---

## 19.3 Pi-hole Exporter

If Pi-hole metrics disappear while system metrics remain available, investigate Pi-hole Exporter separately.

The problem could be:

```text
Pi-hole
   │
   └── Working
        │
        ▼
Pi-hole Exporter
   X
        │
        ▼
Prometheus
```

---

## 19.4 Blackbox Exporter

If active service probes fail:

```text
Prometheus
   │
   ▼
Blackbox Exporter
   │
   X
```

Check the Blackbox Exporter service and its logs.

Then test the target independently.

---

# 20. Grafana Troubleshooting

If dashboards show no data:

Check the following:

```text
[ ] Prometheus is running
[ ] Prometheus has recent data
[ ] Target is UP
[ ] Grafana is running
[ ] Prometheus is configured as a data source
[ ] Dashboard queries use the correct metric names
```

A Grafana dashboard showing no data does not necessarily mean the monitored system is down.

The problem could be in the monitoring pipeline.

---

# 21. Alertmanager Troubleshooting

If alerts appear in Prometheus but do not reach Telegram:

Trace the chain:

```text
Prometheus
    │
    ▼
Alert rule
    │
    ▼
Alertmanager
    │
    ▼
Telegram
```

Check Alertmanager:

```bash
systemctl status alertmanager
```

Then inspect logs:

```bash
journalctl -u alertmanager
```

Verify the notification configuration without exposing credentials publicly.

---

# 22. Telegram Troubleshooting

If Telegram notifications stop working, verify:

* Alertmanager is running
* Alert is firing
* Alert is routed correctly
* Telegram receiver configuration is correct
* Bot token is valid
* Chat ID is correct
* Network connectivity is available

Never place the actual bot token in GitHub documentation.

---

# 23. System Resource Problems

Monitoring can also identify resource issues.

Check CPU:

```bash
top
```

or:

```bash
htop
```

Check memory:

```bash
free -h
```

Check disk:

```bash
df -h
```

Check system load:

```bash
uptime
```

Check network:

```bash
ip -s link
```

---

# 24. Service Logs

When a service fails, logs should be inspected before changing configuration.

Common commands:

```bash
journalctl -u <service>
```

Recent logs:

```bash
journalctl -u <service> --since "10 minutes ago"
```

Follow logs:

```bash
journalctl -u <service> -f
```

This approach often reveals the actual cause of the failure.

---

# 25. Configuration Validation

Before restarting services after configuration changes, validate the configuration whenever the software supports it.

Examples:

### Unbound

```bash
sudo unbound-checkconf
```

### Keepalived

```bash
sudo keepalived -t
```

The principle is:

```text
Edit
  │
  ▼
Validate
  │
  ├── FAIL → Fix configuration
  │
  └── PASS
       │
       ▼
    Restart
       │
       ▼
     Test
```

---

# 26. Troubleshooting Lessons Learned

The project demonstrated several important infrastructure troubleshooting principles.

### 1. Validate configuration before restarting

A configuration parser can often identify problems before they become service outages.

### 2. Test one layer at a time

Testing Unbound independently makes it easier to distinguish an Unbound problem from a Pi-hole problem.

### 3. Service health matters

A VM being online does not necessarily mean its DNS service is healthy.

### 4. HA is not backup

Failover protects availability but does not provide historical recovery.

### 5. Monitoring needs multiple signals

Node metrics, application metrics, and active probes provide different information.

### 6. Logs are valuable

Service logs often provide the most direct explanation for failures.

---

# 27. Troubleshooting Decision Tree

The following decision tree provides a general approach to DNS failures:

```text
                 DNS NOT WORKING
                        │
                        ▼
              Can the network reach
                 the DNS server?
                   /          \
                 NO            YES
                 │              │
                 ▼              ▼
             Network        Test Unbound
             problem        directly
                                │
                         ┌──────┴──────┐
                        FAIL           PASS
                         │              │
                         ▼              ▼
                    Fix Unbound     Test Pi-hole
                                         │
                                  ┌──────┴──────┐
                                 FAIL           PASS
                                  │              │
                                  ▼              ▼
                             Fix Pi-hole    Test VIP
                                                 │
                                          ┌──────┴──────┐
                                         FAIL           PASS
                                          │              │
                                          ▼              ▼
                                     Fix Keepalived   DNS OK
```

---

# 28. Emergency Recovery

If DNS service is completely unavailable:

1. Identify which Pi-hole node is healthy.
2. Verify network connectivity.
3. Test Unbound.
4. Test Pi-hole.
5. Check Keepalived.
6. Verify VIP ownership.
7. Test DNS through the VIP.
8. Check monitoring alerts.
9. Review recent configuration changes.
10. Restore from backup if required.

Do not immediately rebuild both nodes unless the failure requires it.

---

# 29. Final Health Check

After resolving a problem, verify the complete stack:

```bash
ip addr
```

```bash
pihole status
```

```bash
sudo systemctl status unbound
```

```bash
sudo systemctl status keepalived
```

```bash
dig example.com @127.0.0.1 -p 5335
```

```bash
dig example.com @127.0.0.1
```

```bash
dig example.com @172.29.144.4
```

Then verify monitoring and alerting.

The goal is not merely to make one command work, but to confirm that the complete DNS infrastructure is healthy.

---

# 30. Summary

The troubleshooting philosophy of this project is based on **layered diagnosis**.

```text
Network
   ↓
Operating System
   ↓
Unbound
   ↓
Pi-hole
   ↓
Keepalived
   ↓
VIP
   ↓
Monitoring
   ↓
Alerting
```

The most important principle is:

> Test the smallest possible component first, then move outward through the architecture.

This approach reduces unnecessary configuration changes and makes failures easier to isolate.

The troubleshooting experience gained from this homelab is an important part of the project's value because it demonstrates practical Linux, networking, DNS, high-availability, and monitoring skills.
