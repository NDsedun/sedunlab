---
title: "Як я підняв Jekyll блог на VPS з AlmaLinux 9"
date: 2025-01-15 12:00:00 +0200
categories: [homelab, devops]
tags: [vps, jekyll, almalinux, cyberpanel, ruby, chirpy]
---

Вирішив завести технічний блог. Домен є, VPS є, CyberPanel крутиться —
залишилось обрати платформу. Зупинився на Jekyll: статичний генератор,
пишеш у Markdown, деплоїш просто файли. Жодних баз даних, жодного PHP.

## Що знадобилось

- VPS з AlmaLinux 9
- CyberPanel (вже був встановлений)
- Домен sedunlab.com
- Трохи терпіння з rbenv 😄

## Встановлення Ruby через rbenv

Системний Ruby в AlmaLinux 9 застарілий, тому ставимо через rbenv —
менеджер версій Ruby.

```bash
# Клонуємо rbenv
git clone https://github.com/rbenv/rbenv.git ~/.rbenv

# Додаємо в PATH
echo 'export PATH="/root/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init - bash)"' >> ~/.bashrc
source ~/.bashrc

# Встановлюємо ruby-build
git clone https://github.com/rbenv/ruby-build.git \
  ~/.rbenv/plugins/ruby-build

# Залежності для збірки
dnf install -y openssl-devel readline-devel zlib-devel libyaml-devel

# Встановлюємо Ruby 3.3.0
rbenv install 3.3.0
rbenv global 3.3.0
```

Збірка Ruby зайняла хвилин 7. Після цього:

```bash
ruby -v
# ruby 3.3.0 (2023-12-25 revision 5124f9ac75) [x86_64-linux]

## Встановлення Jekyll

gem install jekyll bundler
jekyll -v
# jekyll 4.4.1
```

## Тема Chirpy

Обрав тему Chirpy — виглядає сучасно, має темний режим, пошук,
теги і категорії. Ідеально для tech-блогу.

```bash
git clone https://github.com/cotes2020/chirpy-starter.git ~/sedunlab
cd ~/sedunlab
bundle install
```

## Налаштування _config.yml

Відкриваємо головний конфіг і заповнюємо:

```yaml
title: SedunLab
tagline: Лабораторія з IT у руках
description: DevOps, Homelab та туторіали українською
url: "https://sedunlab.com"
timezone: Europe/Kyiv
lang: uk
```

## Деплой через CyberPanel

Jekyll генерує статичні файли — просто копіюємо їх в публічну папку сайту:

```bash
bundle exec jekyll build
cp -r _site/* /home/sedunlab.com/public_html/
```

В CyberPanel одним кліком видаємо SSL сертифікат — і сайт доступний
по HTTPS.

## Що далі

Наступний крок — автодеплой через Git hook, щоб після кожного
`git push` сайт оновлювався автоматично. Але це вже тема окремої статті.


