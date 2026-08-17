---
title: "Налаштовуємо Homepage Dashboard: інтеграція Docker-контейнерів та моніторингу TrueNAS SCALE"
date: 2026-08-13 14:20:00 +0300
categories: [homelab, software]
tags: [dashboard, homepage, docker, truenas, моніторинг]
image:
  path: /assets/img/posts/homepage-setup-cover.webp
  alt: Дашборд Homepage у домашній лабораторії
alt_lang_url: /posts/homepage-dashboard-truenas-docker-setup-en/
---

[🇬🇧 Read this article in English](/posts/homepage-dashboard-truenas-docker-setup-en/)

---

У попередніх статтях ми розгорнули повноцінну локальну інфраструктуру, навели лад із доменами та захистили трафік за допомогою SSL. Настав час об'єднати всі наші сервіси (Grafana, AdGuard, Plex, Transmission, UrBackup) в одну красиву, сучасну та інформативну стартову сторінку.

Сьогодні ми розберемо налаштування популярного дашборду **Homepage** (від *gethomepage.dev*), підключимо до нього статистику Docker-контейнерів з віддаленої Ubuntu VM та налаштуємо моніторинг заліза сервера **TrueNAS SCALE** через API.

## Крок 1. Встановлення Homepage на TrueNAS SCALE

Для запуску дашборду ми використаємо готовий офіційний додаток із каталогу TrueNAS SCALE:

1. Перейдіть у розділ **Apps** -> **Discover Apps**.
2. Введіть у пошуку **Homepage** та натисніть **Install**.
3. **Налаштування збереження (Важливо):** За замовчуванням система пропонує тимчасове сховище (`ixVolume`). Щоб конфігураційні файли не зникали при оновленні чи перезапуску додатку, у розділі **Storage** виберіть тип сховища **Host Path** та вкажіть шлях до заздалегідь створеної папки на вашому ZFS-пулі (наприклад, `/mnt/Mega/configs/homepage`).
4. **Права доступу (Permissions):** Переконайтеся, що власником або користувачем із правами запису на цю папку є системний користувач **`apps`** (UID/GID `568` у TrueNAS SCALE). Якщо папка належатиме `root` без прав для `apps`, контейнер не зможе записати дефолтні конфігураційні файли та впаде при запуску.
5. Збережіть та запустіть додаток.

---

## Крок 2. Підключаємо статус Docker-контейнерів

Homepage вміє показувати навантаження процесора/пам'яті та стан роботи (`RUNNING` / `STOPPED`) для будь-якої плитки. Оскільки наш дашборд крутиться на TrueNAS SCALE, а контейнери запущені на окремій віртуальній машині Ubuntu, нам потрібно безпечно зв'язати їх.

Для цього на Ubuntu VM ми запускаємо легкий контейнер-проксі для Docker-сокету: **`docker-socket-proxy`**:

```yaml
# Додаємо у docker-compose.yml на Ubuntu VM:
services:
  docker-proxy:
    image: tecnativa/docker-socket-proxy:latest
    container_name: docker-proxy
    restart: unless-stopped
    environment:
      - CONTAINERS=1 # Дозволяємо читати дані контейнерів
      - POST=0       # Забороняємо будь-які запити на зміну (безпека!)
    ports:
      - "2375:2375"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

### Налаштування `docker.yaml` у Homepage:
Тепер у папці конфігів Homepage відкриваємо файл `docker.yaml` та прописуємо нашу віртуальну машину (для надійності використовуємо її **Tailscale IP**, щоб обійти обмеження ізоляції KVM):

```yaml
ubuntu-docker:
  host: 100.x.y.w # Tailscale IP твого Ubuntu-сервера
  port: 2375      # Порт докер-проксі
```

Тепер у файлі `services.yaml` ми можемо додати параметри `server` та `container` до будь-якої плитки:
```yaml
    - Grafana:
        icon: grafana.png
        href: https://grafana.home.myhomelab.org
        server: ubuntu-docker
        container: grafana
```

---

## Крок 3. Моніторинг сервера TrueNAS SCALE через API

Щоб виводити температуру процесора, завантаження оперативної пам'яті та вільне місце на ZFS-пулах нашого головного сховища TrueNAS, ми використаємо вбудований віджет.

### Отримання API Key в TrueNAS:
1. Зайдіть у веб-панель TrueNAS SCALE.
2. Клікніть на свій профіль користувача у правому верхньому кутку та виберіть **API Keys**.
3. Згенеруйте новий ключ (наприклад, `homepage-key`) та збережіть його.
![Генерація API Key в інтерфейсі TrueNAS SCALE](/assets/img/posts/truenas-api.png)
_Мал. 1. Генерація API-ключа для доступу Homepage_

### Налаштування `services.yaml`:
Тепер пропишемо віджет TrueNAS у конфігураційний файл Homepage:

```yaml
- Залізо та Сховища:
    - TrueNAS:
        icon: truenas.png
        href: https://truenas.home.myhomelab.org
        widget:
          type: truenas
          url: http://100.x.y.z:4434 # Tailscale IP твого TrueNAS та веб-порт
          key: ТВІЙ_ЗГЕНЕРОВАНИЙ_API_KEY
          enablealerts: true
```

---

## Крок 4. Налаштування віджетів та кастомного стилю

За замовчуванням Homepage вирівнює системні віджети (годинник, погоду) горизонтально в один рядок. Якщо ви хочете вирівняти їх по вертикалі один під одним:

1. Відкрийте файл `custom.css` в папці конфігурацій Homepage.
2. Додайте туди такий стиль:
   ```css
   .header-widgets, div[class*="HeaderWidgets"] {
     flex-direction: column;
     align-items: flex-start;
     gap: 0.5rem;
   }
   ```
![Готовий та налаштований дашборд Homepage у домашній мережі](/assets/img/posts/homepage-monitoring.png)
_Мал. 2. Готовий робочий дашборд Homepage із моніторингом контейнерів та сервісів_

## Висновок

Дашборд Homepage — це ідеальне обличчя для домашньої лабораторії. Завдяки розумній інтеграції з Docker та системними API, він перетворюється з простого набору посилань на потужний пульт моніторингу та керування домашньою інфраструктурою.
