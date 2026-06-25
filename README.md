# Microservices Interaction Diagram

## Sync (gRPC) vs Async (Kafka)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                        │
│                           (Web / Mobile)                                    │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               GATEWAY                                       │
│                    (HTTP → gRPC → REST)                                      │
│                                                                             │
│  gRPC calls:                                                                │
│  ├── sso-service         (auth: login, register, refresh)                   │
│  ├── chat-service        (chats: create, list, delete)                      │
│  ├── message-service     (messages: send, edit, delete)                     │
│  ├── profile-service     (profile: get, update)                             │
│  ├── file-service        (files: upload, download)                          │
│  ├── permissions-service (permissions: has)                                 │
│  ├── presence-service    (online status)                                    │
│  └── search-service      (search: users, messages)                          │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        │ gRPC (sync)
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SSO-SERVICE                                       │
│                                                                             │
│  Kafka Producer:                                                            │
│  └── user.registered ─────────────────────────────────────────────────┐     │
└───────────────────────────────────────────────────────────────────────┘     │
                                  │                                            │
                                  │ Kafka                                      │
                                  ▼                                            │
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KAFKA TOPIC: user.registered                             │
│                                                                             │
│  Consumers:                                                                 │
│  ├── profile-service    (create empty profile)                              │
│  └── permissions-service (assign default role)                              │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                          CHAT-SERVICE                                       │
│                                                                             │
│  gRPC:                                                                      │
│  ├── permissions-service   (check permissions)                              │
│  └── chat-service          (CRUD chats, users)                              │
│                                                                             │
│  Kafka Producers:                                                           │
│  ├── chat.created ──────────────────────────────────────┐                   │
│  └── chat.deleted ──────────────────────────────────┐   │                   │
└────────────────────────────────────────────────────┘   │                   │
                                  │                       │                   │
                                  │ Kafka                 │ Kafka             │
                                  ▼                       ▼                   │
┌──────────────────────────┐  ┌──────────────────────────────────────────┐   │
│  TOPIC: chat.created      │  │  TOPIC: chat.deleted                     │   │
│                           │  │                                          │   │
│  Consumer:                │  │  Consumer:                                │   │
│  └── search-service       │  │  └── search-service                      │   │
│      (index chat)         │  │      (delete chat index)                 │   │
└──────────────────────────┘  └──────────────────────────────────────────┘   │


┌─────────────────────────────────────────────────────────────────────────────┐
│                         MESSAGE-SERVICE                                     │
│                                                                             │
│  gRPC:                                                                      │
│  ├── permissions-service   (check permissions)                              │
│  ├── chat-service          (validate chat membership)                       │
│  └── sessions-service      (encrypt messages for delivery)                  │
│                                                                             │
│  Kafka Producers:                                                           │
│  ├── message.created ──────────────────────────────────┐                    │
│  ├── message.updated ──────────────────────────────┐   │                    │
│  └── message.deleted ──────────────────────────┐   │   │                    │
└────────────────────────────────────────────┘   │   │   │                    │
                                  │               │   │   │                    │
                                  │ Kafka         │   │   │                    │
                                  ▼               ▼   │   │                    │
┌──────────────────────────┐  ┌──────────────────┐  │   │                    │
│  TOPIC: message.created   │  │  TOPIC: msg.upd   │  │   │                    │
│                           │  │                   │  │   │                    │
│  Consumers:               │  │  Consumer:        │  │   │                    │
│  ├── search-service       │  │  └── search-svc   │  │   │                    │
│  │   (index message)      │  │      (reindex)    │  │   │                    │
│  └── device-worker        │  └──────────────────┘  │   │                    │
│      (encrypt & deliver)  │                        │   │                    │
└──────────────────────────┘                        │   │                    │
                                                    │   │                    │
                                                    ▼   │                    │
                                          ┌──────────────────┐               │
                                          │  TOPIC: msg.del   │               │
                                          │                   │               │
                                          │  Consumer:        │               │
                                          │  └── search-svc   │               │
                                          │      (del index)  │               │
                                          └──────────────────┘               │


┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROFILE-SERVICE                                     │
│                                                                             │
│  gRPC:                                                                      │
│  └── (none — only gRPC server)                                              │
│                                                                             │
│  Kafka Consumers:                                                           │
│  └── user.registered (create empty profile)                                 │
│                                                                             │
│  Kafka Producers:                                                           │
│  └── profile.updated ──────────────────────────────────┐                    │
└────────────────────────────────────────────────────┘   │                    │
                                  │                       │                    │
                                  │ Kafka                 │ Kafka              │
                                  ▼                       ▼                    │
┌──────────────────────────┐  ┌──────────────────────────────────────────┐   │
│  TOPIC: profile.updated   │  │  (consumed by search-service)            │   │
│                           │  │  (update user index)                     │   │
│  Consumer:                │  │                                          │   │
│  └── search-service       │  └──────────────────────────────────────────┘   │
│      (index user)         │                                                │
└──────────────────────────┘                                                │


┌─────────────────────────────────────────────────────────────────────────────┐
│                        PERMISSIONS-SERVICE                                   │
│                                                                             │
│  gRPC:                                                                      │
│  └── (gRPC server only — no outbound gRPC calls)                            │
│                                                                             │
│  Kafka Consumers:                                                           │
│  └── user.registered (assign default role)                                  │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                         SEARCH-SERVICE                                       │
│                                                                             │
│  gRPC:                                                                      │
│  └── (search queries from gateway)                                          │
│                                                                             │
│  Kafka Consumers (6 topics):                                                │
│  ├── chat.created     (index chat)                                          │
│  ├── chat.deleted     (delete chat index)                                   │
│  ├── message.created  (index message)                                       │
│  ├── message.updated  (reindex message)                                     │
│  ├── message.deleted  (delete message index)                                │
│  └── profile.updated  (index/update user)                                   │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                        FILE-SERVICE                                          │
│                                                                             │
│  gRPC:                                                                      │
│  └── (file upload/download)                                                 │
│                                                                             │
│  Kafka:                                                                     │
│  └── (none)                                                                 │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                     SESSIONS-SERVICE                                         │
│                                                                             │
│  gRPC:                                                                      │
│  └── (key management, device sessions)                                      │
│                                                                             │
│  Kafka Producer:                                                            │
│  └── (session events — existing, not changed)                               │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                   NOTIFICATION-SERVICE                                       │
│                                                                             │
│  Kafka Consumer:                                                            │
│  └── (existing — chat/message events for push notifications)                │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Topic Summary

| Topic             | Producer         | Consumer(s)                          |
| -------------------| ------------------| --------------------------------------|
| `user.registered` | sso-service      | profile-service, permissions-service |
| `chat.created`    | chat-service     | search-service                       |
| `chat.deleted`    | chat-service     | search-service                       |
| `message.created` | message-service  | search-service, device-worker        |
| `message.updated` | message-service  | search-service                       |
| `message.deleted` | message-service  | search-service                       |
| `profile.updated` | profile-service  | search-service                       |
| `message-events`  | message-service  | device-worker (existing)             |
| `chat-events`     | chat-service     | notification-service (existing)      |
| `session-events`  | sessions-service | notification-service (existing)      |

## Rules

1. **gRPC** = sync request-response (auth check, CRUD, search queries)
2. **Kafka** = async fire-and-forget (indexing, notifications, side effects)
3. Events published **AFTER** successful DB write
4. Consumer groups ensure each topic processed by exactly one instance
5. Dead-letter topics (`{topic}.dlq`) for failed messages
