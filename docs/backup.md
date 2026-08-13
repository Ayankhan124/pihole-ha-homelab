# Backup and Synchronization Framework

## 1. Overview

The Pi-hole HA homelab includes a backup and synchronization framework designed to improve recoverability and maintain consistency between the two DNS nodes.

The system has two related but different goals:

* **Backup** — Protect configuration and operational data from loss.
* **Synchronization** — Keep important configuration between `pihole01` and `pihole02` consistent.

High availability alone does not provide protection against accidental configuration changes, corruption, or deletion.

The overall architecture is:

```text
                 Pi-hole HA
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
       Backup              Synchronization
          │                     │
          ▼                     ▼
     Recovery data       pihole01 ↔ pihole02
```

---

## 2. Why Backup Is Required

Keepalived provides service availability, but it does not protect configuration data.

For example:

```text
pihole01
   │
   ├── Pi-hole configuration
   ├── DNS configuration
   ├── Custom settings
   └── Other operational data
```

If a configuration is accidentally deleted or incorrectly modified, Keepalived can still fail over to another node, but the underlying configuration problem may remain.

Therefore:

```text
High Availability ≠ Backup
```

The two systems complement each other.

---

## 3. High Availability vs Backup

| Feature                             | High Availability | Backup       |
| ----------------------------------- | ----------------- | ------------ |
| Protects against node failure       | Yes               | Not directly |
| Provides service continuity         | Yes               | No           |
| Protects configuration history      | No                | Yes          |
| Helps recover deleted data          | No                | Yes          |
| Protects against accidental changes | Limited           | Yes          |
| Provides another DNS node           | Yes               | No           |
| Used for disaster recovery          | Limited           | Yes          |

The homelab uses both mechanisms.

---

## 4. Backup Strategy

The backup framework should preserve important configuration and operational data required to rebuild a Pi-hole node.

Typical backup categories include:

```text
/etc/pihole/
/etc/unbound/
/etc/keepalived/
/etc/systemd/
/etc/network/
```

The exact paths should be verified against the current deployment before being treated as the authoritative backup list.

Sensitive files should be handled carefully.

---

## 5. Sensitive Data

Some configuration files can contain sensitive information.

Examples include:

* Passwords
* Authentication secrets
* API tokens
* Telegram bot tokens
* Private keys
* Network credentials

These values must not be committed to a public GitHub repository.

A public documentation example should use placeholders:

```text
TELEGRAM_BOT_TOKEN=<redacted>
```

rather than the actual secret.

---

## 6. Backup Design

The backup system can be viewed as:

```text
                 DNS Infrastructure
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
          pihole01               pihole02
              │                     │
              └──────────┬──────────┘
                         │
                         ▼
                     Backups
                         │
                         ▼
                  Recovery Storage
```

The important principle is that the backup copy should not depend entirely on the health of the node being backed up.

---

## 7. Synchronization

Synchronization keeps selected configuration data consistent between the two Pi-hole nodes.

Conceptually:

```text
pihole01
   │
   │ Synchronization
   ▼
pihole02
```

Depending on the synchronization mechanism, the direction may be:

```text
pihole01 → pihole02
```

or a controlled bidirectional process.

The synchronization design should avoid simultaneous conflicting changes.

---

## 8. Synchronization vs Backup

Synchronization should not be treated as a replacement for backups.

For example:

```text
pihole01
   │
   │ bad configuration change
   ▼
pihole02
```

If the bad configuration is automatically synchronized, both nodes can become incorrect.

A backup provides a separate recovery point.

Therefore:

```text
Synchronization = Consistency
Backup          = Recovery
```

Both are required for a robust infrastructure.

---

## 9. Synchronization Scope

Only required configuration and data should be synchronized.

The synchronization process should avoid copying:

* Temporary files
* Runtime state
* Logs unless specifically required
* System-generated caches
* Secrets that should remain node-specific
* Hardware-specific configuration
* Network configuration that differs between nodes

For example:

```text
pihole01                         pihole02
   │                                │
   ├── Shared configuration ───────►│
   │                                │
   ├── Shared DNS settings ────────►│
   │                                │
   └── Node-specific settings       │
```

The exact synchronization list should match the scripts currently deployed in the homelab.

---

## 10. Backup Integrity

A backup is useful only if it can actually be restored.

Therefore backups should be periodically verified.

A basic verification process is:

```text
Create backup
     │
     ▼
Verify files exist
     │
     ▼
Verify archive integrity
     │
     ▼
Test restoration
     │
     ▼
Confirm services work
```

For archive-based backups, integrity can be checked using the appropriate archive verification command.

---

## 11. Restoration Process

A typical recovery process is:

```text
Node failure
     │
     ▼
Identify failed component
     │
     ▼
Reinstall/rebuild node if required
     │
     ▼
Restore configuration
     │
     ▼
Validate configuration
     │
     ▼
Start services
     │
     ▼
Test DNS
     │
     ▼
Return node to HA cluster
```

The recovery process should be performed carefully so that a partially restored node does not immediately become the active DNS node.

---

## 12. Pi-hole Recovery

After restoring Pi-hole configuration, verify:

```bash
pihole status
```

Then test DNS:

```bash
dig example.com @127.0.0.1
```

Also verify the web interface and relevant Pi-hole configuration.

---

## 13. Unbound Recovery

After restoring Unbound configuration:

```bash
sudo unbound-checkconf
```

If the configuration is valid:

```bash
sudo systemctl restart unbound
```

Then test:

```bash
dig example.com @127.0.0.1 -p 5335
```

Only after successful DNS testing should the node be considered ready for HA operation.

---

## 14. Keepalived Recovery

Keepalived configuration should also be restored carefully.

First validate the configuration:

```bash
sudo keepalived -t
```

Then check the service:

```bash
sudo systemctl status keepalived
```

Verify the VIP:

```bash
ip addr
```

The node should not unexpectedly acquire the VIP while it is still being restored or tested.

---

## 15. Recovery Order

A safe recovery sequence is:

```text
1. Operating system
       │
       ▼
2. Network configuration
       │
       ▼
3. Pi-hole
       │
       ▼
4. Unbound
       │
       ▼
5. Configuration restoration
       │
       ▼
6. DNS testing
       │
       ▼
7. Keepalived
       │
       ▼
8. VIP verification
       │
       ▼
9. Monitoring
       │
       ▼
10. Synchronization
```

The exact order can vary depending on the failure scenario.

---

## 16. Backup Verification Checklist

A backup should periodically be checked using:

```text
[ ] Backup completes successfully
[ ] Expected files are present
[ ] Backup is readable
[ ] Backup is not corrupted
[ ] Sensitive information is protected
[ ] Storage has sufficient capacity
[ ] Restoration procedure is documented
[ ] Restoration has been tested
```

---

## 17. Synchronization Verification

After synchronization:

```text
[ ] pihole01 is reachable
[ ] pihole02 is reachable
[ ] Synchronization completes successfully
[ ] Expected configuration exists on both nodes
[ ] Node-specific settings remain correct
[ ] Pi-hole remains operational
[ ] Unbound remains operational
[ ] Keepalived remains operational
```

Synchronization should be verified rather than assuming that a successful script execution means the nodes are identical.

---

## 18. Failure Scenarios

### Scenario 1 — Single Node Failure

```text
pihole01
   X

pihole02
   │
   └── VIP
```

Keepalived maintains DNS availability through the surviving node.

The failed node can later be rebuilt and restored from backup.

---

### Scenario 2 — Configuration Corruption

```text
Configuration
      │
      ▼
Incorrect change
      │
      ▼
DNS problems
```

The previous known-good backup can be used as a recovery point.

---

### Scenario 3 — Synchronization Propagates an Incorrect Change

If an incorrect configuration is synchronized to both nodes:

```text
pihole01 ───────► pihole02
   │                  │
   └── Bad config ────┘
```

HA alone cannot solve this problem.

A separate backup or recovery point is required.

---

## 19. Disaster Recovery

For a complete infrastructure failure, the recovery objective is to rebuild the DNS environment from a clean Debian installation.

The general process is:

```text
Fresh Debian installation
          │
          ▼
Network configuration
          │
          ▼
Install Pi-hole
          │
          ▼
Install Unbound
          │
          ▼
Restore configuration
          │
          ▼
Validate DNS
          │
          ▼
Install/configure Keepalived
          │
          ▼
Restore HA configuration
          │
          ▼
Test failover
          │
          ▼
Return node to production
```

This makes the project reproducible rather than depending entirely on the original VM.

---

## 20. Backup Security

Backups can contain information that is more sensitive than the running system.

Backup storage should therefore be protected using appropriate permissions and access controls.

At minimum:

```bash
chmod 600 <sensitive-backup>
```

can be used for sensitive individual files where appropriate.

Backup locations should also be protected from unauthorized access.

---

## 21. GitHub and Backup Separation

GitHub should be used for:

* Documentation
* Sanitized configuration examples
* Scripts
* Infrastructure diagrams
* Version-controlled project files

GitHub should **not** be used as a place to publish raw private backups.

For example, do not commit:

```text
/etc/shadow
private keys
API tokens
password files
Pi-hole authentication secrets
Telegram bot tokens
```

Instead, use sanitized examples:

```text
configs/
├── keepalived.conf.example
├── unbound.conf.example
└── pihole-settings.example
```

---

## 22. Version Control

Configuration changes that are safe to publish can be tracked through Git.

For example:

```bash
git status
```

Review changes:

```bash
git diff
```

Stage changes:

```bash
git add .
```

Commit them:

```bash
git commit -m "config: update DNS settings"
```

This provides a history of infrastructure changes.

Git version history and operational backups serve different purposes and should both be maintained where appropriate.

---

## 23. Recovery Testing

A backup system should not be considered reliable merely because backup files exist.

The most important test is restoration.

A recovery test should answer:

> Can a new Debian VM be rebuilt and returned to a functioning DNS node using the available documentation and backup?

A successful recovery test should include:

```text
[ ] Rebuild Debian
[ ] Configure network
[ ] Install required packages
[ ] Restore configuration
[ ] Validate Unbound
[ ] Validate Pi-hole
[ ] Test DNS
[ ] Configure Keepalived
[ ] Test VIP
[ ] Test failover
[ ] Reconnect monitoring
```

---

## 24. Operational Philosophy

The backup and synchronization framework follows three principles:

### Redundancy

Keep two operational DNS nodes.

### Recoverability

Maintain backups that can be used to rebuild a node.

### Reproducibility

Document the installation and configuration process so the infrastructure can be recreated.

The combination is:

```text
Redundancy
    +
Backup
    +
Documentation
    =
Recoverable Infrastructure
```

---

## 25. Summary

The Pi-hole HA homelab uses multiple layers of resilience.

```text
                    DNS Infrastructure
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
        High Availability             Backup
          Keepalived                    │
             │                          ▼
             ▼                     Recovery Data
       pihole01 ↔ pihole02
             │
             ▼
      Synchronization
```

Keepalived maintains service availability.

Synchronization maintains consistency between nodes.

Backups provide recovery from configuration loss, corruption, or other failures.

Together with the project's installation documentation, these mechanisms make the DNS infrastructure more resilient and reproducible.
