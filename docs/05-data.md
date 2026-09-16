# 5. Данные и жизненный цикл

## Источники и потребители

Источники: события комментариев и контента, псевдонимизированный профиль риска, жалобы, решения модераторов, апелляции, версии политики и модели. Потребители: online inference/policy, Review UI, аналитика, quality monitoring, feedback и training pipeline. Raw-текст передаётся только компонентам, которым он необходим.

## Логическая модель

| Сущность и ключевые поля | Тип/обязательность | Назначение и чувствительность |
|---|---|---|
| `User(user_id PK)` | UUID, required | Псевдоним; confidential. Связь с реальным ID хранится вне ML-контура. |
| `ContentItem(content_item_id PK, type, created_at)` | UUID, enum, timestamptz; required | Контекст публикации; internal. |
| `Comment(comment_id PK, content_item_id FK, user_id FK, text_ref, language, created_at, deleted_at)` | UUID/FK/ref/text/time | `text_ref` указывает на защищённое raw-хранилище; текст highly sensitive. |
| `Model(model_id PK, name, task)` | UUID/text/enum; required | Логическая модель; internal. |
| `ModelVersion(model_version_id PK, model_id FK, semantic_version, artifact_uri, checksum, status, created_at)` | UUID/FK/text/time | Воспроизводимость; artifact URI confidential. |
| `Prediction(prediction_id PK, comment_id FK, model_version_id FK, scores_json, explanation_json, inferred_at, latency_ms)` | UUID/FK/JSON/time/int | Вероятности и объяснение; sensitive inference. |
| `ModerationDecision(decision_id PK, comment_id FK, prediction_id FK nullable, parent_decision_id FK nullable, actor_type, action, reason_code, policy_version, created_at)` | UUID/FK/enum/text/time | История решений; sensitive. Append-only revisions. |
| `Moderator(moderator_id PK, team, active)` | UUID/text/bool | Псевдоним сотрудника; confidential. |
| `Appeal(appeal_id PK, decision_id FK, user_id FK, status, reason_ref, resolution_decision_id FK nullable, created_at, resolved_at)` | UUID/FK/enum/ref/time | Апелляция; highly sensitive текст по ссылке. |
| `Label(label_id PK, comment_id FK, class, value, source, moderator_id FK nullable, created_at)` | UUID/FK/enum/float/time | Разметка и provenance; sensitive. |
| `DatasetVersion(dataset_version_id PK, version, manifest_uri, checksum, cutoff_at, created_at)` | UUID/text/URI/hash/time | Снимок данных; confidential. |
| `DatasetItem(dataset_version_id PK/FK, label_id PK/FK, split, included_at)` | UUID/FK/enum/time | Связующая сущность состава датасета; фиксирует train/validation/test и lineage. |
| `FeedbackEvent(feedback_event_id PK, comment_id FK, prediction_id FK, decision_id FK, event_type, created_at)` | UUID/FK/enum/time | Связь прогноза с финальным решением. |
| `AuditEvent(audit_event_id PK, aggregate_type, aggregate_id, actor_ref, action, payload_hash, prev_hash, occurred_at)` | UUID/text/hash/time | Неизменяемая цепочка аудита; restricted. |

Полный raw-текст не дублируется в `Prediction`, `ModerationDecision`, аналитике и обычных логах. ER: [data-model.md](../diagrams/data-model.md).

## Кардинальности и история

User и ContentItem имеют много Comment. Comment имеет много Prediction, Label, ModerationDecision и FeedbackEvent. Model имеет много ModelVersion; ModelVersion — много Prediction. DatasetVersion включает Label через DatasetItem, где зафиксирован split. Решение ссылается на использованный Prediction и предыдущее решение, поэтому override/апелляция не перезаписывают историю. Appeal относится к исходному решению и при разрешении — к новому решению.

## Физическое размещение

| Слой | Что хранит | Почему отдельно |
|---|---|---|
| Операционная PostgreSQL-совместимая БД | задачи, решения, апелляции, metadata | ACID, индексы, согласованная история |
| Event bus | immutable domain events на ограниченный срок | буфер, replay, независимые consumers |
| Аналитическое хранилище | обезличенные факты/агрегаты | колоночные сканы без нагрузки на OLTP |
| Object storage | raw по ссылкам, datasets, models, reports | крупные immutable объекты, versioning/lifecycle |
| Audit store | хешированные append-only события | WORM, отдельная роль и срок хранения |

## Партиционирование и индексы

Event bus партиционируется по `comment_id` для порядка событий комментария. Большие таблицы Prediction/Decision/Audit — по месяцу `created_at`; горячие partitions имеют индексы `(comment_id, created_at desc)`, `(status, priority, created_at)` для очереди, `(model_version_id, inferred_at)` для мониторинга. Уникальный `(source, event_id)` обеспечивает идемпотентность. Аналитика партиционируется по дате и кластеризуется по policy/model version.

## Сроки хранения и удаление

Проектные допущения до privacy/legal review:

- raw-текст и защищённые ссылки — 90 дней, затем удаление или необратимая анонимизация;
- operational decisions/appeals — 1 год;
- обезличенные агрегаты — 2 года;
- event bus replay — 7 дней;
- audit — 3 года либо обязательный юридический срок;
- dataset/model artifacts — пока версия поддерживается + 1 год, с legal hold override.

Удаление по субъекту проходит через deletion workflow: resolve pseudonym вне ML-контура, удалить raw, заменить идентификатор необратимым tombstone, исключить записи из будущих datasets; уже обученную модель оценивают на риск memorization и переобучают при установленной обязанности. Факт удаления также аудируется без сохранения удаляемого PII.

## Версионирование и качество

Dataset manifest фиксирует source windows, query/code version, schema, label taxonomy, exclusions, row counts, checksums и lineage. Model registry связывает модель с dataset version, code commit, hyperparameters, metrics, approver и policy compatibility. Gates: schema/null/range, referential integrity, duplicates, leakage, prevalence/drift, annotation agreement, text-language и отсутствие прямых identifiers в аналитике.
