# Архитектура потоковой обработки данных (Kafka)

## Топики Kafka и их назначение

| Топик | Тип событий | Партиции | Retention |
|-------|-------------|----------|------------|
| `campaign-events` | CampaignCreated, CampaignUpdated, CampaignDeleted, BudgetChanged | 6 | 7 дней |
| `bidding-events` | BidRequestReceived, BidMade | 12 | 3 дня |
| `stats-events` | Impression, Click, Conversion | 24 | 30 дней (для пересчёта аналитики) |
| `financial-events` | PaymentProcessed, BalanceChanged | 6 | 90 дней (аудит) |

**Почему разные топики**: изоляция нагрузок и сроков хранения. События статистики требуют долгого хранения, финансовые – ещё дольше, bidding – короткий.

## Схема событий

Используем **Avro** с Schema Registry.

**Преимущества**:
- Компактный бинарный формат (быстрее JSON).
- Эволюция схем (можно добавлять поля без обратной совместимости).
- Schema Registry централизует управление схемами.

**Пример схемы события `Impression` (Avro)**:

```json
{
  "type": "record",
  "name": "Impression",
  "namespace": "com.adscale.events",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "timestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "campaign_id", "type": "string"},
    {"name": "bid_request_id", "type": "string"},
    {"name": "user_agent", "type": "string"},
    {"name": "ip", "type": "string"},
    {"name": "site_id", "type": "string"},
    {"name": "cost", "type": "double"}
  ]
}