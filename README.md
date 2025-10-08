# Protos - gRPC API с HTTP Gateway

Репозиторий содержит Protocol Buffers схемы для микросервисной архитектуры с поддержкой grpc-gateway для автоматического преобразования HTTP → gRPC.

## Структура

```
protos/
├── auth_v1/          # Сервис аутентификации
├── user_v1/          # Сервис пользователей
├── post_v1/          # Сервис постов
├── feed_v1/          # Сервис ленты постов
├── gen/              # Сгенерированный Go код
├── bin/              # Protoc плагины
└── third_party/      # googleapis (google/api/annotations.proto)
```

## Быстрый старт

### 1. Установка зависимостей

```bash
make install-deps
```

### 2. Генерация кода

```bash
make generate
```

Создаются файлы:

- `*.pb.go` - proto message структуры
- `*_grpc.pb.go` - gRPC сервис интерфейсы
- `*.pb.gw.go` - HTTP gateway handlers ✨

### 3. Использование в сервере

См. [GATEWAY_USAGE.md](./GATEWAY_USAGE.md) для примеров кода и HTTP endpoints.

## Архитектура

### Схема взаимодействия

```
┌─────────────────┐
│  Mobile / Web   │
│     Client      │
└────────┬────────┘
         │ HTTP + JWT
         │
         v
┌─────────────────┐
│  grpc-gateway   │◄──────┐
│   (API Gateway) │       │
└────────┬────────┘       │
         │ gRPC           │ ValidateToken
         │                │
    ┌────┴─────┬──────┬───┴───┐
    v          v      v       v
┌──────┐  ┌──────┐ ┌──────┐ ┌──────┐
│ Auth │  │ User │ │ Post │ │ Feed │
└──────┘  └──────┘ └──────┘ └──────┘
```

**grpc-gateway** - единая точка входа:

- Принимает HTTP запросы от клиентов
- Извлекает JWT токен из заголовка `Authorization`
- Валидирует токен через Auth Service
- Прокидывает `user_id` в metadata к целевому сервису
- Преобразует gRPC ответы в HTTP JSON

### Авторизация

**Все защищенные endpoints требуют JWT токен:**

```http
GET /v1/feed?limit=20
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Flow:**

1. Клиент логинится → получает `access_token` от Auth Service
2. Клиент отправляет запрос с токеном в header
3. grpc-gateway валидирует токен через `Auth.ValidateToken()`
4. grpc-gateway добавляет `user_id` в gRPC metadata
5. Целевой сервис использует `user_id` из metadata

## HTTP API

Все gRPC методы доступны через REST API:

- **Auth**: `/v1/auth/*` - регистрация, логин, токены
- **Users**: `/v1/users/*` - профили, подписки, настройки
- **Posts**: `/v1/posts/*` - посты, лайки, комментарии
- **Feed**: `/v1/feed/*` - персональная лента, геолента, тренды

Полный список endpoints в [GATEWAY_USAGE.md](./GATEWAY_USAGE.md).

## Сервисы

Каждый сервис имеет детальную документацию в `<service>/README.md`:

- [auth_v1/README.md](./auth_v1/README.md) - JWT авторизация, управление пользователями
- [user_v1/README.md](./user_v1/README.md) - Профили, подписки, аватары
- [post_v1/README.md](./post_v1/README.md) - Посты, медиа, лайки, комментарии
- [feed_v1/README.md](./feed_v1/README.md) - Алгоритмы ленты, геопоиск, рекомендации

## Команды Make

- `make install-deps` - установка protoc плагинов
- `make generate` - генерация Go кода для всех сервисов
- `make generate-auth-api` - генерация только auth сервиса
- `make generate-user-api` - генерация только user сервиса
- `make generate-post-api` - генерация только post сервиса
- `make generate-feed-api` - генерация только feed сервиса

## Требования

- Go 1.24+
- protoc (Protocol Buffers compiler)
