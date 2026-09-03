---
title: "Private Self-Hosted ChatGPT in Homelab: Running Ollama and Open WebUI in Docker"
date: 2026-09-03 14:00:00 +0300
categories: [homelab, ai]
tags: [ollama, open-webui, docker, ai, llm, self-hosted]
image:
  path: /assets/img/posts/ollama-openwebui-cover.webp
  alt: Self-Hosted AI with Ollama and Open WebUI in Docker
lang: en
hidden: true
alt_lang_url: /posts/local-ai-ollama-open-webui-docker/
---

[🇺🇦 Читати цю статтю в оригіналі українською](/posts/local-ai-ollama-open-webui-docker/)

---

Artificial intelligence has become an indispensable part of a developer's and sysadmin's daily workflow. However, commercial cloud services like ChatGPT or Claude come with two substantial downsides: **recurring subscription costs** ($20/month per user) and a complete **lack of privacy**. Sending internal server configuration files, proprietary codebases, or sensitive documents to third-party cloud servers represents a major data leak hazard.

Fortunately, the open-source LLM ecosystem has taken a massive leap forward. Today, any enthusiast can deploy their own full-featured ChatGPT alternative on their home server in under 10 minutes — running **completely offline, with zero external tracking, and 100% free forever**.

To achieve this, we combine two best-in-class tools:
* **Ollama** — a high-performance, lightweight backend engine for downloading and running local neural network weights (Llama 3, DeepSeek, Qwen, Mistral).
* **Open WebUI** — a feature-rich web frontend that looks and feels just like ChatGPT (chat history, document RAG, voice interaction, multi-user support).

---

## How Does the Stack Work?

```text
[ Browser / Mobile Device ]
        │
        ▼ (HTTPS via Nginx Proxy Manager)
[ Open WebUI ] ── (File uploads / RAG / Prompts)
        │
        ▼ (Internal Docker Network: port 11434)
[ Ollama Core ] ── (Runs model inference on GPU or CPU)
```

All computations occur locally: Open WebUI communicates with Ollama's internal API socket, streams tokens back in real time, and renders markdown with syntax highlighting directly in your browser.

---

## Step 1. Preparing Docker Compose

Let's create a dedicated folder for our AI stack on the server, such as `/opt/local-ai`:

```bash
sudo mkdir -p /opt/local-ai && cd /opt/local-ai
```

Create a `docker-compose.yml` file:

```yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ./ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    # If your machine has an NVIDIA GPU, uncomment the block below for hardware acceleration:
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=super_secret_homelab_key_change_me
    volumes:
      - ./openwebui_data:/app/backend/data
    depends_on:
      - ollama
```

> 💡 **CPU vs. GPU?**  
> If your home server lacks an NVIDIA discrete graphics card, don't worry! Ollama automatically optimizes model execution for standard modern CPUs using AVX2 instructions. Compact modern architectures (such as `llama3.2:3b` or `qwen2.5:3b`) respond with impressive speed even on modest hardware.

Launch the stack:

```bash
docker compose up -d
```

---

## Step 2. Pulling Your First Models

Once the containers are up, the Ollama backend is ready to download model weights from the official library. You can pull models directly from the server CLI:

```bash
# Meta's lightweight, ultra-fast model (runs smoothly on CPU)
docker exec -it ollama ollama run llama3.2:3b

# Exceptional model specialized in coding, Bash scripts, and configurations
docker exec -it ollama ollama run qwen2.5-coder:7b

# Deep reasoning model with chain-of-thought capabilities
docker exec -it ollama ollama run deepseek-r1:8b
```

When the download finishes, the model loads into memory and allows interactive testing right in your terminal. Press `Ctrl + D` to exit.

---

## Step 3. Configuring Open WebUI

Open your browser and navigate to `http://192.168.50.125:3000`.

1. **Create the Admin Account**: The very first account registered becomes the instance administrator. Provide your name, email, and a secure password.
2. **Instance Hardening**: Head to **Admin Panel** -> **Settings** -> **General** and disable public registration (**Enable New Sign Ups: Off**) so that unintended local network guests cannot consume your server hardware resources.
3. In the model selector dropdown at the top, you will immediately see all models pulled into Ollama!

---

## Step 4. Setting Up a Custom Domain and HTTPS

Using the **AdGuard Home** and **Nginx Proxy Manager** gateway we established previously, let's configure a clean, secured domain:

1. **DNS Rewrite in AdGuard**:
   * **Domain**: `ai.home.myhomelab.org`
   * **IP Address**: Your proxy server IP (e.g., `192.168.50.125`).
2. **Proxy Host in Nginx Proxy Manager**:
   * **Domain**: `ai.home.myhomelab.org`
   * **Scheme**: `http`
   * **Forward Host**: Your VM IP running Open WebUI.
   * **Forward Port**: `3000`
   * **Websockets Support**: Must be **enabled** (required for zero-latency streaming token responses).
   * **SSL**: Select your wildcard certificate `*.home.myhomelab.org` and toggle **Force SSL**.

Your self-hosted AI is now accessible at `https://ai.home.myhomelab.org` with a green lock!

---

## Superpowers of Open WebUI

### 1. Document Chat (Retrieval-Augmented Generation / RAG)
Click the paperclip attachment icon next to the input box and upload any PDF, server log, or reference manual. Open WebUI embeds and indexes the file on the fly, allowing the model to answer questions strictly grounded in your private document.

### 2. Custom Persona Agents (Modelfiles)
Under **Workspace** -> **Models**, you can build specialized assistants (e.g., a "DevOps Assistant") with tailored system prompts:
> *"You are a senior DevOps engineer. Your task is to diagnose Docker errors and craft concise, safe Bash automation scripts."*

---

## Conclusion

Running an autonomous LLM within your Homelab is more than just a tech demonstration — it is an essential step toward digital self-reliance. You gain a powerful co-pilot for coding, analysis, and brainstorming that does not depend on cloud uptime, incurs zero recurring costs, and guarantees that your private code and personal data never leave your premises.
