# Post Service (Сервис постов)

## Описание

Сервис для создания, управления постами и взаимодействия с ними (лайки, комментарии).

## Функциональность

- Создание и управление постами
- Загрузка медиа-контента (фото/видео)
- Лайки и комментарии
- Геолокация постов
- Теги: виды рыбы и снасти

## Методы API

### Управление постами

### 1. CreatePost

Создание нового поста.

```
POST /v1/posts
{
  "user_id": "uuid",
  "description": "Отличная рыбалка! Поймал щуку на 3кг",
  "location": {
    "latitude": 55.751244,
    "longitude": 37.618423
  },
  "media": ["media_id_1", "media_id_2"],
  "fish_type_ids": [1, 5],
  "tackle_type_ids": [3]
}
```

**Response:** объект `Post`

### 2. UploadMedia

Загрузка медиа-файла (перед созданием поста).

```
POST /v1/posts/media
{
  "image": "base64_encoded_data",
  "filename": "photo.jpg",
  "postId": "uuid",
  "is_thumbnail": false,
  "type": "PHOTO"
}
```

**Response:** `{ "id": "media_uuid" }`

**Workflow:**

1. Загрузить медиа → получить `media_id`
2. Создать пост с массивом `media_ids`

### 3. GetPost

Получение поста по ID.

```
GET /v1/posts/{id}
```

**Response:**

```json
{
  "id": "uuid",
  "user": {
    "id": "user_uuid",
    "username": "Иван Рыбаков",
    "avatar_url": "/avatars/user.jpg"
  },
  "description": "Отличная рыбалка!",
  "location": {
    "latitude": 55.751244,
    "longitude": 37.618423
  },
  "media": [...],
  "likes_count": 42,
  "comments_count": 12,
  "fish_types": [...],
  "tackle_types": [...],
  "created_at": "2024-01-01T12:00:00Z"
}
```

### 4. GetPosts

Получение списка постов (с фильтрами).

```
GET /v1/posts?user_id={id}&limit=20
```

### 5. UpdatePost

Обновление поста.

```
PUT /v1/posts/{id}
{
  "description": "Обновленное описание",
  "fish_type_ids": [1, 2],
  "tackle_type_ids": [3],
  "location": {...}
}
```

### 6. DeletePost

Удаление поста.

```
DELETE /v1/posts/{id}
```

## Лайки

### 7. AddLike

Лайкнуть пост.

```
POST /v1/posts/{post_id}/likes
{
  "user_id": "uuid"
}
```

**Response:**

```json
{
  "likes_count": 43,
  "success": true
}
```

### 8. RemoveLike

Убрать лайк.

```
DELETE /v1/posts/{post_id}/likes
{
  "user_id": "uuid"
}
```

### 9. GetLikes

Получить список пользователей, лайкнувших пост.

```
GET /v1/posts/{post_id}/likes
```

**Response:**

```json
{
  "likes": [
    {
      "id": "user1",
      "username": "Петр",
      "avatar_url": "/avatars/user1.jpg"
    }
  ]
}
```

## Комментарии

### 10. AddComment

Добавить комментарий.

```
POST /v1/posts/{post_id}/comments
{
  "user_id": "uuid",
  "content": "Отличный улов!",
  "parent_comment_id": null,  // Для вложенных ответов
  "reply_to_user_id": null
}
```

### 11. RemoveComment

Удалить комментарий.

```
DELETE /v1/posts/comments/{comment_id}
{
  "user_id": "uuid"
}
```

### 12. GetComments

Получить комментарии к посту.

```
GET /v1/posts/{post_id}/comments
```

**Response:**

```json
{
  "comments": [
    {
      "id": "comment_uuid",
      "user": {...},
      "content": "Отличный улов!",
      "created_at": "2024-01-01T12:05:00Z",
      "replies": [
        {
          "id": "reply_uuid",
          "user": {...},
          "content": "Спасибо!",
          "is_reply": true,
          "parent_comment_id": "comment_uuid"
        }
      ]
    }
  ]
}
```

## Модели данных

### Post

```protobuf
message Post {
  string id = 1;
  User user = 2;                    // Автор
  string description = 3;
  LatLng location = 4;
  repeated Media media = 5;
  int32 likes_count = 6;
  int32 comments_count = 7;
  repeated FishType fish_types = 8;
  repeated TackleType tackle_types = 9;
  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}
```

### Media

```protobuf
message Media {
  MediaType type = 1;               // PHOTO | VIDEO
  string url = 2;
  string thumbnail_url = 3;         // Для видео
  int32 width = 4;
  int32 height = 5;
  float duration = 6;               // Для видео (секунды)
  string id = 7;
  google.protobuf.Timestamp created_at = 8;
}
```

### Comment

```protobuf
message Comment {
  string id = 1;
  User user = 2;
  string post_id = 3;
  string content = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
  optional string parent_comment_id = 7;    // Для вложенных
  optional string reply_to_user_id = 8;
  repeated Comment replies = 10;            // Вложенные ответы
  bool is_reply = 11;
}
```

### FishType (Виды рыбы)

```protobuf
message FishType {
  string id = 1;
  string name = 2;            // "Щука", "Окунь", "Карп"
  string description = 3;
}
```

### TackleType (Снасти)

```protobuf
message TackleType {
  string id = 1;
  string name = 2;            // "Спиннинг", "Фидер", "Блесна"
  string description = 3;     // Категория
}
```

## Бизнес-логика

### Загрузка медиа

1. **Максимум 10 фото/видео** на пост
2. **Фото:** max 10MB, форматы: jpg, png, webp
3. **Видео:** max 100MB, форматы: mp4, mov
4. **Автоматический resize:** 1920x1080 для фото
5. **Thumbnails:** генерируются автоматически для видео

### Комментарии

- **Вложенность:** максимум 2 уровня (комментарий → ответ)
- **Лимит:** 500 символов на комментарий
- **Удаление:** автор комментария или автор поста

### Геолокация

- **Приватность:** можно скрыть точные координаты (показывать только регион)
- **Интеграция:** с картами для отображения мест рыбалки

## Технический стек

**Рекомендуется:**

- **База данных:** PostgreSQL (посты, комментарии, лайки)
- **Хранилище медиа:** S3 / MinIO
- **Кэш:** Redis (счетчики лайков/комментариев)
- **Очередь:** Kafka (события создания постов)
- **CDN:** для раздачи медиа-контента

## События

### PostCreatedEvent

При создании поста отправляется в Kafka:

```json
{
  "post_id": "uuid",
  "author_id": "user_uuid",
  "created_at": "2024-01-01T12:00:00Z"
}
```

**Подписчики:**

- **Feed Service** → обновляет ленты подписчиков
- **Notification Service** → отправляет уведомления

## Интеграция

**User Service:**

- Получение информации об авторе
- Проверка is_public для видимости постов

**Feed Service:**

- Запрашивает посты для формирования ленты
- Использует геолокацию для GetLocationFeed

## Генерация кода

```bash
make generate-post-api
# или
make generate
```

## Оптимизация

**Счетчики (likes_count, comments_count):**

- Хранятся денормализованно в таблице posts
- Обновляются через Redis INCR/DECR
- Синхронизация с БД раз в минуту

**Медиа:**

- Сжатие изображений при загрузке
- Генерация нескольких размеров (thumbnail, medium, full)
- Lazy loading в клиенте

## Roadmap

- [x] v1: CRUD постов
- [x] v2: Лайки и комментарии
- [x] v3: Загрузка медиа
- [x] v4: Геолокация
- [x] v5: Теги (рыба, снасти)
- [ ] v6: Репосты
- [ ] v7: Сохранение в избранное
- [ ] v8: Жалобы и модерация
- [ ] v9: Хештеги
