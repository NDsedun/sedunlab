---
title: "Свій сейф паролів: запускаємо Vaultwarden (Bitwarden) в Docker з HTTPS та автобекапами"
date: 2026-08-27 15:40:00 +0300
categories: [homelab, security]
tags: [vaultwarden, bitwarden, docker, security, backup, безпека]
image:
  path: /assets/img/posts/vaultwarden-cover.webp
  alt: Власний менеджер паролів Vaultwarden у Docker
alt_lang_url: /posts/vaultwarden-bitwarden-docker-ssl-backups-en/
---

[🇬🇧 Read this article in English](/posts/vaultwarden-bitwarden-docker-ssl-backups-en/)

---

Паролі — це ключ до нашого цифрового життя. Після масштабних зломів популярних хмарних менеджерів (як-от LastPass) та постійного подорожчання платних підписок, зберігання паролів на власних серверах стало справжнім трендом безпеки.

Найкращим рішенням для цього є **Vaultwarden** — неофіційний легкий бекенд для популярного менеджера **Bitwarden**, написаний на мові Rust. Він повністю безкоштовний, споживає всього 20–30 МБ оперативної пам'яті (на відміну від оригінального Bitwarden, якому потрібно 2–4 ГБ ОЗУ) та сумісний з усіма офіційними мобільними додатками та розширеннями для браузерів Bitwarden.

Сьогодні ми розгорнемо власний сейф паролів у Docker, захистимо його HTTPS-шифруванням та налаштуємо надійний автоматичний бекап бази даних.

---

## Крок 1. Запуск Vaultwarden через Docker Compose

Для зручності ми об'єднаємо запуск у Docker Compose. Створимо окрему папку `/opt/vaultwarden` та файл `docker-compose.yml`:

```yaml
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - WEBSOCKET_ENABLED=true  # Потрібно для миттєвої синхронізації пристроїв
      - SIGNUPS_ALLOWED=true    # Тимчасово дозволяємо реєстрацію (потім вимкнемо!)
    volumes:
      - ./data:/data
    ports:
      - "8088:80"     # Веб-інтерфейс Vaultwarden
      - "3012:3012"   # WebSocket порт
```

Запусти контейнер командою:
```bash
docker-compose up -d
```

Тепер веб-панель доступна за локальною адресою `http://192.168.50.125:8088`. Зайди туди й **одразу створи свій основний акаунт** (вказавши надійну майстер-пароль фразу).

---

## Крок 2. Налаштування HTTPS (Обов'язково!)

> ⚠️ **Важливо:** Офіційні додатки Bitwarden для Chrome, iOS та Android **категорично відмовляються працювати** без шифрованого HTTPS-з'єднання. Веб-криптографія у браузерах вимагає безпечного середовища (Secure Context).

Ми скористаємося нашою зв'язкою **Nginx Proxy Manager** та **AdGuard Home**, яку налаштували у попередніх статтях:

1. **DNS-запис**: В AdGuard Home додаємо перенаправлення домену `vault.home.myhomelab.org` на IP-адресу нашого проксі-сервера.
2. **Nginx Proxy Manager**: Створюємо новий Proxy Host:
   * **Domain**: `vault.home.myhomelab.org`
   * **Scheme**: `http`
   * **Forward Host**: IP-адреса хоста з Vaultwarden (наприклад, твій Tailscale IP або локальний IP).
   * **Forward Port**: `8088`
   * **Websockets Support**: Увімкнути! (важливо для роботи синхронізації пристроїв).
3. **SSL**: Вибираємо наш Let's Encrypt сертифікат `*.home.myhomelab.org` та вмикаємо **Force SSL**.

Тепер твій сейф працює за захищеною адресою `https://vault.home.myhomelab.org` із зеленим замочком. Можеш увійти в додаток на телефоні, вказавши у полі "Адреса сервера" свій новий домен.

---

## Крок 3. Закриваємо двері (Hardening)

Оскільки твій менеджер паролів тепер може бути доступний у локальній мережі (або через VPN), потрібно заблокувати можливість реєстрації для сторонніх людей.

1. Зупини контейнер: `docker-compose down`.
2. Зміни конфіг `docker-compose.yml`, встановивши параметр **`SIGNUPS_ALLOWED=false`**:
   ```yaml
   environment:
     - WEBSOCKET_ENABLED=true
     - SIGNUPS_ALLOWED=false  # Блокуємо реєстрацію нових користувачів
   ```
3. Запусти контейнер знову: `docker-compose up -d`.

Тепер ніхто сторонній не зможе створити акаунт на твоєму сервері.

---

## Крок 4. Налаштування автоматичних бекапів

Vaultwarden за замовчуванням використовує базу даних **SQLite**. Просто копіювати файл `db.sqlite3` «на ходу» небезпечно — якщо в цей момент відбуватиметься запис пароля, копія бази даних виявиться битою (corrupted).

Правильний спосіб — використовувати команду `sqlite3 .backup`.

Створимо простий скрипт бекапу `/opt/vaultwarden/backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/opt/vaultwarden/backups"
DATA_DIR="/opt/vaultwarden/data"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

mkdir -p "$BACKUP_DIR"

# 1. Безпечний бекап бази даних через утиліту sqlite3
sqlite3 "$DATA_DIR/db.sqlite3" ".backup '$BACKUP_DIR/db_$TIMESTAMP.sqlite3'"

# 2. Архівація папки з ключами шифрування та вкладеннями (attachments)
tar -czf "$BACKUP_DIR/attachments_$TIMESTAMP.tar.gz" -C "$DATA_DIR" attachments key.der RSA-key.der

# 3. Видаляємо бекапи, старіші за 7 днів
find "$BACKUP_DIR" -type f -mtime +7 -delete

echo "Backup completed successfully at $TIMESTAMP"
```

Зроби скрипт виконуваним:
```bash
chmod +x /opt/vaultwarden/backup.sh
```

Додамо його в планувальник завдань `cron` для щоденного запуску о 3:00 ночі. Виконай `crontab -e` та додай рядок:
```text
0 3 * * * /bin/bash /opt/vaultwarden/backup.sh >> /var/log/vaultwarden_backup.log 2>&1
```

> 💡 **Порада щодо максимальної безпеки**: Отриману папку `/opt/vaultwarden/backups` вкрай рекомендовано раз на тиждень завантажувати на віддалене хмарне сховище за допомогою утиліти `restic` або `rclone`, про які ми писали раніше.

---

## Висновок

Власний сервер Vaultwarden — це свобода від лімітів платних підписок та впевненість у тому, що твої паролі зберігаються в зашифрованому вигляді на твоєму власному обладнанні. Завдяки роботі в Docker, HTTPS-захисту та надійному скрипту бекапів, твоя особиста кібербезпека виходить на абсолютно новий рівень!
