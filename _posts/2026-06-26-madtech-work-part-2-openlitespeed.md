---
layout: post
title: "Як я зробив сайт який кожного разу інший: частина 2"
subtitle: "Налаштування reverse proxy в OpenLiteSpeed через CyberPanel"
date: 2026-06-26 11:00:00 +0200
categories: [tutorial, devops]
tags: [openlitespeed, cyberpanel, nginx, reverse-proxy, sse, systemd]
---

У [першій частині](/2026/06/26/madtech-work-part-1-fastapi) ми підняли FastAPI додаток на порту 8001. Тепер треба зробити так щоб OpenLiteSpeed передавав запити з `madtech.work` на нього.

Здавалось би — стандартна задача. Але з безкоштовною версією CyberPanel і OLS вона перетворилась на справжній детективний квест.

## Чому не спрацювала документація

Перший інстинкт — додати nginx-конфіг у Rewrite Rules CyberPanel:

```nginx
location / {
    proxy_pass http://127.0.0.1:8001;
    ...
}
```

**Не працює.** CyberPanel використовує OpenLiteSpeed, а не nginx. OLS має власний синтаксис конфігурації, і nginx-директиви він просто ігнорує.

## Правильний шлях: vhost.conf

Конфіг кожного сайту в OLS знаходиться тут:

```bash
/usr/local/lsws/conf/vhosts/madtech.work/vhost.conf
```

> Зверніть увагу: файл називається `vhost.conf`, а не `vhconf.conf` — обидва існують, але активний саме `vhost.conf`.

Щоб налаштувати reverse proxy, потрібні два блоки.

**Перший** — `extprocessor`, який описує бекенд:

```
extprocessor fastapi {
  type                    proxy
  address                 127.0.0.1:8001
  maxConns                100
  initTimeout             120
  retryTimeout            0
  respBuffer              0
  keepAliveTimeout        120
}
```

**Другий** — `context`, який каже OLS використовувати цей бекенд для всіх запитів:

```
context / {
  type                    proxy
  handler                 fastapi
  allowBrowse             1
  addDefaultCharset       off
  accessControl  {
    allow                 *
  }
}
```

Критичний момент: `handler` має посилатись на **ім'я** extprocessor, а не на URL напряму. Якщо написати `location http://127.0.0.1:8001` — OLS видасть помилку `Can not find handler with type: 12`.

Перевірити помилки конфігурації можна у веб-інтерфейсі OLS Admin (`http://IP:7080`) або в лозі:

```bash
tail -f /usr/local/lsws/logs/error.log
```

## SSE і буферизація

Після налаштування проксі сайт відкривався, але стримінг не працював — браузер отримував помилку з'єднання одразу після старту.

Причина: OLS буферизує відповіді від бекенду. Для SSE це смерть — дані мають йти одразу, без буфера.

Виправлення в двох місцях:

**1. В `extprocessor`** — `respBuffer 0` вже виключає буферизацію на рівні OLS.

**2. У FastAPI** — додаємо заголовок який просить проксі не буферизувати:

```python
return StreamingResponse(
    stream(),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "X-Accel-Buffering": "no",  # ← критично!
    },
)
```

## Systemd сервіс

Запускати uvicorn вручну незручно — після перезавантаження сервера він не стартує. Створюємо systemd сервіс:

```ini
# /etc/systemd/system/madtech.service
[Unit]
Description=madtech.work FastAPI app
After=network.target

[Service]
User=root
WorkingDirectory=/home/madtech.work/app
EnvironmentFile=/home/madtech.work/app/.env
ExecStart=/home/madtech.work/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8001 --workers 1
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable madtech
systemctl start madtech

# Перевірка
systemctl status madtech
journalctl -u madtech -f
```

## Редірект HTTP → HTTPS

В CyberPanel → Websites → madtech.work → **Rewrite Rules** додаємо:

```
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```

SSL сертифікат від Let's Encrypt видається в один клік: CyberPanel → Websites → madtech.work → **Issue SSL**.

## Корисні команди для діагностики

```bash
# Чи слухає FastAPI на порту?
curl http://127.0.0.1:8001/

# Логи OLS в реальному часі
tail -f /usr/local/lsws/logs/error.log

# Перезапуск OLS (не systemctl restart lsws!)
/usr/local/lsws/bin/lswsctrl restart

# Логи FastAPI
journalctl -u madtech -f
```

У **частині 3** поговоримо про найцікавіше — фронтенд з 6 випадковими темами, стримінг JSON і SVG favicon у вигляді кубика.
