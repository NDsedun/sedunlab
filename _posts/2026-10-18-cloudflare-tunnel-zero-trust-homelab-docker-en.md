---
title: "Cloudflare Tunnel & Zero Trust: Exposing Your Homelab and Docker Without Open Ports or Public IP"
date: 2026-10-18 17:00:00 +0300
categories: [homelab, networking]
tags: [cloudflare, zero-trust, docker, homelab, networking, security, self-hosted, vps]
image:
  path: /assets/img/posts/cloudflare-tunnel-cover.jpg
  alt: Securely exposing homelab services with Cloudflare Tunnel
lang: en
hidden: true
alt_lang_url: /posts/cloudflare-tunnel-zero-trust-homelab-docker/
---

[🇺🇦 Читайте цю статтю українською](/posts/cloudflare-tunnel-zero-trust-homelab-docker/)

---

Anyone running a self-hosted Homelab or personal home server (Proxmox, TrueNAS, or an old desktop running Docker) eventually hits the same roadblock: **how do you access your private services safely when away from home?**

The traditional methods carry significant pain points:
1. **Carrier-Grade NAT (CGNAT) / Dynamic IP:** Over 90% of residential ISPs place subscribers behind CGNAT. You rarely get a dedicated static public IPv4, and ISPs often block inbound port forwarding altogether.
2. **Router Port Forwarding is Dangerous:** Punching holes in your router (forwarding ports 80 and 443) broadcasts your residential IP to the world. As we demonstrated in our [Cowrie Honeypot experiment](/posts/cowrie-ssh-honeypot-vps-en/), automated scanners probe open ports within **3 to 5 minutes** of discovery. A single vulnerability in your web server or router exposes your entire home subnet.
3. **VPNs (like [Tailscale](/posts/tailscale-in-homelab-en/)) Aren’t for Everyone:** While Tailscale is unbeatable for private admin work, you cannot easily share a private URL with non-technical family members to view your [Immich](/posts/immich-selfhosted-google-photos-homelab-en/) photos without installing VPN client apps on their devices.

The modern solution to this challenge is **Cloudflare Tunnel (formerly Argo Tunnel)** coupled with **Cloudflare Zero Trust Access**.

![Cloudflare Zero Trust Tunnel Architecture Diagram](/assets/img/posts/cloudflare-tunnel-architecture.jpg)
_Fig. 1. Cloudflare Tunnel architecture: outbound encrypted connection with zero inbound ports, traffic filtering through Cloudflare Edge, and custom domain access_

---

## How It Works: The Power of Outbound Tunnels

Instead of opening incoming doors (inbound firewall ports) on your home router, the lightweight **`cloudflared`** daemon running inside a Docker container initiates a persistent, **outbound encrypted connection** to Cloudflare’s nearest global edge data center.

When someone browses to `vault.yourdomain.com`:
1. The request hits Cloudflare’s nearest global Edge Network node.
2. Cloudflare inspects traffic, blocks DDoS floods, and filters malicious bots.
3. If **Zero Trust Access** is configured, an authentication barrier appears (such as Google SSO or a one-time email PIN).
4. Approved traffic is proxied through the encrypted tunnel directly into your local Docker container.

**The Advantages:**
* **Zero inbound open ports on your router.** Firewall remains strictly closed.
* **Bypasses CGNAT, dynamic IPs, and restrictive ISPs** (works over 4G/5G mobile modems or Starlink).
* **Your residential IP address is 100% hidden.**
* **Free, automatic SSL/TLS certificate generation and renewal.**

---

## Prerequisites

1. A custom domain name (costs $2–$10/year from any registrar).
2. A free [Cloudflare](https://dash.cloudflare.com/) account with your domain’s nameservers pointed to Cloudflare.
3. Any machine or server running Docker and Docker Compose.

---

## Step 1. Create a Tunnel in Cloudflare Zero Trust

Modern Cloudflare Tunnels are managed directly via the cloud dashboard without wrestling with manual configuration files:

1. In your Cloudflare dashboard, navigate to **Zero Trust** in the left sidebar (or visit [one.dash.cloudflare.com](https://one.dash.cloudflare.com/)).
2. Register your free Zero Trust organization (free tier supports up to 50 users!).
3. Head to **Networks ➔ Tunnels**.
4. Click **Add a tunnel**, choose the default **Cloudflare (cloudflared)** type, and name it (e.g., `homelab-docker`).
5. Click **Save tunnel**.

---

## Step 2. Deploy `cloudflared` via Docker Compose

On the next screen, Cloudflare provides installation commands for various environments. Select **Docker**. You will see a command containing a long secret tunnel token:

```bash
docker run cloudflare/cloudflared:latest tunnel --no-autoupdate run --token YOUR_TOKEN_HERE
```

To run this as a persistent, reliable service, let's configure `docker-compose.yml`.

Create a dedicated directory:
```bash
mkdir -p /opt/cloudflared
cd /opt/cloudflared
```

Create `docker-compose.yml`:

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=eyJhIjoiYmMy...YOUR_SECRET_TOKEN...
    # Connect to the shared Docker network:
    networks:
      - web-network

networks:
  web-network:
    external: true
```

> **Pro-Tip:** Placing `cloudflared` into a shared user-defined Docker bridge network (`web-network`) alongside your other services (e.g., Uptime Kuma or Vaultwarden) means **you don't even need to publish host ports** via `ports:`! Containers communicate internally using service names (`http://vaultwarden:80`).

Start the container:

```bash
docker compose up -d
```

Return to your Cloudflare browser tab—within a few seconds, the connector status will switch to green (**HEALTHY**).

---

## Step 3. Routing Subdomains (Public Hostnames)

Now map your private internal containers to clean public subdomains. Navigate to the **Public Hostname** tab in your tunnel settings and add your routes:

### Example 1: Uptime Kuma Status Page
* **Subdomain:** `status`
* **Domain:** `yourdomain.com`
* **Type:** `HTTP`
* **URL:** `uptime-kuma:3001` (or local IP `192.168.1.100:3001` if running on another node).

### Example 2: Vaultwarden Password Manager
* **Subdomain:** `vault`
* **Domain:** `yourdomain.com`
* **Type:** `HTTP`
* **URL:** `vaultwarden:80`

### Example 3: Immich Photo Gallery
* **Subdomain:** `photos`
* **Domain:** `yourdomain.com`
* **Type:** `HTTP`
* **URL:** `immich-server:2283`

Click **Save Hostname**. Cloudflare automatically generates the required DNS CNAME record, issues a valid SSL certificate, and binds the traffic to your container.

Visit `https://status.yourdomain.com` from your smartphone using cellular data—it loads instantly!

---

## Step 4. Secret Weapon: Cloudflare Access (Zero Trust SSO)

Exposing sensitive administrative dashboards (like Proxmox, Portainer, or [Beszel](/posts/beszel-lightweight-docker-monitoring-vps-en/)) to the raw public internet carries risks of zero-day exploits and brute-force attempts.

With **Cloudflare Access**, you can place an identity verification shield in front of any internal web app:

1. In the Zero Trust dashboard, open **Access ➔ Applications**.
2. Click **Add an application** ➔ select **Self-hosted**.
3. Name it (e.g., `Beszel Monitoring`) and assign the subdomain `mon.yourdomain.com`.
4. Create an access policy (**Policy**):
   * Action: **Allow**.
   * Rule: **Include ➔ Emails** — enter your authorized email addresses (e.g., `you@gmail.com`).
   * Optionally integrate one-click login with **Google or GitHub OAuth**.
5. Save the policy.

**What Happens in Practice:**  
When an external visitor browses to `mon.yourdomain.com`, the server never serves the Beszel login screen. Instead, Cloudflare intercepts the request with a branded authentication challenge requiring an authorized email and a 6-digit one-time PIN (or OAuth token).

Automated scanners, bots, and Shodan scrapers hit an impenetrable wall.

---

## ⚠️ Important Considerations & Constraints

While Cloudflare Tunnel feels like magic, keep these operational limits in mind:

1. **Video Streaming Restrictions (TOS Section 2.8):**  
   Cloudflare’s free tier terms explicitly prohibit routing excessive non-HTML video streaming (such as terabytes of 4K streams via **Plex** or **Jellyfin**). Abusing bandwidth can result in tunnel suspensions. Use direct [Tailscale](/posts/tailscale-in-homelab-en/) connections for media streaming.
2. **File Upload Size Limits:**  
   Cloudflare enforces a default **100 MB maximum request payload size** on free plans. Uploading a 5 GB archive via Nextcloud or syncing a raw 4K video to Immich in a single request will return a *413 Request Entity Too Large* error.  
   *Solution:* Enable chunked uploads in your client application settings.
3. **SSL Termination at the Edge:**  
   Because Cloudflare operates as a reverse proxy, traffic is decrypted at their edge nodes for WAF analysis before being re-encrypted into your tunnel. If your threat model demands absolute 100% end-to-end zero-knowledge encryption, stick to peer-to-peer WireGuard or Tailscale.

---

## Which One to Pick: Tailscale vs. Cloudflare Tunnel?

Experienced sysadmins universally recommend a **hybrid architecture**:

| Use Case | Best Tool | Rationale |
| :--- | :--- | :--- |
| **Public & Family Access** (Immich, Vaultwarden, blogs) | **Cloudflare Tunnel** | Clean URLs, zero client apps needed, automated SSL |
| **Protected Admin UIs** (Beszel, Portainer, Proxmox) | **Cloudflare Tunnel + Access** | Secured with email PIN or Google SSO before app loads |
| **Core Infrastructure Admin** (SSH, TrueNAS, raw shell) | **Tailscale** | Direct subnet mesh routing without public DNS visibility |
| **Heavy Media Streaming** (Plex, Jellyfin, torrents) | **Tailscale** | Unrestricted bandwidth with zero TOS compliance risks |

---

## Summary

Cloudflare Tunnel is one of the most transformative tools in modern self-hosting. It completely eradicates the frustration of CGNAT, keeps your home network completely sealed against external port scans, and provides enterprise-grade Zero Trust security for free.

Deploy your first tunnel, set up your subdomains, and never worry about residential port forwarding again!
