# Troubleshooting Nginx 502 Bad Gateway

A `502 Bad Gateway` usually means Nginx is reachable, but it cannot successfully communicate with the upstream application.

This guide documents a practical troubleshooting workflow for Linux-based cloud environments.

---

## Request Flow

```text
Client
  ↓
Nginx
  ↓
Application Port
  ↓
Application / Container
```

---

## 1. Check Nginx

Verify that Nginx is running:

```bash
sudo systemctl status nginx
```

Validate the configuration:

```bash
sudo nginx -t
```

---

## 2. Check the Upstream Port

Inspect the Nginx configuration:

```bash
sudo nginx -T
```

Example:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
}
```

The `proxy_pass` port must match the port where the application is listening.

---

## 3. Check the Application

Check listening ports:

```bash
sudo ss -lntp
```

Check a specific port:

```bash
sudo ss -lntp | grep :3000
```

Test the application directly:

```bash
curl http://127.0.0.1:3000
```

If this fails, the problem is probably with the application rather than Nginx.

---

## 4. Check the Application Process

For PM2:

```bash
pm2 status
pm2 logs <application-name> --lines 50
```

For systemd:

```bash
sudo systemctl status <service-name>
```

---

## 5. Check Logs

Nginx error log:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

Application logs should also be checked for crashes, port conflicts, or connection errors.

---

## 6. Docker Applications

Check containers:

```bash
docker ps
docker ps -a
```

Check logs:

```bash
docker logs <container-name> --tail 50
```

Check port mapping:

```bash
docker port <container-name>
```

---

## Common Causes

| Problem | Check |
|---|---|
| Application stopped | `pm2 status`, `systemctl`, or `docker ps` |
| Wrong upstream port | Compare `proxy_pass` with listening ports |
| Application crashed | Check application logs |
| Docker port mismatch | `docker port` |
| Nginx configuration issue | `sudo nginx -t` |
| Upstream timeout | Nginx and application logs |
| Network restriction | Firewall or cloud security rules |

---

## Troubleshooting Order

```text
502 Bad Gateway
      ↓
Check Nginx
      ↓
Check proxy_pass
      ↓
Check application process
      ↓
Check listening port
      ↓
Test localhost
      ↓
Check logs
      ↓
Check Docker / networking
      ↓
Fix root cause
```

## Key Principle

Troubleshoot the request path layer by layer instead of restarting services blindly.
