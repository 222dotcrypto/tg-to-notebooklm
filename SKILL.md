---
name: tg-to-notebooklm
description: "Конвертация экспорта Telegram Desktop чатов (JSON + медиа) и загрузка в NotebookLM. Триггеры: 'загрузи телеграм чат', 'конвертируй телеграм', 'telegram в notebooklm', 'tg чат в notebooklm', путь к папке ChatExport или result.json из Telegram. Используй этот скилл всегда, когда пользователь упоминает Telegram экспорт, конвертацию чатов, или даёт путь к папке/файлу Telegram Desktop экспорта."
---

# Telegram Chat → NotebookLM

## Goal
Принять папку Telegram Desktop экспорта (JSON + photos/ + files/), обработать сообщения в структурированный текст, загрузить в NotebookLM для анализа, поиска важного, создания презентаций и подкастов.

## Requirements
- NotebookLM MCP сервер (настроен и авторизован)

## Process

### Step 1: Проверка авторизации
- Вызвать `notebook_list(max_results=1)` — если ошибка авторизации:
  1. Запустить Brave: `"/Applications/Brave Browser.app/Contents/MacOS/Brave Browser" --remote-debugging-port=18800 --no-first-run --no-default-browser-check &`
  2. Авторизоваться: `nlm login --provider openclaw --cdp-url http://127.0.0.1:18800`
  3. Обновить токены: `refresh_auth()`

### Step 2: Парсинг входа
- Определить путь к экспорту: пользователь может дать путь к папке `ChatExport_*` или к `result.json`
- Если путь к папке — искать `result.json` внутри
- Прочитать JSON, извлечь: `name` (имя чата), `type` (тип), `messages` (массив)
- Определить наличие подпапок `photos/`, `files/`
- Показать пользователю сводку: название чата, количество сообщений, период, количество фото/файлов
- Спросить подтверждение

### Step 3: Обработка сообщений
Конвертировать JSON-сообщения в чистый структурированный текст, оптимизированный для NotebookLM.

**Формат одного сообщения:**
```
[2026-02-18 17:26] User1:
/start

[2026-02-18 17:28] User2:
Yep, I'm here.

[2026-02-18 18:22] User2:
Сделал первую аватарку для бота
[Фото: photo_1@18-02-2026_18-22-51.jpg]
```

**Правила обработки текста:**
- `text` может быть строкой или массивом объектов `{type, text}`
- Для массива — конкатенировать значения `text` из каждого элемента
- Типы текстовых сущностей (bold, italic, code, link и пр.) — брать только `text`, форматирование не нужно
- Экранирование Markdown НЕ нужно — NotebookLM работает с plain text

**Медиа-вложения:**
- `photo` → `[Фото: <имя файла>]`
- `file` (если не начинается с "(File not included") → `[Файл: <имя файла>]`
- Стикеры, видео, голосовые → `[<тип>: <имя файла>]`
- Сами медиа-файлы НЕ загружаются в NotebookLM — только текстовое описание

**Служебные сообщения (type=service):**
- Включать как: `[2026-02-18 17:22] === Служебное: <action> (by <actor>) ===`

**Ответы (reply_to_message_id):**
- Если сообщение — ответ, добавить: `> В ответ на #<id>`

**Дата:** формат `YYYY-MM-DD HH:MM` (без секунд)

### Step 4: Разбивка на части
NotebookLM имеет лимит на размер текстового источника (~200K символов рекомендуется для стабильной работы).

- Подсчитать общий размер текста
- Если > 200K символов — разбить на части по ~150K символов, на границе сообщений (не разрезать сообщение пополам)
- Каждой части дать заголовок: `<Название чата> — Часть N (даты от-до)`

### Step 5: Создание блокнота и загрузка
1. `notebook_create(title=<название чата>)`
2. Для каждой части:
   - `source_add(notebook_id=..., source_type="text", title="<Название> — Часть N", text=<текст части>, wait=true)`
3. Проверить: `notebook_get(notebook_id=...)` — все источники добавлены

### Step 6: Предложить действия (human-in-the-loop)
Спросить пользователя через AskUserQuestion (multiSelect=true):
- **Summary (отчёт)** — краткий обзор ключевых тем переписки (Briefing Doc)
- **Аудиоподкаст** — deep dive диалог по переписке
- **Презентация** — slide deck с основными моментами
- **Mind Map** — визуальная карта тем и связей
- **Оставить как есть** — только источники в блокноте

### Step 7: Генерация и скачивание
Для каждого выбранного материала:
1. `studio_create(notebook_id=..., artifact_type=..., confirm=True)`
2. Поллить `studio_status(notebook_id=...)` до завершения
3. `download_artifact(notebook_id=..., artifact_type=..., output_path=~/tg-to-notebooklm/downloads/<filename>)`
4. Сообщить путь к файлу

Маппинг:
- Summary → report (report_format="Briefing Doc")
- Аудиоподкаст → audio (audio_format="deep_dive")
- Презентация → slide_deck
- Mind Map → mind_map

## Telegram JSON Structure Reference

Структура `result.json` из Telegram Desktop экспорта:

```json
{
  "name": "Chat Name",
  "type": "personal_chat|bot_chat|private_group|private_supergroup",
  "id": 123456789,
  "messages": [
    {
      "id": 12345,
      "type": "message|service",
      "date": "2026-02-18T17:26:12",
      "date_unixtime": "1771410372",
      "from": "Username",
      "from_id": "user123",
      "text": "string or array",
      "text_entities": [...],
      "reply_to_message_id": 12344,
      "photo": "photos/photo_1@....jpg",
      "file": "files/document.pdf",
      "edited": "2026-02-18T17:35:52",
      "forwarded_from": "Channel Name"
    }
  ]
}
```

Поле `text` может быть:
- Строка: `"hello"`
- Массив: `["text", {"type": "bold", "text": "bold text"}, "more text"]`

## Rules
- ВСЕГДА проверять авторизацию NotebookLM перед началом (Step 1)
- Медиа-файлы НЕ загружать в NotebookLM — только текстовые описания вложений
- При большом чате — разбивать на части, не пытаться загрузить всё одним источником
- Показывать прогресс на каждом этапе
- Предлагать несколько вариантов, не один
- Создать папку `~/tg-to-notebooklm/downloads/` для скачанных артефактов
