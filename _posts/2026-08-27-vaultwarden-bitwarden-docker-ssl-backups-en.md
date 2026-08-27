---
title: "Your Own Password Vault: Running Vaultwarden (Bitwarden) in Docker with HTTPS and Auto-Backups"
date: 2026-08-27 15:40:00 +0300
categories: [homelab, security]
tags: [vaultwarden, bitwarden, docker, security, backup]
image:
  path: /assets/img/posts/vaultwarden-cover.webp
  alt: Vaultwarden Password Manager in Docker
lang: en
hidden: true
alt_lang_url: /posts/vaultwarden-bitwarden-docker-ssl-backups/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/vaultwarden-bitwarden-docker-ssl-backups/)

---

Passwords are the keys to our digital lives. Following major breaches of popular cloud-based password managers (such as LastPass) and continuous price increases for paid subscriptions, storing passwords on self-hosted hardware has become a significant security trend.

The best solution for this is **Vaultwarden** — an unofficial, lightweight backend for the popular **Bitwarden** manager, written in Rust. It is completely free, consumes a mere 20–30 MB of RAM (unlike the original Bitwarden which requires 2–4 GB of RAM), and is fully compatible with all official Bitwarden mobile apps and browser extensions.

Today, we will deploy our own password vault in Docker, protect it with HTTPS encryption, and configure a reliable, automated database backup system.

---

## Step 1. Running Vaultwarden via Docker Compose

For convenience, we will manage the deployment using Docker Compose. Let's create a dedicated folder `/opt/vaultwarden` and write a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - WEBSOCKET_ENABLED=true  # Required for real-time client sync
      - SIGNUPS_ALLOWED=true    # Temporarily enable registrations (disabled later!)
    volumes:
      - ./data:/data
    ports:
      - "8088:80"     # Vaultwarden Web UI
      - "3012:3012"   # WebSocket port
```

Launch the container with this command:
```bash
docker-compose up -d
```

Now, the web interface is available at your local address `http://192.168.50.125:8088`. Open it and **immediately create your master account** (specifying a highly secure master password phrase).

---

## Step 2. Configuring HTTPS (Mandatory!)

> ⚠️ **Important:** Official Bitwarden apps for Chrome, iOS, and Android **strictly refuse to work** without an encrypted HTTPS connection. Web cryptography APIs in modern browsers require a secure environment (Secure Context).

We will utilize the **Nginx Proxy Manager** and **AdGuard Home** combination configured in our previous articles:

1. **DNS Record**: In AdGuard Home, add a rewrite routing `vault.home.myhomelab.org` to the IP address of our proxy server.
2. **Nginx Proxy Manager**: Create a new Proxy Host:
   * **Domain**: `vault.home.myhomelab.org`
   * **Scheme**: `http`
   * **Forward Host**: The IP address of the host running Vaultwarden (e.g., your Tailscale IP or local IP).
   * **Forward Port**: `8088`
   * **Websockets Support**: Enabled! (critical for real-time synchronization across devices).
3. **SSL**: Select our Let's Encrypt wildcard certificate `*.home.myhomelab.org` and toggle **Force SSL**.

Now your vault operates at the secure address `https://vault.home.myhomelab.org` with a green padlock. You can log into the phone app by specifying your new domain in the "Server Address" field.

---

## Step 3. Hardening: Locking the Doors

Since your password manager might now be accessible on your local network (or via VPN), you must block registration access for unauthorized users.

1. Stop the container: `docker-compose down`.
2. Edit the `docker-compose.yml` file, setting the environment parameter **`SIGNUPS_ALLOWED=false`**:
   ```yaml
   environment:
     - WEBSOCKET_ENABLED=true
     - SIGNUPS_ALLOWED=false  # Block new user registrations
   ```
3. Restart the container: `docker-compose up -d`.

Now, no unauthorized visitors can create accounts on your server.

---

## Step 4. Setting Up Automated Backups

By default, Vaultwarden uses a **SQLite** database. Simply copying the `db.sqlite3` file "on the fly" is risky — if a password write operation occurs at that exact moment, the copied database file will be corrupted.

The correct way is to use the `sqlite3 .backup` command.

Let's create a simple backup script `/opt/vaultwarden/backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/opt/vaultwarden/backups"
DATA_DIR="/opt/vaultwarden/data"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

mkdir -p "$BACKUP_DIR"

# 1. Safely backup the database using sqlite3 utility
sqlite3 "$DATA_DIR/db.sqlite3" ".backup '$BACKUP_DIR/db_$TIMESTAMP.sqlite3'"

# 2. Archive encryption keys and attachments folder
tar -czf "$BACKUP_DIR/attachments_$TIMESTAMP.tar.gz" -C "$DATA_DIR" attachments key.der RSA-key.der

# 3. Delete backups older than 7 days
find "$BACKUP_DIR" -type f -mtime +7 -delete

echo "Backup completed successfully at $TIMESTAMP"
```

Make the script executable:
```bash
chmod +x /opt/vaultwarden/backup.sh
```

Let's add it to the `cron` scheduler for a daily run at 3:00 AM. Run `crontab -e` and append the following line:
```text
0 3 * * * /bin/bash /opt/vaultwarden/backup.sh >> /var/log/vaultwarden_backup.log 2>&1
```

> 💡 **Maximum Security Tip**: It is highly recommended to upload the resulting `/opt/vaultwarden/backups` directory once a week to a remote cloud storage using `restic` or `rclone` (which we wrote about earlier).

---

## Conclusion

Your own Vaultwarden server provides freedom from paid subscription limitations and the peace of mind that your passwords are stored in encrypted form on your own hardware. Thanks to running in Docker, HTTPS protection, and a reliable backup script, your personal cybersecurity reaches a whole new level!
