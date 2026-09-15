# Deployment Health-Check Workflow

A successful deployment should not be considered complete only because the deployment command finished without errors.

The application must be validated at multiple layers to confirm that the new version is actually running and accessible.

This guide documents a practical post-deployment health-check workflow for Linux-based cloud application environments.

---

## Validation Flow

```text
Deployment Completed
        ↓
Check process / container
        ↓
Check application port
        ↓
Test localhost
        ↓
Check health endpoint
        ↓
Validate Nginx / proxy
        ↓
Check external URL
        ↓
Review logs
        ↓
Check load balancer health
        ↓
Confirm deployment
```

---

## 1. Check Application Process

For systemd:

```bash
sudo systemctl status <service-name>
```

For PM2:

```bash
pm2 status
```

For Docker:

```bash
docker ps
```

Confirm that the expected application is:

```text
running
online
healthy
```

depending on the runtime.

---

## 2. Check the Application Port

Check listening ports:

```bash
sudo ss -lntp
```

For a specific port:

```bash
sudo ss -lntp | grep :3000
```

The application must be listening on the expected port after deployment.

---

## 3. Test the Application Locally

Test from the server:

```bash
curl http://127.0.0.1:3000
```

For a dedicated health endpoint:

```bash
curl http://127.0.0.1:3000/health
```

Example healthy response:

```json
{
  "status": "healthy"
}
```

If localhost fails, investigate the application before checking external infrastructure.

---

## 4. Validate HTTP Status

```bash
curl -I http://127.0.0.1:3000
```

Typical healthy status:

```text
HTTP/1.1 200 OK
```

Possible failure responses:

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
404 Not Found
```

The response code helps determine the next troubleshooting step.

---

## 5. Check Application Logs

For PM2:

```bash
pm2 logs <application-name> --lines 50
```

For systemd:

```bash
sudo journalctl -u <service-name> -n 50
```

For Docker:

```bash
docker logs <container-name> --tail 50
```

Check for:

```text
startup failures
database errors
missing environment variables
port conflicts
permission errors
dependency failures
```

---

## 6. Validate Nginx

Check Nginx status:

```bash
sudo systemctl status nginx
```

Validate configuration:

```bash
sudo nginx -t
```

Check the proxied application through Nginx:

```bash
curl -I http://localhost
```

For HTTPS:

```bash
curl -I https://example.com
```

---

## 7. Verify External Access

Test the public endpoint:

```bash
curl -I https://example.com
```

Verify:

```text
HTTP status
redirect behavior
HTTPS
response headers
application accessibility
```

A locally healthy application may still fail externally because of:

```text
Nginx
DNS
SSL/TLS
Load balancer
Firewall
Security Group
Routing
```

---

## 8. Check Load Balancer Health

In load-balanced environments, verify target health.

Typical states include:

```text
healthy
unhealthy
initial
draining
```

If a target is unhealthy, verify:

```text
health-check path
health-check port
application listener
security rules
response status
timeout settings
```

---

## 9. Validate the Correct Version

A deployment may succeed while an older application version is still running.

Useful approaches include exposing:

```text
application version
Git commit SHA
build number
deployment timestamp
```

Example endpoint:

```text
/version
```

Example response:

```json
{
  "version": "a1b2c3d"
}
```

This helps verify that the intended build is actually deployed.

---

## 10. Check Recent Error Logs

After deployment, monitor logs for a short period.

Example:

```bash
pm2 logs <application-name> --lines 100
```

or:

```bash
docker logs -f <container-name>
```

A process may appear healthy immediately but fail once traffic reaches it.

---

## 11. Verify Dependencies

The application may depend on:

```text
Database
Redis
External API
Object Storage
Message Queue
Internal Services
```

A good health check should confirm that critical dependencies are available.

---

## 12. Decide Whether to Roll Back

Rollback may be appropriate when:

```text
application fails to start
health check remains unhealthy
critical endpoint returns 5xx
database connection fails
new errors appear after deployment
major functionality is unavailable
```

A deployment should not remain active simply because the CI/CD pipeline reported success.

---

## Post-Deployment Checklist

| Check | Expected Result |
|---|---|
| Process / container | Running |
| Application port | Listening |
| Local request | Successful |
| Health endpoint | Healthy |
| Application logs | No critical errors |
| Nginx config | Valid |
| External URL | Accessible |
| HTTPS | Valid |
| Load balancer target | Healthy |
| Correct version | Confirmed |
| Dependencies | Reachable |

---

## Recommended Validation Order

```text
Process
   ↓
Port
   ↓
Local Application
   ↓
Health Endpoint
   ↓
Reverse Proxy
   ↓
Load Balancer
   ↓
HTTPS / DNS
   ↓
External User Request
```

---

## Key Principle

Deployment success and application health are not the same thing.

A reliable deployment process should confirm:

```text
Application Started
        +
Application Reachable
        +
Application Healthy
        +
Correct Version Running
        +
No Critical Errors
```

Only after these checks pass should the deployment be considered successful.
