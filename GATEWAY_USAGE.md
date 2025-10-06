# Использование grpc-gateway

## Что было настроено

1. **Proto файлы** дополнены HTTP аннотациями `google.api.http`
2. **Makefile** настроен для генерации gateway кода
3. **googleapis** скачаны в `third_party/googleapis`
4. **Сгенерированы файлы**: `*.pb.gw.go` - HTTP-обработчики для каждого сервиса

## Примеры HTTP маршрутов

### AuthV1

- `POST /v1/auth/register` → CreateUser
- `POST /v1/auth/login` → Login
- `POST /v1/auth/refresh` → GetRefreshToken
- `POST /v1/auth/access` → GetAccessToken
- `GET /v1/auth/check` → Check
- `POST /v1/auth/users/{id}/block` → BlockUser
- `DELETE /v1/auth/users/{id}` → DeleteUser
- `PUT /v1/auth/users/{id}/role` → UpdateUserRole

### UserV1

- `GET /v1/users/{id}/profile` → GetProfile
- `PUT /v1/users/profile` → UpdateProfile
- `GET /v1/users/{id}/settings` → GetSettings
- `PUT /v1/users/settings` → UpdateSettings
- `POST /v1/users/{user_id}/settings/reset` → ResetSettings
- `GET /v1/users/{id}/subscriptions` → GetSubscriptions
- `POST /v1/users/{user_id}/subscribe/{subscription_id}` → Subscribe
- `DELETE /v1/users/{user_id}/subscribe/{subscription_id}` → UnSubscribe
- `POST /v1/users/avatar` → UploadAvatar
- `DELETE /v1/users/{userId}/avatar` → RemoveAvatar

### PostService

- `POST /v1/posts` → CreatePost
- `POST /v1/posts/media` → UploadMedia
- `GET /v1/posts/{id}` → GetPost
- `GET /v1/posts` → GetPosts
- `PUT /v1/posts/{id}` → UpdatePost
- `DELETE /v1/posts/{id}` → DeletePost
- `POST /v1/posts/{post_id}/likes` → AddLike
- `GET /v1/posts/{post_id}/likes` → GetLikes
- `DELETE /v1/posts/{post_id}/likes` → RemoveLike
- `POST /v1/posts/{post_id}/comments` → AddComment
- `GET /v1/posts/{post_id}/comments` → GetComments
- `DELETE /v1/posts/comments/{comment_id}` → RemoveComment

## Пример кода сервера (Go)

```go
package main

import (
    "context"
    "log"
    "net"
    "net/http"

    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    authpb "protos/gen/auth_v1"
    userpb "protos/gen/user_v1"
    postpb "protos/gen/post_v1"
)

func main() {
    // 1. Запуск gRPC сервера
    go func() {
        lis, err := net.Listen("tcp", ":50051")
        if err != nil {
            log.Fatalf("failed to listen: %v", err)
        }

        grpcServer := grpc.NewServer()

        // Регистрация ваших gRPC сервисов
        // authpb.RegisterAuthV1Server(grpcServer, &yourAuthService{})
        // userpb.RegisterUserV1Server(grpcServer, &yourUserService{})
        // postpb.RegisterPostServiceServer(grpcServer, &yourPostService{})

        log.Println("gRPC server listening on :50051")
        if err := grpcServer.Serve(lis); err != nil {
            log.Fatalf("failed to serve: %v", err)
        }
    }()

    // 2. Настройка HTTP gateway
    ctx := context.Background()
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}

    // Регистрация gateway handlers (они проксируют HTTP → gRPC)
    err := authpb.RegisterAuthV1HandlerFromEndpoint(ctx, mux, "localhost:50051", opts)
    if err != nil {
        log.Fatalf("failed to register auth gateway: %v", err)
    }

    err = userpb.RegisterUserV1HandlerFromEndpoint(ctx, mux, "localhost:50051", opts)
    if err != nil {
        log.Fatalf("failed to register user gateway: %v", err)
    }

    err = postpb.RegisterPostServiceHandlerFromEndpoint(ctx, mux, "localhost:50051", opts)
    if err != nil {
        log.Fatalf("failed to register post gateway: %v", err)
    }

    // 3. Запуск HTTP сервера
    log.Println("HTTP gateway listening on :8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

## Пример HTTP запроса

```bash
# Создание поста
curl -X POST http://localhost:8080/v1/posts \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user123",
    "description": "Отличная рыбалка!",
    "location": {
      "latitude": 55.7558,
      "longitude": 37.6173
    },
    "media": ["media-id-1", "media-id-2"]
  }'

# Получение поста
curl http://localhost:8080/v1/posts/post-id-123

# Добавление лайка
curl -X POST http://localhost:8080/v1/posts/post-id-123/likes \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user123"
  }'
```

## Регенерация кода

После изменения proto файлов:

```bash
make generate
```

## Как работает grpc-gateway

1. HTTP запрос приходит на gateway (порт 8080)
2. Gateway декодирует JSON в proto message
3. Gateway делает gRPC вызов к вашему gRPC серверу (порт 50051)
4. gRPC сервер обрабатывает запрос
5. Gateway кодирует proto response обратно в JSON
6. HTTP ответ отправляется клиенту

Это позволяет использовать один и тот же бизнес-логику как для gRPC, так и для REST API клиентов.
