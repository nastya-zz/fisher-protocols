# Protos - gRPC API с HTTP Gateway

Репозиторий содержит Protocol Buffers схемы для микросервисной архитектуры с поддержкой grpc-gateway для автоматического преобразования HTTP → gRPC.

## Структура

```
protos/
├── auth_v1/          # Сервис аутентификации
├── user_v1/          # Сервис пользователей
├── post_v1/          # Сервис постов
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

## HTTP API

Все gRPC методы доступны через REST API:

- **Auth**: `/v1/auth/*`
- **Users**: `/v1/users/*`
- **Posts**: `/v1/posts/*`

Полный список endpoints в [GATEWAY_USAGE.md](./GATEWAY_USAGE.md).

## Команды Make

- `make install-deps` - установка protoc плагинов
- `make generate` - генерация Go кода для всех сервисов
- `make generate-auth-api` - генерация только auth сервиса
- `make generate-user-api` - генерация только user сервиса
- `make generate-post-api` - генерация только post сервиса

## Требования

- Go 1.24+
- protoc (Protocol Buffers compiler)
