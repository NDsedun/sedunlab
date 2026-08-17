---
title: "Setting Up Homepage Dashboard: Integrating Docker Containers and TrueNAS SCALE Monitoring"
date: 2026-08-13 14:20:00 +0300
categories: [homelab, software]
tags: [dashboard, homepage, docker, truenas, monitoring]
image:
  path: /assets/img/posts/homepage-setup-cover.webp
  alt: Homepage Dashboard in a Home Laboratory
lang: en
hidden: true
alt_lang_url: /posts/homepage-dashboard-truenas-docker-setup/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/homepage-dashboard-truenas-docker-setup/)

---

In previous articles, we deployed a full local infrastructure, organized our domains, and secured our traffic using SSL. Now it is time to combine all of our services (Grafana, AdGuard, Plex, Transmission, UrBackup) into a single, beautiful, modern, and informative starting page.

Today, we will dissect the configuration of the popular **Homepage** dashboard (by *gethomepage.dev*), connect it to the Docker container stats of a remote Ubuntu VM, and configure system hardware monitoring for **TrueNAS SCALE** via its API.

---

## Step 1. Installing Homepage on TrueNAS SCALE

To run the dashboard, we will use the ready-made official app from the TrueNAS SCALE catalog:

1. Go to **Apps** -> **Discover Apps**.
2. Search for **Homepage** and click **Install**.
3. **Storage Settings (Important):** By default, the system suggests temporary storage (`ixVolume`). To prevent your configurations from disappearing whenever the app updates or restarts, under the **Storage** section, choose the **Host Path** type of storage and specify the path to a pre-created directory on your ZFS pool (e.g., `/mnt/Mega/configs/homepage`).
4. **Permissions:** Ensure that the owner or a user with write permissions to this folder is the system user **`apps`** (UID/GID `568` in TrueNAS SCALE). If the folder is owned by `root` with no write permissions for `apps`, the container won't be able to write the default configuration files and will crash on launch.
5. Save and start the application.

---

## Step 2. Connecting Docker Container Statuses

Homepage can show the CPU/RAM load and running state (`RUNNING` / `STOPPED`) for any dashboard card. Since our dashboard runs on TrueNAS SCALE and the containers are launched on a separate Ubuntu VM, we need to link them securely.

To do this, we run a lightweight Docker socket proxy on the Ubuntu VM: **`docker-socket-proxy`**:

```yaml
# Add to docker-compose.yml on the Ubuntu VM:
services:
  docker-proxy:
    image: tecnativa/docker-socket-proxy:latest
    container_name: docker-proxy
    restart: unless-stopped
    environment:
      - CONTAINERS=1 # Allow reading container info
      - POST=0       # Restrict write requests (security!)
    ports:
      - "2375:2375"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

### Configuring `docker.yaml` in Homepage:
Now, in your Homepage configs folder, open the `docker.yaml` file and define our VM (to be safe, we use its **Tailscale IP** to bypass KVM network isolation):

```yaml
ubuntu-docker:
  host: 100.x.y.w # Tailscale IP of your Ubuntu server
  port: 2375      # Docker proxy port
```

Now, in the `services.yaml` file, we can add the `server` and `container` parameters to any service card:
```yaml
    - Grafana:
        icon: grafana.png
        href: https://grafana.home.myhomelab.org
        server: ubuntu-docker
        container: grafana
```

---

## Step 3. Monitoring TrueNAS SCALE Host via API

To display the CPU temperature, RAM load, and free ZFS pool space of our main storage, we will use the built-in widget.

### Generating an API Key in TrueNAS:
1. Log into your TrueNAS SCALE web panel.
2. Click on your profile icon in the top right corner and choose **API Keys**.
3. Generate a new key (e.g., `homepage-key`) and save it.

![Generating API Key in TrueNAS SCALE Interface](/assets/img/posts/truenas-api.png)
_Fig. 1. Generating API Key for Homepage access_

### Configuring `services.yaml`:
Now, write the TrueNAS widget configuration in the Homepage `services.yaml` file:

```yaml
- Hardware & Storage:
    - TrueNAS:
        icon: truenas.png
        href: https://truenas.home.myhomelab.org
        widget:
          type: truenas
          url: http://100.x.y.z:4434 # Tailscale IP of your TrueNAS and its web port
          key: YOUR_GENERATED_API_KEY
          enablealerts: true
```

---

## Step 4. Widget Alignment and Custom Styling

By default, Homepage aligns header system widgets (clock, weather) horizontally in a single row. If you want to stack them vertically:

1. Open the `custom.css` file in your Homepage configurations directory.
2. Add the following CSS rules:
   ```css
   .header-widgets, div[class*="HeaderWidgets"] {
     flex-direction: column;
     align-items: flex-start;
     gap: 0.5rem;
   }
   ```

![Ready and configured Homepage dashboard in home network](/assets/img/posts/homepage-monitoring.png)
_Fig. 2. Fully configured Homepage dashboard with container and service monitoring_

---

## Conclusion

The Homepage dashboard is the perfect face for a home laboratory. Thanks to smart integration with Docker and system APIs, it transforms from a simple page of bookmarks into a powerful interface for monitoring and managing your home infrastructure.
