# Troubleshooting Docker Container Failures

Docker container issues are commonly caused by application crashes, incorrect environment variables, port conflicts, missing dependencies, resource limits, or container networking problems.

This guide documents a practical troubleshooting workflow for containerized applications.

---

## Troubleshooting Flow

```text
Application Unavailable
        ↓
Check container status
        ↓
Check container logs
        ↓
Inspect exit code
        ↓
Check port mappings
        ↓
Check environment/config
        ↓
Check resource usage
        ↓
Check container networking
        ↓
Fix root cause
```

---

## 1. Check Running Containers

```bash
docker ps
```

Check all containers, including stopped ones:

```bash
docker ps -a
```

Important fields:

- container name;
- image;
- status;
- exposed ports;
- restart state.

If the container immediately exits after starting, continue with logs and exit-code checks.

---

## 2. Check Container Logs

```bash
docker logs <container-name>
```

View the latest log entries:

```bash
docker logs <container-name> --tail 50
```

Follow logs:

```bash
docker logs -f <container-name>
```

Common application problems visible in logs include:

```text
Connection refused
Port already in use
Database connection failed
Missing environment variable
Permission denied
Application startup error
```

---

## 3. Inspect Container State

```bash
docker inspect <container-name>
```

Check the container exit code:

```bash
docker inspect <container-name> \
  --format='{{.State.ExitCode}}'
```

Check the container status:

```bash
docker inspect <container-name> \
  --format='{{.State.Status}}'
```

Useful states include:

```text
running
exited
restarting
dead
```

---

## 4. Understand Exit Codes

Some common exit codes:

| Exit Code | Meaning |
|---|---|
| `0` | Process completed successfully |
| `1` | General application error |
| `125` | Docker failed to start the container |
| `126` | Command cannot be invoked |
| `127` | Command not found |
| `137` | Container killed, often due to SIGKILL or memory pressure |
| `143` | Container stopped with SIGTERM |

The exit code should always be investigated together with application logs.

---

## 5. Check Port Mappings

```bash
docker port <container-name>
```

Example:

```text
3000/tcp -> 0.0.0.0:3000
```

Also check:

```bash
docker ps
```

A container may be running correctly but still be inaccessible if the host port is mapped incorrectly.

Test locally:

```bash
curl http://127.0.0.1:3000
```

---

## 6. Check Environment Variables

Inspect environment variables:

```bash
docker inspect <container-name> \
  --format='{{range .Config.Env}}{{println .}}{{end}}'
```

Do not expose sensitive values when sharing output.

Common configuration problems include:

- incorrect database host;
- missing application port;
- incorrect environment mode;
- missing API configuration;
- incorrect service URLs.

---

## 7. Enter a Running Container

```bash
docker exec -it <container-name> sh
```

If Bash is available:

```bash
docker exec -it <container-name> bash
```

Inside the container, you can verify:

```text
application files
environment
network connectivity
running processes
configuration
```

---

## 8. Check Resource Usage

```bash
docker stats
```

Useful metrics include:

```text
CPU
Memory
Network I/O
Block I/O
Process count
```

A container repeatedly exiting with code `137` may indicate memory pressure.

---

## 9. Check Restart Policy

```bash
docker inspect <container-name> \
  --format='{{.HostConfig.RestartPolicy.Name}}'
```

Common policies:

```text
no
always
unless-stopped
on-failure
```

A bad restart loop can hide the actual application failure, so check logs before repeatedly restarting the container.

---

## 10. Check Container Networking

List Docker networks:

```bash
docker network ls
```

Inspect a network:

```bash
docker network inspect <network-name>
```

Check the container network configuration:

```bash
docker inspect <container-name>
```

For multi-container applications, verify that containers are attached to the expected Docker network and can resolve each other correctly.

---

## 11. Check Disk Usage

```bash
docker system df
```

Large numbers of unused images, stopped containers, or build cache can consume significant disk space.

Inspect before cleaning anything.

Never run destructive cleanup commands blindly on production systems.

---

## Common Problems

| Problem | What to Check |
|---|---|
| Container exits immediately | Logs and exit code |
| Container keeps restarting | Restart policy and application logs |
| Application inaccessible | Port mapping and application listener |
| Environment configuration failure | Container environment variables |
| High memory consumption | `docker stats` |
| Exit code 137 | Memory pressure / SIGKILL |
| Containers cannot communicate | Docker networks |
| Application command not found | Dockerfile / entrypoint |
| Disk usage increasing | `docker system df` |

---

## Key Principle

Troubleshoot the container from the inside out:

```text
Application
    ↓
Process
    ↓
Container
    ↓
Port
    ↓
Docker Network
    ↓
Host
    ↓
External Network
```

Do not assume Docker itself is the problem. First determine whether the failure is caused by the application, configuration, container runtime, or network path.
