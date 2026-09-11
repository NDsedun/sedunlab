---
title: "Self-Hosted Obsidian Sync Without Subscriptions: Deploying CouchDB and LiveSync in Docker"
date: 2026-09-11 14:30:00 +0300
categories: [homelab, software]
tags: [obsidian, couchdb, docker, sync, self-hosted, homelab]
image:
  path: /assets/img/posts/obsidian-sync-cover.webp
  alt: Self-Hosted Obsidian Sync with CouchDB in Docker
lang: en
hidden: true
alt_lang_url: /posts/selfhosted-obsidian-sync-couchdb-docker/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/selfhosted-obsidian-sync-couchdb-docker/)

---

**Obsidian** is widely regarded as one of the finest tools for personal knowledge management (Second Brain). Its greatest strength is that notes are stored as plain Markdown files locally on your computer, free from proprietary cloud locks.

However, as soon as you need to review notes on an iPad during a commute or reference a checklist from your smartphone during a meeting, friction arises. The official *Obsidian Sync* subscription costs between \$4 and \$10 per month per user. Free alternatives like **Syncthing** perform well across Windows and Android but fall flat on **iOS (iPad/iPhone)** due to aggressive sandbox background restrictions. Meanwhile, syncing through iCloud onto Windows machines frequently spawns messy duplicate conflicted files.

The ultimate solution is running your own self-hosted sync server using **CouchDB** combined with the **Self-hosted LiveSync** plugin. This setup delivers near real-time, character-by-character synchronization, robust End-to-End Encryption (E2EE), and native support across **Windows, Android, and iOS**!

![Obsidian Self-hosted LiveSync Settings Interface](/assets/img/posts/obsidian-livesync-ui.webp)
_Fig. 1. Self-hosted LiveSync plugin configuration window in Obsidian's dark theme_

---

## Why CouchDB and LiveSync?

* **Full Obsidian Sync Parity**: Edits propagate in real time without waiting for full vault rescans.
* **Universal Multi-Platform Support**: Works seamlessly on iPad, iPhone, Android, Windows, macOS, and Linux.
* **Zero-Knowledge End-to-End Encryption (E2EE)**: Notes are encrypted locally on the device prior to transmission. Even the database administrator cannot read your files.
* **Intelligent Conflict Resolution**: Simultaneous modifications on multiple devices are merged cleanly at the line level without data loss.

---

## Architecture Stack

1. **CouchDB** — A NoSQL document database uniquely suited for distributed bidirectional document synchronization.
2. **Nginx Proxy Manager** — Reverse proxy providing strict HTTPS encryption (mandatory, as iOS actively rejects non-HTTPS endpoints within WebCrypto contexts).
3. **AdGuard Home** — Local authoritative DNS handler resolving `notes.home.myhomelab.org` to our internal proxy IP.

---

## Step 1. Deploying CouchDB in Docker Compose

Prepare a dedicated project workspace on your host (e.g., your Ubuntu VM or Docker host):

```bash
sudo mkdir -p /opt/couchdb && cd /opt/couchdb
```

### 1. Attachment Size Tuning with `local.ini`
By default, CouchDB caps document payload sizes. To ensure you can freely attach heavy images, scans, and PDFs to your notes, configure enhanced buffer ceilings:

```bash
cat <<EOF > /opt/couchdb/local.ini
[chttpd]
max_http_request_size = 4294967296

[couchdb]
max_document_size = 50000000
EOF
```

> ⚠️ **Field Tip:** Ensure `local.ini` exists **PRIOR** to starting the container. If absent, Docker will automatically instantiate a *directory* with that name on the host, causing an initialization mount failure (`mount: not a directory`).

### 2. Crafting `docker-compose.yml`

```yaml
version: '3.8'

services:
  couchdb:
    image: couchdb:latest
    container_name: couchdb
    restart: unless-stopped
    environment:
      - COUCHDB_USER=admin
      - COUCHDB_PASSWORD=SuperSecretPassword2026!  # Specify your own secure password
    volumes:
      - ./data:/opt/couchdb/data
      - ./local.ini:/opt/couchdb/etc/local.ini
    ports:
      - "5984:5984"
```

Spin up the container:

```bash
docker compose up -d
```

Confirm health via `docker ps`. The container should reflect an `Up` status.

---

## Step 2. Establishing HTTPS via Nginx Proxy Manager

Because mobile clients require transport-layer security, open your Nginx Proxy Manager dashboard and add a new **Proxy Host**:

1. **Details Tab**:
   * **Domain Names**: `notes.home.myhomelab.org`
   * **Scheme**: `http`
   * **Forward Host / IP**: IP address of your CouchDB host (e.g., `192.168.50.125`).
   * **Forward Port**: `5984`
   * **Websockets Support**: Must be **Enabled** (essential for LiveSync).
2. **SSL Tab**:
   * Select your wildcard certificate (e.g., `*.home.myhomelab.org`).
   * Enable **Force SSL** and **HTTP/2 Support**.
3. **Advanced Tab** (Lift payload body limits):
   ```nginx
   client_max_body_size 100M;
   ```
4. Save configuration.

Test the endpoint in your browser: `https://notes.home.myhomelab.org` — you should see a JSON welcome response: `{"couchdb":"Welcome","version":"..."}`.

---

## Step 3. Primary Device Configuration (PC with Existing Notes)

1. Open **Obsidian** on your desktop.
2. Navigate to **Settings** ➡️ **Community plugins** (disable Restricted Mode).
3. Search for and install **Self-hosted LiveSync**, then toggle it **Enabled**.
4. Access the plugin settings and select **Rerun Onboarding Wizard** ➡️ **Configure a remote manually**:
   * Server Type: **CouchDB**.
   * **Server URL**: `https://notes.home.myhomelab.org`
   * **User**: `admin`
   * **Password**: Your configured database password.
   * **Database name**: `obsidian`.
   * Click **Test** (if the database does not exist yet, click **Create**).
5. **End-to-End Encryption (E2EE)**:
   * Toggle encryption on and input a strong passphrase. Store this safely in your password manager!
6. **Initial Data Push**:
   * Choose **Push local database to remote** to upload all existing notes to the server.
7. **Export Setup Parameters**:
   * In the plugin options, click **Copy Setup URI** or **Show QR Code**.

---

## Step 4. Linking iPad, iPhone, and Android

The onboarding for secondary mobile clients takes less than a minute:

1. Install **Obsidian** from the App Store or Google Play.
2. Create a clean, empty vault (e.g., `Notes`).
3. Install and enable **Self-hosted LiveSync**.
4. In the prompt window, select **Connect with Setup URI (Recommended)**.
5. Scan the QR code displayed on your PC monitor (or paste the copied string).
6. Enter your E2EE encryption passphrase.
7. Tap **Rebuild local database** (Fetch from remote).

Your complete folder hierarchy, notes, canvas boards, and embedded images will immediately populate!

---

## Common Troubleshooting & Gotchas

* **`Checking configuration failed`**: If Obsidian cannot establish a connection, inspect client DNS resolution. Verify the device queries your local DNS (AdGuard Home) rather than a stock router fallback that cannot resolve local records.
* **Syncing on the Go (Cellular / Remote Wi-Fi)**: To sync outside your home, run **Tailscale** or publish the endpoint safely via **Cloudflare Tunnel**. Because LiveSync applies client-side AES encryption, passing data over the public internet remains zero-trust and completely safe.
* **Multi-User Isolation**: Multiple household members can use the same server by pointing to dedicated database namespaces (e.g., `obsidian_user1` and `obsidian_user2`) paired with unique E2EE passphrases.

---

## Conclusion

Combining CouchDB with the Self-hosted LiveSync plugin represents an exceptional homelab engineering pattern for knowledge workers. It eliminates monthly recurring fees, provides flawless cross-platform harmony between desktop and mobile devices, and ensures you retain total ownership over your private intellectual property!
