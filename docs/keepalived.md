# 🔄 Keepalived High Availability Configuration

Keepalived provides the High Availability (HA) layer for the Pi-hole DNS infrastructure. By utilizing the Virtual Router Redundancy Protocol (VRRP), Keepalived shares a single Virtual IP (VIP) between the two nodes. 

## 📖 The "Why"
If DNS clients are pointed directly at a single Pi-hole node and that node fails, DNS resolution stops and internet access drops. Keepalived abstracts the DNS server by presenting a single VIP (`172.29.144.4`) to clients. If the active node goes offline, the VIP seamlessly fails over to the standby node, requiring zero configuration changes on the client side.

---

## 🏗️ Architecture & State

Both nodes must share the same Virtual Router ID (VRID) to participate in the HA group. The node with the higher configured priority assumes the active role.

| Node       | IP Address     | VRRP State | Priority |
| ---------- | -------------: | ---------- | -------- |
| `pihole01` | `172.29.144.3` | `MASTER`   | Higher (e.g., 150) |
| `pihole02` | `172.29.144.2` | `BACKUP`   | Lower (e.g., 100)  |
| **HA VIP** | **`172.29.144.4`**| **Client-facing DNS** | **N/A** |

```mermaid
flowchart TD
    Client[DNS Clients] -->|Query: 172.29.144.4| VIP(Virtual IP: 172.29.144.4)
    
    subgraph VRRP HA Cluster
        Active[pihole01: .3<br>State: MASTER]
        Standby[pihole02: .2<br>State: BACKUP]
        Active <..>|VRRP Heartbeat| Standby
    end

    VIP -->|Routes to| Active
    Active -.->|If Node/Service Fails| Standby
    
    subgraph Local Services
        DNS1(Pi-hole + Unbound)
        DNS2(Pi-hole + Unbound)
    end
    
    Active --> DNS1
    Standby --> DNS2
```

---

## ⚙️ Installation & Configuration

Keepalived must be installed and attached to the primary network interface on both nodes.

```bash
sudo apt update && sudo apt install keepalived -y
```

### Keepalived Configuration (`/etc/keepalived/keepalived.conf`)
*Note: This configuration includes a service-level health check. If Pi-hole fails but the OS remains online, Keepalived will aggressively fail over to protect DNS resolution.*

**On `pihole01` (MASTER):**
```text
global_defs {
    router_id PIHOLE01
    enable_script_security
    script_user root
}

vrrp_script chk_pihole {
    script "/opt/homelab/health/health-score.sh"
    interval 2
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eno1
    virtual_router_id 51
    priority 200
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass                      # <---- password here 
    }

    virtual_ipaddress {
        172.29.144.4/27 dev eno1
    }

    track_script {
        chk_pihole
    }
}

```

**On `pihole02` (BACKUP):**
Copy the exact configuration above to `pihole02`, but change the following two lines:
```text
    state BACKUP
    priority 100
```

Enable and start the service on both nodes:
```bash
sudo systemctl enable --now keepalived
```

---

## 🧪 Validation & Failover Testing

High Availability should be verified deliberately rather than assuming it works. 

**1. Verify VIP Ownership:**
Run `ip addr` on both nodes. Only `pihole01` should show the `172.29.144.4` address.

**2. Test Service-Level Failover:**
Simulate a DNS failure on the MASTER node by stopping Pi-hole:
```bash
# On pihole01
pihole disable
```
*Expected Behavior:* The health check will fail, priority will drop, and `pihole02` will immediately take ownership of the VIP. Verify with `ip addr` on `pihole02`.

**3. Test Client Resolution:**
From a client machine, verify DNS still resolves during the failover:
```bash
dig example.com @172.29.144.4
```

**4. Test Failback:**
Enable Pi-hole on `pihole01` again (`pihole enable`). Because it has a higher base priority (150 vs 100), it should preempt `pihole02` and take the VIP back.

---

## 🛠️ Troubleshooting & Logs

If the VIP is missing, or if a split-brain scenario occurs where both nodes claim ownership of the VIP:

**1. Validate the Configuration:**
```bash
sudo keepalived -t
```
**2. Check the Service State:**
```bash
sudo systemctl status keepalived
```
**3. Monitor VRRP State Transitions:**
```bash
sudo journalctl -u keepalived -f
```
*(Common issues include mismatched `virtual_router_id` values, incorrect interface names, or firewall rules blocking VRRP multicast traffic).*