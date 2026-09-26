# Потоковая обработка данных
## Архитектура потоковой обработки событий: топики Kafka 

|Топик|Cобытие| Срок хранения, дней|
|---|---|---|
|campaign-events| CRUD кампаний, изменения бюджета|7|
|bidding-events| события сервиса ставок|2|
|stats-events| клики, показы|10|
|financial-events|события финансовых операций|30| 
|user-profiles-events|события профиля пользователя|30|
|service-monitoring-events|сервисные сообщения для мониторинга и логирования|90|



## Схема событий (Avro или JSON) 
Ко всем топикам применяется гарантия At-least-once, кроме платежных и финансовых операций. Для них - Exactly-once.
Идемпотентность обеспечивается наличием уникального идентификатора в сообщении.
Каждое событие содержит event_id (UUID для deduplication), event_type, timestamp, correlation_id (для tracing), и business_data (impression details, click info и т.д.). Используется JSON формат.


**Пример схемы события**:
***Формат json***
{
  "type": "record",
  "name": "Click",
  "namespace": "adscale.events",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "timestamp", "type": "long", "logicalType": "timestamp"},
    {"name": "campaign_id", "type": "string"},
    {"name": "id_bid_request", "type": "string"},
    {"name": "user_agent", "type": "string"},
    {"name": "ip", "type": "string"},
    {"name": "site", "type": "string"},
    {"name": "price", "type": "double"}
  ]
}

## Группы потребителей  
Один поток событий могут читать несколько независимых consumer groups. Каждая группа ведёт свой offset

- group-stats: Агрегация и запись в ClickHouse
- group-financial: Идемпотентное списание PostgreSQL
- group-cache-compaigns: Обновление кампаний/таргетинга в Redis
- group-analytics: Построение аналитики для отчётов

## Политика хранения
Все сообщения хранятся в топиках заданное кол-во часов (дней) с момента их записи в топик;

