---
layout: page
title: Мапа повітряних тривог України — Віджет для Windows 11
permalink: /alerts-widget/
sitemap: false
---

<div style="display: flex; align-items: center; gap: 18px; margin-bottom: 1.5rem;">
  <img src="/assets/img/alerts-widget-icon.png" alt="Ukraine Alerts Widget Icon" width="76" height="76" style="border-radius: 16px;" />
  <div>
    <h2 style="margin: 0 0 4px 0; border: none; padding: 0;">Ukraine Alerts Widget</h2>
    <p style="margin: 0; opacity: 0.8;">Легкий настільний віджет живої мапи повітряних тривог України для Windows 10 / 11 • <strong>Created by ND Sedun</strong></p>
  </div>
</div>

Настільний віджет відображає інтерактивну векторну мапу України поверх вікон або безпосередньо на шпалерах робочого столу, оновлюючи дані кожні 15 секунд у реальному часі. Працює виключно через системний трей і не займає місця на панелі завдань.

### Основні можливості

- **Два режими мапи:** перемикання між **областями (27 регіонів)** та окремими **районами (139 районів)** в один клік.
- **Статуси в реальному часі:**
  - 🔴 **Повна тривога** — тривога оголошена в усій області.
  - 🟠 **Часткова тривога** — тривога лише в окремих районах області.
  - 🟢 **Спокійно** — відсутність активних тривог.
- **Деталі при наведенні курсора:** назва області або району, тривалість поточної тривоги та список активних районів.
- **Налаштування вигляду:**
  - Перемикач **«Фон»** — від напівпрозорої картки у стилі Windows 11 до повністю прозорого фону (лише контури мапи на шпалерах).
  - Перемикач **«Назви»** — увімкнення/вимкнення підписів областей.
  - Закріплення поверх усіх вікон (**📌**) та режим **Click-Through** (кліки миші проходять крізь вікно віджета).
- **Системний трей Windows:** іконка в треї динамічно показує поточну кількість регіонів із тривогою. Лівий клік миттєво ховає або показує віджет.

---

### Завантаження

Оберіть потрібний варіант виконуваного файлу (інсталяція не потрібна — просто запустіть завантажений `.exe`):

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(270px, 1fr)); gap: 16px; margin: 1.5rem 0;">

  <div style="border: 1px solid rgba(56, 189, 248, 0.35); border-radius: 12px; padding: 18px; background: rgba(15, 23, 42, 0.35);">
    <h4 style="margin-top: 0;">📦 Автономна версія (Рекомендовано)</h4>
    <p style="font-size: 0.92rem; opacity: 0.85;">Працює на <strong>будь-якому ПК з Windows 10 / 11 (x64)</strong>. Усі бібліотеки та середовище виконання вже вшиті всередину одного файлу.</p>
    <p style="font-size: 0.85rem; opacity: 0.7; margin-bottom: 14px;">Розмір: ~72.4 МБ • Один файл</p>
    <a href="/downloads/UkraineAlertsWidget_Standalone.exe" download style="display: inline-block; background: #0284c7; color: #ffffff; padding: 9px 18px; border-radius: 8px; font-weight: 600; text-decoration: none;">⬇ Завантажити Standalone (.exe)</a>
  </div>

  <div style="border: 1px solid rgba(148, 163, 184, 0.25); border-radius: 12px; padding: 18px; background: rgba(15, 23, 42, 0.2);">
    <h4 style="margin-top: 0;">⚡ Компактна версія</h4>
    <p style="font-size: 0.92rem; opacity: 0.85;">Полегшений єдиний файл для комп'ютерів, де вже встановлено середовище <strong>.NET 10 Desktop Runtime</strong>.</p>
    <p style="font-size: 0.85rem; opacity: 0.7; margin-bottom: 14px;">Розмір: ~903 КБ • Один файл</p>
    <a href="/downloads/UkraineAlertsWidget.exe" download style="display: inline-block; background: #334155; color: #ffffff; padding: 9px 18px; border-radius: 8px; font-weight: 600; text-decoration: none;">⬇ Завантажити Compact (.exe)</a>
  </div>

</div>

> **Примітка Windows SmartScreen:** оскільки файл не підписаний комерційним сертифікатом Microsoft Store, при першому запуску Windows може показати вікно SmartScreen. Натисніть **«Докладніше» (More info) → «Виконати в будь-якому разі» (Run anyway)**.
{: .prompt-info }
