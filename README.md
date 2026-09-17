🇷🇺 [Читать на русском языке](README_RU.md)

# 🎲 Gamler Hub — Real-Time Game Session Infrastructure

A Go 1.23 real-time game-session hub that activates persisted sessions, manages concurrent players, routes game actions over WebSockets, and optionally attaches WebRTC audio/video conferencing to the same session.

> **Scope:** This document describes only the implementation in `src/modules/hub`.

## 📌 Table of Contents

- [🎯 Project Goal](#-project-goal)
- [🏗 Architecture](#-architecture)
- [💬 Usage Examples](#-usage-examples)
- [🚀 Local Setup](#-local-setup)
- [⚠️ Error Handling & Edge Cases](#️-error-handling--edge-cases)
- [📁 Project Structure](#-project-structure)

## 🎯 Project Goal

The module provides the runtime boundary between a persisted game session and connected clients. It keeps active sessions in memory, instantiates game-specific logic through a registry, synchronizes player membership, and distributes game events to one player, all players, or all players except the sender.

### Data Flow

1. A client opens `GET /api/session/ws/:game_name/:session_id/:user_id` as a WebSocket connection.
2. The handler parses the numeric `session_id` and loads the source session from the PostgreSQL-backed trash repository.
3. The hub checks its in-memory registry. If the session is inactive, it loads or creates a MongoDB `game_sessions` document.
4. The game registry resolves `game_name` to a factory. The factory loads game settings from MongoDB and creates a `Game` implementation.
5. The session is initialized with broadcast callbacks and stored in the process-local hub.
6. The client is added to the session. Host status is derived from the persisted `HostID`; the first connection is **not** promoted automatically by this module.
7. Incoming Socket.IO `game_action` payloads are decoded into `Action{type, payload}` and delegated to the game implementation.
8. The game implementation emits serialized events using the session delivery callbacks.
9. On disconnect, the connection is removed from the active session and the WebSocket-to-session mapping is deleted.

### Responsibilities

- Runtime lifecycle of active game sessions.
- Concurrent player registry with `sync.RWMutex` protection.
- Pluggable game implementations through a factory registry.
- WebSocket action routing and targeted/broadcast delivery.
- Persistence of session metadata in MongoDB.
- Optional WebRTC signaling integration for conference rooms.

## 🏗 Architecture

```text
┌──────────────────────────────────────────────────────────────────┐
│ Client                                                           │
│ WebSocket / Socket.IO events + optional WebRTC signaling         │
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
│ Hub → GameSession → Game interface → registered game logic       │
│ In-memory maps + RWMutexes + player lifecycle + fan-out          │
└───────────────┬───────────────────────────────┬──────────────────┘
                │                               │
                ▼                               ▼
┌─────────────────────────────┐  ┌────────────────────────────────┐
│ Data / integration layer     │  │ Conference layer                │
│ MongoDB: game_sessions       │  │ WebRTC PeerConnection           │
│ MongoDB: game_configs        │  │ SDP offer/answer + ICE          │
│ PostgreSQL: source sessions  │  │ Optional, CONFERENCE_ENABLED    │
└─────────────────────────────┘  └────────────────────────────────┘
```

### Application Layers

| Layer | Implementation | Responsibility |
|---|---|---|
| Delivery | `game_session/delivery/http`, `game_session/delivery/ws` | Registers Fiber routes, upgrades WebSocket connections, decodes envelopes, and forwards actions. |
| Session orchestration | `game_session/usecase/hub.go` | Activates sessions, coordinates repositories, creates game instances, and maintains the process-local active-session map. |
| Domain entities | `game_session/entity` | Defines persisted session metadata, active sessions, connections, player maps, and synchronized access. |
| Game contract | `game_session/contracts` | Defines `Game`, `Action`, `GameFactory`, and callback contracts used by all games. |
| Game registry | `game_session/usecase/registry` | Maps a game name to a factory and settings provider; protects the registry with an `RWMutex`. |
| Persistence | `game_session/infrastructure/repository` | Reads and writes MongoDB `game_sessions`; converts database models to domain entities. |
| Source-session integration | `game_session/infrastructure/trash` | Resolves the source session and host from PostgreSQL-backed application data. |
| Game implementations | `games/sales_courage`, `submodules/deckboard` | Loads game settings and handles game-specific actions and events. |
| Conference integration | `conference` | Attaches WebRTC peer connections, processes SDP/ICE signaling, and manages media rooms. |

### Technology Stack and Integrations

| Category | Technology / component |
|---|---|
| Language | Go 1.23.4 |
| HTTP server | Fiber v2 |
| Real-time transport | `github.com/gofiber/contrib/socketio` over WebSocket |
| Session persistence | MongoDB (`game_sessions`, `game_configs`) via MongoDB Go Driver v2 |
| Source-session storage | PostgreSQL via GORM; accessed by the trash repository |
| Game engine | Internal `Game` interface, registry/factory pattern, Deckboard submodule |
| Conference media | Pion WebRTC v4; SDP offer/answer and ICE candidate signaling |
| Configuration | Environment variables loaded with Viper plus base YAML logger configuration |
| Concurrency | Go goroutines, `sync.RWMutex`, synchronized player/session maps |
| Logging | Project logger and Fiber logging |

## 💬 Usage Examples

### 1. List persisted game sessions

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

The endpoint reads from the MongoDB `game_sessions` collection. Session creation through REST is currently not enabled; the route is read-only in the current implementation.

### 2. Join a session and send a game action

```text
Client → WebSocket: ws://localhost:6069/api/session/ws/sales_courage/42/user_17

Server → Client:
{"message":"Добро пожаловать в игру!"}

Client → Socket.IO event: game_action
{
  "type": "add_coins",
  "payload": {
    "player_id": "user_17",
    "coins": 10
  }
}

Server:
- resolves session 42;
- routes the action to sales_courage;
- applies the host-only rule implemented by the game;
- emits resulting game events through the session callbacks.
```

The generic action shape is:

```json
{
  "type": "<action_name>",
  "payload": {}
}
```

### 3. WebRTC conference signaling in the same connection

When `CONFERENCE_ENABLED=true`, the session WebSocket is also attached to the conference handler.

```json
Client → Server
{
  "event": "offer",
  "data": "{\"type\":\"offer\",\"sdp\":\"...\"}"
}

Server → Client
{
  "event": "answer",
  "data": "{\"type\":\"answer\",\"sdp\":\"...\"}"
}
```

The same signaling channel supports `candidate` and `answer` events. The conference layer creates audio/video send-receive transceivers and publishes a participant list when the connection is established.

## 🚀 Local Setup

### Requirements

- Go **1.23.4** or a compatible Go 1.23 toolchain.
- MongoDB, with access to the configured database.
- PostgreSQL, with the application schema required by the trash repository.
- Docker and Docker Compose are optional. The included Compose file builds only the backend and expects external databases.
- A game configuration document for the selected registered game, for example `sales_courage`, in MongoDB.

### 1. Clone and install dependencies

```bash
git clone <repository-url>
cd gamler_conference
go mod download
```

### 2. Configure the environment

Create a local `.env` file. Do not copy production credentials or commit secrets.

```dotenv
HTTP_HOST=localhost
HTTP_PORT=6069
DATABASE_URL=mongodb://localhost:27017
DATABASE_NAME=gamer_defi_local
POSTGRES_URL=postgresql://user:password@localhost:5432/gamler
CONFERENCE_ENABLED=false

# Required by the wider application configuration loader; use local test values.
TON_CONNECT=https://ton-blockchain.github.io/global.config.json
PLATFORM_SMART_CONTRACT=<local-test-value>
SMART_CONTRACT_JETTON_WALLET=<local-test-value>
TARGET_JETTON_MASTER=<local-test-value>
CONTRACT_ADMIN=<local-test-value>
WALLET_SEED=<local-test-seed>
PRIVATE_KEY=<local-test-private-key>
PUBLIC_KEY=<local-test-public-key>
```

The module also reads logger defaults from `src/config/configs/base.yaml`. The repository's configuration loader validates several application-wide fields before the server starts.

### 3. Standalone run

The Dockerfile builds `./src/core`, which is the application entry point used by the repository's container workflow. Run the same entry point locally after configuring dependencies:

```bash
go run ./src/core
```

### 4. Run with Docker Compose

```bash
docker compose up --build
```

The provided Compose configuration exposes port `8080` and uses host networking. Adjust the port or network mode if your local environment requires it. MongoDB and PostgreSQL are not defined as Compose services, so they must be available separately.

## ⚠️ Error Handling & Edge Cases

| Case | Current behavior |
|---|---|
| Invalid `session_id` | `strconv.ParseUint` rejects the value; activation fails and the WebSocket is closed. |
| Missing source session | The trash repository error is returned; the connection is closed during activation. |
| Missing MongoDB session document | The hub creates a new `game_sessions` document using the source session host and requested game name. |
| Unknown game name | The registry returns `game '<name>' not found`; activation fails. |
| Missing game configuration | A game factory returns `game config not found`; activation fails. |
| Malformed WebSocket envelope or action JSON | The handler emits the error text to the current socket and does not dispatch the invalid action. |
| Action for an unmapped socket | The handler emits `session not found`. |
| Player disconnect | The player is removed from the active session and the UUID mapping is deleted. Game-level removal callbacks are invoked. |
| Empty player set / target player absent | Broadcast functions return without sending; targeted delivery returns `игрок не найден в сессии`. |
| Concurrent access | Hub and player maps are guarded with `sync.RWMutex`; session activation is serialized per hub map. |
| WebRTC signaling failure | Invalid SDP/ICE data, missing room/peer connection, and invalid signaling state are logged and ignored for that message. |
| Authorization | The intended access check exists as commented code in `ActiveteSession`; the current module does not enforce user authorization. |
| Panic/recover | No module-local `recover` boundary is implemented. Panics are not converted into protocol errors by this module. |
| Transactions and limits | Session creation uses MongoDB `InsertOne`; no cross-database transaction or explicit player/rate limit is implemented in this module. |
| Secrets | Runtime credentials must be supplied through environment configuration; production values must not be copied into local documentation or source control. |

## 📁 Project Structure

```text
src/modules/hub/
├── conference/                         # WebRTC conference integration
│   ├── entity/                         # Rooms, users, connections, tracks
│   ├── handler/                        # Socket.IO and SDP/ICE handlers
│   ├── service/                        # Room and peer lifecycle services
│   └── util/                           # Signaling message types and helpers
├── game_session/                       # Core real-time game-session module
│   ├── contracts/                      # Game and action interfaces
│   ├── delivery/
│   │   ├── http/                       # GET /api/game/sessions
│   │   └── ws/                         # WebSocket route and event handlers
│   ├── dto/                            # Transport DTOs
│   ├── entity/                         # Session, hub, and connection entities
│   ├── infrastructure/
│   │   ├── repository/game_session/    # MongoDB game_sessions repository
│   │   ├── repository/models/          # MongoDB persistence models
│   │   └── trash/repository/           # PostgreSQL source-session adapter
│   ├── usecase/hub.go/                 # Active-session orchestration and fan-out
│   └── usecase/registry/               # Game factory registry
├── games/                              # Registered game implementations
│   ├── game_config/                    # MongoDB game-config repository/model
│   └── sales_courage/                  # Sales Courage factory and game actions
├── submodules/deckboard/               # Reusable board, player, deck, and dice logic
└── doc.md                              # Detailed legacy WebSocket protocol notes
```
