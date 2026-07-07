---
title: "Конвертація SSH ключів: PuTTY ↔ OpenSSH"
date: 2026-07-07 12:00:00 +0200
categories: [homelab, windows]
tags: [ssh, putty, windows, openssh, ключі]
image:
  path: /assets/img/posts/puttygen-export.png
  alt: PuTTYgen — конвертація SSH ключів
---

Якщо ти використовуєш і PuTTY, і нативний SSH у Windows або Linux,
рано чи пізно виникне питання: як перетворити `.ppk` файл у звичайний
OpenSSH ключ і навпаки. Вирішується це через PuTTYgen — графічний
інструмент з пакету PuTTY.

## Що таке PuTTYgen

PuTTYgen — утиліта для роботи з SSH ключами яка входить до складу
пакету PuTTY. Вміє генерувати нові ключі, конвертувати між форматами
і змінювати парольну фразу. Завантажити можна на
[putty.org](https://www.putty.org).

## Формати ключів

- **`.ppk`** — формат PuTTY, використовується в PuTTY і WinSCP
- **`id_rsa` / `id_ed25519`** — формат OpenSSH, використовується
  в PowerShell, Linux терміналі, Git, VS Code Remote SSH

## PuTTY → OpenSSH

Цей варіант потрібен коли є старий `.ppk` ключ і хочеш підключатись
через PowerShell або VS Code Remote SSH без PuTTY.

**Крок 1** — відкриваємо PuTTYgen і завантажуємо `.ppk` файл:
`File → Load private key` або просто двічі клікаємо на `.ppk` файл.

**Крок 2** — експортуємо в OpenSSH формат:
`Conversions → Export OpenSSH key`

![PuTTYgen — експорт в OpenSSH формат](/assets/img/posts/puttygen-export.png)
_Меню Conversions → Export OpenSSH key_

Є два варіанти експорту:
- **Export OpenSSH key** — класичний формат, сумісний з усім
- **Export OpenSSH key (force new format)** — новіший формат,
  рекомендується для Ed25519 ключів

**Крок 3** — зберігаємо файл. Зазвичай кладемо в:

C: \ Users \ твій_юзер\ .ssh \ id_rsa

Якщо ключ захищений паролем — PuTTYgen запитає його перед експортом.

**Крок 4** — встановлюємо правильні права на файл (важливо!):

```powershell
icacls "$env:USERPROFILE\.ssh\id_rsa" /inheritance:r /grant:r "$env:USERNAME`:F"
```

Без цього SSH відмовиться використовувати ключ через надто широкі права.

**Перевірка:**

```powershell
ssh -i $env:USERPROFILE\.ssh\id_rsa root@sedunlab.com
```

## OpenSSH → PuTTY

Цей варіант потрібен коли згенерував ключ на Linux або через
`ssh-keygen` і хочеш використовувати його в PuTTY або WinSCP.

**Крок 1** — відкриваємо PuTTYgen і натискаємо **Load**

**Крок 2** — у діалозі вибору файлу міняємо фільтр на
`All Files (*.*)` — інакше `.ppk` файли не показуватимуться,
а нам потрібен `id_rsa` або `id_ed25519`

**Крок 3** — вибираємо OpenSSH ключ. PuTTYgen покаже повідомлення:

![PuTTYgen — успішний імпорт OpenSSH ключа](/assets/img/posts/puttygen-import.png)
_PuTTYgen повідомляє що ключ успішно імпортовано_

"Successfully imported foreign key" — все добре, натискаємо OK.

**Крок 4** — зберігаємо в форматі PuTTY:
натискаємо **Save private key** → зберігаємо як `.ppk`

Тепер цей `.ppk` можна використовувати в PuTTY і WinSCP.

## Командний рядок (без графіки)

Якщо треба конвертувати без графічного інтерфейсу — наприклад
на сервері — використовуємо утиліту `ssh-keygen`:

```bash
# OpenSSH → PuTTY (потрібен puttygen)
puttygen id_rsa -o mykey.ppk

# PuTTY → OpenSSH
puttygen mykey.ppk -O private-openssh -o id_rsa
```

На AlmaLinux встановлюємо puttygen:

```bash
dnf install -y putty
```

## Підсумок

| Напрямок | Інструмент | Результат |
|----------|-----------|-----------|
| `.ppk` → OpenSSH | PuTTYgen → Conversions → Export OpenSSH key | `id_rsa` |
| OpenSSH → `.ppk` | PuTTYgen → Load → Save private key | `.ppk` |
| Командний рядок | `puttygen` | обидва формати |

Конвертація займає хвилину і позбавляє потреби тримати два
окремих ключі для PuTTY і OpenSSH.