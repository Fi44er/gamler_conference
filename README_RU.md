🇬🇧 [Read in English](README.md)

# 🎲 Gamler Hub — инфраструктура игровых сессий в реальном времени

Real-time модуль на Go 1.23 для активации сохранённых игровых сессий, управления конкурентными подключениями игроков, маршрутизации игровых действий через WebSocket и, опционально, подключения WebRTC-аудио/видеоконференции к той же сессии.

> **Область документации:** описана только реализация в `src/modules/hub`.

## 📌 Содержание

- [🎯 Задача проекта](#-задача-проекта)
- [🏗 Архитектура](#-архитектура)
- [💬 Примеры взаимодействия](#-примеры-взаимодействия)
- [🚀 Запуск локально](#-запуск-локально)
- [⚠️ Обработка ошибок и edge cases](#️-обработка-ошибок-и-edge-cases)
- [📁 Структура проекта](#-структура-проекта)

## 🎯 Задача проекта

Модуль связывает сохранённую игровую сессию с подключёнными клиентами на этапе выполнения. Он хранит активные сессии в памяти, создаёт игровую логику через registry, синхронизирует состав игроков и доставляет игровые события одному игроку, всем игрокам или всем игрокам, кроме отправителя.

### Data Flow

1. Клиент открывает WebSocket-соединение `GET /api/session/ws/:game_name/:session_id/:user_id`.
2. Handler разбирает числовой `session_id` и получает исходную сессию из PostgreSQL-backed trash repository.
3. Hub проверяет in-memory registry. Если сессия неактивна, он загружает или создаёт документ MongoDB `game_sessions`.
4. Game registry разрешает `game_name` в factory. Factory загружает настройки игры из MongoDB и создаёт реализацию `Game`.
5. Сессия инициализируется callback-функциями для broadcast-доставки и сохраняется в process-local hub.
6. Клиент добавляется в сессию. Статус host определяется по сохранённому `HostID`; первое подключение автоматически не назначается host этим модулем.
7. Входящие Socket.IO payload события `game_action` декодируются в `Action{type, payload}` и передаются реализации игры.
8. Реализация игры отправляет сериализованные события через callback-функции сессии.
9. При отключении соединение удаляется из активной сессии, а mapping WebSocket-to-session удаляется.

### Ответственность модуля

- Runtime lifecycle активных игровых сессий.
- Конкурентный registry игроков с защитой через `sync.RWMutex`.
- Подключаемые реализации игр через factory registry.
- Маршрутизация WebSocket-действий и targeted/broadcast-доставка.
- Хранение метаданных сессий в MongoDB.
- Опциональная интеграция с WebRTC signaling для conference rooms.

## 🏗 Архитектура

```text
┌──────────────────────────────────────────────────────────────────┐
│ Клиент                                                           │
│ WebSocket / Socket.IO events + опциональный WebRTC signaling     │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ Delivery layer                                                   │
│ Fiber routes:                                                    │
│   GET /api/session/ws/:game_name/:session_id/:user_id            │
│   GET /api/game/sessions                                         │
│ Socket.IO: connect, message, game_action, disconnect             │
└──────────────────────────────┬───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│ Session / business layer                                         │
│ Hub → GameSession → Game interface → зарегистрированная логика    │
│ In-memory maps + RWMutexes + lifecycle игроков + fan-out          │
└───────────────┬───────────────────────────────┬──────────────────┘
                │                               │
                ▼                               ▼
┌─────────────────────────────┐  ┌────────────────────────────────┐
│ Data / integration layer     │  │ Conference layer                │
│ MongoDB: game_sessions       │  │ WebRTC PeerConnection           │
│ MongoDB: game_configs        │  │ SDP offer/answer + ICE          │
│ PostgreSQL: исходные сессии  │  │ Опционально, CONFERENCE_ENABLED │
└─────────────────────────────┘  └────────────────────────────────┘
```

### Слои приложения

| Слой | Реализация | Ответственность |
|---|---|---|
| Delivery | `game_session/delivery/http`, `game_session/delivery/ws` | Регистрирует Fiber routes, переводит WebSocket-соединения в рабочий режим, разбирает envelopes и передаёт actions. |
| Session orchestration | `game_session/usecase/hub.go` | Активирует сессии, координирует repositories, создаёт game instances и поддерживает process-local map активных сессий. |
| Domain entities | `game_session/entity` | Описывает сохранённые метаданные сессии, активные сессии, соединения, maps игроков и synchronized access. |
| Game contract | `game_session/contracts` | Определяет `Game`, `Action`, `GameFactory` и callback-контракты для всех игр. |
| Game registry | `game_session/usecase/registry` | Связывает имя игры с factory и settings provider; защищает registry через `RWMutex`. |
| Persistence | `game_session/infrastructure/repository` | Читает и записывает MongoDB `game_sessions`; преобразует database models в domain entities. |
| Source-session integration | `game_session/infrastructure/trash` | Получает исходную сессию и host из PostgreSQL-backed application data. |
| Game implementations | `games/sales_courage`, `submodules/deckboard` | Загружает настройки игры и обрабатывает game-specific actions и events. |
| Conference integration | `conference` | Создаёт WebRTC peer connections, обрабатывает SDP/ICE signaling и управляет media rooms. |

### Технологический стек и интеграции

| Категория | Технология / компонент |
|---|---|
| Язык | Go 1.23.4 |
| HTTP-сервер | Fiber v2 |
| Real-time transport | `github.com/gofiber/contrib/socketio` поверх WebSocket |
| Хранение сессий | MongoDB (`game_sessions`, `game_configs`) через MongoDB Go Driver v2 |
| Хранилище исходных сессий | PostgreSQL через GORM; доступ через trash repository |
| Game engine | Внутренний интерфейс `Game`, registry/factory pattern, Deckboard submodule |
| Conference media | Pion WebRTC v4; SDP offer/answer и ICE candidate signaling |
| Конфигурация | Environment variables через Viper и базовая YAML-конфигурация logger |
| Concurrency | Go goroutines, `sync.RWMutex`, synchronized maps игроков и сессий |
| Логирование | Project logger и Fiber logging |

## 💬 Примеры взаимодействия

### 1. Получение сохранённых игровых сессий

```bash
$ curl http://localhost:6069/api/game/sessions
[
  {
    "id": "42",
    "gameName": "sales_courage",
    "hostId": "user_host_1"
  }
]
```

Endpoint читает данные из MongoDB collection `game_sessions`. Создание сессии через REST сейчас отключено: в текущей реализации route работает только на чтение.

### 2. Подключение к сессии и отправка game action

```text
Клиент → WebSocket: ws://localhost:6069/api/session/ws/sales_courage/42/user_17

Сервер → Клиент:
{"message":"Добро пожаловать в игру!"}

Клиент → Socket.IO event: game_action
{
  "type": "add_coins",
  "payload": {
    "player_id": "user_17",
    "coins": 10
  }
}

Сервер:
- находит session 42;
- передаёт action в sales_courage;
- применяет host-only rule, реализованное игрой;
- отправляет результирующие game events через callbacks сессии.
```

Общий формат action:

```json
{
  "type": "<action_name>",
  "payload": {}
}
```

### 3. WebRTC conference signaling через то же соединение

Если `CONFERENCE_ENABLED=true`, WebSocket-соединение игровой сессии также подключается к conference handler.

```json
Клиент → Сервер
{
  "event": "offer",
  "data": "{\"type\":\"offer\",\"sdp\":\"...\"}"
}

Сервер → Клиент
{
  "event": "answer",
  "data": "{\"type\":\"answer\",\"sdp\":\"...\"}"
}
```

Тот же signaling channel поддерживает events `candidate` и `answer`. Conference layer создаёт send/receive transceivers для аудио и видео и публикует список участников после установки соединения.

## 🚀 Запуск локально

### Системные требования

- Go **1.23.4** или совместимый toolchain Go 1.23.
- MongoDB с доступом к настроенной database.
- PostgreSQL со схемой приложения, которую использует trash repository.
- Docker и Docker Compose опциональны. Включённый Compose-файл собирает только backend и ожидает внешние databases.
- Документ конфигурации выбранной зарегистрированной игры, например `sales_courage`, в MongoDB.

### 1. Клонирование и установка зависимостей

```bash
git clone <repository-url>
cd gamler_conference
go mod download
```

### 2. Конфигурация окружения

Создайте локальный `.env`. Не копируйте production credentials и не коммитьте secrets.

```dotenv
HTTP_HOST=localhost
HTTP_PORT=6069
DATABASE_URL=mongodb://localhost:27017
DATABASE_NAME=gamer_defi_local
POSTGRES_URL=postgresql://user:password@localhost:5432/gamler
CONFERENCE_ENABLED=false

# Требуется общему configuration loader приложения; используйте local test values.
TON_CONNECT=https://ton-blockchain.github.io/global.config.json
PLATFORM_SMART_CONTRACT=<local-test-value>
SMART_CONTRACT_JETTON_WALLET=<local-test-value>
TARGET_JETTON_MASTER=<local-test-value>
CONTRACT_ADMIN=<local-test-value>
WALLET_SEED=<local-test-seed>
PRIVATE_KEY=<local-test-private-key>
PUBLIC_KEY=<local-test-public-key>
```

Модуль также использует настройки logger из `src/config/configs/base.yaml`. Configuration loader репозитория проверяет несколько application-wide полей до запуска сервера.

### 3. Standalone-запуск

Dockerfile собирает `./src/core` — entry point приложения для container workflow. После настройки зависимостей этот же entry point можно запускать локально:

```bash
go run ./src/core
```

### 4. Запуск через Docker Compose

```bash
docker compose up --build
```

Предоставленная Compose-конфигурация открывает порт `8080` и использует host networking. При необходимости измените port или network mode для локального окружения. MongoDB и PostgreSQL не объявлены как Compose services, поэтому должны быть доступны отдельно.

## ⚠️ Обработка ошибок и edge cases

| Случай | Текущее поведение |
|---|---|
| Некорректный `session_id` | `strconv.ParseUint` отклоняет значение; activation завершается с ошибкой, WebSocket закрывается. |
| Исходная сессия не найдена | Ошибка trash repository возвращается наружу; во время activation соединение закрывается. |
| Документ сессии отсутствует в MongoDB | Hub создаёт новый документ `game_sessions`, используя host исходной сессии и запрошенное имя игры. |
| Неизвестное имя игры | Registry возвращает `game '<name>' not found`; activation завершается с ошибкой. |
| Отсутствует game configuration | Game factory возвращает `game config not found`; activation завершается с ошибкой. |
| Некорректный WebSocket envelope или action JSON | Handler отправляет текст ошибки в текущее socket-соединение и не dispatch-ит invalid action. |
| Action от неизвестного socket | Handler отправляет `session not found`. |
| Отключение игрока | Игрок удаляется из активной сессии, UUID mapping удаляется. Вызываются game-level callbacks удаления. |
| Пустой список игроков / target player отсутствует | Broadcast-функции завершаются без отправки; targeted delivery возвращает `игрок не найден в сессии`. |
| Concurrent access | Hub и player maps защищены `sync.RWMutex`; activation сессий сериализуется через mutex hub map. |
| Ошибка WebRTC signaling | Некорректные SDP/ICE данные, отсутствующая room/peer connection и неверное signaling state логируются, сообщение игнорируется. |
| Authorization | Предусмотренная проверка доступа находится в виде commented code в `ActiveteSession`; текущий модуль не проверяет authorization пользователя. |
| Panic/recover | Локальная граница `recover` в модуле не реализована. Panic не преобразуется этим модулем в protocol error. |
| Transactions и limits | Создание сессии использует MongoDB `InsertOne`; cross-database transaction и явные player/rate limits в модуле не реализованы. |
| Secrets | Runtime credentials должны передаваться через environment configuration; production values нельзя копировать в документацию или source control. |

## 📁 Структура проекта

```text
src/modules/hub/
├── conference/                         # Интеграция WebRTC conference
│   ├── entity/                         # Rooms, users, connections, tracks
│   ├── handler/                        # Socket.IO и SDP/ICE handlers
│   ├── service/                        # Сервисы lifecycle rooms и peers
│   └── util/                           # Типы signaling messages и helpers
├── game_session/                       # Основной real-time модуль игровых сессий
│   ├── contracts/                      # Интерфейсы games и actions
│   ├── delivery/
│   │   ├── http/                       # GET /api/game/sessions
│   │   └── ws/                         # WebSocket route и event handlers
│   ├── dto/                            # Transport DTOs
│   ├── entity/                         # Сущности session, hub и connection
│   ├── infrastructure/
│   │   ├── repository/game_session/    # MongoDB repository game_sessions
│   │   ├── repository/models/          # Persistence models MongoDB
│   │   └── trash/repository/           # PostgreSQL source-session adapter
│   ├── usecase/hub.go/                 # Orchestration активных сессий и fan-out
│   └── usecase/registry/               # Registry game factories
├── games/                              # Зарегистрированные реализации игр
│   ├── game_config/                    # MongoDB repository/model game config
│   └── sales_courage/                  # Factory и game actions Sales Courage
├── submodules/deckboard/               # Переиспользуемая логика board, players, decks и dice
└── doc.md                              # Подробные legacy notes WebSocket-протокола
```
