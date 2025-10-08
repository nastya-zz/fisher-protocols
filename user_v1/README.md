# User Service (Сервис пользователей)

## Описание

Сервис для управления профилями пользователей, настройками и подписками.

## Функциональность

- Управление профилями пользователей
- Настройки аккаунта (язык, приватность)
- Система подписок (subscriptions/subscribers)
- Загрузка и управление аватарами

## Методы API

### 1. GetProfile

Получение профиля пользователя.

```
GET /v1/users/{id}/profile
```

**Response:**

```json
{
  "profile": {
    "id": "uuid",
    "name": "Иван Рыбаков",
    "email": "ivan@fishers.ru",
    "avatar_path": "/avatars/uuid.jpg",
    "bio": "Рыбачу 10 лет, люблю щуку на спиннинг",
    "experience_level": 3,
    "is_verified": true,
    "is_public": true,
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

### 2. UpdateProfile

Обновление профиля.

```
PUT /v1/users/profile
{
  "info": {
    "id": "uuid",
    "name": "Новое имя",
    "bio": "Обновленная биография",
    "is_public": false
  }
}
```

**Поля (опциональные):** name, email, bio, avatar_path, is_public

### 3. GetSettings

Получение настроек аккаунта.

```
GET /v1/users/{id}/settings
```

**Response:**

```json
{
  "settings": {
    "language": "RU",
    "availability": "PUBLIC"
  }
}
```

### 4. UpdateSettings

Обновление настроек.

```
PUT /v1/users/settings
{
  "user_id": "uuid",
  "settings_info": {
    "language": "ENG",
    "availability": "PRIVATE"
  }
}
```

### 5. ResetSettings

Сброс настроек к значениям по умолчанию.

```
POST /v1/users/{user_id}/settings/reset
```

## Система подписок

### 6. GetSubscriptions

Получение подписок и подписчиков.

```
GET /v1/users/{id}/subscriptions
```

**Response:**

```json
{
  "subscriptions": [
    {
      "id": "uuid1",
      "name": "Петр Карпов",
      "avatar_path": "/avatars/uuid1.jpg"
    }
  ],
  "subscribers": [
    {
      "id": "uuid2",
      "name": "Сергей Лещев",
      "avatar_path": "/avatars/uuid2.jpg"
    }
  ]
}
```

### 7. Subscribe

Подписаться на пользователя.

```
POST /v1/users/{user_id}/subscribe/{subscription_id}
```

### 8. UnSubscribe

Отписаться от пользователя.

```
DELETE /v1/users/{user_id}/subscribe/{subscription_id}
```

## Управление аватарами

### 9. UploadAvatar

Загрузка аватара.

```
POST /v1/users/avatar
{
  "image": "base64_encoded_image",
  "filename": "avatar.jpg",
  "userId": "uuid"
}
```

**Response:** `{ "link": "/avatars/uuid_timestamp.jpg" }`

### 10. RemoveAvatar

Удаление аватара.

```
DELETE /v1/users/{userId}/avatar?filename=avatar.jpg
```

## Модели данных

### UserProfile

```protobuf
message UserProfile {
  string id = 1;
  string name = 2;
  string email = 3;
  string avatar_path = 4;
  string bio = 5;
  int32 experience_level = 6;  // 1-5 уровень опыта
  bool is_verified = 7;         // Верифицированный аккаунт
  bool is_public = 8;           // Публичный профиль
  google.protobuf.Timestamp created_at = 9;
}
```

### AccountSettings

```protobuf
message AccountSettings {
  Language language = 1;        // RU | ENG
  Availability availability = 2; // PUBLIC | PRIVATE
}
```

### SubscriptionUser

Упрощенная модель для списка подписок.

```protobuf
message SubscriptionUser {
  string id = 1;
  string name = 2;
  string avatar_path = 3;
}
```

## Бизнес-логика

### Уровни опыта (experience_level)

- 1 = Новичок (0-1 год)
- 2 = Любитель (1-3 года)
- 3 = Опытный (3-7 лет)
- 4 = Эксперт (7-15 лет)
- 5 = Мастер (15+ лет)

### Верификация (is_verified)

Значок ✓ для:

- Профессиональных рыбаков
- Гидов
- Магазинов снастей
- Популярных блогеров

### Приватность (is_public/availability)

**PUBLIC:**

- Профиль виден всем
- Посты в общей ленте
- Можно найти в поиске

**PRIVATE:**

- Профиль только для подписчиков
- Посты только для подписчиков
- Не отображается в поиске

## Технический стек

**Рекомендуется:**

- **База данных:** PostgreSQL
- **Хранилище файлов:** S3 / MinIO (аватары)
- **Кэш:** Redis (профили, подписки)
- **Resize изображений:** imaging/go

## Интеграция

**Feed Service:**

- Запрашивает список подписок (`GetSubscriptions`)
- Использует для формирования ленты

**Post Service:**

- Получает информацию об авторе поста
- Проверяет is_public для видимости

## Генерация кода

```bash
make generate-user-api
# или
make generate
```

## Метрики

**Поля для аналитики:**

- Количество подписок/подписчиков
- Уровень активности (посты за неделю)
- Популярность (views профиля)

## Roadmap

- [x] v1: Базовый профиль
- [x] v2: Система подписок
- [x] v3: Настройки аккаунта
- [x] v4: Загрузка аватаров
- [ ] v5: Статистика профиля
- [ ] v6: Достижения и badges
- [ ] v7: Приватные сообщения
