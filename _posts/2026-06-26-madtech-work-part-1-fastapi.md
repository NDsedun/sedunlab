---
layout: post
title: "Як я зробив сайт який кожного разу інший: частина 1"
subtitle: "Від ідеї до FastAPI на VPS з CyberPanel"
date: 2026-06-26 09:00:00 +0200
categories: [tutorial, python]
tags: [fastapi, python, vps, cyberpanel, openlitespeed, ai]
---

Одного дня я подумав: а що якби сайт при кожному відкритті показував щось нове? Не просто рандомну сторінку з бази — а повністю згенерований контент: унікальна стаття з картинкою, інший колір, інший лейаут. Щоразу інший.

Так народився [madtech.work](https://madtech.work) — сайт про комп'ютерні технології з гумором, де кожне відвідування це окремий досвід.

У цій серії з трьох статей я розповім як це зробити з нуля на VPS з CyberPanel та OpenLiteSpeed.

## Що в результаті вийде

- **FastAPI** бекенд на Python
- **Groq API** (безкоштовно) для генерації тексту — модель Llama 3.3 70B
- **Pollinations.ai** (безкоштовно, без ключа) для генерації зображень
- **SSE стримінг** — текст з'являється поступово, як у ChatGPT
- **6 випадкових тем** з різними кольорами та лейаутами
- Автоматичне визначення мови браузера

## Частина 1: Структура проєкту і FastAPI

### Що потрібно

- VPS з CyberPanel (OpenLiteSpeed)
- Python 3.9+
- Безкоштовний акаунт на [console.groq.com](https://console.groq.com)

### Структура файлів

```
/home/madtech.work/
├── app/
│   ├── main.py
│   ├── requirements.txt
│   ├── .env
│   ├── static/
│   │   └── favicon.svg
│   └── templates/
│       └── index.html
└── venv/
```

### Встановлення залежностей

```bash
# Створюємо директорію
mkdir -p /home/madtech.work/app
cd /home/madtech.work/app

# Віртуальне середовище
python3 -m venv /home/madtech.work/venv
source /home/madtech.work/venv/bin/activate

# Залежності
pip install fastapi uvicorn openai httpx python-dotenv jinja2 aiofiles
```

> **Важливо для Python 3.9:** бібліотека `openai` нових версій конфліктує з `httpx`. Фіксуємо версії:
> ```bash
> pip install "openai==1.54.4" "httpx==0.27.2"
> ```

### Основний файл main.py

Ключова ідея: Groq має OpenAI-сумісний API, тому використовуємо `AsyncOpenAI` просто змінивши `base_url`:

```python
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key=os.getenv("GROQ_API_KEY"),
    base_url="https://api.groq.com/openai/v1",
)
```

Для генерації контенту просимо модель повертати **строго JSON** — це дозволяє парсити відповідь прямо під час стримінгу:

{% raw %}
```python
system_prompt = (
    f"You are a witty tech blogger. Write EXCLUSIVELY {lang_name}. "
    f"Response format — STRICTLY valid JSON (no markdown, no extra text):\n"
    f'{{"title": "...", "subtitle": "...", "body": "...HTML..."}}\n'
    f"body must contain 4-6 paragraphs in <p> tags."
)
```
{% endraw %}

### SSE стримінг

Замість звичайного HTTP-відповіді використовуємо **Server-Sent Events** — браузер отримує дані шматками в реальному часі:

```python
@app.get("/generate")
async def generate(lang: str = "en"):
    async def stream():
        # Спочатку відправляємо URL зображення
        yield f"data: {json.dumps({'type': 'image', 'url': image_url})}\n\n"

        # Потім стримімо текст
        stream_response = await client.chat.completions.create(
            model="llama-3.3-70b-versatile",
            stream=True,
            messages=[...]
        )
        async for chunk in stream_response:
            delta = chunk.choices[0].delta.content or ""
            if delta:
                yield f"data: {json.dumps({'type': 'text', 'chunk': delta})}\n\n"

        yield f"data: {json.dumps({'type': 'done'})}\n\n"

    return StreamingResponse(
        stream(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

Заголовок `X-Accel-Buffering: no` критично важливий — без нього nginx/OLS буферизує SSE і стримінг не працює.

### Зображення через Pollinations.ai

Для зображень використовуємо безкоштовний сервіс без реєстрації — просто формуємо URL:

```python
import urllib.parse, random

def get_image_url(topic: str) -> str:
    prompt = f"funny tech illustration about: {topic}, digital art"
    encoded = urllib.parse.quote(prompt)
    seed = random.randint(1, 999999)
    return f"https://image.pollinations.ai/prompt/{encoded}?width=1792&height=1024&seed={seed}&nologo=true"
```

Параметр `seed` гарантує що кожен раз зображення нове, навіть для однієї теми.

### .env файл

```bash
GROQ_API_KEY=gsk_ваш_ключ_тут
```

Ключ отримати безкоштовно на [console.groq.com/keys](https://console.groq.com/keys). Groq дає 6000 запитів на день — більш ніж достатньо.

### Перший запуск

```bash
source /home/madtech.work/venv/bin/activate
cd /home/madtech.work/app
uvicorn main:app --host 127.0.0.1 --port 8001
```

```bash
# В іншому терміналі перевіряємо
curl http://127.0.0.1:8001/
# Має повернути HTML
```

У **частині 2** розберемо як налаштувати OpenLiteSpeed в CyberPanel щоб проксіювати запити на FastAPI — це виявилось найцікавішою частиною.
