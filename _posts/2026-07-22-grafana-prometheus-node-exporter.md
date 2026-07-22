---
title: "Grafana + Prometheus + Node Exporter: будуємо професійний моніторинг для серверів"
date: 2026-07-22 18:00:00 +0300
categories: [homelab, devops]
tags: [grafana, prometheus, node-exporter, docker, моніторинг]
image:
  path: /assets/img/posts/grafana-monitoring.webp
  alt: Дашборд моніторингу Grafana
---

У попередніх статтях ми розглядали **Netdata** як швидкий інструмент для миттєвої діагностики сервера. Проте, якщо вам потрібен довгостроковий аналіз метрик, збір даних із кількох серверів в одну панель та гнучкі сповіщення в Telegram, стандартною зв'язкою в індустрії є **Grafana + Prometheus + Node Exporter**.

Сьогодні ми розгорнемо цей стек у Docker за кілька хвилин та підключимо готовий професійний дашборд.

---

## Як працює цей стек?

* 🔍 **Node Exporter** — легкий агент від розробників Prometheus, який встановлюється на сервер, зчитує системні метрики (процесор, пам'ять, диски, мережу) і віддає їх у вигляді простого тексту на порту `9100`.
* 🗄️ **Prometheus** — спеціалізована база даних часових рядів (Time Series Database). Вона регулярно самостійно «ходить» на Node Exporter, забирає нові метрики та зберігає їх у себе.
* 📊 **Grafana** — платформа для візуалізації. Вона підключається до Prometheus, робить запити та будує красиві графіки на інтерактивних панелях.

---

## Крок 1. Структура проекту та конфігурація Prometheus

Створимо окрему робочу директорію для моніторингу на сервері:

```bash
mkdir -p /opt/monitoring/prometheus
cd /opt/monitoring
```

Створимо файл конфігурації для Prometheus, який вкаже йому, звідки збирати метрики.

Створіть файл `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s # Частота збору метрик

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

---

## Крок 2. Налаштування Docker Compose

Створимо файл `docker-compose.yml` в папці `/opt/monitoring`, який запустить усі три компоненти в одній мережі:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped
    ports:
      - "9090:9090"

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
    expose:
      - "9100"

  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=super_secret_pass # Вкажіть свій надійний пароль
    restart: unless-stopped
    ports:
      - "3000:3000"

volumes:
  prometheus_data:
  grafana_data:
```

> 💡 **Зверніть увагу:** ми не прописували `ports` для `node-exporter`, а використали `expose`. Оскільки Prometheus та Node Exporter працюють в одній Docker-мережі, Prometheus може забирати метрики напряму без відкриття порту `9100` у зовнішній інтернет, що безпечніше.

Запускаємо весь стек:

```bash
docker compose up -d
```

---

## Крок 3. Налаштування джерела даних (Data Source) у Grafana

1. Відкрийте у браузері адресу `http://<IP-вашого-сервера>:3000`.
2. Увійдіть за допомогою логіна `admin` та пароля, який ви вказали у змінній `GF_SECURITY_ADMIN_PASSWORD` (за замовчуванням це `admin`, якщо змінна не була задана).
3. Перейдіть до меню: **Connections** -> **Data sources** -> натисніть кнопку **Add data source**.
4. Оберіть **Prometheus**.
5. У полі **Connection** -> **URL** вкажіть внутрішню Docker-адресу контейнера Prometheus: `http://prometheus:9090`.
6. Прокрутіть сторінку до самого низу та натисніть кнопку **Save & test**. Ви маєте побачити зелене повідомлення: *«Data source is working»*.

---

## Крок 4. Імпорт професійного дашборду

Малювати графіки вручну з нуля — довга справа. Спільнота Grafana має тисячі готових дашбордів. Один із найкращих та найдетальніших для Node Exporter має ID **`1860`** (*Node Exporter Full*).

Щоб імпортувати його:
1. У лівому боковому меню натисніть на іконку **Dashboards**.
2. Натисніть кнопку **New** (зверху справа) -> **Import**.
3. У полі **Find and import dashboards via grafana.com** введіть число `1860` та натисніть **Load**.
4. Унизу сторінки в спадному списку для Prometheus виберіть ваше нещодавно створене джерело даних.
5. Натисніть **Import**.

Перед вами відкриється неймовірно красивий та інформативний дашборд: завантаження ядер процесора, оперативна пам'ять, дискові IOPS, вільне місце на розділах, мережева швидкість та температура!

---

## Підсумок

Тепер у вас є надійна промислова система моніторингу. Найбільший плюс цієї зв'язки — масштабованість. Якщо ви піднімете новий сервер, вам достатньо запустити на ньому лише легкий контейнер `node-exporter`, а Prometheus на основному сервері сам забиратиме з нього дані. Усі ваші сервери будуть як на долоні в одній панелі Grafana!
