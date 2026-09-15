# Troubleshooting Application Ports and Processes

Application availability issues are often caused by a service not running, listening on the wrong port, binding to the wrong interface, or being blocked by another process.

This guide documents a structured approach for troubleshooting application ports and processes on Linux-based cloud servers.

---

## Request Flow

```text
Client
  ↓
Network / Load Balancer
  ↓
Server Port
  ↓
Process / Container
  ↓
Application
```

---

## 1. Check Listening Ports

List TCP ports currently listening:

```bash
sudo ss -lntp
```

Check a specific port:

```bash
sudo ss -lntp | grep :3000
```

Example output:

```text
LISTEN 0 511 0.0.0.0:3000 0.0.0.0:* users:(("node",pid=1234,fd=20))
```

This confirms:

- the application is listening;
- the port is `3000`;
- the process is `node`;
- the PID is `1234`.

---

## 2. Identify the Process Using a Port

Using `lsof`:

```bash
sudo lsof -i :3000
```

Using `ss`:

```bash
sudo ss -lntp | grep :3000
```

If another process is already using the required port, the application may fail to start with an error such as:

```text
Address already in use
EADDRINUSE
```

---

## 3. Check Running Processes

Search for an application process:

```bash
ps aux | grep <application-name>
```

Example:

```bash
ps aux | grep node
```

This helps verify whether the expected application process exists.

---

## 4. Test the Application Locally

Test the service directly from the server:

```bash
curl http://127.0.0.1:3000
```

For a health endpoint:

```bash
curl http://127.0.0.1:3000/health
```

If localhost works but the application is inaccessible externally, investigate:

- reverse proxy configuration;
- security groups;
- firewall rules;
- load balancer configuration;
- DNS;
- application binding.

---

## 5. Understand Application Binding

An application may listen on:

```text
127.0.0.1:3000
```

or:

```text
0.0.0.0:3000
```

### `127.0.0.1`

The application accepts connections only from the local machine.

### `0.0.0.0`

The application accepts connections through all available network interfaces, subject to firewall and security rules.

Incorrect interface binding can cause connectivity problems even when the application is running.

---

## 6. Check systemd Services

Check service status:

```bash
sudo systemctl status <service-name>
```

Restart if required:

```bash
sudo systemctl restart <service-name>
```

Check recent service logs:

```bash
sudo journalctl -u <service-name> -n 50
```

Follow logs in real time:

```bash
sudo journalctl -u <service-name> -f
```

---

## 7. Check PM2 Applications

List processes:

```bash
pm2 status
```

Inspect an application:

```bash
pm2 describe <application-name>
```

Check logs:

```bash
pm2 logs <application-name> --lines 50
```

Important details to verify:

- process status;
- restart count;
- application port;
- environment;
- error logs.

---

## 8. Check Docker Containers

List running containers:

```bash
docker ps
```

List all containers:

```bash
docker ps -a
```

Check container logs:

```bash
docker logs <container-name> --tail 50
```

Check exposed port mappings:

```bash
docker port <container-name>
```

Example:

```text
3000/tcp -> 0.0.0.0:3000
```

A container may be running correctly while the expected host port is not mapped correctly.

---

## 9. Test Port Connectivity

Using `curl` for HTTP services:

```bash
curl -I http://127.0.0.1:3000
```

Using `nc` for basic TCP connectivity:

```bash
nc -zv 127.0.0.1 3000
```

Possible outcomes include:

```text
Connection succeeded
Connection refused
Connection timed out
```

Each result points toward a different troubleshooting path.

---

## Common Problems

| Problem | Check |
|---|---|
| Application not running | `ps`, `systemctl`, `pm2 status`, `docker ps` |
| Wrong application port | `ss -lntp` |
| Port already occupied | `lsof -i :PORT` |
| Wrong interface binding | `127.0.0.1` vs `0.0.0.0` |
| Application crash | Service/application logs |
| Container port mismatch | `docker port` |
| Local service works but external access fails | Proxy, firewall, security groups, load balancer |
| Connection refused | Nothing listening or process unavailable |
| Connection timeout | Network or firewall path issue |

---

## Troubleshooting Flow

```text
Application Unavailable
        ↓
Check process
        ↓
Check listening port
        ↓
Identify process using port
        ↓
Test localhost
        ↓
Check binding address
        ↓
Check logs
        ↓
Check PM2 / systemd / Docker
        ↓
Check network path
        ↓
Fix root cause
```

---

## Key Principle

Separate application problems from networking problems.

First prove that the application works locally:

```text
Process
  ↓
Port
  ↓
localhost
```

Then troubleshoot the external path:

```text
Reverse Proxy
  ↓
Firewall / Security Group
  ↓
Load Balancer
  ↓
DNS
```

This approach makes troubleshooting faster and avoids changing infrastructure before confirming that the application itself is healthy.
