---
title: "Uptime Kuma: моніторинг доступності сайтів з повідомленнями в Telegram"
date: 2026-07-06 12:00:00 +0200
categories: [homelab, devops]
tags: [uptime-kuma, monitoring, docker, telegram, vps]
image:
  path: /assets/img/posts/uptime-kuma-dashboard.png
  alt: Uptime Kuma дашборд — моніторинг доступності сайтів
---

Netdata показує що відбувається всередині сервера. Але він не скаже
тобі якщо сайт впав і недоступний ззовні. Для цього є Uptime Kuma —
безкоштовний self-hosted моніторинг доступності з красивим інтерфейсом
і повідомленнями куди завгодно.

## Що таке Uptime Kuma

Uptime Kuma перевіряє твої сайти кожну хвилину і одразу сповіщає
якщо щось пішло не так. Підтримує HTTP/HTTPS, TCP, DNS, ping та інші
протоколи. Інтерфейс локалізований українською — приємний бонус.

Головне — він повністю self-hosted. Ніяких платних планів,
ніяких обмежень на кількість моніторів.

## Встановлення через Docker

Спочатку встановлюємо Docker на AlmaLinux 9:

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
```

Запускаємо Uptime Kuma одним рядком:

```bash
docker run -d \
  --name uptime-kuma \
  --restart unless-stopped \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1
```

Прапорець `--restart unless-stopped` означає що контейнер
автоматично запуститься після перезавантаження сервера.
Том `uptime-kuma:/app/data` зберігає всі дані постійно —
навіть якщо контейнер перестворити, все збережеться.

Перевіряємо що запустився:

```bash
docker ps
```

## Доступ до інтерфейсу

Як і з Netdata — не відкриваємо порт назовні, використовуємо
SSH-тунель:

```powershell
# На Windows
ssh -L 3001:localhost:3001 root@sedunlab.com
```

Відкриваємо `http://localhost:3001` — при першому вході
створюємо акаунт адміністратора.

## Додаємо монітори

Натискаємо **"Новий монітор"** і заповнюємо:

- **Тип монітора:** HTTP(s)
- **Назва:** SedunLab
- **URL:** `https://sedunlab.com`
- **Інтервал перевірки:** 60 секунд

Зберігаємо і повторюємо для інших сайтів. Uptime Kuma одразу
починає перевірки і показує статус, час відповіді і uptime
за 24 години та 30 днів.

Мої результати після першого запуску:
- `sedunlab.com` — відповідь 19ms, середня 38ms, uptime 100%
- `madtech.work` — теж зелений

SSL сертифікат спливає через 30 днів — Uptime Kuma попередить
заздалегідь. Дуже зручно.

## Повідомлення в Telegram

Це найкорисніша функція — знаєш про проблему раніше ніж
помітять відвідувачі.

**Крок 1 — створюємо бота:**

Знаходимо в Telegram `@BotFather`, пишемо `/newbot`,
вводимо назву і username. Отримуємо токен.

**Крок 2 — отримуємо Chat ID:**

Знаходимо `@userinfobot`, пишемо `/start` — він покаже
наш Chat ID.

**Крок 3 — підключаємо в Uptime Kuma:**

Settings → Notifications → Add Notification:
- Тип: **Telegram**
- Bot Token: токен від BotFather
- Chat ID: наш ID

Натискаємо **Test** — в Telegram одразу приходить тестове
повідомлення. Зберігаємо і призначаємо нотифікацію до моніторів.

Тепер якщо `sedunlab.com` впаде — телефон повідомить протягом
хвилини.

## Корисні команди Docker

```bash
# Статус контейнера
docker ps

# Логи Uptime Kuma
docker logs uptime-kuma

# Перезапустити
docker restart uptime-kuma

# Оновити до нової версії
docker pull louislam/uptime-kuma:1
docker restart uptime-kuma
```

## Підсумок

Два інструменти — два різних рівні моніторингу:

- **Netdata** — що відбувається всередині сервера
- **Uptime Kuma** — чи доступні сайти ззовні

Разом вони дають повну картину. І все це безкоштовно,
self-hosted, без реєстрацій і платних планів.