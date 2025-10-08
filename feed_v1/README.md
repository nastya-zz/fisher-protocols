# Feed Service (Сервис ленты постов)

## Описание

Сервис для формирования персонализированной ленты постов для приложения рыбаков.

## Архитектура

**Гибридная модель (Push + Pull):**

- Push-модель для обычных пользователей (<10k подписчиков)
- Pull-модель для популярных авторов (>10k подписчиков)
- Event-driven обработка через `OnPostCreated`

## Методы API

### 1. GetFeed

Основная лента подписок пользователя.

```
GET /v1/feed?user_id={id}&limit=20
```

**Фичи:**

- Сортировка: RECENT, POPULAR, RELEVANT
- Фильтры: виды рыбы, снасти, даты
- Cursor-based пагинация

### 2. GetLocationFeed

Лента по геолокации (посты рядом с вами).

```
POST /v1/feed/location
{
  "center": {"latitude": 55.75, "longitude": 37.57},
  "radius_km": 50
}
```

**Кейсы:**

- Поиск рыбных мест рядом
- Узнать что клюет в данном водоеме
- Найти рыбаков поблизости

### 3. GetTrendingFeed

Популярные посты (трендовое).

```
GET /v1/feed/trending?scope=REGIONAL&period=WEEK
```

**Scope:** GLOBAL | REGIONAL | COUNTRY
**Period:** TODAY | WEEK | MONTH

### 4. GetRecommendations

Персональные рекомендации на основе интересов.

```
GET /v1/feed/recommendations?user_id={id}
```

## Внутренние методы

### OnPostCreated

Event handler для обработки создания постов.

- Получает событие от Post Service
- Обновляет кэш лент подписчиков (fan-out on write)

### RefreshUserFeed

Принудительное обновление кэша ленты пользователя.

## Технический стек

**Рекомендуется:**

- **Кэш:** Redis Sorted Sets (ключ: `feed:{user_id}`, score: timestamp)
- **Геопоиск:** PostGIS или ElasticSearch с geo_distance
- **Message Queue:** Kafka/RabbitMQ для событий
- **Персонализация:** Ranking алгоритм (позже ML)

## Зависимости

- **User Service:** подписки пользователя
- **Post Service:** посты, лайки, комментарии
- **Auth Service:** валидация токенов

## Генерация кода

```bash
make generate-feed-api
# или
make generate
```

## Метрики производительности

**Цели:**

- P95 latency < 200ms для GetFeed
- P95 latency < 300ms для GetLocationFeed
- Throughput > 1000 RPS

## Roadmap

- [ ] v1: Базовая лента подписок
- [ ] v2: Геолокационная лента
- [ ] v3: Трендовая лента
- [ ] v4: ML персонализация
- [ ] v5: Real-time обновления (WebSocket)
