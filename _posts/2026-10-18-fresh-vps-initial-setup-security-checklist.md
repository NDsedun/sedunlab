---
title: "Чекліст безпеки свіжого Linux VPS: 7 обов'язкових кроків після першого логіну"
date: 2026-10-18 14:00:00 +0300
categories: [vps, security]
tags: [vps, security, linux, ssh, ufw, firewalld, fail2ban, hardening, devops, almalinux, ubuntu]
image:
  path: /assets/img/posts/vps-hardening-cover.jpg
  alt: Базовий чекліст безпеки та налаштування Linux VPS
alt_lang_url: /posts/fresh-vps-initial-setup-security-checklist-en/
---

[🇬🇧 Read this article in English](/posts/fresh-vps-initial-setup-security-checklist-en/)

---

Ви щойно орендували новенький VPS на Hetzner, DigitalOcean, OVH чи Netcup. Провайдер надіслав на пошту IP-адресу та тимчасовий пароль для `root`. Руки сверблять одразу накотити Docker, веб-сервер чи базу даних і пустити трафік.

Але зачекайте. **Сучасний публічний інтернет — це дике поле.**

Як ми наочно переконалися в нашій [статті про Cowrie SSH Honeypot](/posts/cowrie-ssh-honeypot-vps/), автоматизовані сканери та ботнети починають «стукати» у 22-й порт новоствореного сервера вже за **3–5 хвилин** після виділення публічної IP-адреси. Якщо залишити сервер у стані «за замовчуванням», його злам або перетворення на вузол ботнету — це лише питання часу.

Нижче — вичерпний покроковий чекліст із 7 обов'язкових дій, який перетворить ваш свіжий сервер на неприступну фортецю. 

> **Примітка щодо дистрибутивів:** Усі команди нижче протестовано та наведено окремо як для сімейства **Debian / Ubuntu**, так і для корпоративних RHEL-сумісних систем (**AlmaLinux 9 / Rocky Linux 9 / RHEL**).

![Термінал перевірки безпеки Linux VPS](/assets/img/posts/vps-checklist-terminal.jpg)
_Мал. 1. Результат виконання базового аудиту безпеки: криптографія Ed25519, закритий прямий root, активний фаєрвол та оптимізований Swap_

---

## Чекліст коротко (Quick Reference)

- [ ] **Крок 1**: Повне оновлення репозиторіїв та налаштування автооновлень безпеки (`unattended-upgrades` або `dnf-automatic`).
- [ ] **Крок 2**: Створення непривілейованого користувача та надання прав (`sudo` або `wheel`).
- [ ] **Крок 3**: Захист SSH: сучасні ключі Ed25519, повне вимкнення паролів і заборона входу root.
- [ ] **Крок 4**: Налаштування фаєрволу (`ufw` або `firewalld`) та нейтралізація обходу через Docker.
- [ ] **Крок 5**: Захист від брутфорс-атак через Fail2ban.
- [ ] **Крок 6**: Системна гігієна: часовий пояс, NTP та налаштування Swap з `swappiness=10`.
- [ ] **Крок 7**: Фінальний аудит відкритих портів та мережевих служб.

---

## Крок 1. Первинне оновлення та автооновлення безпеки

Базові образи операційних систем у дата-центрах часто збираються місяцями раніше, тому в них гарантовано є незакриті вразливості нульового дня (0-day).

### Для Ubuntu / Debian:
Оновлюємо списки пакетів та систему:
```bash
sudo apt update && sudo apt upgrade -y
```

Встановлюємо утиліту автоматичного встановлення security-патчів:
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```
Цей пакет налаштовує фонову службу, яка щодня автоматично встановлює лише критичні security-виправлення без ламання конфігурацій.

### Для AlmaLinux / Rocky Linux:
Оновлюємо систему через DNF:
```bash
sudo dnf update -y
```

Встановлюємо та активуємо `dnf-automatic`:
```bash
sudo dnf install dnf-automatic -y

# Налаштовуємо автозастосування оновлень безпеки:
sudo sed -i 's/upgrade_type = default/upgrade_type = security/' /etc/dnf/automatic.conf
sudo sed -i 's/apply_updates = no/apply_updates = yes/' /etc/dnf/automatic.conf

# Вмикаємо таймер:
sudo systemctl enable --now dnf-automatic.timer
```

---

## Крок 2. Створення sudo-користувача та відмова від root

Робота під обліковим записом `root` — це небезпечна звичка. Будь-яка друкарська помилка в команді `rm` або запущений скрипт із зашитим бекдором одразу отримує необмежений контроль над ядром.

### Для Ubuntu / Debian:
Створюємо нового системного користувача (наприклад, `adminuser`) і додаємо його в групу `sudo`:
```bash
adduser adminuser
usermod -aG sudo adminuser
```

### Для AlmaLinux / Rocky Linux:
У сімействі RHEL користувача створюють через `adduser`, пароль задають через `passwd`, а адміністративна група називається **`wheel`** (а не `sudo`):
```bash
adduser adminuser
passwd adminuser
usermod -aG wheel adminuser
```

**Перевірка для обох систем:**  
Перемкніться на нового користувача командою `su - adminuser` і перевірте команду `sudo whoami`. Результатом має бути слово `root`.

---

## Крок 3. Безпека SSH: ключі Ed25519 та заборона паролів

Більшість атак на сервери — це банальний брутфорс (словниковий перебір паролів). Якщо ви вимкнете автентифікацію за паролем і залишите лише криптографічні ключі, брутфорс втрачає будь-який сенс.

### 1. Згенеруйте ключ Ed25519 на своєму локальному комп'ютері
Забудьте про застарілий RSA 2048/4096. Сьогодні золотим стандартом є швидкий і криптографічно стійкий алгоритм **Ed25519**:

```bash
# Виконується на вашому робочому комп'ютері (macOS / Linux / Windows PowerShell):
ssh-keygen -t ed25519 -C "adminuser@sedunlab"
```

### 2. Скопіюйте публічний ключ на VPS
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub adminuser@IP_ВАШОГО_СЕРВЕРА
```

### 3. Налаштуйте конфігурацію SSH-демона
Замість редагування монолітного файлу `/etc/ssh/sshd_config`, створимо ізольований файл перевизначень:

```bash
sudo nano /etc/ssh/sshd_config.d/hardening.conf
```

Вставте такі директиви:

```text
# Забороняємо прямий вхід під root
PermitRootLogin no

# Повністю вимикаємо автентифікацію за паролями
PasswordAuthentication no
KbdInteractiveAuthentication no

# Дозволяємо лише вхід за публічними ключами
PubkeyAuthentication yes

# Безпека сесій
MaxAuthTries 3
X11Forwarding no
AllowTcpForwarding yes
```

Перевірте синтаксис конфігурації перед перезапуском, щоб не заблокувати себе:

```bash
sudo sshd -t
```

Якщо помилок немає, перезапускаємо службу SSH:

* На **Ubuntu / Debian**:
  ```bash
  sudo systemctl restart ssh
  ```
* На **AlmaLinux / Rocky Linux**:
  ```bash
  sudo systemctl restart sshd
  ```

> **Підказка для AlmaLinux / Rocky Linux (SELinux):**  
> Якщо ви вирішите змінити стандартний порт SSH (наприклад, на `2222`), на Alma/Rocky обов'язково повідомте про це SELinux, інакше служба не запуститься:  
> `sudo semanage port -a -t ssh_port_t -p tcp 2222`

> **КРИТИЧНО ВАЖЛИВО:** Ніколи не закривайте поточне термінальне вікно! Відкрийте нове вікно на комп'ютері та перевірте підключення: `ssh adminuser@IP_ВАШОГО_СЕРВЕРА`. Переконайтеся, що вхід за ключем працює без введення пароля.

---

## Крок 4. Налаштування фаєрволу: UFW vs Firewalld

Фаєрвол має пропускати лише необхідний трафік, блокуючи всі інші порти за замовчуванням.

### Варіант А: Для Ubuntu / Debian (через UFW)

```bash
# Забороняємо всі вхідні з'єднання за замовчуванням
sudo ufw default deny incoming

# Дозволяємо всі вихідні з'єднання
sudo ufw default allow outgoing

# Відкриваємо SSH (порт 22 або ваш кастомний порт)
sudo ufw allow 22/tcp comment 'SSH'

# Відкриваємо веб-порти
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Активуємо фаєрвол
sudo ufw enable
```
Перевірити статус: `sudo ufw status verbose`.

---

### Варіант Б: Для AlmaLinux / Rocky Linux (через Firewalld)

В AlmaLinux та Rocky Linux стандартом є **firewalld**:

```bash
# Переконуємось, що firewalld увімкнений та запущений
sudo systemctl enable --now firewalld

# Дозволяємо базові сервіси
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Застосовуємо правила
sudo firewall-cmd --reload

# Перевіряємо активну зону
sudo firewall-cmd --list-all
```

---

### ⚠️ Головний підводний камінь для всіх систем: Docker обходить фаєрвол!
Якщо ви розгортаєте додатки через Docker (наприклад, публікуєте порт `-p 8080:8080`), **Docker напряму модифікує ланцюжки iptables PREROUTING в обхід UFW та Firewalld!**

В результаті опублікований порт стає доступним усьому інтернету, навіть якщо фаєрвол каже зворотне.

**Як це виправити:**
1. Прив'язуйте порти контейнерів виключно до локального інтерфейсу (`127.0.0.1`), якщо сервіс працює за Reverse Proxy:
   ```yaml
   ports:
     - "127.0.0.1:8080:8080"
   ```
2. Або для систем з UFW використовуйте скрипт [ufw-docker](https://github.com/chaifeng/ufw-docker), який змушує Docker шанувати правила UFW.

---

## Крок 5. Захист від зловмисників: Fail2ban

Навіть якщо паролі вимкнено, боти продовжують забивати порт SSH спробами підключення. **Fail2ban** аналізує журнал авторизацій і автоматично відправляє IP-адреси нападників у бан.

### Встановлення:

* На **Ubuntu / Debian**:
  ```bash
  sudo apt install fail2ban -y
  ```
* На **AlmaLinux / Rocky Linux**:  
  Fail2ban знаходиться в репозиторії EPEL:
  ```bash
  sudo dnf install epel-release -y
  sudo dnf install fail2ban fail2ban-firewalld -y
  ```

### Налаштування:
Створимо локальний конфіг:

```bash
sudo nano /etc/fail2ban/jail.local
```

Вставте конфігурацію:

```ini
[DEFAULT]
bantime  = 1d
findtime = 10m
maxretry = 3
backend  = systemd

[sshd]
enabled = true
port    = 22
mode    = aggressive
```

Запускаємо та активуємо службу:

```bash
sudo systemctl enable --now fail2ban
```

Перевірити статус забанених ботів:
```bash
sudo fail2ban-client status sshd
```

---

## Крок 6. Системна гігієна: Час, NTP та Swap

Для коректної роботи аналітики логів, SSL-сертифікатів та cron-завдань життєво необхідно мати точний час.

### 1. Налаштуйте правильний часовий пояс (однаково для всіх систем):
```bash
sudo timedatectl set-timezone Europe/Kyiv
sudo timedatectl set-ntp on
```

### 2. Створення Swap-файлу (Файл підкачки)
Більшість недорогих VPS мають 1–2 ГБ оперативної пам’яті. Під час складання проекту чи раптового сплеску трафіку Linux Out-Of-Memory (OOM) Killer просто «приб'є» вашу базу даних або процес веб-сервера.

Створимо 2 ГБ підкачки на випадок пікових навантажень:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Додаємо у fstab для збереження після перезавантаження:
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 3. Оптимізація пам'яті (`swappiness`)
За замовчуванням Linux скидає дані у Swap занадто рано, що уповільнює SSD. Налаштуємо систему скидати дані у Swap лише тоді, коли вільної RAM залишається менше 10%:

```bash
sudo nano /etc/sysctl.d/99-custom.conf
```

Додайте параметри:

```ini
vm.swappiness = 10
vm.vfs_cache_pressure = 50
```

Застосуйте зміни: `sudo sysctl --system`.

---

## Крок 7. Фінальний аудит: що слухає мережу?

Перед тим як оголосити сервер готовим до роботи, перевірте відкриті порти:

```bash
sudo ss -tulpn
```

У виводі не повинно бути жодних зайвих сервісів, які слухають інтерфейс `0.0.0.0` або `:::`, окрім вашого SSH-демона та налаштованого фаєрволу.

---

## Підсумок: Захищений сервер за 5 хвилин

Виконавши ці 7 кроків, незалежно від обраного дистрибутива (Ubuntu чи Alma/Rocky), ви отримуєте:
1. **Імунітет проти брутфорсу**: паролі вимкнено, працює тільки ключ Ed25519.
2. **Захищений фаєрвол**: відкриті тільки потрібні порти (UFW або Firewalld), а Docker не має неконтрольованого доступу до мережі.
3. **Стабільність ресурсів**: налаштований Swap вбереже від падінь при браку RAM.
4. **Автоматичні патчі**: ядро та вразливості системи оновлюються автоматично.

Тепер ваш сервер на 100% готовий до розгортання [Beszel](/posts/beszel-lightweight-docker-monitoring-vps/) для моніторингу або [Coolify](/posts/coolify-selfhosted-heroku-vercel-vps/) для керування додатками!
