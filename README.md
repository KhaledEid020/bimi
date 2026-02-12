# SOC Chat Platform - Full System Guide


```mermaid
erDiagram
    AUTH_USERS ||--o{ AUTH_REFRESH_TOKENS : has
    AUTH_USERS ||--o{ CHAT_SESSION_META : owns
    AUTH_USERS ||--o{ PUBLIC_SESSIONS : logical_user_id_match
    PUBLIC_SESSIONS ||--o{ PUBLIC_EVENTS : contains

    AUTH_USERS {
      uuid id PK
      string username
      string email
      string password_hash
      string role
      bool is_active
      timestamptz created_at
      timestamptz updated_at
    }

    AUTH_REFRESH_TOKENS {
      uuid id PK
      uuid user_id FK
      string token_hash
      timestamptz expires_at
      timestamptz revoked_at
      uuid replaced_by_token_id
      timestamptz created_at
    }

    CHAT_SESSION_META {
      uuid id PK
      string app_name
      uuid user_id FK
      string session_id
      string title
      bool archived
      timestamptz created_at
      timestamptz updated_at
      timestamptz last_message_at
    }

    PUBLIC_SESSIONS {
      string id
      string app_name
      string user_id
      json state
      timestamptz last_update_time
    }

    PUBLIC_EVENTS {
      string id
      string session_id
      string author
      json content
      json actions
      timestamptz timestamp
    }

```


This document explains the full system in simple language:
- what each project does
- where data is stored
- how a request moves across services
- how sessions, users, and memory are connected

## 1) Projects and Roles

### A) ADK Backend (`/home/administrator/google-agent`)
This is your core AI agent runtime.

Main file:
- `/home/administrator/google-agent/main.py`

What it does:
- Starts ADK FastAPI server (port `8080`)
- Loads agent apps from the agent directory
- Exposes ADK routes like `/list-apps`, `/run`, `/apps/{app}/users/{user}/sessions/...`
- Stores ADK sessions/events using `SESSION_SERVICE_URI`

Current ADK session storage config:
- In `main.py`, it now defaults to Postgres via env-friendly config.

### B) Chat Gateway (`/home/administrator/chat_gateway`)
This is your authenticated API in front of ADK.

Main file:
- `/home/administrator/chat_gateway/app/main.py`

What it does:
- User auth (register/login/refresh/logout)
- JWT validation + role checks
- Chat tab metadata management
- Calls ADK backend APIs for actual AI runs
- Returns final responses/events to your chat UI

### C) OpenMemory (`/home/administrator/google-agent/OpenMemory-1.2.3`)
This is long-term memory service used by ADK memory integration.

Default behavior in its own compose:
- Metadata/vector backend defaults to SQLite unless overridden.

## 2) Storage - Exactly Where Data Lives

There are **three storage categories**:

### 1) Gateway Auth + Chat Metadata (Postgres)
Stored by `chat_gateway` migrations in DB `session` (container `chat_gateway_postgres`):
- `auth.users`
- `auth.refresh_tokens`
- `chat.session_meta`

### 2) ADK Conversation Sessions + Events (Postgres)
Stored by ADK `DatabaseSessionService` using `SESSION_SERVICE_URI`.
ADK creates tables in default schema (usually `public`), such as:
- `adk_internal_metadata`
- `sessions`
- `events`
- `app_states`
- `user_states`

### 3) OpenMemory Long-Term Memory
By default in OpenMemory compose:
- SQLite file (`OM_DB_PATH`, default `/data/openmemory.sqlite`)
- unless switched to Postgres via OpenMemory env vars.

## 3) Database-Level Relationship Between Gateway and ADK Data

There is **no direct foreign key** from ADK tables (`public.sessions`, `public.events`) to gateway tables (`auth.*`, `chat.*`).

The relationship is **logical** and enforced by application behavior:
- Gateway sets `adk_user_id = JWT.sub` (user UUID string)
- Gateway calls ADK with:
  - `app_name`
  - `user_id` (from JWT)
  - `session_id`

So records align by values, not DB FK constraints.

## 4) End-to-End Request Flow (Simple)

1. User logs in to gateway (`/v1/auth/login`)
2. Gateway returns JWTs
3. User opens/creates chat tab (`/v1/chats`)
4. Gateway creates ADK session (`/apps/{app}/users/{user}/sessions`)
5. User sends message (`/v1/chats/{session_id}/messages`)
6. Gateway calls ADK `/run` (or `/run_sse`)
7. ADK writes session/events to its session tables
8. Gateway updates `chat.session_meta.last_message_at`
9. Response returns to UI
10. ADK callback can push session to OpenMemory for long-term memory

## 4A) Login Dataflow Diagram (Files + Tables + Columns)

```mermaid
sequenceDiagram
    autonumber
    participant UI as Chat UI
    participant AuthAPI as chat_gateway/app/routers/auth.py::login_user()
    participant Sec as chat_gateway/app/security.py
    participant PG as Postgres (session DB)
    participant U as auth.users
    participant RT as auth.refresh_tokens

    UI->>AuthAPI: POST /v1/auth/login {email,password}
    AuthAPI->>U: SELECT by email (email, password_hash, is_active, role, id)
    U-->>AuthAPI: user row
    AuthAPI->>Sec: verify_password(password, password_hash)
    Sec-->>AuthAPI: true/false

    alt valid credentials + active user
        AuthAPI->>Sec: create_access_token(sub=id, role)
        Sec-->>AuthAPI: access_token (+ exp)
        AuthAPI->>Sec: create_refresh_token(sub=id, jti=uuid)
        Sec-->>AuthAPI: refresh_token (+ exp)
        AuthAPI->>Sec: hash_token(refresh_token)
        Sec-->>AuthAPI: token_hash
        AuthAPI->>RT: INSERT (id, user_id, token_hash, expires_at, revoked_at, replaced_by_token_id)
        RT-->>AuthAPI: committed
        AuthAPI-->>UI: 200 {access_token, refresh_token, expires_in, user{id,email,role}}
    else invalid credentials/inactive
        AuthAPI-->>UI: 401 / 403
    end
```

### Login Path - Exact Files
- API entrypoint: `app/routers/auth.py:95` (`login_user`)
- Password verify: `app/security.py:29`
- Token issue: `app/security.py:39`, `app/security.py:52`
- Refresh row insert: `app/routers/auth.py:42`, `app/routers/auth.py:47`

### Login Path - Exact Tables/Columns
- Read: `auth.users`
  - `id`, `email`, `password_hash`, `role`, `is_active`
- Write: `auth.refresh_tokens`
  - `id`, `user_id`, `token_hash`, `expires_at`, `revoked_at`, `replaced_by_token_id`, `created_at`

## 4B) After Login: Chat/Session Dataflow Diagram

```mermaid
sequenceDiagram
    autonumber
    participant UI as Chat UI
    participant Deps as chat_gateway/app/deps.py
    participant ChatsAPI as chat_gateway/app/routers/chats.py
    participant Meta as chat.session_meta
    participant ADKClient as chat_gateway/app/adk_client.py
    participant ADK as ADK API (:8080)
    participant ADKDB as public.sessions/public.events

    UI->>Deps: Authorization: Bearer <access_token>
    Deps->>Deps: decode JWT -> sub(user_id), role
    Deps-->>ChatsAPI: current user

    UI->>ChatsAPI: POST /v1/chats
    ChatsAPI->>ADKClient: create_session(user_id=sub, session_id=generated)
    ADKClient->>ADK: POST /apps/{app}/users/{user_id}/sessions/{session_id}
    ADK-->>ADKClient: session created
    ChatsAPI->>Meta: INSERT (app_name, user_id, session_id, title, archived, last_message_at)
    ChatsAPI-->>UI: Chat tab created

    UI->>ChatsAPI: POST /v1/chats/{session_id}/messages
    ChatsAPI->>ADKClient: run_agent(app_name,user_id,session_id,new_message)
    ADKClient->>ADK: POST /run or /run_sse
    ADK->>ADKDB: write session/events
    ADK-->>ADKClient: events/final response
    ChatsAPI->>Meta: UPDATE last_message_at, updated_at
    ChatsAPI-->>UI: assistant response
```

### Notes
- Gateway does not directly write ADK `public.events`.
- Gateway writes auth/chat metadata; ADK writes conversation history.
- Cross-system link is value-based: `user_id` + `session_id` + `app_name`.

## 5) Active Runtime Ports and Services

Typical local runtime:
- ADK backend: `http://localhost:8080`
- Chat gateway: `http://localhost:8090`
- Gateway Postgres exposed host port: `5433` (container internal `5432`)
- OpenMemory: `http://localhost:8081`

## 6) Key Config Files

ADK backend:
- `/home/administrator/google-agent/main.py`
- `/home/administrator/google-agent/Dockerfile`

Gateway:
- `/home/administrator/chat_gateway/docker-compose.yml`
- `/home/administrator/chat_gateway/.env`
- `/home/administrator/chat_gateway/.env.example`
- `/home/administrator/chat_gateway/alembic/versions/0001_create_auth_and_chat_schemas.py`

OpenMemory:
- `/home/administrator/google-agent/OpenMemory-1.2.3/docker-compose.yml`

## 7) Gateway API Surface

Auth:
- `POST /v1/auth/register`
- `POST /v1/auth/login`
- `POST /v1/auth/refresh`
- `POST /v1/auth/logout`
- `GET /v1/me`

Chats:
- `GET /v1/chats?archived=false`
- `POST /v1/chats`
- `GET /v1/chats/{session_id}`
- `PATCH /v1/chats/{session_id}`
- `POST /v1/chats/{session_id}/messages`

Health:
- `GET /healthz`

## 8) How to Verify Data in Postgres

Enter DB:
```bash
docker exec -it chat_gateway_postgres psql -U postgres -d session
```

Check schemas:
```sql
\dn
```

Gateway tables:
```sql
\dt auth.*
\dt chat.*
```

ADK tables:
```sql
\dt public.*
```

Quick row counts:
```sql
SELECT count(*) FROM auth.users;
SELECT count(*) FROM chat.session_meta;
SELECT count(*) FROM public.sessions;
SELECT count(*) FROM public.events;
```

## 9) Docker Notes (Important)

Gateway compose overrides env to avoid common Docker mistakes:
- `DATABASE_URL` uses service DNS `postgres`
- `ADK_BASE_URL` uses `host.docker.internal:8080`
- `ADK_APP_NAME` is set to `my_agent` (must exist in ADK `/list-apps`)

If ADK app name mismatch happens, gateway startup fails with:
- `Configured ADK app '...' not found. Available apps: [...]`

## 10) Common Problems and Fixes

### Problem: Gateway cannot connect to DB
Cause:
- `DATABASE_URL` points to `localhost` inside container
Fix:
- Use `postgres` as host in container network

### Problem: ADK app validation fails on startup
Cause:
- `ADK_APP_NAME` not matching `/list-apps`
Fix:
- Set `ADK_APP_NAME` to actual app name (currently `my_agent`)

### Problem: Postgres volume/image compatibility errors
Fix:
```bash
docker compose down -v
# then up again
```

## 11) Security Model (Current)

- Password hashing: Argon2
- JWT access + refresh with rotation
- Refresh token is hashed in DB
- Chat routes require valid JWT + allowed role (`soc_analyst`/`soc_admin`)
- User identity for ADK is derived from JWT (`sub`) and cannot be overridden by frontend

## 12) What Is Not Yet Coupled by DB Constraints

Not enforced by FK today:
- `chat.session_meta` <-> `public.sessions`

They are coupled by API-level identity mapping and shared values.

## 13) Current Mental Model (One-line)

**Gateway controls identity and chat UX; ADK performs agent reasoning and stores conversation events; OpenMemory stores long-term memory.**
