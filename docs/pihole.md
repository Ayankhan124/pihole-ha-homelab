# 🕳️ Pi-hole Configuration & Operations

Pi-hole is the primary DNS filtering and network-level ad-blocking component of this homelab. 

Two independent Pi-hole instances (`pihole01` and `pihole02`) are deployed behind a Keepalived Virtual IP (`172.29.144.4`), ensuring that DNS filtering remains highly available.

---

## 🏗️ Architecture & Traffic Flow

Pi-hole does not perform recursive DNS resolution itself. Its primary functions are:
1. Receiving DNS queries from clients via the Keepalived VIP.
2. Checking the requested domain against local blocklists (the Gravity database).
3. Forwarding allowed queries to the local Unbound resolver running on `127.0.0.1:5335`.

```text
Client -> VIP (172.29.144.4) -> Pi-hole (Filtering) -> Unbound (127.0.0.1:5335) -> Authoritative Servers
```

---

## ⚙️ Core Configuration

After installing Pi-hole, you must configure it to forward traffic to your local Unbound instance.

1. Log into the Pi-hole Web Interface (`http://172.29.144.3/admin` and `http://172.29.144.2/admin`).
2. Navigate to **Settings** -> **DNS**.
3. Under **Upstream DNS Servers**, uncheck all third-party providers (e.g., Google, Cloudflare).
4. Under **Custom 1 (IPv4)**, enter: `127.0.0.1#5335`
5. Save the configuration.

### Managing Blocklists (Gravity)
Pi-hole uses the Gravity database to maintain blocked domains. The project uses multiple blocklist sources to improve filtering coverage, such as the StevenBlack hosts collection:
`https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts`

*Caution: Blocklists should be selected carefully; overly aggressive lists can break legitimate domains and services.*

To manually update the Gravity database and process new lists:
```bash
pihole -g
```

---

## 🛠️ Layered Diagnostics

When DNS is failing, do not immediately assume the entire stack is broken. Test the infrastructure systematically from the inside out to isolate the failure point.

**1. Test the Unbound Resolver:**
*Bypasses Pi-hole and tests if Unbound can reach the internet.*
```bash
dig example.com @127.0.0.1 -p 5335
```

**2. Test the Pi-hole Filter:**
*Bypasses the network VIP to ensure Pi-hole is filtering and forwarding correctly.*
```bash
dig example.com @127.0.0.1
```

**3. Test the High Availability VIP:**
*Tests the full client-facing stack.*
```bash
dig example.com @172.29.144.4
```

### Service Management Commands
If a specific layer fails, use these commands to inspect the service status:
*   **Pi-hole Engine:** `systemctl status pihole-FTL`
*   **Pi-hole Logs:** `journalctl -u pihole-FTL -f`
*   **Unbound Status:** `systemctl status unbound`

---

## 🔒 Security & Maintenance

*   **Version Control:** The components of Pi-hole (Core, Web, FTL) update independently. Check your installed versions periodically using `pihole -v`.
*   **Access Control:** Never expose the Pi-hole administration web interface or SSH access directly to the public Internet. Restrict DNS access to trusted local networks where appropriate.
*   **Credential Management:** Protect administrative passwords and never commit API tokens to GitHub.