# Photo Upload MCP (OpenCode)

MCP-сервер, который отправляет фото с локального диска в чат OpenCode через его внутренний HTTP API. В отличие от «вставки из буфера», здесь не нужны горячие клавиши, фокус окна или доступ к буферу — только `POST /session/{sid}/prompt_async`.

> Исследование и обоснование решения: [RESEARCH.md](RESEARCH.md)

---

## Русский

### Что делает
- Инструмент **`send_photo(path, text?, session_id?)`**.
- Читает файл изображения с диска, кодирует в base64 (data-URL) и отправляет как `FilePartInput`.
- Фото рендерится в desktop UI OpenCode (проверено живым тестом: HTTP 204).
- **Сам находит** адрес API: PID процессов `OpenCode.exe` (`tasklist`) ∩ LISTENING-сокеты на `127.0.0.1` (`netstat`), с проверкой через `GET /session`.
- **Сам выбирает** сессию: самая недавняя по `time.updated` (если не задан `session_id`).

### Параметры инструмента
| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `path` | string | да | Абсолютный путь к файлу изображения на ПК |
| `text` | string | нет | Подпись/комментарий к фото |
| `session_id` | string | нет | ID сессии; по умолчанию — самая недавняя |

### Требования
- **Windows** (поиск порта использует `tasklist` и `netstat`).
- **Python 3.x** с пакетами: `mcp` (MCP SDK) и `requests`.
- Запущенный **OpenCode Desktop**, слушающий на `127.0.0.1`, с установленным паролем сервера.

### Переменные окружения
| Переменная | По умолчанию | Описание |
|---|---|---|
| `OPENCODE_SERVER_USERNAME` | `opencode` | Логин для Basic-авторизации |
| `OPENCODE_SERVER_PASSWORD` | — | Пароль сервера (наследуется дочерним процессом MCP) |

### Установка / регистрация
1. Положите `server.py` в любую папку, например `C:/tests_model/photo-upload-mcp/`.
2. В `opencode.json` добавьте запись под ключом `mcpServers`:
   ```json
   {
     "mcpServers": {
       "photo-upload": {
         "command": "C:\\Python314\\python.exe",
         "args": ["C:/tests_model/photo-upload-mcp/server.py"]
       }
     }
   }
   ```
   (замените путь к `python.exe` и к `server.py` на свои)
3. Перезапустите OpenCode — инструмент `send_photo` станет доступен агенту.

### Использование
- Вызов агентом: `send_photo(path="C:/path/to/image.png", text="Вот фото")`.
- Поле `session_id` можно не указывать — сессия подхватится автоматически.

---

## English

### What it does
- Tool **`send_photo(path, text?, session_id?)`**.
- Reads an image file from disk, base64-encodes it (data-URL) and sends it as a `FilePartInput`.
- The photo renders in the OpenCode desktop UI (verified with a live test: HTTP 204).
- **Finds the API address itself**: `OpenCode.exe` PIDs (`tasklist`) ∩ LISTENING sockets on `127.0.0.1` (`netstat`), confirmed via `GET /session`.
- **Picks the session itself**: most recent by `time.updated` (unless `session_id` is given).

### Tool parameters
| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | string | yes | Absolute path to an image file on the machine |
| `text` | string | no | Caption/comment for the photo |
| `session_id` | string | no | Session ID; defaults to the most recent one |

### Requirements
- **Windows** (port discovery uses `tasklist` and `netstat`).
- **Python 3.x** with packages: `mcp` (MCP SDK) and `requests`.
- A running **OpenCode Desktop** listening on `127.0.0.1`, with a server password set.

### Environment variables
| Variable | Default | Description |
|---|---|---|
| `OPENCODE_SERVER_USERNAME` | `opencode` | Basic-auth username |
| `OPENCODE_SERVER_PASSWORD` | — | Server password (inherited by the MCP child process) |

### Installation / registration
1. Place `server.py` in any folder, e.g. `C:/tests_model/photo-upload-mcp/`.
2. Add an entry under the `mcpServers` key in `opencode.json`:
   ```json
   {
     "mcpServers": {
       "photo-upload": {
         "command": "C:\\Python314\\python.exe",
         "args": ["C:/tests_model/photo-upload-mcp/server.py"]
       }
     }
   }
   ```
   (replace the `python.exe` and `server.py` paths with your own)
3. Restart OpenCode — the `send_photo` tool becomes available to the agent.

### Usage
- Agent call: `send_photo(path="C:/path/to/image.png", text="Here is the photo")`.
- The `session_id` field is optional — a session is picked automatically.

---

## Files
- `server.py` — the MCP server.
- `RESEARCH.md` — research and rationale for the chosen approach.
- `.gitignore`
