# Keepalived High Availability Configuration

## 1. Overview

Keepalived provides the High Availability (HA) layer for the Pi-hole DNS infrastructure.

The two Pi-hole servers have individual IP addresses, while Keepalived provides a shared Virtual IP (VIP) that DNS clients use.

```text
pihole01 → 172.29.144.3
pihole02 → 172.29.144.2
VIP      → 172.29.144.4
```

The primary purpose of Keepalived is to prevent a single Pi-hole server failure from interrupting DNS service.

---

## 2. High Availability Architecture

The DNS clients use only the virtual IP:

```text
DNS Client
     │
     │ DNS query
     ▼
172.29.144.4
   Virtual IP
     │
     ▼
┌───────────────┐
│   Keepalived  │
└───────┬───────┘
        │
   ┌────┴────┐
   │         │
   ▼         ▼
pihole01   pihole02
 .3          .2
```

Only the active Keepalived node owns the VIP at a given time.

---

## 3. Nodes

| Node       |     IP Address | HA Role           |
| ---------- | -------------: | ----------------- |
| `pihole01` | `172.29.144.3` | Primary           |
| `pihole02` | `172.29.144.2` | Secondary         |
| HA VIP     | `172.29.144.4` | Client-facing DNS |

The exact MASTER/BACKUP state is determined by the Keepalived configuration and priority values.

---

## 4. Virtual IP

The shared virtual IP is:

```text
172.29.144.4
```

Clients should use this address as their DNS server.

This creates an abstraction between the clients and the individual Pi-hole nodes.

Instead of configuring:

```text
DNS → 172.29.144.3
```

clients use:

```text
DNS → 172.29.144.4
```

If the active node fails, the client configuration does not need to change.

---

## 5. VRRP

Keepalived uses **VRRP (Virtual Router Redundancy Protocol)** to manage ownership of the virtual IP.

The nodes communicate with each other to determine which node should own the VIP.

Conceptually:

```text
              VRRP
        ┌──────────────┐
        │              │
        ▼              ▼
  pihole01          pihole02
   .3                  .2
        │              │
        └──────┬───────┘
               │
               ▼
          VIP .4
```

The node with the appropriate priority becomes the active owner of the VIP.

---

## 6. Keepalived Installation

Keepalived was installed on both Pi-hole nodes.

On Debian:

```bash
sudo apt update
sudo apt install keepalived
```

The service can be checked with:

```bash
sudo systemctl status keepalived
```

Keepalived can be enabled at boot:

```bash
sudo systemctl enable keepalived
```

---

## 7. Configuration File

The main Keepalived configuration file is:

```text
/etc/keepalived/keepalived.conf
```

The configuration defines:

* VRRP instance
* Node state
* Interface
* Virtual Router ID
* Priority
* Authentication
* Virtual IP
* Health-check logic

The exact configuration should be kept in the repository only after removing secrets and environment-specific credentials.

---

## 8. MASTER and BACKUP

The two nodes use different priorities.

The node with the higher priority normally becomes the active node.

Conceptually:

```text
pihole01
Priority: higher
State:    MASTER

pihole02
Priority: lower
State:    BACKUP
```

The exact values should match the configuration deployed on the servers.

---

## 9. Virtual Router ID

The VRRP instance uses a Virtual Router ID (VRID).

Both Keepalived nodes participating in the same HA group must use the same VRID.

Conceptually:

```text
pihole01 ── VRID ── pihole02
```

The VRID identifies the virtual router represented by the Keepalived instance.

---

## 10. Interface

Keepalived must be attached to the network interface used by the Pi-hole nodes.

The interface can be identified using:

```bash
ip addr
```

or:

```bash
ip link
```

The configured interface should be verified before deploying the Keepalived configuration.

---

## 11. Health Checking

A major part of the HA design is health checking.

A simple VIP failover is not enough if the operating system remains online while the DNS service itself has failed.

The health-check mechanism can therefore verify that the required DNS service is operational.

The general concept is:

```text
Keepalived
    │
    ▼
Health Check
    │
    ├── Healthy
    │      │
    │      ▼
    │   Keep VIP
    │
    └── Failed
           │
           ▼
       Reduce priority
           │
           ▼
       VIP failover
```

This makes the HA system service-aware rather than relying only on whether the VM is reachable.

---

## 12. Why Service Health Matters

Consider this failure:

```text
pihole01 VM
     │
     ├── Network: UP
     ├── SSH: UP
     ├── Keepalived: UP
     └── Pi-hole DNS: DOWN
```

If Keepalived only checks whether the VM is reachable, it might continue keeping the VIP on `pihole01`.

DNS clients would then continue sending queries to a node that cannot answer them.

A service-level health check helps prevent this situation.

---

## 13. Normal Operation

Under normal conditions:

```text
                    VIP
                172.29.144.4
                      │
                      ▼
                 pihole01
                172.29.144.3
                      │
                ┌─────┴─────┐
                ▼           ▼
             Pi-hole     Unbound
```

`pihole02` remains available as the standby node.

```text
pihole02
172.29.144.2
    │
    └── Standby
```

---

## 14. Failover Scenario

If `pihole01` fails its HA health requirements:

```text
pihole01
172.29.144.3
    X
```

Keepalived on `pihole02` can take ownership of the VIP:

```text
                VIP
            172.29.144.4
                  │
                  ▼
             pihole02
            172.29.144.2
                  │
             ┌────┴────┐
             ▼         ▼
          Pi-hole    Unbound
```

Clients continue using:

```text
172.29.144.4
```

No client-side DNS configuration change is required.

---

## 15. Failback

When the failed node becomes healthy again, Keepalived can restore the normal HA state depending on the configured priorities and preemption behavior.

The desired behavior should be documented explicitly because automatic failback is not always desirable in production environments.

For this homelab, the priority configuration determines which node should normally own the VIP.

---

## 16. Checking VIP Ownership

On either Pi-hole node:

```bash
ip addr
```

The active node should show:

```text
172.29.144.4
```

The standby node should not normally have the VIP assigned.

This is one of the simplest ways to determine which node currently owns the virtual IP.

---

## 17. Checking Keepalived

Check the service:

```bash
sudo systemctl status keepalived
```

Check recent logs:

```bash
sudo journalctl -u keepalived --since "10 minutes ago"
```

For continuous log monitoring:

```bash
sudo journalctl -u keepalived -f
```

These logs are useful when troubleshooting state transitions.

---

## 18. Testing Failover

HA should be tested deliberately rather than assuming it works.

### Test 1 — Identify the active node

On both nodes:

```bash
ip addr
```

Determine which node owns:

```text
172.29.144.4
```

---

### Test 2 — Test DNS through the VIP

From a client:

```bash
dig example.com @172.29.144.4
```

Verify that DNS resolution works.

---

### Test 3 — Stop Keepalived on the active node

On the active node:

```bash
sudo systemctl stop keepalived
```

Then check the VIP on the other node:

```bash
ip addr
```

The VIP should move to the surviving node if the configuration is functioning correctly.

---

### Test 4 — Test DNS again

From a client:

```bash
dig example.com @172.29.144.4
```

DNS should continue working through the surviving Pi-hole node.

---

### Test 5 — Restore Keepalived

Start Keepalived again:

```bash
sudo systemctl start keepalived
```

Then verify the resulting VIP ownership:

```bash
ip addr
```

---

## 19. Testing Service-Level Failure

A more meaningful HA test is to simulate a DNS service failure rather than shutting down the entire VM.

For example, if the configured health check monitors Pi-hole FTL, temporarily stopping the relevant service can be used to test whether Keepalived detects the failure.

Before performing this test, make sure you have console or SSH access to both nodes.

The expected behavior is:

```text
Pi-hole service failure
        │
        ▼
Health check fails
        │
        ▼
Keepalived reacts
        │
        ▼
VIP moves to healthy node
        │
        ▼
DNS remains available
```

---

## 20. Verification After Failover

After a failover test, verify all layers.

### Check VIP

```bash
ip addr
```

### Check Pi-hole

```bash
pihole status
```

### Check Unbound

```bash
sudo systemctl status unbound
```

### Test DNS

```bash
dig example.com @172.29.144.4
```

### Check Keepalived

```bash
sudo systemctl status keepalived
```

### Check logs

```bash
sudo journalctl -u keepalived --since "10 minutes ago"
```

---

## 21. Failure Model

The HA design handles failures at multiple levels:

```text
                 Failure
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
     Node failure          Service failure
        │                       │
        ▼                       ▼
   Keepalived detects     Health check detects
        │                       │
        └───────────┬───────────┘
                    ▼
              VIP failover
                    │
                    ▼
             Healthy Pi-hole
                    │
                    ▼
                Unbound
                    │
                    ▼
               DNS works
```

This provides more meaningful redundancy than simply running two DNS servers.

---

## 22. Security Considerations

Keepalived configuration should be protected because it controls the DNS service's virtual IP.

Important considerations include:

* Protect `/etc/keepalived/keepalived.conf`.
* Do not publish sensitive authentication values.
* Restrict network access where appropriate.
* Use appropriate file permissions.
* Do not commit secrets to GitHub.
* Verify VRRP traffic is limited to the intended network.
* Monitor unexpected VIP changes.

If authentication is configured, the actual secret should never be stored in public documentation.

Use a placeholder such as:

```text
<VRRP_AUTH_PASSWORD>
```

in public documentation.

---

## 23. Troubleshooting

### VIP does not appear

Check:

```bash
sudo systemctl status keepalived
```

Then:

```bash
sudo journalctl -u keepalived
```

Check the interface:

```bash
ip addr
```

Check the configuration:

```bash
sudo keepalived -t
```

---

### Both nodes appear to own the VIP

This can indicate a VRRP communication or configuration problem.

Check:

```bash
sudo journalctl -u keepalived
```

Also verify:

* VRID
* Interface
* Priority
* VRRP configuration
* Network connectivity
* Firewall rules

Two nodes should not normally behave as simultaneous owners of the same VIP.

---

### VIP does not fail over

Check:

```bash
systemctl status keepalived
```

Then verify the health-check configuration.

Also test whether the health-check command itself works correctly when executed manually.

---

### DNS fails after failover

Check the surviving node independently:

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

This determines whether the problem is:

```text
VIP / Keepalived
        or
Pi-hole
        or
Unbound
```

---

## 24. Operational Checklist

Before considering the HA system healthy:

```text
[ ] pihole01 is reachable
[ ] pihole02 is reachable
[ ] Keepalived is active on both nodes
[ ] VIP exists on the active node
[ ] VIP is absent from the standby node
[ ] Pi-hole is running on both nodes
[ ] Unbound is running on both nodes
[ ] DNS works through each individual node
[ ] DNS works through the VIP
[ ] Node failure causes VIP failover
[ ] DNS remains available after failover
[ ] Monitoring detects node/service failures
```

---

## 25. Summary

Keepalived provides the high-availability mechanism for the Pi-hole DNS infrastructure.

The important design principle is that clients use a stable virtual IP:

```text
172.29.144.4
```

rather than depending directly on either Pi-hole server.

The final HA architecture is:

```text
                    DNS Clients
                         │
                         ▼
                  172.29.144.4
                     HA VIP
                         │
               ┌─────────┴─────────┐
               │                   │
               ▼                   ▼
          pihole01             pihole02
       172.29.144.3         172.29.144.2
               │                   │
               ▼                   ▼
            Pi-hole             Pi-hole
               │                   │
               ▼                   ▼
            Unbound              Unbound
               │                   │
               └─────────┬─────────┘
                         ▼
                    Recursive DNS
```

This design provides redundancy at the DNS server level while keeping the client-facing DNS address unchanged during node failure.
