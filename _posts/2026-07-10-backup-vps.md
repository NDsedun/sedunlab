---
title: "Автоматичний бекап VPS на домашній сервер через rsync"
date: 2026-07-10 12:00:00 +0200
categories: [homelab, devops]
tags: [backup, rsync, cron, linux, ssh]
image:
  path: /assets/img/posts/backup-art.png
  alt: Структура бекапів на домашньому сервері
---

Моніторинг є, Fail2ban є, сайт працює — але якщо диск на VPS
здохне, все зникне за секунду. Час налаштувати бекап.

У мене є Ubuntu server в домашній локалці з виділеною IP —
ідеальне місце для бекапів з VPS. Використовую `rsync` по SSH
і `cron` для автоматизації.

## Що бекапимо

- `/home/` — всі сайти, включно з WordPress
- `/etc/` — всі конфіги (Fail2ban, SSH, Nginx)
- MySQL — дамп всіх баз даних
- Docker volumes — дані Uptime Kuma та інших контейнерів

## Налаштування SSH ключів

Бекап має працювати без пароля — через SSH ключі.
Генеруємо окремий ключ спеціально для бекапу:

```bash
ssh-keygen -t ed25519 -C "vps-backup" -f ~/.ssh/backup_key -N ""
cat ~/.ssh/backup_key.pub
```

Копіюємо публічний ключ на домашній сервер:

```bash
ssh-copy-id -i ~/.ssh/backup_key.pub -p 22253 nd@home.sedunlab.com
```

Або вручну — на Ubuntu server:

```bash
echo "ssh-ed25519 AAAA...ключ..." >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Перевіряємо підключення:

```bash
ssh -i ~/.ssh/backup_key nd@home.sedunlab.com -p22253 "echo OK"
# OK
```

Створюємо папки для бекапів на Ubuntu:

```bash
ssh -i ~/.ssh/backup_key nd@home.sedunlab.com -p22253 \
  "mkdir -p ~/backups/vps/{home,etc,db,docker}"
```

## Скрипт бекапу

```bash
nano /root/backup.sh
```

```bash
#!/bin/bash

# === Налаштування ===
REMOTE_USER="nd"
REMOTE_HOST="home.sedunlab.com"
REMOTE_PORT="22253"
REMOTE_DIR="/home/nd/backups/vps"
SSH_KEY="/root/.ssh/backup_key"
DATE=$(date +%Y-%m-%d)
LOG="/var/log/backup.log"

echo "=== Бекап розпочато: $(date) ===" >> $LOG

# === База даних ===
echo "Дамп БД..." >> $LOG
mkdir -p /tmp/backup_db
mysqldump --all-databases > /tmp/backup_db/all-databases-$DATE.sql
gzip /tmp/backup_db/all-databases-$DATE.sql

# === Rsync /home/ ===
echo "Rsync /home/..." >> $LOG
rsync -avz --delete \
  -e "ssh -i $SSH_KEY -p $REMOTE_PORT" \
  /home/ \
  $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/home/

# === Rsync /etc/ ===
echo "Rsync /etc/..." >> $LOG
rsync -avz --delete \
  -e "ssh -i $SSH_KEY -p $REMOTE_PORT" \
  /etc/ \
  $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/etc/

# === Бекап БД ===
echo "Копіюємо дамп БД..." >> $LOG
rsync -avz \
  -e "ssh -i $SSH_KEY -p $REMOTE_PORT" \
  /tmp/backup_db/ \
  $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/db/

# === Docker volumes ===
echo "Бекап Docker volumes..." >> $LOG
rsync -avz --delete \
  -e "ssh -i $SSH_KEY -p $REMOTE_PORT" \
  /var/lib/docker/volumes/ \
  $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/docker/

# === Очищення ===
rm -rf /tmp/backup_db

echo "=== Бекап завершено: $(date) ===" >> $LOG
```

Робимо виконуваним і тестуємо:

```bash
chmod +x /root/backup.sh
bash /root/backup.sh
```

## Результат першого запуску

![Лог бекапу](/assets/img/posts/backup-log.png)
_Перший запуск — бекап завершено за 2.5 хвилини_

Перевіряємо що є на Ubuntu:

```bash
ssh -i ~/.ssh/backup_key nd@home.sedunlab.com -p22253 \
  "ls -lh ~/backups/vps/"
```

![Структура бекапів](/assets/img/posts/backup-files.png)
_Чотири папки — home, etc, db, docker_

## Автоматизація через cron

Запускаємо бекап щодня о 3:00 ночі:

```bash
crontab -e
```

Додаємо: