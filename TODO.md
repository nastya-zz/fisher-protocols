# TODO: Улучшения архитектуры

Список рекомендаций для улучшения архитектуры приложения на основе современных стандартов.

## ✅ Прогресс

**Всего задач:** 15  
**Выполнено:** 1 (6.7%)  
**В работе:** 0  
**Осталось:** 14

**Последнее обновление:** Исправлена уязвимость user impersonation (задача #1)

## 🔴 Критичные (необходимо для production)

### ✅ 1. Убрать user_id из параметров запросов

**Статус:** ✅ **ВЫПОЛНЕНО** (устранена уязвимость user impersonation)

**Проблема:** user_id передается и в JWT, и в параметрах - риск подделки и рассинхронизации.

**Решение (реализовано):**

- ✅ Убран `user_id` из всех Request message
- ✅ Gateway извлекает user_id из JWT и добавляет в gRPC metadata
- ✅ Сервисы читают user_id только из metadata

**Изменённые файлы:**

- ✅ `feed_v1/feed.proto` - убран user_id из GetFeedRequest, GetLocationFeedRequest, GetTrendingFeedRequest, GetRecommendationsRequest
- ✅ `post_v1/post.proto` - убран из CreatePostRequest, AddLikeRequest, AddCommentRequest, RemoveLikeRequest, RemoveCommentRequest
- ✅ `user_v1/user.proto` - убран из UpdateProfile, SubscribeRequest, UpdateSettingsRequest, ResetSettingsRequest, UploadAvatarRequest, RemoveAvatarRequest
- ✅ HTTP endpoints обновлены (убран {user_id} из path где нужно)

**Результат:**

- 🔒 Клиент НЕ может подделать user_id
- 🔒 user_id гарантированно соответствует JWT токену
- 🔒 Единый источник правды для user_id (JWT)

**Приоритет:** 🔴 HIGH  
**Сложность:** Medium  
**Время:** 2-3 часа → ✅ Выполнено

---

### 2. Кэширование валидации токенов

**Проблема:** Auth Service становится bottleneck - каждый запрос требует ValidateToken.

**Решение:**

1. Gateway валидирует JWT локально (проверка signature + expiration)
2. Redis кэш для revoked tokens (blacklist)
3. Auth.ValidateToken только для критичных операций (смена пароля, удаление и т.д.)

**Реализация:**

```go
// В gateway middleware
func validateTokenLocal(token string, redisClient *redis.Client) (*Claims, error) {
  // 1. Парсинг JWT локально
  // 2. Проверка signature
  // 3. Проверка expiration
  // 4. Проверка в Redis blacklist
}
```

**Приоритет:** 🔴 HIGH  
**Сложность:** Medium  
**Время:** 4-5 часов

---

### 3. Добавить Observability (Tracing, Metrics)

**Проблема:** Нет распределенного трейсинга и метрик в proto-контрактах.

**Решение:**
Создать `common_v1/observability.proto`:

```protobuf
message RequestContext {
  string request_id = 1;      // Уникальный ID запроса
  string trace_id = 2;        // Distributed tracing (OpenTelemetry)
  string correlation_id = 3;  // Связь между связанными запросами
  google.protobuf.Timestamp timestamp = 4;
  string user_agent = 5;
  string client_ip = 6;
}

message ResponseMetrics {
  int32 response_time_ms = 1;
  int32 db_queries_count = 2;
  int32 cache_hit_count = 3;
  string service_version = 4;
}
```

Добавить в каждый Request/Response.

**Приоритет:** 🔴 HIGH  
**Сложность:** High  
**Время:** 1 день

---

### 4. Rate Limiting

**Проблема:** Нет защиты от DDoS и abuse.

**Решение:**

1. Добавить в proto:

```protobuf
message RateLimitInfo {
  int32 limit = 1;          // Лимит запросов
  int32 remaining = 2;      // Осталось
  int64 reset_at = 3;       // Когда сбросится (unix timestamp)
  string policy = 4;        // "100 per hour" или "10 per minute"
}

message GetFeedResponse {
  repeated FeedPost posts = 1;
  Pagination pagination = 2;
  RateLimitInfo rate_limit = 10;
}
```

2. Реализовать в gateway через Redis (sliding window algorithm)

**Приоритет:** 🔴 HIGH  
**Сложность:** Medium  
**Время:** 1 день

---

## 🟡 Важные (желательно в ближайшее время)

### 5. Создать common_v1 для переиспользуемых типов

**Проблема:** Дублирование моделей (User, Media) между сервисами.

**Решение:**
Создать `common_v1/models.proto`:

```protobuf
syntax = "proto3";

package common_v1;

message User {
  string id = 1;
  string username = 2;
  string avatar_url = 3;
  bool is_verified = 4;
}

message Media {
  MediaType type = 1;
  string url = 2;
  string thumbnail_url = 3;
  int32 width = 4;
  int32 height = 5;
}

message Location {
  double latitude = 1;
  double longitude = 2;
  string place_name = 3;
}
```

Импортировать в других сервисах:

```protobuf
import "common_v1/models.proto";
```

**Приоритет:** 🟡 MEDIUM  
**Сложность:** Medium  
**Время:** 3-4 часа

---

### 6. Idempotency Keys

**Проблема:** Нет защиты от дублей при создании постов/лайков.

**Решение:**
Добавить во все POST/PUT операции:

```protobuf
message CreatePostRequest {
  string idempotency_key = 1;  // UUID от клиента
  string description = 2;
  // ...
}
```

Сервис:

1. Проверяет idempotency_key в Redis
2. Если существует - возвращает cached response
3. Иначе выполняет и кэширует на 24 часа

**Приоритет:** 🟡 MEDIUM  
**Сложность:** Medium  
**Время:** 4-5 часов

---

### 7. Bulk Operations

**Проблема:** Feed Service делает N запросов к Post Service для получения N постов.

**Решение:**
Добавить batch методы:

```protobuf
// В post_v1/post.proto
rpc GetPostsBatch(GetPostsBatchRequest) returns (GetPostsBatchResponse) {
  option (google.api.http) = {
    post : "/v1/posts/batch"
    body : "*"
  };
}

message GetPostsBatchRequest {
  repeated string post_ids = 1;  // Max 100
}

message GetPostsBatchResponse {
  repeated Post posts = 1;
  map<string, string> errors = 2;  // post_id -> error message
}
```

**Приоритет:** 🟡 MEDIUM  
**Сложность:** Low  
**Время:** 2 часа

---

### 8. Field Masks (выборочная загрузка полей)

**Проблема:** Клиент получает все поля, даже если нужны только name + avatar.

**Решение:**

```protobuf
import "google/protobuf/field_mask.proto";

message GetProfileRequest {
  string id = 1;
  google.protobuf.FieldMask fields = 2;  // ["name", "avatar_path", "bio"]
}
```

Клиент:

```http
GET /v1/users/123/profile?fields=name,avatar_path
```

Экономия: до 70% трафика на мобильных устройствах.

**Приоритет:** 🟡 MEDIUM  
**Сложность:** Medium  
**Время:** 1 день

---

### 9. Pagination с total_count

**Проблема:** UI не знает общее количество (для "1 из 100").

**Решение:**

```protobuf
message Pagination {
  int32 limit = 1;
  int32 offset = 2;
  string cursor = 3;
  int32 total_count = 4;  // ✅ Добавить
}
```

**Компромисс:** total_count дорогой для больших выборок.  
**Решение:** Возвращать только если offset < 1000, иначе null.

**Приоритет:** 🟡 MEDIUM  
**Сложность:** Low  
**Время:** 1 час

---

## 🟢 Желательные (для масштабирования)

### 10. Notification Service

**Описание:** Отдельный сервис для уведомлений.

**Proto:**

```protobuf
service NotificationV1 {
  rpc SendPushNotification(SendPushRequest) returns (SendPushResponse);
  rpc SendEmail(SendEmailRequest) returns (SendEmailResponse);
  rpc GetUserNotifications(GetNotificationsRequest) returns (GetNotificationsResponse);
  rpc MarkAsRead(MarkAsReadRequest) returns (google.protobuf.Empty);
}
```

**События:**

- Новый подписчик → уведомление
- Лайк на пост → уведомление
- Комментарий → уведомление
- Кто-то рядом поймал рыбу → push

**Приоритет:** 🟢 LOW  
**Сложность:** High  
**Время:** 2-3 дня

---

### 11. Soft Delete (мягкое удаление)

**Проблема:** Удаленные данные теряются навсегда.

**Решение:**
Добавить во все основные сущности:

```protobuf
message Post {
  // ...
  google.protobuf.Timestamp deleted_at = 20;
  bool is_deleted = 21;
  string deleted_by = 22;  // user_id кто удалил
}
```

Фильтровать в запросах: `WHERE deleted_at IS NULL`

**Приоритет:** 🟢 LOW  
**Сложность:** Low  
**Время:** 2-3 часа

---

### 12. WebSocket для real-time

**Описание:** Real-time обновления (новые посты в ленте, лайки, комментарии).

**Решение:**

```protobuf
service StreamV1 {
  rpc SubscribeToFeed(SubscribeRequest) returns (stream FeedUpdate);
  rpc SubscribeToPost(SubscribeRequest) returns (stream PostUpdate);
}

message FeedUpdate {
  UpdateType type = 1;  // NEW_POST, NEW_LIKE, NEW_COMMENT
  FeedPost post = 2;
  google.protobuf.Timestamp timestamp = 3;
}
```

**Технологии:** gRPC streaming или WebSocket через gateway.

**Приоритет:** 🟢 LOW  
**Сложность:** Very High  
**Время:** 1 неделя

---

### 13. Analytics Service

**Описание:** Сервис для аналитики и метрик.

**Методы:**

```protobuf
service AnalyticsV1 {
  rpc TrackEvent(TrackEventRequest) returns (google.protobuf.Empty);
  rpc GetUserStats(GetUserStatsRequest) returns (UserStats);
  rpc GetPostPerformance(GetPostPerformanceRequest) returns (PostPerformance);
}
```

**Метрики:**

- Просмотры постов
- Время в приложении
- Популярные места рыбалки
- Популярные виды рыбы

**Приоритет:** 🟢 LOW  
**Сложность:** High  
**Время:** 3-4 дня

---

### 14. Search Service

**Описание:** Полнотекстовый поиск (ElasticSearch/Typesense).

**Методы:**

```protobuf
service SearchV1 {
  rpc SearchPosts(SearchPostsRequest) returns (SearchPostsResponse);
  rpc SearchUsers(SearchUsersRequest) returns (SearchUsersResponse);
  rpc SearchLocations(SearchLocationsRequest) returns (SearchLocationsResponse);
  rpc GetSuggestions(GetSuggestionsRequest) returns (GetSuggestionsResponse);
}
```

**Фильтры:**

- По описанию
- По геолокации (radius search)
- По видам рыбы
- По типам снастей

**Приоритет:** 🟢 LOW  
**Сложность:** High  
**Время:** 1 неделя

---

### 15. Content Moderation Service

**Описание:** Автоматическая модерация контента.

**Методы:**

```protobuf
service ModerationV1 {
  rpc ModeratePost(ModeratePostRequest) returns (ModerationResult);
  rpc ModerateComment(ModerateCommentRequest) returns (ModerationResult);
  rpc ReportContent(ReportContentRequest) returns (google.protobuf.Empty);
  rpc GetReports(GetReportsRequest) returns (GetReportsResponse);
}

message ModerationResult {
  bool is_approved = 1;
  repeated string violations = 2;  // "spam", "nudity", "hate_speech"
  float confidence = 3;  // 0.0-1.0
}
```

**Интеграция:** AWS Rekognition, Google Vision API.

**Приоритет:** 🟢 LOW  
**Сложность:** Very High  
**Время:** 1-2 недели

---

## 📊 Приоритезация

### Sprint 1 (критичное для production)

- [x] #1: Убрать user_id из параметров ✅ **ВЫПОЛНЕНО**
- [ ] #2: Кэширование валидации токенов
- [ ] #3: Observability (tracing)
- [ ] #4: Rate Limiting

**Оценка:** 1.5 недели  
**Прогресс:** 1/4 (25% выполнено)

### Sprint 2 (стабилизация)

- [ ] #5: common_v1 переиспользуемые типы
- [ ] #6: Idempotency Keys
- [ ] #7: Bulk Operations
- [ ] #8: Field Masks
- [ ] #9: Pagination total_count

**Оценка:** 1 неделя

### Sprint 3+ (масштабирование)

- [ ] #10: Notification Service
- [ ] #11: Soft Delete
- [ ] #12: WebSocket real-time
- [ ] #13: Analytics Service
- [ ] #14: Search Service
- [ ] #15: Moderation Service

**Оценка:** 1-2 месяца

---

## 🔧 Технический долг

### Рефакторинг

- Выделить валидацию в отдельные proto validators
- Добавить proto linting (buf.build)
- CI/CD для автоматической генерации кода
- Contract testing между сервисами

### Документация

- OpenAPI/Swagger генерация из proto
- Postman коллекции
- Архитектурные диаграммы (C4 model)
- Runbook для production incidents

### Тестирование

- Integration tests между сервисами
- Load testing (k6, Gatling)
- Chaos engineering (Chaos Monkey)

---

## 📝 Примечания

**Версионирование:**
При breaking changes создавать новую версию (например, `feed_v2`). Поддерживать v1 минимум 6 месяцев.

**Мониторинг:**

- Prometheus + Grafana для метрик
- Jaeger/Tempo для tracing
- ELK/Loki для логов
- Sentry для ошибок

**Документация изменений:**
Каждое изменение proto фиксировать в CHANGELOG.md с указанием breaking changes.
