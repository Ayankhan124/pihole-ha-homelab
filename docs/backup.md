# 💾 Backup, Synchronization & Disaster Recovery

High availability (Keepalived) guarantees service continuity if a node fails, but it does not protect against accidental configuration changes, data corruption, or catastrophic VM loss. 

This framework establishes the methods used to keep `pihole01` and `pihole02` consistent, and outlines the recovery procedures for a total node failure.

---

## 1. Architecture & Philosophy

A robust infrastructure requires three distinct layers of resilience:

1. **High Availability (Availability):** Keepalived handles real-time node failure.
2. **Synchronization (Consistency):** Keeps core configurations (adlists, whitelists, local DNS) identical between `pihole01` and `pihole02`.
3. **Backup (Recovery):** Offline archives of configurations and OS states to recover from data loss or bad synchronizations.

**Redundancy + Backup + Documentation = Recoverable Infrastructure**

---

## 2. Synchronization Strategy

Synchronization ensures both nodes resolve queries identically. However, synchronization is *not* a backup. If a bad configuration change is made on `pihole01`, it will sync to `pihole02` and break both nodes.

### Synchronization Scope
We only synchronize application-layer configurations. We **do not** synchronize:
* Network interfaces (`/etc/network/interfaces`)
* Keepalived node states (`/etc/keepalived/keepalived.conf`)
* Hostnames or machine-specific secrets

### Execution
*(Replace this section with your specific sync method, e.g., Gravity Sync, rsync, or a custom script)*
```bash
# Example: Using a custom rsync script
[INSERT YOUR SYNC TOOL/SCRIPT HERE]
```

---

## 3. Backup Strategy

Backups should be stored externally from the Pi-hole VMs (e.g., on a NAS or isolated storage). 

### Critical Backup Paths
To successfully rebuild a node from a fresh Debian install, ensure the following paths are backed up:
* `/etc/pihole/` (Pi-hole configurations and databases)
* `/etc/unbound/` (Recursive DNS settings)
* `/etc/keepalived/` (VRRP configurations)
* `/etc/network/interfaces` (Static IP configurations)

### Execution
*(Specify your backup method here, e.g., Teleport/cron script/VM snapshots)*
```bash
# Example: Creating a tarball of critical configs
sudo tar -czvf /mnt/nas/pihole01_backup_$(date +%F).tar.gz /etc/pihole /etc/unbound /etc/keepalived /etc/network/interfaces
```
*Note: Ensure backup archives containing sensitive data (API tokens, passwords) are secured with `chmod 600` or encrypted.*

---

## 4. Disaster Recovery (DR) Procedure

If a node suffers total failure or corruption, follow this strict recovery sequence to prevent a partially restored node from stealing the Keepalived VIP and dropping DNS queries.

### Phase 1: Foundation
1. **Rebuild VM:** Deploy a fresh Debian 13 installation.
2. **Network Setup:** Restore `/etc/network/interfaces` and verify the static IP.
3. **Install Packages:** Install Pi-hole, Unbound, and Keepalived (Do *not* start Keepalived yet).

### Phase 2: Configuration Restoration
1. Restore `/etc/pihole/` and `/etc/unbound/` from the backup archive.
2. **Validate Unbound:**
   ```bash
   sudo unbound-checkconf
   sudo systemctl restart unbound
   dig example.com @127.0.0.1 -p 5335
   ```
3. **Validate Pi-hole:**
   ```bash
   pihole status
   dig example.com @127.0.0.1
   ```

### Phase 3: High Availability Re-entry
Only proceed when Phase 2 DNS validation is 100% successful.
1. Restore `/etc/keepalived/keepalived.conf`.
2. **Validate Keepalived:**
   ```bash
   sudo keepalived -t
   sudo systemctl start keepalived
   ip addr # Verify VIP behavior
   ```
3. Reconnect monitoring exporters and verify synchronization state.

---

## 5. Routine Verification

Backups and syncs are only valid if they are tested. Periodically verify:
- [ ] Sync completes without overwriting node-specific data (IPs/hostnames).
- [ ] Backup archives exist and are not corrupted (`tar -tzf archive.tar.gz`).
- [ ] A full Disaster Recovery test (rebuilding a VM from scratch) succeeds using only the documentation and backup files.