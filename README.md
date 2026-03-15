# Docker Data Server Infrastructure

A Docker Compose infrastructure stack for managing containerized services on a
data/web server. It provides a reverse proxy with SSL termination, a web server,
container management via Portainer, and comprehensive intrusion prevention
through Fail2ban with custom jails and filters tailored for Docker environments.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Services](#services)
  - [Fail2ban](#fail2ban)
  - [Nginx Proxy Manager](#nginx-proxy-manager)
  - [Nginx Main](#nginx-main)
  - [Portainer Agent](#portainer-agent)
  - [Portainer Server](#portainer-server)
- [Network Architecture](#network-architecture)
- [Security Architecture](#security-architecture)
  - [Jails](#jails)
  - [Filters](#filters)
  - [Actions](#actions)
  - [Whitelisted IPs](#whitelisted-ips)
- [Environment Variables](#environment-variables)
- [Persistent Storage](#persistent-storage)
- [Common Commands](#common-commands)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
                          Internet
                             |
                     +-------+-------+
                     |  Host Network  |
                     |   Fail2ban     |
                     |  (iptables)    |
                     +-------+-------+
                             |
               +-------------+-------------+
               |             |             |
          Port 80/443     Port 81     Port 22 (SSH)
               |             |
     +---------+---------+   |
     | Nginx Proxy       +---+
     | Manager           |
     | (reverse proxy)   |
     +--------+----------+
              |
     reverse-proxy-nw
              |
     +--------+----------+        portainer-default
     |                    |              |
  +--+---+          +-----+------+  +---+--------+
  | Nginx|          | Portainer  |  | Portainer  |
  | Main |          | Server     +--+ Agent      |
  +------+          +-----+------+  +------------+
                          |
                   Port 9000/9443/8000
```

**Traffic flow:** All inbound traffic passes through the host firewall where
Fail2ban inspects and filters requests. Legitimate traffic reaches Nginx Proxy
Manager, which terminates SSL and forwards requests to backend services over the
`reverse-proxy-nw` Docker network. Portainer components communicate over their
own internal `portainer-default` network.

---

## Prerequisites

| Requirement        | Minimum Version | Notes                           |
| ------------------ | --------------- | ------------------------------- |
| Docker Engine      | 20.10+          | With Docker CLI                 |
| Docker Compose     | 1.29+ or v2     | Compose file version 3.9        |
| Linux host         | --              | iptables support required        |
| Open ports         | 80, 443, 81    | Plus 9000, 9443, 8000, 9001     |

### Directory structure

Create the following directories on the host before starting the stack:

```bash
sudo mkdir -p /srv/fail2ban/data
sudo mkdir -p /srv/nginx-proxy-manager/data
sudo mkdir -p /srv/portainer/data
sudo mkdir -p /srv/logs/fail2ban
sudo mkdir -p /srv/logs/nginx
sudo mkdir -p /srv/logs/nginx-proxy-manager
```

### External Docker network

The `reverse-proxy-nw` network must exist before running the stack:

```bash
docker network create reverse-proxy-nw
```

---

## Quick Start

1. **Clone the repository:**

   ```bash
   git clone https://github.com/lgdweb/docker-data-server-infra.git
   cd docker-data-server-infra
   ```

2. **Create the required directories:**

   ```bash
   sudo mkdir -p /srv/{fail2ban/data,nginx-proxy-manager/data,portainer/data}
   sudo mkdir -p /srv/logs/{fail2ban,nginx,nginx-proxy-manager}
   ```

3. **Create the external network:**

   ```bash
   docker network create reverse-proxy-nw
   ```

4. **Configure environment variables:**

   ```bash
   cat > .env << 'EOF'
   F2B_DEST_EMAIL=your-email@example.com
   F2B_USER=your-smtp-user@example.com
   F2B_PASSWORD=your-smtp-password
   F2B_HOSTNAME=your-hostname.example.com
   EOF
   chmod 600 .env
   ```

5. **Copy Fail2ban configuration into the persistent volume:**

   ```bash
   sudo cp -r fail2ban/data/* /srv/fail2ban/data/
   ```

6. **Start the stack:**

   ```bash
   docker compose up -d
   ```

7. **Verify all services are running:**

   ```bash
   docker compose ps
   ```

8. **Access the services:**

   - Nginx Proxy Manager admin: `http://<server-ip>:81`
     (default credentials: `admin@example.com` / `changeme`)
   - Portainer: `https://<server-ip>:9443`

---

## Services

### Fail2ban

| Property       | Value                           |
| -------------- | ------------------------------- |
| Image          | `crazymax/fail2ban:0.11.2`      |
| Container name | `fail2ban`                      |
| Network mode   | `host`                          |
| Capabilities   | `NET_ADMIN`, `NET_RAW`          |
| Privileged     | Yes                             |
| Restart policy | `unless-stopped`                |
| Timezone       | `Europe/Paris`                  |

Monitors SSH and web server logs for malicious activity. Runs on the host
network with elevated privileges to manipulate iptables rules directly. Sends
email alerts through SSMTP via `mail.infomaniak.com:587` with STARTTLS.

**Log targets:** Fail2ban logs to `STDOUT` at `INFO` level, making logs
accessible via `docker logs fail2ban`.

### Nginx Proxy Manager

| Property       | Value                              |
| -------------- | ---------------------------------- |
| Image          | `jc21/nginx-proxy-manager:2.9.18`  |
| Container name | `nginx-proxy-manager`              |
| Ports          | `80:80`, `443:443`, `81:81`        |
| Network        | `reverse-proxy-nw`                 |
| Timezone       | `Europe/Paris`                     |

Acts as the main reverse proxy and SSL termination point. Port 81 exposes the
web-based admin interface for managing proxy hosts, SSL certificates (Let's
Encrypt), redirections, and access lists. Logs are stored under
`/srv/logs/nginx-proxy-manager/` and monitored by the Fail2ban `proxy` jail.

### Nginx Main

| Property       | Value                        |
| -------------- | ---------------------------- |
| Image          | `lgdweb/nginx-test:latest`   |
| Container name | `nginx-main`                 |
| Ports          | None (behind reverse proxy)  |
| Network        | `reverse-proxy-nw`           |

The primary web server, accessible only through Nginx Proxy Manager. It does not
expose any ports directly to the host. Access and error logs are stored under
`/srv/logs/nginx/` and monitored by the `nginx-http-auth` and `nginx-badbots`
jails.

### Portainer Agent

| Property       | Value                               |
| -------------- | ----------------------------------- |
| Image          | `portainer/agent:2.14.2-alpine`     |
| Container name | `portainer-agent`                   |
| Ports          | `9001:9001`                         |
| Network        | `portainer-default`                 |
| Restart policy | `unless-stopped`                    |

Lightweight agent that communicates with Portainer Server to provide container
management capabilities. Mounts the Docker socket and volumes directory for full
visibility into the Docker environment.

### Portainer Server

| Property       | Value                                    |
| -------------- | ---------------------------------------- |
| Image          | `portainer/portainer-ce:2.14.2-alpine`   |
| Container name | `portainer-server`                       |
| Ports          | `9000:9000`, `9443:9443`, `8000:8000`    |
| Networks       | `portainer-default`, `reverse-proxy-nw`  |
| Restart policy | `unless-stopped`                         |
| Security       | `no-new-privileges:true`                 |

Web-based Docker management interface. Port 9443 serves the HTTPS UI, port 9000
the HTTP UI, and port 8000 is used for edge agent communication. Connected to
both networks so it can be proxied through Nginx Proxy Manager while
communicating with its agent on the internal network.

---

## Network Architecture

| Network            | Type       | Purpose                                       |
| ------------------ | ---------- | --------------------------------------------- |
| `reverse-proxy-nw` | External   | Connects reverse proxy to backend services     |
| `portainer-default`| Internal   | Isolated communication between Portainer components (attachable) |
| `host`             | Host       | Fail2ban operates on the host network stack    |

The `reverse-proxy-nw` network must be created manually before starting the
stack because it may be shared with other Docker Compose projects. The
`portainer-default` network is created automatically and marked as `attachable`
so additional containers can join it if needed.

---

## Security Architecture

### Jails

| Jail               | Log Source                          | Max Retries | Find Time | Ban Time | Ban Action      |
| ------------------ | ----------------------------------- | ----------- | --------- | -------- | --------------- |
| `sshd`             | `/var/log/auth.log` (system)        | 5           | 1 hour    | 15 min   | `action_mwl`    |
| `proxy`            | `/srv/logs/nginx-proxy-manager/*.log` | 4         | 1 day*    | 15 min*  | `docker-action` |
| `nginx-http-auth`  | `/srv/logs/nginx/error.log`         | 2           | 1 day*    | 15 min*  | `docker-action` |
| `nginx-badbots`    | `/srv/logs/nginx/access.log`        | 2           | 1 day*    | 15 min*  | `docker-action` |
| `recidive`         | `/srv/logs/fail2ban/fail2ban.log`   | 4           | 4 days    | 24 hours | `banaction_allports` |

*Values marked with \* inherit from the `[DEFAULT]` section in `jail.d/default.conf`.

**Default settings:** `maxretry=10`, `findtime=1d`, `bantime=15m` with
incremental ban times enabled (`bantime.increment=true`) across all jails
(`bantime.overalljails=true`). The database purge age is set to 8 days.

**Recidive jail:** A meta-jail that monitors Fail2ban's own log. If an IP gets
banned 4 times across any jail within 4 days, recidive issues a 24-hour ban on
all ports.

### Filters

**proxy** (`filter.d/proxy.conf`)

Detects malicious web requests in Nginx Proxy Manager logs:

- `w00tw00t` scanner signatures
- Attempts to access admin panels (`/admin`, `/myadmin`, `/pma`, `/wp-*`, etc.)
- Requests for sensitive files (`/.ssh/`, `/.git/`, `/.env`, `/web.config`, `/backup`)
- HTTP 4xx/5xx error responses (excluding 404)

**nginx-http-auth** (`filter.d/nginx-http-auth.conf`)

Catches failed HTTP Basic Authentication attempts:

- Password mismatches
- Missing credentials for protected resources

**badbots** (`filter.d/badbots.conf`)

Identifies known malicious user agents (200+ signatures), including:

- Email harvesters (`Atomic_Email_Hunter`, `EmailSiphon`, `EmailWolf`, etc.)
- Malicious crawlers (`psycheclone`, `WebVulnCrawl`, etc.)
- Spam bots (`Guestbook Auto Submitter`, `TrackBack`, etc.)
- Fake browser signatures and known scanner tools

### Actions

**docker-action** (`action.d/docker-action.conf`)

Custom iptables action designed for Docker environments:

```
Start  -> Creates f2b-<name> chain, inserts into FORWARD chain
Ban    -> Adds DROP rule for offending IP in f2b-<name> chain
Unban  -> Removes DROP rule for the IP
Stop   -> Cleans up chain and rules
```

This action works with the Docker `FORWARD` chain, which is where Docker routes
container traffic. Standard iptables `INPUT` chain rules do not affect traffic
destined for Docker containers.

### Whitelisted IPs

The following IPs are excluded from all jails via `ignoreip`:

| IP / Range            | Description              |
| --------------------- | ------------------------ |
| `127.0.0.1/8`        | Localhost                 |
| `89.233.108.164`     | Trusted host              |
| `57.128.171.245`     | Trusted host              |
| `107.155.122.53`     | Trusted host              |
| `162.19.249.202`     | Trusted host              |
| `208.87.129.64`      | Trusted host              |
| `92.222.216.54`      | Trusted host              |
| `192.168.10.1/24`    | Local network             |
| `lgdweb.duckdns.org` | Dynamic DNS hostname      |

---

## Environment Variables

Create a `.env` file in the project root with the following variables:

| Variable         | Required | Description                                      |
| ---------------- | -------- | ------------------------------------------------ |
| `F2B_DEST_EMAIL` | Yes      | Destination email for Fail2ban ban notifications  |
| `F2B_USER`       | Yes      | SSMTP username for `mail.infomaniak.com`          |
| `F2B_PASSWORD`   | Yes      | SSMTP password for the mail account               |
| `F2B_HOSTNAME`   | Yes      | Hostname used in SSMTP EHLO/HELO                  |

Additional environment variables set in `docker-compose.yaml`:

| Variable           | Service              | Value                   |
| ------------------ | -------------------- | ----------------------- |
| `TZ`               | fail2ban, manager    | `Europe/Paris`          |
| `F2B_LOG_TARGET`   | fail2ban             | `STDOUT`                |
| `F2B_LOG_LEVEL`    | fail2ban             | `INFO`                  |
| `SSMTP_HOST`       | fail2ban             | `mail.infomaniak.com`   |
| `SSMTP_PORT`       | fail2ban             | `587`                   |
| `SSMTP_TLS`        | fail2ban             | `NO`                    |
| `SSMTP_STARTTLS`   | fail2ban             | `YES`                   |

---

## Persistent Storage

All persistent data is stored under `/srv/` on the host:

| Host Path                                  | Container Path           | Service             | Access |
| ------------------------------------------ | ------------------------ | ------------------- | ------ |
| `/srv/fail2ban/data`                       | `/data`                  | fail2ban            | rw     |
| `/srv/logs`                                | `/srv/logs`              | fail2ban            | ro     |
| `/var/log`                                 | `/var/log`               | fail2ban            | ro     |
| `/srv/nginx-proxy-manager/data`            | `/data`                  | nginx-proxy-manager | rw     |
| `/srv/nginx-proxy-manager/data/letsencrypt`| `/etc/letsencrypt`       | nginx-proxy-manager | rw     |
| `/srv/logs/nginx-proxy-manager`            | `/data/logs`             | nginx-proxy-manager | rw     |
| `/srv/logs/nginx`                          | `/var/log/nginx`         | nginx-main          | rw     |
| `/srv/portainer/data`                      | `/data`                  | portainer-server    | rw     |
| `/var/run/docker.sock`                     | `/var/run/docker.sock`   | agent, server       | rw     |
| `/var/lib/docker/volumes`                  | `/var/lib/docker/volumes`| agent, server       | rw     |

---

## Common Commands

### Stack management

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart a single service
docker compose restart fail2ban

# View logs for a service
docker compose logs -f fail2ban

# Check service status
docker compose ps
```

### Fail2ban operations

```bash
# Check overall Fail2ban status
docker exec fail2ban fail2ban-client status

# Check a specific jail
docker exec fail2ban fail2ban-client status sshd
docker exec fail2ban fail2ban-client status proxy
docker exec fail2ban fail2ban-client status nginx-http-auth
docker exec fail2ban fail2ban-client status nginx-badbots
docker exec fail2ban fail2ban-client status recidive

# Manually ban an IP in a jail
docker exec fail2ban fail2ban-client set proxy banip 1.2.3.4

# Unban an IP from a jail
docker exec fail2ban fail2ban-client set proxy unbanip 1.2.3.4

# Unban an IP from all jails
docker exec fail2ban fail2ban-client unban 1.2.3.4

# Unban all IPs
docker exec fail2ban fail2ban-client unban --all

# Reload Fail2ban configuration
docker exec fail2ban fail2ban-client reload

# Test a filter against a log file
docker exec fail2ban fail2ban-regex /srv/logs/nginx-proxy-manager/fallback_access.log /data/filter.d/proxy.conf
```

### Firewall inspection

```bash
# List all iptables rules (look for f2b-* chains)
sudo iptables -L -n -v

# List FORWARD chain rules
sudo iptables -L FORWARD -n -v

# List rules in a specific Fail2ban chain
sudo iptables -L f2b-proxy -n -v
```

### Log inspection

```bash
# View Nginx Proxy Manager access logs
tail -f /srv/logs/nginx-proxy-manager/proxy-host-*.log

# View Nginx error logs
tail -f /srv/logs/nginx/error.log

# View Fail2ban log
docker logs -f fail2ban
```

---

## Troubleshooting

### Fail2ban container will not start

- Verify the `/srv/fail2ban/data` directory contains the jail, filter, and
  action configuration files. Copy them from the repository if missing:
  ```bash
  sudo cp -r fail2ban/data/* /srv/fail2ban/data/
  ```
- Check that the container has `NET_ADMIN` and `NET_RAW` capabilities. These are
  required for iptables manipulation.

### Bans are not working on Docker containers

- Standard iptables `INPUT` rules do not apply to Docker container traffic.
  Verify that jails targeting Docker services use `banaction = docker-action`,
  which operates on the `FORWARD` chain.
- Inspect the iptables rules to confirm the `f2b-*` chains are present:
  ```bash
  sudo iptables -L FORWARD -n | grep f2b
  ```

### Email notifications are not sent

- Verify the `.env` file contains valid `F2B_USER`, `F2B_PASSWORD`, and
  `F2B_HOSTNAME` values.
- Check SSMTP connectivity from inside the container:
  ```bash
  docker exec fail2ban cat /etc/ssmtp/ssmtp.conf
  ```
- Review Fail2ban logs for mail-related errors:
  ```bash
  docker logs fail2ban 2>&1 | grep -i mail
  ```

### Nginx Proxy Manager admin panel is inaccessible

- Confirm port 81 is open and not blocked by a host firewall.
- Check container status: `docker compose ps manager`
- Review logs: `docker compose logs manager`
- Default login credentials are `admin@example.com` / `changeme` on first run.

### "Network reverse-proxy-nw declared as external but could not be found"

- Create the external network before starting the stack:
  ```bash
  docker network create reverse-proxy-nw
  ```

### A legitimate IP was banned

- Unban it immediately:
  ```bash
  docker exec fail2ban fail2ban-client unban <IP>
  ```
- Add the IP to the `ignoreip` list in `fail2ban/data/jail.d/default.conf`, then
  reload:
  ```bash
  docker exec fail2ban fail2ban-client reload
  ```

### Portainer cannot connect to the agent

- Verify both containers are on the `portainer-default` network:
  ```bash
  docker network inspect portainer-default
  ```
- Confirm port 9001 is accessible between the containers.
- Check agent logs: `docker compose logs agent`

---

## License

See the repository for license information.
