# 4. Production-архитектура

## Нагрузка

Факт условия: 500 000 комментариев/сутки. Среднее: `500 000 / 86 400 = 5,79 RPS`. **Допущение A-02:** пик равен 10× среднего, то есть около 58 RPS. Запас 2× задаёт проектную пропускную способность 120 RPS. При среднем размере события 4 KiB входящий поток составляет около 2 GiB/сутки до репликации и служебных событий; это допущение уточняется измерением.

## Контуры

Online: Social Network → API Gateway/Event Producer → Event Bus → Preprocessor → Inference → Policy Engine → Moderation Router → публикация/очередь/временное скрытие. Все этапы передают `event_id`, `comment_id`, trace и версии; consumers идемпотентны.

Offline: обезличенные события и решения → аналитическое/object storage → quality gates → обучение → registry → canary deployment. Production-модель не обучается напрямую на необработанном online-потоке.

Схемы: [architecture.md](../diagrams/architecture.md), процессы: [diagrams/README.md](../diagrams/README.md).

## Компоненты

| Компонент | Вход | Выход/хранение | Масштабирование |
|---|---|---|---|
| API Gateway / producer | HTTPS/gRPC команды платформы | Событие в bus | Stateless replicas, rate limit |
| Event Bus | `CommentCreated/Updated` | Partitioned durable log | Партиции по `comment_id`, ≥3 replicas |
| Preprocessor | Raw event | Нормализованный текст, language | Consumer group, CPU autoscale |
| Inference Service | Текст + допустимый контекст | Scores, explanation, model version | Независимые CPU/GPU replicas, batching |
| Policy Engine | Scores + rules + policy version | `ALLOW/REVIEW/TEMP_HIDE` | Stateless replicas, cached policy |
| Moderation Router | Decision recommendation | Platform action, review task | Consumer group, idempotency key |
| Review UI/API | Review task | Final decision, reason | Web replicas; priority queues |
| Appeal Service | Appeal | Appeal decision, restore action | Stateless + operational DB |
| Feedback Pipeline | Final decisions/audits | Curated labels | Batch/stream workers |
| Training Pipeline | Dataset version | Model artifact + report | Isolated jobs, scheduled/on demand |
| Model Registry | Artifact metadata | Approved model version | HA metadata DB + object storage |
| Operational DB | Tasks, decisions, appeals | Transactional state | Primary + replicas, partition/archive |
| Analytics Warehouse | Sanitized events | Aggregates/dashboards | Date partitioning |
| Object Storage | Datasets/models/reports | Immutable versioned objects | Managed replication/lifecycle |
| Audit Log | Security/moderation events | Append-only evidence | WORM/object lock, separate access |

## Синхронность и деградация

Публикация и приём события разделены: источник получает подтверждение durable enqueue, дальнейшая обработка асинхронна. Внутри inference допустим gRPC/HTTP с timeout и circuit breaker. Если model service недоступен или нарушает latency budget, Policy Engine применяет только детерминированные критические правила, остальные случаи отправляет в ручную очередь; событие повторяется из bus. При недоступности review UI задачи остаются в очереди. При недоступности аналитики online-контур продолжает работу.

Dead-letter queue используется только после ограниченного числа повторов; алерт обязателен, а replay сохраняет исходный event ID. Backpressure: autoscaling по lag, ограничение batch, priority partition для high-risk; при перегрузке запрещено молча отбрасывать события.

## SLO и восстановление

| Показатель | Проектная цель | Измерение |
|---|---|---|
| Availability online routing | 99,9%/месяц | Успешный маршрут / валидные события |
| End-to-end latency | p95 ≤ 800 мс; p99 ≤ 2 с | От durable accept до route |
| Throughput | 120 RPS устойчиво, burst 180 RPS/5 мин | Нагрузочный тест |
| Event loss | ≤ 0,001% подтверждённых событий | Reconciliation producer/bus/sink |
| RTO | ≤ 30 мин | DR exercise |
| RPO operational DB | ≤ 5 мин | Backup/replication drill |
| RPO event bus/audit | близко к 0 для acknowledged writes | Quorum replication verification |

## Хранение и распределённость

Система распределённая, потому что поток событий нужно буферизовать и повторно проигрывать, inference масштабировать независимо от review API, а транзакционные задачи, аналитические сканы, крупные артефакты и неизменяемый аудит имеют разные профили согласованности и стоимости. Компромисс — больше операционной сложности, eventual consistency и необходимость идемпотентности/наблюдаемости.

## Границы доверия и безопасность

Внешний пользовательский ввод считается недоверенным; gateway ограничивает размер/частоту. Межсервисный доступ — mTLS и короткоживущие identities. Текст не пишется в технические логи. Доступ модератора — least privilege, MFA и аудит просмотра. PII отделены от ML-идентификаторов; registry и pipeline не имеют произвольного доступа к production PII. Артефакты подписываются; deployment допускает только approved version.

## Мониторинг

- Инфраструктура: RPS, latency, error, saturation, consumer lag, DLQ, DB locks, cost.
- ML: prevalence, score distribution, calibration, delayed precision/recall, slice quality, drift.
- Продукт: time-to-action, review rate, handling time, complaints, appeals и overturn rate.
- Алерты привязаны к runbook: scale, circuit break, rollback model/policy, replay и incident response.

## Альтернативы

Синхронный монолит проще, но связывает публикацию с latency модели и хуже переносит пики. Общая БД дешевле в начале, но смешивает OLTP, аналитику и WORM-аудит. Для индивидуально спроектированного решения особенно уместны managed event bus, relational DB, object storage и monitoring; конкретный облачный провайдер не фиксируется.
