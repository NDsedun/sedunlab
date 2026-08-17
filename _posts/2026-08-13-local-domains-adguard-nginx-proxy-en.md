---
title: "Local Domains Without Ports: Setting Up AdGuard Home + Nginx Proxy Manager in Homelab"
date: 2026-08-13 14:00:00 +0300
categories: [homelab, network]
tags: [dns, adguard, reverse-proxy, nginx, homelab, network]
image:
  path: /assets/img/posts/adguard-nginx-cover.webp
  alt: Local Domains in Home Network
lang: en
hidden: true
alt_lang_url: /posts/local-domains-adguard-nginx-proxy/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/local-domains-adguard-nginx-proxy/)

---

When the number of services in a home laboratory (Homelab) starts exceeding three or four, the pain begins. Remembering dozens of addresses like `192.168.50.125:3000` (Grafana), `192.168.50.125:8080` (another service), or `192.168.50.67:32400` (Plex) becomes practically impossible.

It is much more pleasant to type simple and clean names in the browser: `http://grafana.home`, `http://adguard.home`, or `http://truenas.home`.

Today, we will deploy and link **AdGuard Home** (local DNS server) and **Nginx Proxy Manager** (reverse proxy) into a single system using Docker. This will allow you to forget about IP addresses and port numbers in your local network forever.

---

## How Does This Setup Work?

The workflow is very simple and looks like this:
1. You type the address `http://grafana.home` in your browser.
2. Your computer asks the local DNS (**AdGuard Home**): "Where does this domain live?".
3. AdGuard Home answers: "It lives on the IP address of our proxy server (Ubuntu VM)."
4. The request arrives at the proxy server (**Nginx Proxy Manager**) on the standard web port `80`.
5. The proxy server looks at the domain `grafana.home`, understands that this service runs on port `3000`, and silently redirects the traffic there for you.

---

## Step 1. Deploying via Docker Compose

We will combine both services into one stack. To do this, we will create a dedicated working directory `/opt/gateway` on the server and run the following `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    volumes:
      - ./adguard/workdir:/opt/adguardhome/work
      - ./adguard/confdir:/opt/adguardhome/conf
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "3002:3000/tcp" # Port for initial setup
      - "8080:80/tcp"   # AdGuard admin panel port

  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'   # Standard HTTP web port
      - '81:81'   # Proxy admin panel
      - '443:443' # HTTPS port
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
```

> ⚠️ **Important Tip for Ubuntu Server:**  
> By default, Ubuntu runs the `systemd-resolved` service, which occupies port `53` (DNS). To allow the AdGuard container to launch, you need to disable the stub listener in the `/etc/systemd/resolved.conf` file (set `DNSStubListener=no`) and restart the resolver.

---

## Step 2. Configuring AdGuard Home

After launching, open the initial setup wizard on port `3002` (e.g., `http://192.168.50.125:3002`), create a user, and navigate to the main dashboard.

![AdGuard Authentication](/assets/img/posts/adguard-login.png)
_Fig. 1. AdGuard Home Authentication Page_

Now we need to create a rule that routes all local domains to the proxy server.
1. Go to the menu **Filters** -> **DNS Rewrites**.
2. Click **Add DNS Rewrite**.
3. Instead of adding each site manually, we will make a global wildcard record:
   * **Domain**: `*.home`
   * **IP Address**: The IP of your proxy server (e.g., `192.168.50.125`).

![AdGuard DNS Rewrite Settings](/assets/img/posts/adguard-dns.png)
_Fig. 2. Configuring DNS Rewrites in AdGuard Home_

Now, any domain ending in `.home` (such as `grafana.home` or `portainer.home`) will be automatically resolved to the reverse proxy IP address.

---

## Step 3. Configuring Nginx Proxy Manager

Log into the proxy server panel on port `81` (default credentials: `admin@example.com` / `changeme`).

Now create redirection rules for our clean domains to internal ports. For each service, create a **Proxy Host**:
* **Domain Names**: Specify the local domain (e.g., `grafana.home`).
* **Forward Host/IP**: Specify the internal IP address where the service is running.
* **Forward Port**: Specify the port (e.g., `3000` for Grafana or `8080` for AdGuard).

![NPM Proxy Hosts List](/assets/img/posts/nginx-proxy-hosts.png)
_Fig. 3. List of configured proxy hosts in Nginx Proxy Manager_

Once you save the settings, Nginx Proxy Manager will automatically intercept requests to `grafana.home` and proxy them to the correct port.

---

## 🔒 Security and Hardening

Since we are configuring the servers ourselves, it is important to practice good digital hygiene to leave no loopholes for attackers:

1. **Local DNS Only**: Our `.home` domains exist **exclusively** inside your home network. The public internet knows nothing about them. This makes them completely invisible to external scanners and hackers.
2. **Do Not Expose Admin Panels on Your Router**: The proxy manager control panel (port `81`) and the AdGuard panel (port `8080`) must remain closed from the outside. Never forward these ports on your router (Port Forwarding). If remote access is needed, connect only via **Tailscale** or another secure VPN.
3. **Local IP Addresses**: Showing addresses like `192.168.50.X` in screenshots or articles is completely safe, as these are "RFC 1918" (private) IP addresses that do not route on the public internet. No hacker can connect to them from the outside.

---

## Conclusion

Combining AdGuard Home and Nginx Proxy Manager makes managing your home server incredibly convenient. You get not only a clean ad-free internet experience for the whole family, but also beautiful, convenient navigation across all the services of your IT lab.
