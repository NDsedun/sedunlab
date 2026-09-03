---
title: "Серверні сповіщення прямо в Telegram: налаштовуємо моніторинг бекапів і дисків за 5 хвилин"
date: 2026-09-03 15:00:00 +0300
categories: [homelab, devops]
tags: [telegram, monitoring, bash, backup, automation, homelab]
image:
  path: /assets/img/posts/server-alerts-telegram-cover.webp
  alt: Серверні сповіщення у Telegram для Homelab
alt_lang_url: /posts/server-alerts-telegram-ntfy-docker-en/
---

[🇬🇧 Read this article in English](/posts/server-alerts-telegram-ntfy-docker-en/)

---

Панелі моніторингу на кшталт Grafana чи красивий дашборд Homepage — це чудово. Але у них є один фундаментальний недолік: **вони пасивні**. Ніхто не сидить перед екраном моніторингу о 3-й годині ночі, щоб перевірити, чи успішно пройшов бекап баз даних і чи не переповнився випадково SSD-накопичувач.

Справжній спокій системного адміністратора та власника Homelab починається тоді, коли сервери самі надсилають **миттєві активні сповіщення на смартфон**.

Сьогодні ми розберемо найпростіший, найнадійніший та найпопулярніший спосіб організації пуш-сповіщень: створення власного **Telegram-бота без єдиного рядка складного коду** за допомогою простої утиліти `curl`. Також ми інтегруємо його в нічні бекапи та налаштуємо автоматичний моніторинг вільного місця на дисках.

---

## Крок 1. Створюємо Telegram-бота за 2 хвилини

Для надсилання повідомлень нам потрібні два значення: **Token бота** та ваш особистий **Chat ID**.

### 1. Отримання токена бота
1. Відкрийте Telegram і знайдіть офіційного бота [**@BotFather**](https://t.me/BotFather).
2. Надішліть команду `/newbot`.
3. Вкажіть ім'я бота (наприклад, `My Homelab Alert`) та його юзернейм (має закінчуватися на `bot`, наприклад `sedunlab_alert_bot`).
4. BotFather поверне унікальний **API Token** вигляду `123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ`. Збережіть його!

### 2. Отримання вашого Chat ID
1. Знайдіть у Telegram свого щойно створеного бота і натисніть **Start** (надішліть йому будь-яке повідомлення, наприклад «Привіт»).
2. Тепер перейдіть у пошуку до бота [**@userinfobot**](https://t.me/userinfobot) і натисніть Start.
3. Бот покаже ваш числовий **Id** (наприклад, `987654321`).

---

## Крок 2. Тестовий запит через термінал

Перевіримо працездатність однією командою `curl` прямо з термінала вашого сервера або комп'ютера:

```bash
BOT_TOKEN="ВАШ_ТОКЕН_БОТА"
CHAT_ID="ВАШ_CHAT_ID"
MESSAGE="🚀 Тестове сповіщення з мого домашнього сервера!"

curl -s -X POST "https://api.telegram.org/bot$BOT_TOKEN/sendMessage" \
     -d "chat_id=$CHAT_ID" \
     -d "text=$MESSAGE" \
     -d "parse_mode=HTML"
```

Через частку секунди на ваш смартфон надійде сповіщення!

---

## Крок 3. Створюємо універсальний скрипт сповіщень

Щоб не прописувати токени у кожному скрипті бекапу чи кроні, зробимо єдину системну утиліту `/usr/local/bin/tg-notify`:

```bash
sudo nano /usr/local/bin/tg-notify
```

Вставте такий код:

```bash
#!/bin/bash
# Скрипт надсилання сповіщень у Telegram

BOT_TOKEN="ВАШ_ТОКЕН_БОТА"
CHAT_ID="ВАШ_CHAT_ID"

MESSAGE="$1"

if [ -z "$MESSAGE" ]; then
    echo "Використання: tg-notify 'Текст повідомлення'"
    exit 1
fi

curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
     -d "chat_id=${CHAT_ID}" \
     -d "text=${MESSAGE}" \
     -d "parse_mode=HTML" > /dev/null
```

Зробимо скрипт виконуваним для всієї системи:

```bash
sudo chmod +x /usr/local/bin/tg-notify
```

Тепер із будь-якої точки системи ви можете надіслати повідомлення однією простою командою:
```bash
tg-notify "⚠️ <b>Увага!</b> Сервіс перезавантажено."
```

---

## Практичний приклад 1: Сповіщення про успіх або помилку бекапу

Пам'ятаєте наш скрипт бекапу для **Vaultwarden** чи **Restic**? Додамо до нього розумне інформування з кодами завершення:

```bash
#!/bin/bash
BACKUP_DIR="/opt/vaultwarden/backups"
DATA_DIR="/opt/vaultwarden/data"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

# Створюємо резервну копію SQLite
if sqlite3 "$DATA_DIR/db.sqlite3" ".backup '$BACKUP_DIR/db_$TIMESTAMP.sqlite3'"; then
    SIZE=$(du -sh "$BACKUP_DIR/db_$TIMESTAMP.sqlite3" | cut -f1)
    tg-notify "✅ <b>Vaultwarden Backup:</b> Успішно створено копію розміром <code>$SIZE</code> ($TIMESTAMP)."
else
    tg-notify "❌ <b>Vaultwarden Backup:</b> ПОМИЛКА створення бекапу! Перевірте сервер негайно."
    exit 1
fi
```

Тепер щоранку ви бачитимете приємне повідомлення із зеленим прапорцем і точним розміром бекапу, а в разі непередбачуваної помилки миттєво дізнаєтесь про інцидент.

---

## Практичний приклад 2: Автоматичний контроль вільного місця на диску

Дуже часта причина падіння баз даних чи зависання Docker — це забитий на 100% системний диск через надлишок логів.

Створимо простий сторожовий скрипт `/opt/scripts/check-disk.sh`:

```bash
#!/bin/bash
# Перевірка вільного місця на кореневому диску

THRESHOLD=85
CURRENT=$(df / | grep / | awk '{ print $5}' | sed 's/%//g')

if [ "$CURRENT" -gt "$THRESHOLD" ]; then
    HOSTNAME=$(hostname)
    tg-notify "🚨 <b>Критично мало місця!</b>%0AСервер: <code>$HOSTNAME</code>%0AДиск заповнено на: <b>$CURRENT%</b> (Поріг: $THRESHOLD%)"
fi
```

Зробіть його виконуваним (`chmod +x /opt/scripts/check-disk.sh`) та додайте у планувальник `crontab -e` для перевірки кожні 6 годин:

```text
0 */6 * * * /bin/bash /opt/scripts/check-disk.sh
```

---

## Альтернатива для пуристів: Self-Hosted сервіс Ntfy

Якщо ви принципово не хочете залежати від хмари Telegram і віддаєте перевагу локальним Open Source рішенням, чудовою альтернативою є **ntfy** (написаний на Go):

```yaml
# docker-compose.yml для ntfy
services:
  ntfy:
    image: binwiederhier/ntfy
    container_name: ntfy
    restart: unless-stopped
    ports:
      - "8095:80"
    volumes:
      - ./cache:/var/cache/ntfy
    command: serve
```

Він має безкоштовні клієнти для Android та iOS, підтримує Web Push у браузерах і приймає сповіщення звичайним `curl -d "Повідомлення" http://ntfy.home.myhomelab.org/alerts`.

---

## Висновок

Налаштування оперативних сповіщень займає не більше п'яти хвилин, проте заощаджує години нервів та захищає від раптової втрати важливих даних. Інтегруйте такі мікроповідомлення у свої нічні задачі, скрипти бекапів та моніторинг ресурсів — і ваша домашня лабораторія працюватиме під вашим повним контролем!
