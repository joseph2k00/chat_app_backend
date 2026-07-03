# Chat App Backend

A REST + WebSocket API backend for a real-time chat application, built with **Laravel 12**. It supports JWT-based authentication, private/group conversations, real-time message delivery over WebSockets, and AI-powered message translation via the OpenAI API.

## Features

- **JWT Authentication** — stateless login/register/logout via `tymon/jwt-auth`
- **Conversations** — create private conversations, list a user's conversations, fetch conversation details/messages
- **Real-time messaging** — new messages and new conversations are broadcast over private WebSocket channels (Laravel Reverb) so connected clients update instantly
- **AI message translation** — messages can be translated on demand using the OpenAI API, with a strict JSON schema response (translated text + detected source language)
- **User search** — search other users to start a new conversation
- **Form Request validation** — every endpoint validates input through dedicated `FormRequest` classes with custom rules (e.g. preventing duplicate conversations between the same two users)
- **Dockerized** — ships with a `Dockerfile` and `docker-compose.yml` (PHP-Apache + MySQL) for one-command local setup
- **CI** — GitHub Actions workflow runs on every push

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | [Laravel 12](https://laravel.com) (PHP 8.2+) |
| Auth | JWT via [tymon/jwt-auth](https://github.com/tymondesigns/jwt-auth), token revocation via [Laravel Sanctum](https://laravel.com/docs/sanctum) |
| Real-time | [Laravel Reverb](https://github.com/laravel/reverb) (WebSocket server), [Laravel Echo](https://github.com/laravel/echo) on the client side |
| AI | [OpenAI API](https://platform.openai.com/) (GPT-5) for translation |
| Database | MySQL (SQLite supported for local dev) |
| Infra | Docker, Docker Compose, GitHub Actions |

## Architecture

The codebase follows a thin-controller, service-layer pattern:

```
app/
├── Http/
│   ├── Controllers/       # Thin controllers — delegate to services
│   └── Requests/          # Form Request validation + authorization per endpoint
├── Services/               # Business logic (Auth, User, Conversation, OpenAI)
├── Models/                 # Eloquent models (Conversation, ConversationMembers, ConversationMessage, User)
├── Events/                 # Broadcast events (ShouldBroadcast) pushed over Reverb channels
├── Rules/                  # Custom validation rules (e.g. UsersNotAlreadyInConversation)
└── Enum/                   # ConversationType enum (private | group)
```

**Request flow:** `Route → FormRequest (validate/authorize) → Controller → Service (business logic + DB transaction) → Event (broadcast) → Response`

Conversation creation, for example, wraps member creation, message creation, and event dispatch in a single DB transaction (`ConversationService::createNewConversation`), so a failure midway rolls back cleanly instead of leaving orphaned records.

## API Overview

All authenticated routes require a `Bearer` JWT token (`auth:api` middleware).

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/register` | Register a new user |
| POST | `/api/login` | Log in and receive a JWT |
| POST | `/api/logout` | Invalidate the current token |
| GET | `/api/profile` | Get the authenticated user's profile |
| GET | `/api/search-users` | Search users by name/email |
| GET | `/api/conversations` | List the authenticated user's conversations |
| POST | `/api/create-conversation` | Start a new private conversation |
| GET | `/api/conversation/{conversation_id}` | Get conversation details + messages |
| POST | `/api/conversation/send-message` | Send a message in a conversation |
| POST | `/api/translate-message` | Translate a message into a target language via OpenAI |

### Real-time channels

| Channel | Event | Purpose |
|---|---|---|
| `private-message.received.{conversationId}` | `message.received` | Broadcast to conversation members when a new message arrives |
| `private-new.conversation.received.{userId}` | `new.conversation.received` | Notify a user when they're added to a new conversation |

## Getting Started

### Option 1 — Docker (recommended)

```bash
git clone <repo-url>
cd chat_app_be
cp .env.example .env
docker-compose up -d --build
docker-compose exec app php artisan key:generate
docker-compose exec app php artisan migrate
```

The API will be available at `http://localhost:8000`.

### Option 2 — Local (native PHP)

```bash
composer install
cp .env.example .env
php artisan key:generate
php artisan jwt:secret
php artisan migrate
npm install
composer run dev   # runs server, queue listener, log watcher, and vite concurrently
```

### Required environment variables

In addition to standard Laravel `.env` values, configure:

```
JWT_SECRET=
OPENAI_API_KEY=
REVERB_APP_ID=
REVERB_APP_KEY=
REVERB_APP_SECRET=
REVERB_HOST=localhost
REVERB_PORT=8080
```

## Testing

```bash
composer test
```

## License

MIT
