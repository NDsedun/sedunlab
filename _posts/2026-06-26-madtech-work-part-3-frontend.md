---
layout: post
title: "Як я зробив сайт який кожного разу інший: частина 3"
subtitle: "Фронтенд з випадковими темами, стримінг JSON і SVG favicon"
date: 2026-06-26 14:00:00 +0200
categories: [tutorial, frontend]
tags: [javascript, css, sse, streaming, svg, дизайн]
---

У [першій частині](/2026/06/26/madtech-work-part-1-fastapi) ми підняли FastAPI, у [другій](/2026/06/26/madtech-work-part-2-openlitespeed) — налаштували OLS. Тепер найцікавіше: фронтенд який щоразу виглядає інакше.

## Ідея: теми як набори CSS змінних

Замість того щоб мати кілька HTML-файлів або перезавантажувати сторінку, вся магія відбувається через CSS custom properties. Кожна тема — це просто об'єкт з набором змінних:

```javascript
const THEMES = [
  {
    name: 'void',
    layout: 'default',
    vars: {
      '--bg': '#0a0a0f',
      '--accent': '#7c3aed',
      '--accent2': '#06b6d4',
      '--font-head': "'Space Grotesk', sans-serif",
      // ...
    }
  },
  {
    name: 'matrix',
    layout: 'terminal',
    vars: {
      '--bg': '#000d00',
      '--accent': '#00ff41',
      '--font-head': "'Space Mono', monospace",
      // ...
    }
  },
  // ще 4 теми...
];
```

Застосування теми — одна функція:

```javascript
function applyTheme(theme) {
  const root = document.documentElement;
  document.body.classList.remove('layout-default','layout-sidebar','layout-magazine','layout-terminal');
  document.body.classList.add('layout-' + theme.layout);

  for (const [k, v] of Object.entries(theme.vars)) {
    root.style.setProperty(k, v);
  }
}

// При завантаженні і при кожному новому артиклі
const theme = THEMES[Math.floor(Math.random() * THEMES.length)];
applyTheme(theme);
```

Всього 6 тем і 3 лейаути: `default` (стандартний), `sidebar` (фото ліворуч, текст праворуч), `magazine` (широке фото зверху), `terminal` (вузький, моноширинний шрифт).

## Стримінг JSON — найцікавіша частина

Модель повертає JSON, але ми отримуємо його шматками. Як показувати текст поступово якщо JSON ще неповний?

Відповідь: **парсимо часткові дані регулярними виразами**. Поки JSON не завершений, витягуємо поля по мірі їх появи:

```javascript
function renderPartial(text) {
  // Спочатку пробуємо повний парсинг
  const full = tryParse(text);
  if (full) {
    titleEl.innerHTML = full.title;
    subtitleEl.innerHTML = full.subtitle;
    bodyEl.innerHTML = full.body;
    return;
  }

  // Якщо JSON неповний — витягуємо поля частково
  if (!titleDone) {
    // Завершений title
    const tm = text.match(/"title"\s*:\s*"([^"\\]*(?:\\.[^"\\]*)*)"/s);
    if (tm) {
      titleEl.textContent = tm[1];
      titleDone = true;
    } else {
      // Ще стримиться — показуємо з курсором
      const p = text.match(/"title"\s*:\s*"([^"]*)/s);
      if (p) titleEl.innerHTML = p[1] + '<span class="cursor"></span>';
    }
  }

  // Аналогічно для subtitle і body...
}
```

Курсор, що блимає, додає відчуття живого друку:

```css
.cursor {
  display: inline-block;
  width: 2px; height: 1.1em;
  background: var(--accent);
  animation: blink 0.7s step-end infinite;
}
@keyframes blink { 50% { opacity: 0; } }
```

## EventSource на клієнті

```javascript
const evtSource = new EventSource(`/generate?lang=${encodeURIComponent(lang)}`);

evtSource.onmessage = (e) => {
  const data = JSON.parse(e.data);

  if (data.type === 'text') {
    rawBuffer += data.chunk;
    renderPartial(rawBuffer);
  }
  if (data.type === 'image') {
    document.getElementById('article-img').src = data.url;
  }
  if (data.type === 'done') {
    evtSource.close();
    document.querySelectorAll('.cursor').forEach(c => c.remove());
  }
};
```

Мова визначається автоматично з браузера:

```javascript
const lang = navigator.language || navigator.languages?.[0] || 'en';
```

І передається на бекенд, де маппиться до назви мови для промпту: `"uk" → "українською"`, `"de" → "auf Deutsch"` тощо.

## SVG Favicon — кубик для гри в кості

Замість PNG використовуємо SVG — він чітко виглядає на будь-якому розмірі та щільності екрану. Тривимірний кубик складається з трьох паралелограмів:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
  <!-- Верхня грань -->
  <polygon points="100,4 196,52 100,100 4,52" fill="#7c3aed"/>
  <!-- Права грань -->
  <polygon points="100,100 196,52 196,148 100,196" fill="#5b21b6"/>
  <!-- Ліва грань -->
  <polygon points="100,100 4,52 4,148 100,196" fill="#9d4ff7"/>

  <!-- Крапка на верхній грані (1) -->
  <circle cx="100" cy="52" r="8" fill="white" opacity="0.95"/>
  <!-- Крапки на правій грані (2) -->
  <circle cx="166" cy="76" r="7" fill="white" opacity="0.9"/>
  <circle cx="118" cy="172" r="7" fill="white" opacity="0.9"/>
  <!-- Крапки на лівій грані (3) -->
  <circle cx="70" cy="76" r="7" fill="white" opacity="0.9"/>
  <circle cx="40" cy="124" r="7" fill="white" opacity="0.9"/>
  <circle cx="22" cy="172" r="7" fill="white" opacity="0.9"/>
</svg>
```

Підключення в `<head>`:

```html
<link rel="icon" type="image/svg+xml" href="/static/favicon.svg" />
```

## Підсумок усієї серії

За три статті ми зібрали сайт де:

- **Бекенд** — FastAPI + Groq (Llama 3.3 70B) + Pollinations.ai, все безкоштовно
- **Стримінг** — SSE з частковим парсингом JSON в реальному часі
- **Дизайн** — 6 випадкових тем × 3 лейаути = 18 варіантів вигляду
- **Інфраструктура** — OpenLiteSpeed + CyberPanel + systemd + Let's Encrypt

Повний код доступний на [madtech.work](https://madtech.work) — просто оновіть сторінку кілька разів щоб побачити різні теми в дії.
