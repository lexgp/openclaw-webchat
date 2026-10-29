# WebChat — канал чата OpenClaw для браузера

Форк от [https://github.com/wutao667/openclaw-webchat](https://github.com/wutao667/openclaw-webchat)

WebChat позволяет общаться с агентами OpenClaw прямо в браузере, без сторонних мессенджеров.

## Возможности

- **Ноль сторонних зависимостей**: браузер — единственный клиент, полный контроль над данными.
- **Несколько агентов на одном сервере**: один Chat Server и один Gateway обслуживают несколько агентов независимо.
- **Изоляция сессий**: `sessionKey = openclaw-webchat:{userId}:{agentId}` — история разных пользователей не пересекается.
- **Реальное время**: WebSocket‑соединение с индикатором печати.
- **Встроенная панель управления**: создание и отключение App без правки конфигов вручную.
- **Markdown**: ответы агента отображаются с форматированием (жирный, списки, код).

## Архитектура

```
Браузер --WS--> Chat Server --WS--> Channel Plugin --dispatch--> OpenClaw Core → Agent
```

- **Chat Server**: Node.js‑сервер, отдаёт страницу и проксирует WebSocket между браузером и плагином.
- **Channel Plugin**: устанавливается в OpenClaw Gateway и подключается к Chat Server самостоятельно.
- Gateway не требует входящих вебхуков — все соединения исходящие.

## Скриншоты

### Страница входа
![Login](docs/screenshots/01-login.png)

### Список агентов
![Agent list](docs/screenshots/02-agent-list.png)

### Чат (индикатор печати)
![Chat](docs/screenshots/03-chat.png)

### Панель управления
![Admin](docs/screenshots/04-admin.jpg)

Вход в панель: `/admin/`, пароль по умолчанию `admin` — **сразу смените его** (верхняя панель → «Изменить пароль»).

## Структура проекта

```text
webchat3.0/
├── README.md
├── docs/                   # Документация по архитектуре и реализации
├── server/                 # Chat Server (фронтенд + WebSocket)
│   ├── server.js
│   ├── package.json
│   ├── README.md
│   └── public/
└── plugin/                 # OpenClaw Channel Plugin
    ├── index.js
    ├── openclaw.plugin.json
    ├── package.json
    ├── README.md
    └── src/
```

## Быстрый старт

1. Запустите Chat Server:
   ```bash
   npx openclaw-webchat-server
   ```

2. Откройте `http://localhost:3100/admin/`, войдите (`admin`) и создайте App.

3. На машине с OpenClaw Gateway установите плагин:
   ```bash
   openclaw plugins install openclaw-webchat-plugin
   ```

4. Настройте `~/.openclaw/openclaw.json` (см. ниже).

5. Перезапустите Gateway и откройте `http://localhost:3100`:
   ```bash
   openclaw gateway restart
   ```

## Развертывание сервера

### Запуск

```bash
npx openclaw-webchat-server
```

По умолчанию слушает `http://localhost:3100`:

| Адрес | Назначение |
|---|---|
| `http://<host>:3100` | Веб‑интерфейс |
| `ws://<host>:3100/ws` | WebSocket браузера |
| `ws://<host>:3100/plugin` | WebSocket плагина |

Ручной запуск: `cd server && npm install && PORT=3100 node server.js`.

### HTTPS (необязательно)

Для локальных тестов достаточно HTTP/WS; в продакшене используйте HTTPS/WSS.

Пример Caddy:
```caddy
your-domain.com {
    reverse_proxy localhost:3100
}
```

Проверка: `curl https://your-domain.com/healthz`. После включения HTTPS в настройках плагина укажите `wss://your-domain.com/plugin`.

### Создание App

1. Откройте `http://<host>:3100/admin/`.
2. Войдите с паролем `admin`.
3. Смените пароль на надёжный.
4. Нажмите **«+ Создать приложение»** и укажите отображаемое имя агента.

`appId` и `secret` генерируются автоматически. Сохраните их: в списке приложений `secret` можно скопировать кликом.

> В файле `apps.json` хранятся `secret` (открытый текст) и `secretHash` (bcrypt). Плагин проверяет `secretHash`.

## Настройка плагина

### Установка

```bash
openclaw plugins install openclaw-webchat-plugin
openclaw plugins list
```

Для локальной разработки: `openclaw plugins install path:/path/to/webchat3.0/plugin`.

### Конфигурация openclaw.json

```json
{
  "channels": {
    "openclaw-webchat": {
      "enabled": true,
      "serverUrl": "wss://your-domain.com/plugin",
      "accounts": {
        "my-account": {
          "appId": "wch_a1b2c3d4e5f60708",
          "secret": "sk-wch-xxx"
        }
      }
    }
  },
  "bindings": [
    { "channel": "openclaw-webchat", "accountId": "my-account", "agentId": "main" }
  ]
}
```

| Параметр | Описание |
|---|---|
| `serverUrl` | WebSocket‑адрес Chat Server (`/plugin`) |
| `accounts.*.appId` | ID приложения из панели управления |
| `accounts.*.secret` | Секрет приложения (хранится в открытом виде в панели) |
| `bindings` | Связка `{ channel, accountId }` → `agentId` |

### Перезапуск

```bash
openclaw gateway restart
openclaw channels list
```

Ожидаемый статус: `WebChat my-account: installed, configured, enabled`. После этого откройте чат, введите имя и выберите агента.

## Несколько агентов (необязательно)

Создайте отдельные App в панели, добавьте account и привяжите к разным агентам:

```json
{
  "channels": {
    "openclaw-webchat": {
      "enabled": true,
      "serverUrl": "wss://your-domain.com/plugin",
      "accounts": {
        "dev":  { "appId": "wch_dev0000000000000",  "secret": "sk-dev-xxx" },
        "prod": { "appId": "wch_prod000000000000", "secret": "sk-prod-xxx" }
      }
    }
  },
  "bindings": [
    { "channel": "openclaw-webchat", "accountId": "dev",  "agentId": "dev-agent" },
    { "channel": "openclaw-webchat", "accountId": "prod", "agentId": "prod-agent" }
  ]
}
```

К одному Chat Server может подключаться несколько Gateway — у каждого свой `appId`/`secret`/`bindings`.

## Модель сессий

- История привязана к `openclaw-webchat:{userId}:{agentId}`.
- Один пользователь в нескольких браузерах видит одну и ту же историю.
- Разные пользователи изолированы.
- На сервере хранится последние 100 сообщений в памяти.

## Проверка

```bash
curl http://localhost:3100/healthz
cd server && node test-ws.mjs
openclaw channels list
```

- `/healthz` — сервер жив.
- `test-ws.mjs` — WebSocket работает.
- `channels list` — плагин подключён.

## Разработка

```bash
cd server
npm run dev
```

Код плагина: `plugin/src/`. После правок плагина перезапустите Gateway. Подробности — в `docs/ARCHITECTURE.md` и `docs/IMPL-GUIDE.md`.

## Частые проблемы

| Проблема | Решение |
|---|---|
| `not configured` | Проверьте, что в `openclaw.json` есть `accounts` и `bindings`, а `serverUrl` указан на уровне канала. |
| `invalid_app` | Убедитесь, что `appId` создан в панели и не отключён. |
| `invalid_secret` | Скопируйте `secret` из панели управления и сверьте с конфигом плагина. |
| Браузер не подключается | Проверьте `/healthz`, адрес WebSocket в консоли и настройку HTTPS/WSS. |
