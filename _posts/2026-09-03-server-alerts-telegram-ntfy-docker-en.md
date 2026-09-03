---
title: "Instant Server Alerts in Telegram: Monitoring Backups and Disks in 5 Minutes"
date: 2026-09-03 15:00:00 +0300
categories: [homelab, devops]
tags: [telegram, monitoring, bash, backup, automation, homelab]
image:
  path: /assets/img/posts/server-alerts-telegram-cover.webp
  alt: Server Alerts in Telegram for Homelab
lang: en
hidden: true
alt_lang_url: /posts/server-alerts-telegram-ntfy-docker/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/server-alerts-telegram-ntfy-docker/)

---

Monitoring dashboards like Grafana or a sleek Homepage layout are fantastic. However, they share one fundamental limitation: **they are inherently passive**. Nobody sits in front of a monitoring screen at 3:00 AM to verify whether nocturnal database backups completed or if an SSD volume unexpectedly filled to capacity.

True peace of mind for any Homelab engineer and sysadmin arrives when servers autonomously deliver **instant, active notifications directly to your smartphone**.

Today, we will dissect the simplest, most dependable, and widely adopted push notification pattern: creating a personal **Telegram bot without writing a single line of complex code**, powered entirely by a native `curl` command. We will also integrate it with scheduled backup routines and configure automated disk capacity surveillance.

---

## Step 1. Creating a Telegram Bot in 2 Minutes

To push messages from the command line, you need two pieces of information: your **Bot Token** and your **Chat ID**.

### 1. Generating a Bot Token
1. Open Telegram and search for the official [**@BotFather**](https://t.me/BotFather).
2. Send the command `/newbot`.
3. Provide a display name (e.g., `Homelab Monitor`) and a username ending in `bot` (e.g., `sedunlab_alerts_bot`).
4. BotFather returns an **API Token** formatted like `123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ`. Save this credential securely!

### 2. Finding Your Personal Chat ID
1. Find your newly created bot in Telegram and click **Start** (send any arbitrary test greeting like "Hello").
2. Next, search for the utility bot [**@userinfobot**](https://t.me/userinfobot) and press Start.
3. It replies with your numeric **Id** (e.g., `987654321`).

---

## Step 2. Sending a Test Notification via CLI

Test the pipeline instantly with a single `curl` call from your server terminal:

```bash
BOT_TOKEN="YOUR_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"
MESSAGE="🚀 Test alert from my home server infrastructure!"

curl -s -X POST "https://api.telegram.org/bot$BOT_TOKEN/sendMessage" \
     -d "chat_id=$CHAT_ID" \
     -d "text=$MESSAGE" \
     -d "parse_mode=HTML"
```

Within a fraction of a second, an alert buzzes on your phone!

---

## Step 3. Building a System-Wide Helper Script

Instead of scattering tokens across every cron job and backup script, let's establish a centralized binary at `/usr/local/bin/tg-notify`:

```bash
sudo nano /usr/local/bin/tg-notify
```

Paste the following script:

```bash
#!/bin/bash
# Universal Telegram Notification Dispatcher

BOT_TOKEN="YOUR_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"

MESSAGE="$1"

if [ -z "$MESSAGE" ]; then
    echo "Usage: tg-notify 'Your alert text'"
    exit 1
fi

curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
     -d "chat_id=${CHAT_ID}" \
     -d "text=${MESSAGE}" \
     -d "parse_mode=HTML" > /dev/null
```

Grant executable permissions:

```bash
sudo chmod +x /usr/local/bin/tg-notify
```

Now you can trigger rich HTML notifications from anywhere in the OS with a clean one-liner:
```bash
tg-notify "⚠️ <b>Notice:</b> Service stack restarted successfully."
```

---

## Practical Example 1: Backup Success & Failure Alerts

Remember our backup automation for **Vaultwarden** or **Restic**? Let's equip it with status-aware reporting:

```bash
#!/bin/bash
BACKUP_DIR="/opt/vaultwarden/backups"
DATA_DIR="/opt/vaultwarden/data"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

# Execute consistent SQLite backup
if sqlite3 "$DATA_DIR/db.sqlite3" ".backup '$BACKUP_DIR/db_$TIMESTAMP.sqlite3'"; then
    SIZE=$(du -sh "$BACKUP_DIR/db_$TIMESTAMP.sqlite3" | cut -f1)
    tg-notify "✅ <b>Vaultwarden Backup:</b> Completed successfully. Size: <code>$SIZE</code> ($TIMESTAMP)."
else
    tg-notify "❌ <b>Vaultwarden Backup:</b> BACKUP FAILURE! Inspect server logs immediately."
    exit 1
fi
```

Every morning, you receive confirmation with a green status badge and backup size metrics, while any unforeseen failure triggers an urgent alert.

---

## Practical Example 2: Proactive Disk Space Sentry

A classic root cause of container crashes and database corruption is an unmonitored root disk filling up with Docker log files.

Let's configure a lightweight watchdog at `/opt/scripts/check-disk.sh`:

```bash
#!/bin/bash
# Disk Space Threshold Sentry

THRESHOLD=85
CURRENT=$(df / | grep / | awk '{ print $5}' | sed 's/%//g')

if [ "$CURRENT" -gt "$THRESHOLD" ]; then
    HOSTNAME=$(hostname)
    tg-notify "🚨 <b>Storage Alert!</b>%0AServer: <code>$HOSTNAME</code>%0APartition utilization: <b>$CURRENT%</b> (Limit: $THRESHOLD%)"
fi
```

Make it executable (`chmod +x /opt/scripts/check-disk.sh`) and schedule it via `crontab -e` to run every 6 hours:

```text
0 */6 * * * /bin/bash /opt/scripts/check-disk.sh
```

---

## Self-Hosted Purist Alternative: Ntfy

If you prefer avoiding reliance on Telegram's cloud infrastructure and want an open-source, self-hosted push architecture, **ntfy** (written in Go) is the gold standard:

```yaml
# docker-compose.yml for ntfy
services:
  ntfy:
    image: binwiederhier/ntfy
    container_name: ntfy
    restart: unless-stopped
    ports:
      - "8095:80"
    volumes:
      - ./cache:/var/cache/ntfy
    command: serve
```

It offers official native clients for Android and iOS, supports Web Push in desktop browsers, and handles notifications with a simple `curl -d "Disk alert" http://ntfy.home.myhomelab.org/alerts`.

---

## Conclusion

Deploying an operational notification channel takes less than 5 minutes, yet prevents data loss disasters and eliminates monitoring anxiety. Integrate these lightweight webhooks into your scheduled cron routines and disaster recovery workflows to ensure your Homelab remains completely under control at all times!
