---
layout: page
title: Мапа повітряних тривог України — Віджет для Windows 11 та годинників Amazfit
permalink: /alerts-widget/
sitemap: false
---

<div style="display: flex; align-items: center; gap: 18px; margin-bottom: 1.5rem;">
  <img src="/assets/img/alerts-widget-icon.png" alt="Ukraine Alerts Widget Icon" width="76" height="76" style="border-radius: 16px;" />
  <div>
    <h2 style="margin: 0 0 4px 0; border: none; padding: 0;">Ukraine Alerts Widget <span style="font-size: 0.85rem; opacity: 0.65; font-weight: normal;">v1.2</span></h2>
    <p style="margin: 0; opacity: 0.8;">Настільний віджет для Windows 10 / 11 та застосунок для смарт-годинників Amazfit T-Rex 2 / Amazfit Balance • <strong>Created by ND Sedun</strong></p>
  </div>
</div>

Настільний віджет відображає інтерактивну векторну мапу України поверх вікон або безпосередньо на шпалерах робочого столу, оновлюючи дані кожні 15 секунд у реальному часі. Працює виключно через системний трей і не займає місця на панелі завдань.

### Основні можливості (Windows 11)

- **🔍 Зум по кліку на район або область:**
  - Клік лівою кнопкою миші по будь-якому району (або області) плавно масштабує мапу на обраний регіон; повторний клік або колесо миші повертає загальний план.
- **🔔 Сповіщення про тривогу та відбій для обраного району:**
  - Можливість обрати будь-яку **область** та конкретний **район** (або всю область).
  - Спливаючі повідомлення Windows 11 у правому нижньому куті екрана при оголошенні тривоги або відбою.
  - Опціональний **м'який звуковий сигнал** (приємний мелодійний дзвіночок замість сирени) + кнопки миттєвого тестування у налаштуваннях.
- **🛸 Відображення руху ворожих повітряних цілей (Увімк. / Вимк.):**
  - Кнопка **«🛸 Цілі»** вмикає або вимикає відображення зафіксованих повітряних об'єктів (БпЛА Shahed, реактивні БпЛА, ракети, КАБ) із векторами напрямку їхнього руху та деталями при наведенні курсора.
- **Два режими мапи:** перемикання між **областями (27 регіонів)** та окремими **районами (139 районів)** в один клік.
- **Налаштування вигляду:**
  - Перемикач **«Фон»** — від напівпрозорої картки у стилі Windows 11 до повністю прозорого фону (лише контури мапи на шпалерах).
  - Перемикач **«Назви»** — увімкнення/вимкнення підписів областей.
  - Закріплення поверх усіх вікон (**📌**) та режим **Click-Through** (кліки миші проходять крізь вікно віджета).
- **Системний трей Windows:** іконка в треї динамічно показує поточну кількість регіонів із тривогою. Лівий клік миттєво ховає або показує віджет.

---

### Завантаження для Windows 10 / 11

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
    <p style="font-size: 0.85rem; opacity: 0.7; margin-bottom: 14px;">Розмір: ~918 КБ • Один файл</p>
    <a href="/downloads/UkraineAlertsWidget.exe" download style="display: inline-block; background: #334155; color: #ffffff; padding: 9px 18px; border-radius: 8px; font-weight: 600; text-decoration: none;">⬇ Завантажити Compact (.exe)</a>
  </div>

</div>

> **Примітка Windows SmartScreen:** оскільки файл не підписаний комерційним сертифікатом Microsoft Store, при першому запуску Windows може показати вікно SmartScreen. Натисніть **«Докладніше» (More info) → «Виконати в будь-якому разі» (Run anyway)**.
{: .prompt-info }

---

### ⌚ Версія для смарт-годинників Amazfit T-Rex 2 та Amazfit Balance (Zepp OS)

Спеціальний тактичний застосунок **«Тривога UA»** для круглих AMOLED-дисплеїв **Amazfit T-Rex 2 (`454×454`)** та **Amazfit Balance (`480×480`)** із миттєвою візуальною індикацією та вібро-алярмом на зап'ясті:

![Amazfit Watch Preview](/assets/img/trex2-preview.png)

- **🟢 Зелений / 🔴 Червоний статус з першого погляду:** при запуску зовнішнє кільце безеля та верхній банер одразу світяться **смарагдово-зеленим (`СПОКІЙНО`)** або **яскраво-червоним (`! ТРИВОГА !`)** для вашого регіону (за замовчуванням — **`м. Київ`**).
- **🗺️ Жива мапа України на екрані годинника:** усі 27 регіонів відображаються у реальному часі (зелений — спокійно, помаранчевий — часткова тривога в області, червоний — повна тривога).
- **📳 Вібро-алярм на руці:** якщо у вашому регіоні активна тривога (або вона починається під час роботи застосунку), годинник вмикає відчутну тактильну вібрацію.
- **⏰ Автоматична перевірка кожні 5 хв (`Авто: 5хв`):** увімкніть кнопку **«Авто: 5хв»** внизу екрана, щоб годинник сам періодично прокидався, перевіряв статус по Bluetooth через телефон і будив вас вібрацією у разі тривоги.
- **📍 Перемикання регіону:** тапніть по назві `< м. Київ >` угорі екрана, щоб перемкнути моніторинг на Київську область або будь-яку іншу область України (вибір зберігається в пам'яті годинника).

<div style="border: 1px solid rgba(16, 185, 129, 0.35); border-radius: 12px; padding: 20px; background: rgba(15, 23, 42, 0.35); margin: 1.5rem 0;">
  <h4 style="margin-top: 0;">Встановлення через QR-код у додатку Zepp (за 1 хвилину)</h4>
  <ol style="font-size: 0.92rem; line-height: 1.55; padding-left: 1.2rem; margin-bottom: 18px;">
    <li>У додатку <strong>Zepp</strong> на телефоні перейдіть у <strong>Профіль → Налаштування → Про програму</strong> та 7 разів торкніться логотипу Zepp, щоб увімкнути <strong>Режим розробника (Developer Mode)</strong>.</li>
    <li>Перейдіть у <strong>Пристрій → Загальні → Режим розробника → Mini Program</strong>.</li>
    <li>Натисніть кнопку <strong>«+» (Scan)</strong> у правому верхньому куті та відскануйте QR-код для вашої моделі годинника:</li>
  </ol>

  <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(230px, 1fr)); gap: 16px;">
    <div style="border: 1px solid rgba(148, 163, 184, 0.25); border-radius: 10px; padding: 14px; text-align: center; background: rgba(15, 23, 42, 0.4);">
      <div style="font-weight: 700; margin-bottom: 8px;">⌚ Amazfit T-Rex 2 (454×454)</div>
      <img src="/assets/img/trex2-qr.png" alt="QR Amazfit T-Rex 2" width="145" height="145" style="border-radius: 8px; background: #fff; padding: 6px; margin: 0 auto 10px auto; display: block;" />
      <a href="/downloads/UkraineAlerts_TRex2.zpk" download style="display: inline-block; background: #10b981; color: #06281e; padding: 7px 14px; border-radius: 7px; font-weight: 700; text-decoration: none; font-size: 0.86rem;">⬇ Завантажити T-Rex 2 (.zpk)</a>
    </div>

    <div style="border: 1px solid rgba(148, 163, 184, 0.25); border-radius: 10px; padding: 14px; text-align: center; background: rgba(15, 23, 42, 0.4);">
      <div style="font-weight: 700; margin-bottom: 8px;">⌚ Amazfit Balance (480×480)</div>
      <img src="/assets/img/balance-qr.png" alt="QR Amazfit Balance" width="145" height="145" style="border-radius: 8px; background: #fff; padding: 6px; margin: 0 auto 10px auto; display: block;" />
      <a href="/downloads/UkraineAlerts_Balance.zpk" download style="display: inline-block; background: #0284c7; color: #ffffff; padding: 7px 14px; border-radius: 7px; font-weight: 700; text-decoration: none; font-size: 0.86rem;">⬇ Завантажити Balance (.zpk)</a>
    </div>
  </div>

  <div style="margin-top: 14px; text-align: center;">
    <a href="/downloads/UkraineAlerts_TRex2.zab" download style="display: inline-block; background: #334155; color: #ffffff; padding: 7px 16px; border-radius: 7px; font-weight: 600; text-decoration: none; font-size: 0.85rem;">⬇ Універсальний бандл для обох моделей (.zab)</a>
  </div>
</div>
