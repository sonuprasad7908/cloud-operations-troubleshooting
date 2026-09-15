# DNS Troubleshooting Workflow

DNS issues can make an application appear unavailable even when the server, load balancer, and application are healthy.

This guide documents a practical troubleshooting workflow for common DNS problems in cloud environments.

---

## Troubleshooting Flow

```text
Domain Not Resolving
        ↓
Check local DNS resolution
        ↓
Check A / CNAME record
        ↓
Check authoritative nameservers
        ↓
Check record target
        ↓
Check TTL
        ↓
Check propagation
        ↓
Check public vs private DNS
        ↓
Verify application endpoint
```

---

## 1. Check Basic DNS Resolution

Use:

```bash
nslookup example.com
```

or:

```bash
dig example.com
```

A successful DNS lookup should return an IP address or another expected DNS target.

Example:

```text
example.com.  300  IN  A  203.0.113.10
```

If the domain does not resolve, continue checking the DNS configuration.

---

## 2. Check A Records

An `A` record maps a hostname to an IPv4 address.

Example:

```text
example.com -> 203.0.113.10
```

Query specifically:

```bash
dig A example.com
```

Useful short output:

```bash
dig +short example.com
```

Verify that the returned IP matches the intended server or public endpoint.

---

## 3. Check CNAME Records

A `CNAME` maps one hostname to another hostname.

Example:

```text
app.example.com -> loadbalancer.example.net
```

Check:

```bash
dig CNAME app.example.com
```

A common issue is pointing the CNAME to an incorrect or outdated target.

---

## 4. Check Authoritative Nameservers

Check the nameservers for the domain:

```bash
dig NS example.com
```

Example:

```text
ns-100.example-dns.com
ns-200.example-dns.net
```

The domain registrar must point to the same authoritative nameservers configured in the DNS hosting service.

If the nameservers do not match, records created in the DNS zone may never be used.

---

## 5. Trace DNS Resolution

Use:

```bash
dig +trace example.com
```

This shows the DNS resolution path:

```text
Root DNS
   ↓
TLD Nameserver
   ↓
Authoritative Nameserver
   ↓
DNS Record
```

This is useful when trying to identify where resolution is failing.

---

## 6. Check for NXDOMAIN

Example:

```text
status: NXDOMAIN
```

`NXDOMAIN` usually means the requested hostname does not exist in DNS.

Possible causes:

- record was never created;
- incorrect hostname;
- wrong hosted zone;
- domain is using different nameservers;
- record was deleted.

Verify the exact hostname carefully.

---

## 7. Check SERVFAIL

Example:

```text
status: SERVFAIL
```

Possible causes include:

- DNS server configuration problem;
- DNSSEC issue;
- authoritative nameserver failure;
- malformed DNS zone;
- upstream resolver failure.

Try querying another resolver to compare results.

---

## 8. Check TTL

Check the DNS record:

```bash
dig example.com
```

Example:

```text
example.com. 300 IN A 203.0.113.10
```

Here:

```text
300
```

is the TTL in seconds.

TTL determines how long DNS resolvers may cache the record.

A recently changed record may continue returning an older value until cached entries expire.

---

## 9. Query a Specific DNS Resolver

Google DNS:

```bash
dig @8.8.8.8 example.com
```

Cloudflare DNS:

```bash
dig @1.1.1.1 example.com
```

Comparing multiple resolvers can help identify caching or propagation differences.

---

## 10. Check Local DNS Cache

A local system may still have an old DNS result cached.

On Linux using systemd-resolved:

```bash
sudo resolvectl flush-caches
```

Check resolver statistics:

```bash
resolvectl statistics
```

On Windows:

```text
ipconfig /flushdns
```

Do not assume the DNS provider is wrong before checking local caching.

---

## 11. Check `/etc/resolv.conf`

On Linux:

```bash
cat /etc/resolv.conf
```

This shows which DNS resolver the system is using.

Example:

```text
nameserver 8.8.8.8
```

If the resolver is unavailable or incorrectly configured, DNS lookups may fail locally.

---

## 12. Public vs Private DNS

Cloud environments may contain:

```text
Public DNS
Private DNS
```

A private DNS record may resolve only from within a specific VPC or private network.

Example:

```text
database.internal
```

may work from an application server but fail from a public laptop.

Always determine whether the DNS record is intended to be:

```text
Publicly resolvable
or
Private-network only
```

---

## 13. DNS and Load Balancers

Applications commonly use DNS records pointing toward load balancers.

Typical flow:

```text
app.example.com
       ↓
DNS Record
       ↓
Load Balancer
       ↓
Application Server
```

If DNS resolves correctly but the application is unavailable, continue troubleshooting:

```text
Load balancer
Target health
Security rules
Application port
Application service
```

DNS resolution alone does not prove that the application is healthy.

---

## 14. DNS and HTTPS

DNS and SSL/TLS are closely related.

Example:

```text
api.example.com
```

must resolve to the infrastructure serving a certificate valid for:

```text
api.example.com
```

If DNS points to the wrong server, users may receive:

```text
certificate hostname mismatch
wrong certificate
TLS validation failure
```

---

## Route 53 Troubleshooting Concepts

For AWS Route 53 environments, verify:

- correct hosted zone;
- correct record name;
- correct record type;
- correct target;
- public vs private hosted zone;
- authoritative nameservers;
- alias target;
- health-check behavior when used.

Common records include:

```text
A
AAAA
CNAME
TXT
MX
NS
```

---

## Common DNS Problems

| Problem | What to Check |
|---|---|
| Domain not resolving | `dig` / `nslookup` |
| NXDOMAIN | Record existence and hostname |
| Wrong IP returned | A record target |
| Wrong hostname target | CNAME |
| DNS changes not visible | TTL and cache |
| Records exist but ignored | Authoritative nameservers |
| Works internally only | Private DNS / private hosted zone |
| Different results from different users | Resolver cache / propagation |
| HTTPS certificate mismatch | DNS target and certificate hostname |
| Application unavailable after DNS resolves | Load balancer / server / application |

---

## Recommended Troubleshooting Order

```text
Hostname
   ↓
Local Resolver
   ↓
Authoritative Nameservers
   ↓
DNS Record
   ↓
Record Target
   ↓
TTL / Cache
   ↓
Public or Private DNS
   ↓
Load Balancer / Server
   ↓
Application
```

---

## Key Principle

DNS troubleshooting should separate:

```text
Resolution Problem
```

from:

```text
Application Availability Problem
```

First confirm:

```text
Domain
  ↓
DNS
  ↓
Correct Target
```

Then move to:

```text
Network
  ↓
Load Balancer
  ↓
Server
  ↓
Application
```

This prevents application troubleshooting when the actual problem is simply an incorrect DNS record.
