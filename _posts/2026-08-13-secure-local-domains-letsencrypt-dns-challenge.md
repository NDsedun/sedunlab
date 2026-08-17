---
title: "Справжній HTTPS у локальній мережі: Let's Encrypt та DNS-валідація через GoDaddy API"
date: 2026-08-13 14:15:00 +0300
categories: [homelab, security]
tags: [ssl, letsencrypt, godaddy, reverse-proxy, nginx, безпека]
image:
  path: /assets/img/posts/godaddy-ssl-cover.webp
  alt: SSL-сертифікати для локальної мережі
alt_lang_url: /posts/secure-local-domains-letsencrypt-dns-challenge-en/
---

[🇬🇧 Read this article in English](/posts/secure-local-domains-letsencrypt-dns-challenge-en/)

---

Ми вже навчилися давати красиві імена нашим локальним сервісам за допомогою AdGuard Home та Nginx Proxy Manager. Проте відкриття адрес типу `http://grafana.home` все ще показує в адресному рядку неприємне «Не захищено».

Для безпечної передачі паролів і даних всередині домашньої мережі потрібен **HTTPS із зеленим замочком**. 

Оскільки Let's Encrypt не видає сертифікати на вигадані локальні зони (типу `.home`), ми використаємо наш реальний публічний домен — наприклад, **`home.myhomelab.org`** — та випустимо для нього безкоштовний сертифікат за допомогою **DNS-валідації через API реєстратора GoDaddy**.

---

## Чому саме DNS-валідація (DNS-01)?

Зазвичай Let's Encrypt перевіряє право власності на домен (HTTP-01 challenge), намагаючись зайти на ваш сервер з інтернету. Оскільки наші сервери знаходяться за NAT у локальній мережі, Let's Encrypt не зможе підключитися до них ззовні.

При **DNS-валідації** перевірка відбувається інакше:
1. Nginx Proxy Manager (NPM) робить запит до Let's Encrypt.
2. Let's Encrypt просить створити спеціальний TXT-запис у DNS вашого домену.
3. NPM через офіційне API реєстратора (в нашому випадку GoDaddy) автоматично створює цей запис.
4. Let's Encrypt перевіряє його наявність у глобальному DNS і видає **Wildcard-сертифікат (`*.home.myhomelab.org`)**.
5. Запис автоматично видаляється, а сертифікат безкоштовно перевипускається кожні 3 місяці.

---

## Крок 1. Локальний редірект в AdGuard Home

Нам потрібно зробити так, щоб для домашніх пристроїв адреси типу `*.home.myhomelab.org` вели відразу на реверс-проксі:
1. Зайдіть в AdGuard Home -> **Filters** -> **DNS Rewrites**.
2. Створіть правило:
   * **Domain**: `*.home.myhomelab.org`
   * **IP Address**: IP вашого проксі-сервера (наприклад, `192.168.50.125`).

---

## Крок 2. Отримання API-ключів у GoDaddy

Щоб автоматизувати створення записів, отримаємо ключі доступу:
1. Перейдіть на портал розробників: [**developer.godaddy.com/keys**](https://developer.godaddy.com/keys).
2. Натисніть **Create New API Key**.
3. Назвіть його (наприклад, `NPM-SSL`) та виберіть тип середовища: **Production**.
4. Збережіть отримані **Key** та **Secret** (секрет показується один раз!).
![Створення ключів API у кабінеті розробника GoDaddy](/assets/img/posts/godaddy-key.png)
_Мал. 1. Створення ключів API у кабінеті розробника GoDaddy_

---

## Крок 3. Замовлення сертифіката в NPM

1. Відкрийте Nginx Proxy Manager -> **SSL Certificates** -> **Add SSL Certificate** -> **Let's Encrypt**.
2. Введіть домен: `*.home.myhomelab.org` (та обов'язково натисніть **Enter**, щоб текст перетворився на тег).
3. Увімкніть **Use a DNS Challenge** та виберіть провайдера **GoDaddy**.
4. Вставте згенеровані API-ключі замість стандартних плейсхолдерів:
   ```ini
   dns_godaddy_api_key = ТВІЙ_GODADDY_KEY
   dns_godaddy_api_secret = ТВІЙ_GODADDY_SECRET
   ```
5. Натисніть **Save**.
![Запит Wildcard-сертифіката Let's Encrypt через DNS Challenge в NPM](/assets/img/posts/add-le-cert.png)
_Мал. 2. Налаштування DNS-валідації Let's Encrypt в NPM_

---

## Крок 4. Прив'язка сертифіката до сервісів

Тепер відкрийте вкладку **Hosts** -> **Proxy Hosts**, відредагуйте свій хост (наприклад, змініть домен на `grafana.home.myhomelab.org`) і на вкладці **SSL** виберіть випущений сертифікат `*.home.myhomelab.org`. Увімкніть **Force SSL** та збережіть.
![Прив'язка Wildcard-сертифіката та активація Force SSL](/assets/img/posts/Proxy-host-ssl.png)
_Мал. 3. Активація SSL-шифрування для Proxy Host_

---

## Крок 5. Швидкий редірект за коротким ім'ям

Оскільки сертифікат виданий тільки під `*.home.myhomelab.org`, при спробі зайти на `https://grafana.home` браузер сваритиметься на безпеку.

Щоб зберегти зручність короткого імені:
1. В NPM перейдіть у **Redirection Hosts** -> **Add Redirection Host**.
2. Введіть **Domain**: `grafana.home`.
3. Вкажіть **Forward Domain**: `https://grafana.home.myhomelab.org`.

Тепер ви вводите супер-коротку адресу `grafana.home`, а проксі автоматично та непомітно перенаправляє вас на повністю захищену `https://grafana.home.myhomelab.org` із зеленим замочком!
