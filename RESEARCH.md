# Research: making local photos visible in OpenCode chat

This document records the investigation into how to make an image from the local disk appear in the OpenCode desktop chat UI, and why the chosen MCP approach works.

---

## Russian

### Проблема
Отправка изображения через «легаси»-эндпоинт `/session/{sid}/message` сохраняла сообщение, но фото **не рендерилось** в desktop UI OpenCode.

### Корень проблемы
В OpenCode существуют два независимых пути хранения сообщений:
- **SDK/legacy path** — пишет сообщения на серверную сторону (`/api`).
- **Raw path** — тот, из которого читает desktop UI.

UI рендерит только те части (parts), которые прошли через raw path. Поэтому фото, отправленное легаси-эндпоинтом, сохранялось, но не отображалось.

### Рабочее решение
`POST /session/{sid}/prompt_async` с `FilePartInput` (data-URL) в поле `parts`. Возвращает **204**, и фото рендерится в UI. Подтверждено живым тестом.

Форма payload:
```json
{
  "parts": [
    { "type": "text", "text": "..." },
    { "type": "file", "mime": "image/png", "filename": "x.png", "url": "data:image/png;base64,...." }
  ]
}
```

### Поиск адреса API (порт)
1. `tasklist /FI "IMAGENAME eq OpenCode.exe"` → набор PID процессов OpenCode.
2. `netstat -ano -p tcp` → сокеты в состоянии `LISTENING`, чей PID входит в набор из п. 1, на `127.0.0.1`.
3. Проверка кандидата: `GET /session` с Basic-авторизацией → 200 = живой API.

Проверено: возвращается порт `60843`.

### Автовыбор сессии
`GET /session` → массив сессий; сортировка по `time.updated` (убывающе); берётся первая (самая недавняя).

### Авторизация
Basic auth. Логин `opencode`; пароль берётся из переменной окружения `OPENCODE_SERVER_PASSWORD`, которую наследует дочерний процесс MCP.

### Почему API, а не буфер обмена / горячие клавиши
Существующий MCP (`autopaste-gallery`) использует буфер обмена + `pyautogui` (Ctrl+V). Это хрупко: зависимости от фокуса окна, доступ к буферу; в логах наблюдались access violations. Подход через HTTP API детерминирован, не требует GUI и работает headless.

### Факты окружения (на момент тестирования)
- Desktop OpenCode 1.19.3; эталонный source: `C:\temp\opencode-src`, tag `v1.19.3`.
- Live-сервер: порт `60843`, exe `OpenCode.exe`.
- Сессия: `ses_eb59759c2ffeR9DjYYTaqnsUj2`, модель `qwen3.8-27b`.
- Live-тест: `send_photo(...)` → `(sid, 60843, 204, '')`; фото подтверждено пользователем.

---

## English

### Problem
Sending an image through the legacy `/session/{sid}/message` endpoint persisted the message but the photo was **not rendered** in the OpenCode desktop UI.

### Root cause
OpenCode has two independent message-storage paths:
- **SDK/legacy path** — writes messages to the server side (`/api`).
- **Raw path** — the one the desktop UI reads from.

The UI only renders parts that went through the raw path, so a photo sent via the legacy endpoint was stored but not displayed.

### Working solution
`POST /session/{sid}/prompt_async` with a `FilePartInput` (data-URL) in `parts`. Returns **204** and the photo renders in the UI. Confirmed by a live test.

Payload shape:
```json
{
  "parts": [
    { "type": "text", "text": "..." },
    { "type": "file", "mime": "image/png", "filename": "x.png", "url": "data:image/png;base64,...." }
  ]
}
```

### API address (port) discovery
1. `tasklist /FI "IMAGENAME eq OpenCode.exe"` → set of OpenCode PIDs.
2. `netstat -ano -p tcp` → `LISTENING` sockets on `127.0.0.1` whose PID is in the set from step 1.
3. Candidate check: `GET /session` with Basic auth → 200 = live API.

Verified: returns port `60843`.

### Session auto-pick
`GET /session` → array of sessions; sorted by `time.updated` (descending); the first (most recent) is used.

### Authorization
Basic auth. User `opencode`; password from the `OPENCODE_SERVER_PASSWORD` env var, inherited by the MCP child process.

### Why API instead of clipboard / hotkeys
The existing MCP (`autopaste-gallery`) uses the clipboard + `pyautogui` (Ctrl+V). That is fragile: it depends on window focus and clipboard access, and access violations were observed in the logs. The HTTP-API approach is deterministic, needs no GUI, and works headless.

### Environment facts (at test time)
- Desktop OpenCode 1.19.3; reference source: `C:\temp\opencode-src`, tag `v1.19.3`.
- Live server: port `60843`, exe `OpenCode.exe`.
- Session: `ses_eb59759c2ffeR9DjYYTaqnsUj2`, model `qwen3.8-27b`.
- Live test: `send_photo(...)` → `(sid, 60843, 204, '')`; photo confirmed by the user.
