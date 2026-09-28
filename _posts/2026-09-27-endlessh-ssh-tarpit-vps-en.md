---
title: "Endlessh (SSH Tarpit): Trapping Hacker Bots for Weeks on Your VPS"
date: 2026-09-27 12:15:00 +0300
categories: [vps, security]
tags: [endlessh, ssh, tarpit, security, vps, docker, linux]
image:
  path: /assets/img/posts/endlessh-cover.webp
  alt: Endlessh SSH Tarpit Bot Trap on VPS
lang: en
hidden: true
alt_lang_url: /posts/endlessh-ssh-tarpit-vps/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/endlessh-ssh-tarpit-vps/)

---

Anyone who has deployed a public VPS server knows the sobering reality: within 15 minutes of provisioning an IP address, your system log (`/var/log/auth.log`) starts bursting at the seams.

Hundreds of automated botnets and script-kiddies scan port 22 relentlessly around the clock, brute-forcing pedestrian credentials: `admin`, `root`, `test`, `qwerty`.

Traditionally, system administrators respond with standard defensive measures:
1. Moving OpenSSH to a non-standard port.
2. Installing `fail2ban` to drop IP addresses after consecutive failed authentication attempts.

However, consider this: when `fail2ban` severs an attacker's TCP connection, the bot instantly reclaims its network socket and immediately targets its next victim. **What if, instead of kicking the bot out... you forced it to wait for eternity?**

This is the principle behind an **SSH Tarpit**, and the gold standard implementation is a tiny utility called **Endlessh**.

![Live Logs and Metrics of Captured Bots in Endlessh](/assets/img/posts/endlessh-metrics-ui.webp)
_Fig. 1. Endlessh terminal logs: automated bots stuck in the tarpit for 24 hours and over 3 days_

---

## What is Endlessh and How Does a Tarpit Work?

According to the official SSH protocol specification (**RFC 4253, Section 4.2**), before the client and server exchange their protocol version strings, the server is permitted to transmit arbitrary textual header banners. The client is obligated to wait for and parse this input.

Endlessh exploits this standard with mischievous elegance:
1. An automated bot hits port 22 expecting an OpenSSH prompt.
2. Endlessh accepts the connection immediately, but **never prompts for credentials**.
3. Instead, it transmits random lines of banner text at a glacial crawl: **one line every 10 seconds**.
4. Standard automated brute-forcing scripts rarely enforce aggressive timeouts during the pre-auth banner phase. Consequently, they get hooked and **hang open for hours, days, or even weeks**.

```text
[ Malicious Bot ]
        │
        ▼ (Connection attempt on port 22)
[ Endlessh Tarpit ]
        │ ── (Line 1) ──> [Waits 10 seconds...]
        │ ── (Line 2) ──> [Waits 10 seconds...]
        │ ── (Line 3) ──> [Waits 10 seconds...]
        ▼
   Bot socket locked up for weeks! Attacker resources depleted.
```

Written in C (or Go), Endlessh relies on asynchronous non-blocking `poll()` routines, consuming **under 2–3 MB of RAM** while easily maintaining thousands of simultaneous attacker sockets!

---

## Step 1. Relocating Real OpenSSH to a Protected Port

Before opening the trap on port 22, move your genuine administrative SSH listener to a non-standard port (such as `53798` or any alternative high port).

> ⚠️ **Warning:** Keep your existing terminal session alive until you confirm successful login via a secondary window, ensuring you do not lock yourself out!

1. Edit the SSH daemon configuration:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
2. Locate `#Port 22` or `Port 22`, uncomment it, and specify your new port:
   ```text
   Port 53798
   ```
3. Allow the new port through your system firewall:
   * **If using UFW (Ubuntu/Debian):**
     ```bash
     sudo ufw allow 53798/tcp
     sudo ufw reload
     ```
   * **If using Firewalld (CentOS/AlmaLinux/RHEL):**
     ```bash
     sudo firewall-cmd --permanent --add-port=53798/tcp
     sudo firewall-cmd --reload
     ```
4. If you run **Fail2ban**, ensure you update the monitored port in `/etc/fail2ban/jail.local` (`port = 53798` under the `[sshd]` jail section) and restart the service:
   ```bash
   sudo systemctl restart fail2ban
   ```
5. Restart the OpenSSH service:
   ```bash
   sudo systemctl restart sshd   # or 'sudo systemctl restart ssh' on Debian/Ubuntu
   ```
6. **Mandatory Verification**: Open a new terminal on your local machine and verify connectivity:
   ```bash
   ssh -p 53798 user@your-server-ip
   ```
   Once logged in, standard port 22 is officially liberated for our trap!

---

## Step 2. Deploying Endlessh via Docker

The cleanest and most robust deployment method is running the **`shizunge/endlessh-go`** image from Docker Hub. Unlike older builds, this version is written in Go, provides native Prometheus metrics export, and includes automated GeoIP resolution for incoming attackers.

> 💡 **Important:** The `endlessh-go` image is configured via **CLI flags**, not environment variables (`ENV`).

### Option A: Via `docker-compose.yml`

1. Create a service directory:
   ```bash
   sudo mkdir -p /opt/endlessh && cd /opt/endlessh
   ```

2. Write `docker-compose.yml`:
   ```yaml
   version: '3.8'

   services:
     endlessh:
       image: shizunge/endlessh-go:latest
       container_name: endlessh
       restart: unless-stopped
       ports:
         - "22:2222"   # Map external port 22 to internal tarpit port 2222
         - "2112:2112" # Prometheus metrics endpoint
       command:
         - -prometheus_enabled       # Enable Prometheus metrics exporter (disabled by default!)
         - -geoip_supplier=ip-api    # Automatically query country & Geohash via ip-api.com
         - -interval_ms=5000         # Delay between banner lines (5s = 5000ms)
         - -logtostderr
         - -v=1
   ```

3. Launch the container:
   ```bash
   docker compose up -d
   ```

### Option B: Direct `docker run` Command

If you prefer launching it via a single standalone command:

```bash
docker run -d \
  --name endlessh \
  --restart unless-stopped \
  -p 22:2222 \
  -p 2112:2112 \
  shizunge/endlessh-go:latest \
  -prometheus_enabled \
  -geoip_supplier=ip-api \
  -interval_ms=5000 \
  -logtostderr \
  -v=1
```

#### Why these specific flags?
* **`-interval_ms=5000` (5 seconds):** Many scanning scripts enforce a short *idle/read timeout* (aborting after 5–10 seconds of socket silence). Sending banner lines every 5 seconds keeps them engaged without triggering inactivity disconnects.
* **`-geoip_supplier=ip-api`:** Automatically queries the free `ip-api.com` service for geographic coordinates and country names without requiring manual downloads of offline MaxMind GeoLite2 databases.
* Verify port 22 is permitted in your firewall (`sudo ufw allow 22/tcp` or `sudo firewall-cmd --permanent --add-port=22/tcp`).

---

## Step 3. Observing the Trapped Bots (Live Logs)

Within minutes of opening port 22, the first automated probes will stumble into your tarpit. Inspect the live container output:

```bash
docker logs -f endlessh
```

![Real Endlessh Terminal Log](/assets/img/posts/Endlessh_log.png)
_Fig. 1. Live Endlessh logs on our VPS: bots trapped in the tarpit_

The log details the lifecycle of every trapped connection:
* **`ACCEPT host=...`** — An attacker connects to port 22 and begins reading the endless banner.
* **`CLOSE host=... time=... bytes=...`** — The connection terminates, showing exact duration and bytes transferred.

### Why do some bots disconnect after 10–20 seconds?
You will notice several bots disconnecting with `time=10.00...` or `time=20.00...`. This occurs because many automated mass-scanning frameworks (built on Paramiko, Go, or Masscan) enforce a hardcoded client-side timeout of 10 to 20 seconds for the entire SSH handshake. If authentication is not presented within that window, the client initiates a `FIN`/`RST` disconnect.

Even so, a 20-second stall is a **massive win**: it ties up attacker socket worker threads and throttles rapid subnet sweeps. Meanwhile, persistent brute-force engines and less aggressive bots hang open for tens of minutes, hours, or even days!

---

## Bonus: Monitoring with Prometheus & Grafana

With `-prometheus_enabled` active, `endlessh-go` exports detailed telemetry on port `2112/metrics`.

![Live Endlessh Grafana Dashboard](/assets/img/posts/Grafana_Trapped.png)
_Fig. 2. Live Grafana dashboard: over 200 trapped bots, 12 hours of wasted attacker compute, attack world map, and country statistics_

### 1. Adding the Scrape Target in Prometheus
In your monitoring server's `prometheus.yml`, append the new scrape job:

```yaml
scrape_configs:
  - job_name: 'vps-endlessh'
    scrape_interval: 10s
    static_configs:
      - targets: ['your-vps-ip:2112'] # Public IP, domain, or Tailscale IP of your VPS
```

Reload Prometheus (`docker restart prometheus` or POST to `/-/reload`).

### 2. Importing the Official Grafana Dashboard
The official community dashboard on Grafana Labs is ID **`15156`** (*Endlessh*):
1. In Grafana, navigate to: **Dashboards** ➡️ **New** ➡️ **Import**.
2. Enter ID **`15156`** and click **Load**.
3. Select your **Prometheus** data source and click **Import**.

### 3. Two Essential Dashboard Fixes (Pro Tips)

After importing, make the dashboard editable by clicking **`Make editable`** in the top-right corner.

#### 1. Eliminating the "API KEY REQUIRED" Watermark on the Map
By default, the Geomap panel requests Carto raster tiles, which now enforce an API key requirement.
* Click the three dots on the **Locations** panel ➡️ **Edit**.
* In the right-hand settings panel, expand **Map layers ➡️ Basemap (Layer 0)**.
* Change the **Type** / **Theme** to **Open Street Map** (completely free, zero API keys required).

#### 2. Fixing the Blank Map in Modern Grafana (Grafana 9 / 10 / 13)
The original dashboard routed map coordinates through 4 legacy `groupBy` transformations, which break in recent Grafana versions due to aggregation renaming. The cleanest fix is querying Prometheus directly:
* In the **Locations** panel, switch to the **Query** tab (bottom left).
* Change the data source from `-- Dashboard --` to your **Prometheus** source.
* Enter the direct query:
  ```promql
  sum by (geohash, country, location) (endlessh_client_open_count)
  ```
* Toggle **Instant** to enabled and set **Format: Table**.
* Switch to the **Transformations** tab and delete the old legacy transformations (trash can icon 🗑️).
* In the right-hand sidebar under **Map layers ➡️ Layer 1 (Markers)**, ensure **Location mode** is set to `Geohash` and **Geohash field** is set to `geohash`.
* Click **Apply** and save the dashboard.

Attackers' geographic coordinates immediately populate across the globe with their respective cities and countries!

---

## Conclusion: Active Defense and Internet Hygiene

Endlessh is more than just a satisfying counter-trolling mechanism — it represents **active community defense**:
1. **Resource Preservation**: Instead of forcing OpenSSH to fork processes and evaluate thousands of crypto handshakes, Endlessh ties up bots with a single thread.
2. **Global Threat Mitigation**: Botnets operate with finite concurrent socket pools. By locking up hundreds of bot threads on your VPS, you tangibly shield vulnerable unpatched servers worldwide from being scanned.

Deploy Endlessh on your VPS — watching your security logs has never been this rewarding! 🪤✨
