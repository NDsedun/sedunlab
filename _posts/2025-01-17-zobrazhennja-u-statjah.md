---
title: "Як додавати зображення у статті Jekyll"
date: 2025-01-17 12:00:00 +0200
categories: [homelab, jekyll]
tags: [jekyll, chirpy, markdown, images]
image:
  path: /assets/img/posts/images-cover.jpg
  alt: Зображення у статтях Jekyll
---

Зображення роблять статті живішими. У Jekyll і темі Chirpy є
кілька способів їх додавати і розміщувати.

## Де зберігати зображення

Всі зображення кладемо в папку `assets/img/posts/`:

```bash
assets/
└── img/
    ├── avatar.jpg
    └── posts/
        ├── jekyll-setup.png
        └── cyberpanel-ssl.png
```

## Базове вставлення

Стандартний Markdown синтаксис:

```markdown
![Опис зображення](/assets/img/posts/jekyll-setup.png)
```

## Зображення з підписом

Chirpy автоматично робить підпис з тексту в `_`:

```markdown
![Налаштування Jekyll](/assets/img/posts/jekyll-setup.png)
_Так виглядає CyberPanel після налаштування SSL_
```

## Розміри і розміщення

Обмежуємо ширину зображення:

```markdown
![Скріншот](/assets/img/posts/screen.png){: width="400"}
```

Вирівнювання по лівому краю:

```markdown
![Скріншот](/assets/img/posts/screen.png){: .left width="300"}
```

Вирівнювання по правому краю:

```markdown
![Скріншот](/assets/img/posts/screen.png){: .right width="300"}
```

Текст обтікатиме зображення з відповідного боку.

По центру (без обтікання):

```markdown
![Скріншот](/assets/img/posts/screen.png){: .normal}
```

## Обкладинка статті

Додаємо у front matter статті:

```yaml
---
title: "Назва статті"
image:
  path: /assets/img/posts/cover.jpg
  alt: Опис обкладинки
---
```

Chirpy автоматично покаже її у списку статей і вгорі поста.

## Корисні поради

Оптимальна ширина зображень — 800-1200px. Більше не потрібно,
тільки важчий сайт. Формат WebP замість PNG дає менший розмір
при тій самій якості:

```bash
# Конвертація на сервері
dnf install -y libwebp-tools
cwebp -q 80 input.png -o output.webp
```

Тоді в статті:

```markdown
![Опис](/assets/img/posts/screen.webp)
```