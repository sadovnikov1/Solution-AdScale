
---

## Файл 4: api-gateway.md (дизайн API Gateway для DSP)

```markdown
# API Gateway для DSP-интеграции — дизайн

## Назначение

API Gateway (Kong) — единая точка входа для DSP-партнёра. Он принимает OpenRTB-запросы, аутентифицирует партнёра, ограничивает частоту запросов, маршрутизирует трафик в Bidding Service и собирает метрики.

## Почему Kong

- Высокая производительность (основан на Nginx/OpenResty) — критично для 18 000 RPS
- Встроенные плагины: rate limiting, JWT, key-auth, circuit breaker, prometheus
- Горизонтальное масштабирование (stateless, база конфигурации в PostgreSQL/Cassandra)
- Администрирование через REST API (можно автоматизировать)

## Конфигурация маршрутизации

**Один эндпоинт для DSP:**

| Параметр | Значение |
|----------|----------|
| Путь | `/rtb/bid` |
| Метод | POST |
| Upstream URL | `http://bidding-service:8080/v1/bid` |
| Таймаут | 95 мс (чтобы успеть до 100 мс) |

## Аутентификация

Используется API Key (простейший вариант для B2B-интеграции).

**Плагин Kong:** `key-auth`

**Процесс:**
1. DSP-партнёр включает API-ключ в заголовок запроса: `apikey: dsp_12345_secret`
2. Kong проверяет ключ в своей конфигурации (или через внешний сервис)
3. При неверном ключе → HTTP 401 Unauthorized

Для более высокого уровня безопасности (если потребуется) — mTLS (плагин `mtls-auth`).

## Rate Limiting (ограничение частоты запросов)

**Плагин Kong:** `rate-limiting`

**Политика:** 1000 запросов в секунду для каждого DSP-партнёра (среднее 18 000 RPS / 18 DSP = 1000). При превышении лимита Kong возвращает HTTP 429 Too Many Requests и добавляет заголовок `Retry-After`.

**Хранение счётчиков:** Redis (чтобы лимиты работали в кластере Kong)

## Circuit Breaker (автоматический размыкатель)

**Плагин Kong:** `circuit-breaker`

**Настройки:**
- Таймаут вызова Bidding Service: 95 мс (после этого размыкаем цепь)
- Порог ошибок: 50% ошибок за 10 секунд
- Время разомкнутого состояния: 30 секунд
- Полуоткрытое состояние: 1 запрос для проверки

**Действие при разомкнутой цепи:** Kong возвращает fallback-ответ (дефолтный no-bid) без вызова Bidding Service.

## Мониторинг времени отклика

**Плагин Kong:** `prometheus`

Метрики, доступные в Prometheus:
- `kong_http_request_duration_seconds` — гистограмма латенции
- `kong_http_status_code_total` — количество ответов по HTTP-кодам
- `kong_bandwidth_bytes` — трафик

**Алерты:**
- P95 latency > 80 мс в течение 2 минут → предупреждение (можем не уложиться в 100 мс после вызова Bidding)
- Ошибки 5xx > 1% за минуту → критический алерт

## Пример конфигурации Kong (через declarative config)

```yaml
_format_version: "3.0"
services:
  - name: bidding-service
    url: http://bidding-service:8080
    routes:
      - name: dsp-bid-route
        paths:
          - /rtb/bid
        methods:
          - POST
    plugins:
      - name: key-auth
      - name: rate-limiting
        config:
          minute: 1000
          policy: redis
          redis_host: redis-cluster
      - name: circuit-breaker
        config:
          timeout: 95000  # 95 ms в микросекундах
          threshold: 50
          window_size: 10
          window_interval: 10
          fallback_body: '{"status": "no-bid"}'
      - name: prometheus
        config:
          per_consumer: true
consumers:
  - username: dsp-partner-1
    keyauth_credentials:
      - key: dsp_12345_secret