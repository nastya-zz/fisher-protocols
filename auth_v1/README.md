# Auth Service (Сервис авторизации)

## Описание

Сервис для аутентификации и управления пользователями в системе.

## Функциональность

- Регистрация и авторизация пользователей
- JWT токены (Access & Refresh)
- Управление ролями (ADMIN/USER)
- Блокировка/удаление пользователей

## Методы API

### 1. CreateUser

Регистрация нового пользователя.

```
POST /v1/auth/register
{
  "user_info": {
    "name": "Иван Рыбаков",
    "email": "ivan@fishers.ru",
    "password": "secure123",
    "password_confirm": "secure123"
  }
}
```

**Response:** `{ "id": "uuid" }`

### 2. Login

Вход в систему.

```
POST /v1/auth/login
{
  "login": "ivan@fishers.ru",
  "password": "secure123"
}
```

**Response:**

```json
{
  "refresh_token": "jwt_token",
  "role": "USER"
}
```

### 3. GetAccessToken

Получение access токена по refresh токену.

```
POST /v1/auth/access
{
  "refresh_token": "jwt_refresh_token"
}
```

**Response:** `{ "access_token": "jwt_access_token" }`

### 4. GetRefreshToken

Обновление refresh токена.

```
POST /v1/auth/refresh
{
  "refresh_token": "old_jwt_refresh_token"
}
```

**Response:** `{ "refresh_token": "new_jwt_refresh_token" }`

### 5. Logout

Выход из системы.

```
POST /v1/auth/logout
{
  "refresh_token": "jwt_refresh_token"
}
```

### 6. Check

Проверка прав доступа к endpoint.

```
GET /v1/auth/check?endpoint_address=/v1/posts
```

### 7. ValidateToken (Internal)

Валидация JWT токена и извлечение claims.

```protobuf
rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse)
```

**Используется:** другими сервисами для проверки токенов.

## Административные методы

### 8. UpdateUser

Обновление данных пользователя.

```
PUT /v1/auth/users/{id}
```

### 9. BlockUser

Блокировка пользователя.

```
POST /v1/auth/users/{id}/block
```

### 10. DeleteUser

Удаление пользователя.

```
DELETE /v1/auth/users/{id}
```

### 11. UpdateUserRole

Изменение роли пользователя.

```
PUT /v1/auth/users/{id}/role
{
  "role": "ADMIN"
}
```

## Модели данных

### UserInfo

```protobuf
message UserInfo {
  string name = 1;
  string email = 2;
  string password = 3;
  string password_confirm = 4;
  Role role = 5;
  bool is_verified = 6;
}
```

### Role

```protobuf
enum Role {
  ADMIN = 0;  // Администратор
  USER = 1;   // Обычный пользователь
}
```

### Claims

```protobuf
message Claims {
  string id = 1;    // User ID
  string role = 2;  // Роль пользователя
}
```

## Технический стек

**Рекомендуется:**

- **JWT:** библиотека golang-jwt/jwt
- **Хранение токенов:** Redis (refresh tokens)
- **Хеширование паролей:** bcrypt
- **Валидация:** validator/v10

## Безопасность

**Access Token:**

- TTL: 15 минут
- Используется для авторизации запросов

**Refresh Token:**

- TTL: 30 дней
- Хранится в Redis с возможностью отзыва
- Используется для получения новых access токенов

**Password:**

- Минимум 8 символов
- Хеширование bcrypt (cost 10)

## Интеграция с другими сервисами

### Архитектура с grpc-gateway

```
Клиент (Mobile/Web)
    |
    | HTTP + JWT в Header (Authorization: Bearer <token>)
    |
    v
grpc-gateway
    |
    | 1. Извлекает токен из HTTP Header
    | 2. Вызывает Auth.ValidateToken() → получает Claims
    | 3. Добавляет user_id в gRPC metadata
    |
    v
Сервисы (User/Post/Feed)
    |
    | Получают user_id из metadata
    | Выполняют бизнес-логику
```

**Flow авторизации:**

1. **Клиент** отправляет HTTP запрос с JWT:

```http
GET /v1/feed?limit=20
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

2. **grpc-gateway** перехватывает запрос:

```go
// В gateway middleware
func AuthMiddleware(authClient auth_v1.AuthV1Client) grpc.UnaryServerInterceptor {
  return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) {
    // Извлечь токен из HTTP Authorization header
    md, _ := metadata.FromIncomingContext(ctx)
    token := extractToken(md)

    // Валидация через Auth Service
    resp, err := authClient.ValidateToken(ctx, &auth_v1.ValidateTokenRequest{
      Token: token,
    })
    if err != nil {
      return nil, status.Errorf(codes.Unauthenticated, "invalid token")
    }

    // Добавить claims в контекст для downstream сервисов
    ctx = metadata.AppendToOutgoingContext(ctx,
      "user-id", resp.Claims.Id,
      "user-role", resp.Claims.Role,
    )

    return handler(ctx, req)
  }
}
```

3. **Feed/User/Post Service** получает user_id:

```go
// В методе сервиса
func (s *Server) GetFeed(ctx context.Context, req *feed_v1.GetFeedRequest) (*feed_v1.GetFeedResponse, error) {
  // Извлечь user_id из metadata
  md, _ := metadata.FromIncomingContext(ctx)
  userID := md.Get("user-id")[0]

  // Использовать для бизнес-логики
  feed := s.feedService.GetUserFeed(userID, req)
  return feed, nil
}
```

## Генерация кода

```bash
make generate-auth-api
# или
make generate
```

## Roadmap

- [x] v1: Базовая авторизация JWT
- [x] v2: Роли (ADMIN/USER)
- [x] v3: Блокировка пользователей
- [ ] v4: OAuth2 (Google, VK)
- [ ] v5: 2FA (Two-Factor Authentication)
- [ ] v6: Rate limiting по IP
