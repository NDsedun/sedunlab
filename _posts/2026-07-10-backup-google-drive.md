---
title: "Бекап VPS на Google Drive через rclone"
date: 2026-07-10 15:00:00 +0200
categories: [homelab, devops]
tags: [backup, rclone, google-drive, linux, vps]
---

Попередня стаття описувала бекап на домашній Ubuntu server через rsync.
Але що якщо вдома відключать світло або інтернет? Потрібне друге місце —
Google Drive ідеально підходить, безкоштовно до 15GB.

## Встановлення rclone

```bash
curl https://rclone.org/install.sh | bash
rclone --version
```

## Підключення Google Drive

```bash
rclone config
```

Відповідаємо:
- `n` — новий remote
- Name: `gdrive`
- Storage: `drive` (Google Drive)
- client_id, client_secret: порожньо (Enter)
- scope: `1` (повний доступ)
- Use auto config: `n` (ми на сервері без браузера)

Rclone виведе команду — виконуємо її на Windows:

```powershell
.\rclone.exe authorize "drive" "токен_з_сервера"
```

Відкриється браузер — авторизуємось в Google. Копіюємо
отриманий токен назад на сервер.

## Перевірка

```bash
rclone lsd gdrive:
```

Має показати папки з Google Drive.

## Додаємо в скрипт бекапу

Відкриваємо `/root/backup.sh` і додаємо перед останнім `echo`:

```bash
# === Google Drive — раз на тиждень у неділю ===
if [ $(date +%u) -eq 7 ]; then
  echo "Копіюємо на Google Drive..." >> $LOG
  rclone sync /home/nd/backups/vps/ gdrive:backups/vps/ 2>> $LOG
  echo "Google Drive бекап завершено" >> $LOG
fi
```

Синхронізація запускається тільки по неділях — щоб не витрачати
квоту щодня і не уповільнювати щоденний бекап.

## Корисні команди

```bash
# Перевірити що є на Drive
rclone lsd gdrive:backups/

# Розмір бекапів
rclone size gdrive:backups/vps/

# Синхронізувати вручну
rclone sync ~/backups/vps/ gdrive:backups/vps/

# Видалити тестову папку
rclone purge gdrive:backups/test/
```

## Підсумок — три рівні бекапу

| Рівень | Куди | Як часто |
|--------|------|----------|
| 1 | Ubuntu server (локалка) | Щодня о 3:00 |
| 2 | Google Drive | Щонеділі |
| 3 | Git репозиторій | При кожному push |

Три копії в різних місцях — якщо щось піде не так,
завжди є звідки відновити.