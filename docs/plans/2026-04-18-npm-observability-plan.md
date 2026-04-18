# NPM Observability Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use godmode:task-runner to implement this plan task-by-task.

**Goal:** Deploy a GoAccess dashboard that monitors all Nginx Proxy Manager request logs, served securely via NPM with SSL and basic auth.

**Architecture:** A single GoAccess container reads NPM's log directory read-only. NPM proxies the dashboard at a subdomain with Let's Encrypt SSL and an access list for authentication. WebSockets are enabled so the real-time panel updates live.

**Tech Stack:** Docker Compose, `xavierh/goaccess-for-nginxproxymanager:latest`, Nginx Proxy Manager admin UI.

---

### Task 1: Write docker-compose.yml

**Files:**
- Create: `/home/jhh/projects/npm-obs/docker-compose.yml`

**Step 1: Create the file**

```yaml
services:
  goaccess:
    image: xavierh/goaccess-for-nginxproxymanager:latest
    container_name: goaccess
    restart: unless-stopped
    ports:
      - '127.0.0.1:7880:7880'
    volumes:
      - /home/jhh/npm/data/logs:/opt/log:ro
    environment:
      - LOG_TYPE=NPM
      - TZ=Europe/Copenhagen
      - HTML_REFRESH=10
      - KEEP_LAST=30
```

**Step 2: Verify the file looks correct**

Run: `cat /home/jhh/projects/npm-obs/docker-compose.yml`  
Expected: File contents match above exactly.

**Step 3: Commit**

```bash
cd /home/jhh/projects/npm-obs
git init
git add docker-compose.yml docs/
git commit -m "feat: add GoAccess container for NPM log monitoring"
```

---

### Task 2: Start the container and verify it's running

**Files:** None (runtime only)

**Step 1: Pull the image and start**

```bash
cd /home/jhh/projects/npm-obs
docker compose up -d
```
Expected output: `Container goaccess  Started`

**Step 2: Confirm container is healthy**

```bash
docker compose ps
```
Expected: `goaccess` shows `running` status, not `restarting` or `exited`.

**Step 3: Check logs for errors**

```bash
docker compose logs goaccess
```
Expected: No `ERROR` lines. You should see GoAccess starting and parsing log files. Lines like `Parsing /opt/log/proxy-host-*` are normal.

**Step 4: Confirm port is listening**

```bash
ss -tlnp | grep 7880
```
Expected: A line showing `127.0.0.1:7880` is listening.

---

### Task 3: Verify GoAccess dashboard locally

**Files:** None (browser test)

**Step 1: Fetch the dashboard from the server**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:7880
```
Expected: `200`

**Step 2: (Optional) SSH tunnel to preview in browser**

From your local machine:
```bash
ssh -L 7880:localhost:7880 jhh@npm01
```
Then open `http://localhost:7880` in your browser.  
Expected: GoAccess HTML dashboard loads showing proxy host traffic stats.

---

### Task 4: Configure NPM proxy host for the dashboard

**This task is done in the NPM admin UI at http://npm01:81**

**Step 1: Create an Access List**

- Go to: **Access Lists** → **Add Access List**
- Name: `GoAccess Auth`
- Under **Authorization**, add a username and password
- Save

**Step 2: Create the Proxy Host**

- Go to: **Proxy Hosts** → **Add Proxy Host**
- Fill in the **Details** tab:

| Field | Value |
|---|---|
| Domain Names | `stats.yourdomain.com` (your actual subdomain) |
| Scheme | `http` |
| Forward Hostname / IP | `host.docker.internal` |
| Forward Port | `7880` |
| Websockets Support | **ON** |
| Access List | `GoAccess Auth` |

- Click the **SSL** tab:
  - SSL Certificate: Request a new Let's Encrypt certificate
  - Force SSL: **ON**
  - HTTP/2 Support: ON (optional)
- Click **Save**

**Step 3: Verify DNS**

Ensure `stats.yourdomain.com` has an A record pointing to your server's public IP before requesting the certificate.

```bash
dig +short stats.yourdomain.com
```
Expected: Your server's public IP address.

---

### Task 5: End-to-end verification

**Step 1: Open the dashboard in a browser**

Navigate to `https://stats.yourdomain.com`  
Expected: Browser prompts for username/password (basic auth). After entering credentials, GoAccess dashboard loads over HTTPS with a valid certificate.

**Step 2: Verify real-time updates**

- Keep the dashboard open
- From another terminal, make a request through NPM to any proxied service:
  ```bash
  curl https://any-of-your-proxy-hosts.com
  ```
- Watch the dashboard — within `HTML_REFRESH=10` seconds, the request should appear in the real-time panel.

**Step 3: Verify historical data**

The dashboard should show traffic from rotated `.gz` log files as well. Check that the date range selector shows data from past days (not just today).

**Step 4: Final commit**

```bash
cd /home/jhh/projects/npm-obs
git add .
git commit -m "docs: add implementation plan and verify deployment"
```
