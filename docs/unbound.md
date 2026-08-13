# Unbound Recursive DNS Configuration

## 1. Overview

Unbound is used as the recursive DNS resolver for both Pi-hole servers.

Instead of forwarding DNS requests directly to a public DNS resolver, Pi-hole sends allowed DNS queries to the local Unbound instance.

The architecture is:

```text
DNS Client
    │
    ▼
Pi-hole
    │
    ▼
Unbound
    │
    ▼
DNS Root Servers
    │
    ▼
TLD Servers
    │
    ▼
Authoritative DNS Servers
```

This provides a locally controlled recursive DNS resolution layer.

---

## 2. Unbound Nodes

Unbound runs independently on both Pi-hole servers.

| Node       |     Pi-hole IP | Unbound          |
| ---------- | -------------: | ---------------- |
| `pihole01` | `172.29.144.3` | `127.0.0.1:5335` |
| `pihole02` | `172.29.144.2` | `127.0.0.1:5335` |

Unbound listens only on the local loopback interface.

This means external hosts cannot directly access the Unbound listener.

---

## 3. Installation

Unbound was installed on both Debian 13 Pi-hole nodes.

The package can be installed with:

```bash
sudo apt update
sudo apt install unbound
```

After installation, the service can be checked with:

```bash
sudo systemctl status unbound
```

The service should show an active state.

---

## 4. Configuration File

The Pi-hole-specific Unbound configuration is stored in:

```text
/etc/unbound/unbound.conf.d/pi-hole.conf
```

The configuration is separated into its own file rather than modifying the main Unbound configuration directly.

This makes the configuration easier to maintain and troubleshoot.

---

## 5. Listener Configuration

Unbound is configured to listen on the local loopback address:

```text
interface: 127.0.0.1
port: 5335
```

This produces:

```text
Pi-hole
   │
   │ localhost DNS query
   ▼
127.0.0.1:5335
   │
   ▼
Unbound
```

Using port `5335` also avoids conflicting with Pi-hole's DNS service, which normally uses port `53`.

---

## 6. IPv6 Configuration

The current configuration disables IPv6 inside Unbound:

```text
do-ip6: no
prefer-ip6: no
```

This was intentionally configured according to the current homelab requirements.

The system can still have IPv6 capability at the operating-system/network level without Unbound using IPv6 for recursive DNS communication.

---

## 7. Thread Configuration

The current configuration uses:

```text
num-threads: 1
```

The homelab VMs are intentionally lightweight, so a single Unbound worker thread is sufficient for the current workload.

For a larger deployment, the number of threads could be increased according to available CPU resources and DNS workload.

---

## 8. Caching Configuration

Unbound provides DNS caching to reduce repeated recursive lookups.

The configuration includes:

```text
prefetch-key: yes
```

This allows Unbound to refresh DNSSEC-related information before cached data becomes stale.

The configuration also enables:

```text
serve-expired: yes
serve-expired-ttl: 86400
```

This allows Unbound to continue serving expired cached responses for up to 86400 seconds when appropriate.

This can improve DNS availability during temporary upstream resolution problems.

---

## 9. Root Hints

Unbound uses a root hints file to discover the DNS root server infrastructure.

The configuration references the root hints file:

```text
root-hints: <root-hints-file>
```

The actual path used by the system should be verified directly on the server before documenting it as a fixed path.

A missing root hints file can cause configuration validation to fail.

---

## 10. DNSSEC Trust Anchor

DNSSEC validation requires a trust anchor.

Unbound supports an automatically maintained root trust anchor through:

```text
auto-trust-anchor-file
```

The project previously encountered a configuration issue caused by duplicate trust-anchor configuration.

The important lesson is that the trust anchor should be configured **once** and should not be duplicated across configuration files.

---

## 11. Configuration Validation

Before restarting Unbound, the configuration should always be validated.

Run:

```bash
sudo unbound-checkconf
```

A successful validation should return without configuration errors.

Example:

```text
unbound-checkconf
```

If errors are reported, the service should not be restarted until the configuration problem is corrected.

---

## 12. Service Management

Check the service:

```bash
sudo systemctl status unbound
```

Start the service:

```bash
sudo systemctl start unbound
```

Restart after configuration changes:

```bash
sudo systemctl restart unbound
```

Enable Unbound at boot:

```bash
sudo systemctl enable unbound
```

---

## 13. DNS Testing

Unbound can be tested independently from Pi-hole.

Run:

```bash
dig example.com @127.0.0.1 -p 5335
```

A successful response confirms that Unbound is listening and able to perform DNS resolution.

A more detailed response can be obtained with:

```bash
dig example.com @127.0.0.1 -p 5335
```

Important fields to check include:

* `status: NOERROR`
* An answer section
* Query response time
* The DNS server address

---

## 14. Testing DNSSEC

DNSSEC functionality can be tested using a DNSSEC-enabled test domain.

For example:

```bash
dig dnssec-failed.org @127.0.0.1 -p 5335
```

A properly validating resolver should reject deliberately invalid DNSSEC data.

Another useful test is:

```bash
dig cloudflare.com @127.0.0.1 -p 5335
```

The exact response should be inspected rather than relying only on whether the command returned output.

---

## 15. Testing Through Pi-hole

After confirming that Unbound works independently, test the Pi-hole → Unbound path.

First test Unbound:

```bash
dig example.com @127.0.0.1 -p 5335
```

Then test Pi-hole:

```bash
dig example.com @127.0.0.1
```

The expected architecture is:

```text
dig
 │
 ▼
Pi-hole :53
 │
 ▼
Unbound :5335
 │
 ▼
Recursive DNS
```

If Unbound works but Pi-hole does not, the issue is likely in the Pi-hole upstream DNS configuration or Pi-hole DNS service.

---

## 16. Troubleshooting

### 16.1 Unbound fails to start

Check:

```bash
sudo systemctl status unbound
```

Then inspect recent logs:

```bash
sudo journalctl -u unbound --since "10 minutes ago"
```

Validate the configuration:

```bash
sudo unbound-checkconf
```

---

### 16.2 Missing root hints

If `unbound-checkconf` reports that a configured root hints file does not exist, verify the file:

```bash
ls -l <root-hints-file>
```

The root hints file must exist at the path specified by the configuration.

---

### 16.3 Duplicate trust-anchor configuration

If Unbound reports duplicate `auto-trust-anchor-file` configuration, search the configuration files:

```bash
grep -R "auto-trust-anchor-file" /etc/unbound/
```

There should not be conflicting duplicate definitions.

After correcting the configuration:

```bash
sudo unbound-checkconf
```

Then restart:

```bash
sudo systemctl restart unbound
```

---

### 16.4 DNS query fails

Test Unbound directly:

```bash
dig example.com @127.0.0.1 -p 5335
```

Then check whether Unbound is listening:

```bash
sudo ss -lntup | grep 5335
```

The expected listener is on:

```text
127.0.0.1:5335
```

---

### 16.5 Pi-hole works but Internet DNS does not

Check the complete chain:

```text
Pi-hole
   │
   ▼
127.0.0.1:5335
   │
   ▼
Unbound
   │
   ▼
Root DNS
```

Test each layer separately.

---

## 17. Cache Considerations

DNS caching can improve performance, but excessively long minimum TTL settings can cause stale DNS information.

The project therefore avoids unnecessarily aggressive cache settings.

When troubleshooting CDN or DNS propagation issues, inspect the current Unbound cache-related configuration before changing it.

The goal is to balance:

* DNS performance
* Availability
* Freshness
* Recursive query reduction

---

## 18. Security and Privacy

Using Unbound as a recursive resolver provides greater control over DNS resolution.

The Pi-hole servers do not need to send every DNS query directly to a public recursive resolver.

Unbound performs recursive resolution locally.

Additional security benefits include:

* Local DNS resolution
* DNSSEC validation
* Reduced dependency on third-party recursive DNS providers
* Local caching
* Restricted listener interface
* Separation between Pi-hole filtering and recursive resolution

However, recursive DNS does not make DNS traffic invisible to the network provider or authoritative DNS infrastructure. It should therefore be considered a privacy and control improvement, not complete anonymity.

---

## 19. Unbound Architecture

Each Pi-hole node has its own independent Unbound instance:

```text
                 HA DNS VIP
                172.29.144.4
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        pihole01           pihole02
             │                 │
             ▼                 ▼
      127.0.0.1:5335    127.0.0.1:5335
             │                 │
             ▼                 ▼
          Unbound           Unbound
             │                 │
             └────────┬────────┘
                      │
                Recursive DNS
```

This design ensures that a single Unbound instance is not a dependency for the entire DNS infrastructure.

---

## 20. Final Verification

After configuration or maintenance, the following checks should be performed:

```bash
sudo unbound-checkconf
```

```bash
sudo systemctl status unbound
```

```bash
sudo ss -lntup | grep 5335
```

```bash
dig example.com @127.0.0.1 -p 5335
```

Then verify Pi-hole:

```bash
dig example.com @127.0.0.1
```

Finally verify the HA DNS endpoint:

```bash
dig example.com @172.29.144.4
```

A successful result at all three DNS layers indicates that the complete resolution path is operational.

---

## 21. Summary

Unbound provides the recursive DNS layer underneath Pi-hole.

The final DNS stack is:

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
DNS Root
  │
  ▼
TLD
  │
  ▼
Authoritative DNS
```

Both Pi-hole nodes maintain their own independent Unbound resolver.

This design combines DNS filtering, recursive resolution, caching, DNSSEC validation, and high availability into a single homelab DNS infrastructure.
