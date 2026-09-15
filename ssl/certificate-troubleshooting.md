# Troubleshooting SSL/TLS Certificates

SSL/TLS issues can be caused by expired certificates, hostname mismatches, incomplete certificate chains, incorrect Nginx certificate paths, DNS problems, or invalid certificate configuration.

This guide documents a practical workflow for troubleshooting HTTPS certificate issues on Linux-based cloud environments.

---

## Troubleshooting Flow

```text
HTTPS Failure
     ↓
Check DNS resolution
     ↓
Check certificate dates
     ↓
Check hostname / SAN
     ↓
Check issuer and certificate chain
     ↓
Check Nginx certificate paths
     ↓
Validate Nginx configuration
     ↓
Test HTTPS connection
     ↓
Fix root cause
```

---

## 1. Check the Certificate Presented by the Server

Use OpenSSL:

```bash
openssl s_client -connect example.com:443 -servername example.com
```

The `-servername` option sends the hostname using SNI.

This is important when multiple HTTPS sites are hosted behind the same server or load balancer.

To reduce the output:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  </dev/null 2>/dev/null
```

---

## 2. Check Certificate Expiry

Check the certificate dates directly from a remote server:

```bash
echo | openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  2>/dev/null |
openssl x509 -noout -dates
```

Example:

```text
notBefore=Sep 01 00:00:00 2026 GMT
notAfter=Dec 01 23:59:59 2026 GMT
```

Important fields:

- `notBefore` — when the certificate becomes valid
- `notAfter` — when the certificate expires

An expired certificate will cause browser and client validation failures.

---

## 3. Inspect a Local Certificate

If the certificate exists on the server:

```bash
openssl x509 \
  -in /path/to/certificate.crt \
  -noout \
  -subject \
  -issuer \
  -dates
```

This displays:

```text
subject
issuer
notBefore
notAfter
```

---

## 4. Check the Certificate Hostname

Check the Subject Alternative Names:

```bash
openssl x509 \
  -in /path/to/certificate.crt \
  -noout \
  -ext subjectAltName
```

Example:

```text
DNS:example.com
DNS:www.example.com
```

The hostname requested by the user must be covered by the certificate.

For example, a certificate valid only for:

```text
example.com
```

may not automatically cover:

```text
api.example.com
```

unless that hostname is included in the certificate SANs or covered by a wildcard certificate.

---

## 5. Check the Certificate Issuer

```bash
openssl x509 \
  -in /path/to/certificate.crt \
  -noout \
  -issuer
```

Example:

```text
issuer=C=US, O=Let's Encrypt, CN=R13
```

The issuer identifies the Certificate Authority that signed the certificate.

---

## 6. Check the Certificate Chain

A server normally needs to provide:

```text
Server Certificate
       ↓
Intermediate Certificate
       ↓
Trusted Root Certificate
```

An incomplete certificate chain can result in HTTPS validation problems even when the server certificate itself is valid.

Check the certificates presented by the server:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts
```

Look for all certificates returned in the chain.

---

## 7. Test HTTPS with curl

```bash
curl -Iv https://example.com
```

Useful information includes:

```text
TLS handshake
certificate subject
certificate issuer
HTTP response
redirects
connection failures
```

Do not use `-k` as the first troubleshooting step because it disables certificate verification and can hide the actual TLS problem.

---

## 8. Check DNS Resolution

Confirm that the hostname resolves to the expected server:

```bash
dig example.com
```

or:

```bash
nslookup example.com
```

A valid certificate can still appear incorrect if DNS points users to the wrong server.

Troubleshooting should therefore verify both:

```text
Domain
   ↓
DNS
   ↓
Correct Server
   ↓
Correct Certificate
```

---

## 9. Check Nginx SSL Configuration

Search the active configuration:

```bash
sudo nginx -T | grep -n "ssl_certificate"
```

Typical configuration:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/ssl/example/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/example/private.key;
}
```

Verify that:

- `server_name` matches the domain
- certificate path exists
- private key path exists
- correct certificate bundle is configured

---

## 10. Validate Nginx Configuration

Before reloading Nginx:

```bash
sudo nginx -t
```

A successful validation should report that the configuration syntax is valid.

Only then reload:

```bash
sudo systemctl reload nginx
```

Reloading is generally preferable to restarting when only configuration or certificate files have changed because existing connections can remain active.

---

## 11. Verify Certificate and Private Key Match

Compare the public keys derived from both files.

Certificate:

```bash
openssl x509 \
  -in certificate.crt \
  -pubkey \
  -noout |
openssl sha256
```

Private key:

```bash
openssl pkey \
  -in private.key \
  -pubout |
openssl sha256
```

The hashes should match.

If they do not match, the certificate and private key do not belong together.

---

## 12. Check File Permissions

Inspect certificate and key permissions:

```bash
ls -l /path/to/certificate.crt
ls -l /path/to/private.key
```

Private keys should not be unnecessarily readable by other users.

Never commit certificate private keys to Git repositories.

---

## Common SSL/TLS Problems

| Problem | What to Check |
|---|---|
| Certificate expired | `openssl x509 -noout -dates` |
| Certificate not valid yet | `notBefore` value |
| Hostname mismatch | Subject Alternative Names |
| Wrong certificate served | Nginx virtual host / SNI |
| Incomplete chain | `openssl s_client -showcerts` |
| Certificate and key mismatch | Compare public-key hashes |
| Wrong DNS destination | `dig` / `nslookup` |
| Nginx config error | `sudo nginx -t` |
| HTTPS connection fails | `curl -Iv` and OpenSSL |
| Wrong certificate path | Nginx SSL configuration |

---

## Recommended Troubleshooting Order

```text
Domain
  ↓
DNS Resolution
  ↓
TCP 443
  ↓
TLS Handshake
  ↓
Certificate Dates
  ↓
Hostname / SAN
  ↓
Certificate Chain
  ↓
Nginx Configuration
  ↓
Application Response
```

---

## Key Principle

Troubleshoot SSL/TLS layer by layer.

Do not immediately replace certificates when HTTPS fails.

First determine whether the problem is related to:

```text
DNS
Certificate
Hostname
Certificate Chain
Private Key
Web Server Configuration
Network
```

This helps identify the actual root cause without making unnecessary changes.
