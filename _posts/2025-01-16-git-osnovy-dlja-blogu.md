---
title: "Git для блогу: як я організував автодеплой"
date: 2025-01-16 12:00:00 +0200
categories: [devops, git]
tags: [git, deploy, vps, automation]
---

Копіювати файли вручну після кожної статті — це не DevOps, це страждання.
Тому налаштував автодеплой через Git hook: пишу статтю, роблю `git push` —
сайт оновлюється сам.

## Як це працює

Схема проста:

1. Локально редагую файли
2. `git push vps master` — відправляю на сервер
3. На сервері спрацьовує `post-receive` hook
4. Hook будує Jekyll і копіює файли в `public_html`

## Налаштування на сервері

Спочатку створюємо bare репозиторій — це спеціальний тип репо
без робочої директорії, тільки для зберігання і хуків:

```bash
mkdir -p /var/repo/sedunlab.git
cd /var/repo/sedunlab.git
git init --bare
```

Потім створюємо хук `post-receive`:

```bash
nano /var/repo/sedunlab.git/hooks/post-receive
```

```bash
#!/bin/bash
GIT_REPO=/var/repo/sedunlab.git
WORK_DIR=/root/sedunlab_deploy
PUBLIC=/home/sedunlab.com/public_html

export PATH="/root/.rbenv/bin:$PATH"
eval "$(rbenv init - bash)"
export BUNDLE_SILENCE_ROOT_WARNING=1

echo "--- Деплой починається ---"
rm -rf $WORK_DIR
git clone $GIT_REPO $WORK_DIR 2>/dev/null
cd $WORK_DIR
bundle install --quiet 2>/dev/null
bundle exec jekyll build --quiet
rm -rf $PUBLIC/*
cp -r _site/* $PUBLIC/
echo "--- Деплой завершено! ---"
```

Робимо хук виконуваним:

```bash
chmod +x /var/repo/sedunlab.git/hooks/post-receive
```

## Налаштування на Windows

```powershell
cd C:/Users/ndsed/sedunlab
git init
git remote add vps root@sedunlab.com:/var/repo/sedunlab.git
git add .
git commit -m "initial commit"
git push vps master
```

## Робочий процес щодня

Тепер мій workflow виглядає так:

```powershell
# Написав статтю
git add .
git commit -m "нова стаття про Jekyll"
git push vps master
# Сайт оновився автоматично
```

Один рядок замість ручного копіювання. Саме так і має бути.