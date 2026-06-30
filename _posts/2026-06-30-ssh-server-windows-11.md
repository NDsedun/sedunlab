---
title: "Як налаштувати SSH-підключення на Windows 11"
date: 2026-06-30 12:00:00 +0200
categories: [homelab, windows]
tags: [ssh, windows, powershell, vps]
---

Поки працював з VPS, доводилось весь час відкривати окремі SSH-клієнти
типу PuTTY. Виявилось, що Windows 11 має вбудований SSH-клієнт і навіть
SSH-сервер — просто треба їх увімкнути.

## SSH-клієнт (підключатись з Windows на сервер)

Найкраща новина — клієнт вже встановлений за замовчуванням.
Просто відкривай PowerShell і підключайся:

```powershell
ssh root@sedunlab.com
```

Якщо це перше підключення, з'явиться запит підтвердити fingerprint
сервера — пишеш `yes` і вводиш пароль.

### Підключення без пароля через SSH-ключ

Генеруємо пару ключів локально:

```powershell
ssh-keygen -t ed25519 -C "adm@windows"
```

Тричі тисни Enter щоб залишити стандартні шляхи і без пароля
на ключ (або постав пароль, якщо хочеш додатковий захист).

Копіюємо публічний ключ на сервер:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@sedunlab.com "cat >> ~/.ssh/authorized_keys"
```

Тепер `ssh root@sedunlab.com` підключає без запиту пароля.

### Зручний доступ через config файл

Щоб не вводити повну адресу щоразу, створюємо файл конфігурації:

```powershell
notepad $env:USERPROFILE\.ssh\config
```

Додаємо:
Host vps
HostName sedunlab.com
User root
Port 22
IdentityFile ~/.ssh/id_ed25519

Тепер підключення стає коротким:

```powershell
ssh vps
```

## SSH-сервер (підключатись ДО Windows ззовні)

Це знадобиться якщо хочеш керувати своїм Windows-комп'ютером
віддалено — наприклад з VPS або іншого пристрою.

### Встановлення OpenSSH Server

Через PowerShell з правами адміністратора:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Запускаємо службу і ставимо автозапуск:

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

### Відкриваємо порт у фаєрволі

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' `
  -Enabled True -Direction Inbound -Protocol TCP `
  -Action Allow -LocalPort 22
```

### Перевірка

З іншого пристрою (наприклад з VPS) пробуємо підключитись:

```bash
ssh adm@IP_твого_Windows
```

Вводиш пароль свого Windows-акаунту — і ти всередині PowerShell
на своєму комп'ютері.

## Корисні дрібниці

Перевірити статус SSH-служби:

```powershell
Get-Service sshd
```

Подивитись лог підключень:

```powershell
Get-WinEvent -LogName "OpenSSH/Operational" -MaxEvents 20
```

Змінити стандартний шелл з cmd на PowerShell для SSH-сесій:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
  -PropertyType String -Force
```

Тепер Windows і VPS говорять однією мовою — SSH в обидва боки,
без сторонніх програм.