# Pi-hole Configuration

## 1. Overview

Pi-hole is the primary DNS filtering and network-level ad-blocking component of this homelab.

Two Pi-hole instances are deployed:

* `pihole01` — Primary node
* `pihole02` — Secondary node

Both nodes run on Debian 13 and use Unbound as their local recursive DNS resolver.

The Pi-hole layer is combined with Keepalived to provide high availability.

---

## 2. Pi-hole Nodes

| Node       | IP Address     | Role                      |
| ---------- | -------------- | ------------------------- |
| `pihole01` | `172.29.144.3` | Primary                   |
| `pihole02` | `172.29.144.2` | Secondary                 |
| HA VIP     | `172.29.144.4` | Client-facing DNS address |

Clients should use the HA virtual IP rather than relying on a single Pi-hole node.

```text
DNS Client
    │
    ▼
172.29.144.4
    │
    ▼
Keepalived
    │
    ├── pihole01
    │
    └── pihole02
```

---

## 3. Pi-hole Versions

The project currently uses:

| Component    | Version |
| ------------ | ------- |
| Pi-hole Core | 6.4.3   |
| Pi-hole Web  | 6.6     |
| Pi-hole FTL  | 6.7     |

These versions should be checked periodically because Pi-hole components are updated independently.

To check the installed versions:

```bash
pihole -v
```

---

## 4. Pi-hole DNS Architecture

Pi-hole provides the DNS filtering layer.

The DNS request flow is:

```text
Client
   │
   │ DNS query
   ▼
Pi-hole
   │
   │ Blocklists / DNS filtering
   ▼
Unbound
   │
   │ Recursive DNS resolution
   ▼
Authoritative DNS servers
```

Pi-hole therefore performs two important functions:

1. DNS filtering
2. Forwarding allowed queries to the local recursive resolver

---

## 5. Unbound Integration

Unbound runs locally on each Pi-hole node.

The local Unbound listener is:

```text
127.0.0.1:5335
```

Pi-hole forwards allowed DNS queries to Unbound.

This means the two DNS nodes operate independently:

```text
pihole01
 └── Pi-hole
      └── Unbound :5335

pihole02
 └── Pi-hole
      └── Unbound :5335
```

This is important for high availability because either Pi-hole node has its own complete DNS resolution path.

---

## 6. DNS Client Configuration

The HA virtual IP is the preferred DNS address for clients:

```text
172.29.144.4
```

A client can therefore perform:

```bash
dig example.com @172.29.144.4
```

A successful response confirms that the HA DNS endpoint is reachable.

For comparison, each individual node can also be tested directly:

```bash
dig example.com @172.29.144.3
```

and:

```bash
dig example.com @172.29.144.2
```

---

## 7. Pi-hole Status Checks

Pi-hole provides a command-line status check:

```bash
pihole status
```

This can be used to verify whether the Pi-hole DNS filtering service is operational.

The version information can be checked with:

```bash
pihole -v
```

Additional service information can be inspected using:

```bash
systemctl status pihole-FTL
```

---

## 8. Gravity Database and Blocklists

Pi-hole uses the Gravity database to maintain domains that should be blocked.

Gravity can be updated with:

```bash
pihole -g
```

During a gravity update, Pi-hole downloads the configured blocklists, processes them, and updates the local database.

The project uses multiple blocklist sources to improve filtering coverage.

One of the configured sources is the StevenBlack hosts collection.

Example source:

```text
https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts
```

Blocklists should be selected carefully because excessive or aggressive lists can cause legitimate domains to be blocked.

---

## 9. Gravity Update Verification

After updating Gravity:

```bash
pihole -g
```

the output should be reviewed for:

* DNS resolution availability
* successful download of blocklists
* list processing errors
* database creation errors
* invalid URLs
* failed blocklist sources

A successful Gravity update indicates that Pi-hole was able to retrieve and process its configured lists.

---

## 10. DNS Testing

DNS functionality should be tested at multiple layers.

### Test Unbound directly

```bash
dig example.com @127.0.0.1 -p 5335
```

This verifies that Unbound can resolve DNS queries.

### Test Pi-hole locally

```bash
dig example.com @127.0.0.1
```

This verifies Pi-hole's DNS service.

### Test pihole01

```bash
dig example.com @172.29.144.3
```

### Test pihole02

```bash
dig example.com @172.29.144.2
```

### Test the HA VIP

```bash
dig example.com @172.29.144.4
```

Testing all four endpoints helps identify where a DNS failure is occurring.

---

## 11. DNS Testing Model

The testing hierarchy can be represented as:

```text
                    DNS Test
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Unbound       Pi-hole       HA VIP
       :5335          :53          :53
          │            │            │
          ▼            ▼            ▼
       Resolver      Filter       Failover
```

If Unbound works but Pi-hole does not, the problem is likely between Pi-hole and Unbound.

If both individual Pi-hole nodes work but the VIP fails, the problem is likely in the Keepalived/network layer.

---

## 12. Pi-hole and High Availability

Pi-hole itself does not provide the network-level failover mechanism used by this project.

Keepalived provides the shared virtual IP.

The architecture is therefore:

```text
                172.29.144.4
                  HA VIP
                     │
             ┌───────┴───────┐
             │               │
             ▼               ▼
        pihole01         pihole02
       172.29.144.3     172.29.144.2
             │               │
             ▼               ▼
          Unbound          Unbound
```

The active node owns the VIP.

If the active node becomes unavailable, the VIP can move to the other Pi-hole node.

---

## 13. Pi-hole FTL

Pi-hole FTL provides the DNS engine and statistics functionality used by Pi-hole.

Its service can be checked with:

```bash
systemctl status pihole-FTL
```

Logs can be inspected with:

```bash
journalctl -u pihole-FTL
```

For troubleshooting, recent log entries can be viewed with:

```bash
journalctl -u pihole-FTL --since "10 minutes ago"
```

---

## 14. Performance and Reliability

The Pi-hole layer is designed to provide:

* Local DNS filtering
* Recursive DNS resolution through Unbound
* Redundancy through two nodes
* Automatic VIP failover
* DNS monitoring
* Metrics collection
* Alerting
* Backup and synchronization

The architecture avoids making a single Pi-hole VM a critical single point of failure.

---

## 15. Failure Scenarios

### Scenario 1 — pihole01 fails

```text
pihole01
   X

pihole02
   │
   └── 172.29.144.4
```

The virtual IP should move to `pihole02`.

Clients continue using:

```text
172.29.144.4
```

without needing to change their DNS configuration.

---

### Scenario 2 — pihole02 fails

If `pihole01` is the active node, DNS service continues through:

```text
pihole01
172.29.144.3
    │
    └── 172.29.144.4
```

The failure of the standby node does not immediately interrupt DNS service.

The monitoring system should detect the failed node and generate an alert.

---

### Scenario 3 — Unbound failure

If Pi-hole is running but Unbound is unavailable, allowed DNS queries may fail because Pi-hole cannot obtain upstream recursive responses.

This is why Unbound is monitored separately.

A useful diagnostic sequence is:

```bash
systemctl status unbound
```

then:

```bash
sudo unbound-checkconf
```

and finally:

```bash
dig example.com @127.0.0.1 -p 5335
```

---

## 16. Troubleshooting Checklist

When DNS is not working, check the components in this order:

### 1. Network

```bash
ip addr
ip route
ping <gateway>
```

### 2. Unbound

```bash
systemctl status unbound
sudo unbound-checkconf
```

### 3. Test Unbound

```bash
dig example.com @127.0.0.1 -p 5335
```

### 4. Pi-hole FTL

```bash
systemctl status pihole-FTL
```

### 5. Test Pi-hole

```bash
dig example.com @127.0.0.1
```

### 6. Keepalived

```bash
systemctl status keepalived
```

### 7. Check VIP

```bash
ip addr
```

Look for:

```text
172.29.144.4
```

### 8. Test HA DNS

```bash
dig example.com @172.29.144.4
```

This layered approach makes it easier to determine whether the problem is related to networking, Unbound, Pi-hole, or Keepalived.

---

## 17. Security Considerations

The Pi-hole servers should not expose unnecessary services to untrusted networks.

Recommended practices include:

* Use strong administrative credentials.
* Do not publish Pi-hole administration interfaces directly to the Internet.
* Restrict SSH access.
* Keep Debian and Pi-hole updated.
* Do not commit credentials or API tokens to GitHub.
* Protect monitoring credentials and notification tokens.
* Restrict DNS access to trusted networks where appropriate.
* Regularly review blocklists and custom DNS entries.

---

## 18. Summary

Pi-hole forms the DNS filtering layer of the homelab.

The complete DNS stack is:

```text
Client
   │
   ▼
Keepalived VIP
172.29.144.4
   │
   ▼
Pi-hole
   │
   ▼
Unbound
127.0.0.1:5335
   │
   ▼
Recursive DNS
```

Two independent Pi-hole + Unbound nodes provide redundancy, while Keepalived provides the client-facing virtual IP.

Monitoring, alerting, backup, and synchronization provide additional operational reliability.
