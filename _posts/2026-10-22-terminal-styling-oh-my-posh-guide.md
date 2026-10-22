---
title: "Краса та продуктивність у терміналі: повний гайд з Oh My Posh для Windows, Linux та macOS"
date: 2026-10-22 20:00:00 +0300
categories: [devops, tools]
tags: [terminal, oh-my-posh, powershell, zsh, bash, windows-terminal, linux, macos, nerd-fonts, customization]
image:
  path: /assets/img/posts/oh-my-posh-cover.jpg
  alt: Естетичне налаштування термінала з Oh My Posh
alt_lang_url: /posts/terminal-styling-oh-my-posh-guide-en/
---

[🇬🇧 Read this article in English](/posts/terminal-styling-oh-my-posh-guide-en/)

---

Типовий стандартний термінал після чистого встановлення операційної системи зазвичай виглядає як монохромний чорний екран із нудним рядком на кшталт `PS C:\Users\Username>` у Windows або `user@hostname:~$ ` у Linux. У ньому немає контексту гілки Git, інформації про версію Node/Python, часу виконання важкої команди чи банальної індикації статусу помилки.

Роками стандартом краси вважалися зв'язки **Oh My Zsh** та тема **Powerlevel10k** на macOS/Linux, а на Windows люди страждали з важкими скриптами для PowerShell. 

Але часи змінилися. Проєкт **[Oh My Posh](https://ohmyposh.dev/)**, повністю переписаний на швидкій мові **Go**, став абсолютним кросплатформним лідером. Він запускається за частки мілісекунди, однаково бездоганно працює у **Windows Terminal (PowerShell 7 / Git Bash)**, на **Linux (Bash, Zsh, Fish)** та на **macOS**, пропонуючи сотні готових преміальних тем із коробки.

Сьогодні розберемося, як перетворити свій робочий термінал на витвір мистецтва та потужний інструмент продуктивності на будь-якій операційній системі.

![Огляд структури та сегментів Oh My Posh](/assets/img/posts/oh-my-posh-showcase.jpg)
_Мал. 1. Анатомія сучасного промпта Oh My Posh: іконка ОС, поточна папка, статус Git, час виконання та статус коду виходу_

---

## Крок 0: Фундамент естетики — встановлюємо Nerd Fonts

Перш ніж ставити теми та сегменти, запам'ятайте головне правило кастомізації термінала: **без спеціального шрифту замість красивих іконок ви побачите порожні квадратики (▯) або знаки питання**.

Для відображення гліфів (іконки Git , логотипи Linux/Windows/Apple, значки папок, годинники) потрібні **[Nerd Fonts](https://www.nerdfonts.com/)** — звичайні моноширинні шрифти для коду, в які імплантовано тисячі векторних іконок.

### Найпростіший спосіб встановлення: через сам Oh My Posh

Oh My Posh має вбудовану інтерактивну команду завантаження шрифтів:

```bash
oh-my-posh font install
```

Стрілочками на клавіатурі оберіть один із найкращих шрифтів:
* **`Meslo LGS NF`** — класичний, ідеально відкалібрований шрифт для гліфів.
* **`JetBrains Mono NF`** — вибір багатьох розробників, чудові лігатури.
* **`Fira Code NF`** — стильні лігатури стрілок та операторів.

### Обов'язково: активуйте шрифт у налаштуваннях вашого емулятора!

* **Windows Terminal:**  
  Натисніть `Ctrl + ,` (Налаштування) → у лівому меню оберіть **«Параметри за замовчуванням» (Defaults)** → **«Зовнішній вигляд» (Appearance)** → у полі **«Шрифт» (Font face)** виберіть `MesloLGS NF` (або `JetBrainsMono NF`) і збережіть.
* **macOS (Terminal.app або iTerm2):**  
  `Preferences` → `Profiles` → `Text` → `Font` → вкажіть ваш `* Nerd Font`.
* **Linux (GNOME Terminal, Alacritty, Kitty):**  
  У налаштуваннях профілю термінала увімкніть Custom Font та виберіть встановлений Nerd Font.

---

## Крок 1: Встановлення Oh My Posh

Утиліта поширюється як єдиний бінарний файл, тому встановлюється буквально однією командою.

```bash
# 🪟 Windows (через штатний менеджер пакетів winget):
winget install JanDeDobbeleer.OhMyPosh -s winget

# 🍎 macOS (через Homebrew):
brew install jandedobbeleer/oh-my-posh/oh-my-posh

# 🐧 Linux (офіційний скрипт встановлення):
curl -s https://ohmyposh.dev/install.sh | bash -s
```

> [!NOTE]
> На Linux скрипт за замовчуванням кладе бінарник у `/usr/local/bin` або `~/.local/bin`. Переконайтеся, що папка є у вашій змінній `$PATH`. Перевірити встановлення на будь-якій системі: `oh-my-posh --version`.

---

## Крок 2: Активація в конфігурації оболонки (Shell Profile)

Щоб красивий промпт завантажувався щоразу, коли ви відкриваєте вкладку термінала, потрібно додати команду ініціалізації у файл вашого профілю.

### 1. Якщо ви використовуєте PowerShell (Windows / Linux / Mac)

Відкрийте профіль PowerShell у текстовому редакторі:

```powershell
notepad $PROFILE
```
*(Якщо блокнот повідомить, що файлу не існує, створіть його командою `New-Item -Path $PROFILE -Type File -Force` і відкрийте знову).*

Вставте в самий кінець файлу наступний рядок:

```powershell
oh-my-posh init pwsh | Invoke-Expression
```

Збережіть файл. Перезапустіть термінал або виконайте `. $PROFILE`.

> [!TIP]
> Якщо у Windows з'являється помилка *"running scripts is disabled on this system"*, виконайте один раз у консолі від імені користувача:
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

---

### 2. Якщо ви використовуєте Bash (Linux / Git Bash на Windows)

Відкрийте `~/.bashrc`:

```bash
nano ~/.bashrc
```

Додайте в кінець:

```bash
eval "$(oh-my-posh init bash)"
```

Застосуйте зміни: `source ~/.bashrc`.

---

### 3. Якщо ви використовуєте Zsh (macOS за замовчуванням / Linux)

Відкрийте `~/.zshrc`:

```bash
nano ~/.zshrc
```

Додайте в кінець:

```zsh
eval "$(oh-my-posh init zsh)"
```

Застосуйте зміни: `source ~/.zshrc`.

---

## Крок 3: Обираємо та змінюємо теми (Themes)

Разом з Oh My Posh автоматично завантажуються **понад 100 офіційних тем**.

### Як переглянути всі теми наживо?

Виконайте команду інтерактивного попереднього перегляду:

```bash
# У PowerShell:
Get-PoshThemes

# У Bash / Zsh:
oh-my-posh get themes
```

Вам виведеться список усіх доступних тем прямо з вашим шрифтом і кольорами!

### Топ-5 перевірених і найестетичніших тем:

1. **`catppuccin` (або `catppuccin_mocha`):** Ніжні пастельні відтінки, один із найпопулярніших дизайнів сучасності.
2. **`atomic`:** Футуристична багаторядкова тема, що показує рівень заряду батареї, використання пам'яті, Git та час виконання.
3. **`jandedobbeleer`:** Фірмова класика автора Oh My Posh з округлими сегментами Powerline.
4. **`paradox`:** Елегантні стрілочні розділювачі в стилі класичного agnoster.
5. **`tokyonight_storm`:** Глибокі неонові фіолетові та сині кольори в стилі нічного Токіо.

### Як застосувати обрану тему?

Всі теми зберігаються локально. Ми просто передаємо прапорець `--config` під час ініціалізації:

**У Windows PowerShell (`$PROFILE`):**
```powershell
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH/catppuccin.omp.json" | Invoke-Expression
```

**У Linux / macOS (`~/.bashrc` або `~/.zshrc`):**
```bash
eval "$(oh-my-posh init zsh --config ~/.cache/oh-my-posh/themes/catppuccin.omp.json)"
```

> [!TIP]
> **Тема напряму з інтернету:** Якщо не хочете шукати шлях на диску, можна підставити прямий URL з офіційного GitHub:
> ```bash
> --config 'https://raw.githubusercontent.com/JanDeDobbeleer/oh-my-posh/main/themes/atomic.omp.json'
> ```

---

## Крок 4: Тюнінг на максимум — автодоповнення та іконки файлів

Красивий рядок промпта — це лише половина справи. Давайте додамо інтелект і підказки, як у сучасних IDE.

### Для користувачів Windows Terminal & PowerShell:

Встановіть два модулі, які кардинально змінюють досвід роботи:

```powershell
# 1. Іконки для файлів і папок під час виклику 'ls' або 'dir'
Install-Module -Name Terminal-Icons -Repository PSGallery -Scope CurrentUser

# 2. Модуль швидкого автодоповнення
Install-Module -Name PSReadLine -Repository PSGallery -Scope CurrentUser -Force
```

Тепер додайте у ваш `$PROFILE` наступні рядки для активації підказок у стилі Fish-shell (історія команд показується сірим кольором, ви приймаєте її стрілкою вправо `→`):

```powershell
# Імпорт іконок
Import-Module Terminal-Icons

# Розумні підказки з історії команд
Set-PSReadLineOption -PredictionSource HistoryAndPlugin
Set-PSReadLineOption -PredictionViewStyle InlineView
Set-PSReadLineOption -Colors @{ InlinePrediction = '#71717a' }
Set-PSReadLineKeyHandler -Key Tab -Function Complete
```

Тепер ваш `ls` у PowerShell буде виводити логотипи біля `.json`, `.py`, `.js`, папок і архівів, а довгі команди дописуватимуться за одним натисканням стрілочки!

---

### Для користувачів Linux & macOS (Zsh):

Встановіть плагіни `zsh-autosuggestions` та `zsh-syntax-highlighting`:

```bash
# Ubuntu / Debian:
sudo apt install zsh-autosuggestions zsh-syntax-highlighting -y

# macOS (Homebrew):
brew install zsh-autosuggestions zsh-syntax-highlighting
```

Додайте у `~/.zshrc`:
```zsh
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

---

## Крок 5: Розв'язання частих проблем (FAQ)

### 1. Замість іконок відображаються квадратики з хрестиком або `?`
* **Причина:** Ви встановили шрифт у систему, але **не вибрали його в налаштуваннях термінала**.
* **Вирішення:** Обов'язково зайдіть у налаштування Windows Terminal (або iTerm2) і вручну перемкніть `Font face` на `MesloLGS NF` чи `JetBrainsMono NF`.

### 2. Термінал став запускатися на 0.5 секунди повільніше
* **Вирішення:** Додайте кешування у команду ініціалізації:
  ```powershell
  # У PowerShell:
  oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH/catppuccin.omp.json" --manual | Invoke-Expression
  ```
  Також перевірте, чи не вантажаться зайві важкі мережеві модулі у вашому `$PROFILE`.

---

## Шпаргалка швидких команд (Cheat Sheet)

| Дія | Команда |
| :--- | :--- |
| **Оновити Oh My Posh** | `winget upgrade JanDeDobbeleer.OhMyPosh` / `brew upgrade oh-my-posh` |
| **Встановити шрифт** | `oh-my-posh font install` |
| **Переглянути всі теми** | `oh-my-posh get themes` або `Get-PoshThemes` |
| **Редагувати конфіг теми** | `oh-my-posh config export --output ~/.mytheme.omp.json` |
| **Перевірити версію** | `oh-my-posh version` |

---

## Підсумки

Кастомізація термінала — це не просто забавка для естетики. Коли ви бачите перед очима статус поточної гілки Git, кількість незбережених змін, віртуальне середовище Python або Node.js, час виконання збірки та код завершення попередньої утиліти — ви робите менше помилок і працюєте значно продуктивніше.

Oh My Posh дає все це за лічені хвилини без необхідності переналаштовувати операційну систему. Спробуйте кілька тем, оберіть улюблену під свій смак — і консоль перестане бути нудним чорним вікном!
