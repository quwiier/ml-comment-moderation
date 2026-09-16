# Модель данных

Диаграмма показывает логические сущности и кардинальности. Чувствительность, типы и размещение полей описаны в [docs/05-data.md](../docs/05-data.md).

```mermaid
erDiagram
    USER ||--o{ COMMENT : writes
    CONTENT_ITEM ||--o{ COMMENT : contains
    COMMENT ||--o{ PREDICTION : receives
    MODEL ||--o{ MODEL_VERSION : versions
    MODEL_VERSION ||--o{ PREDICTION : produces
    COMMENT ||--o{ MODERATION_DECISION : has_history
    PREDICTION o|--o{ MODERATION_DECISION : informs
    MODERATOR o|--o{ MODERATION_DECISION : makes
    MODERATION_DECISION o|--o{ MODERATION_DECISION : supersedes
    MODERATION_DECISION ||--o{ APPEAL : challenged_by
    USER ||--o{ APPEAL : submits
    COMMENT ||--o{ LABEL : labelled
    MODERATOR o|--o{ LABEL : assigns
    DATASET_VERSION ||--o{ DATASET_ITEM : contains
    LABEL ||--o{ DATASET_ITEM : included_as
    COMMENT ||--o{ FEEDBACK_EVENT : generates
    PREDICTION ||--o{ FEEDBACK_EVENT : compared_in
    MODERATION_DECISION ||--o{ FEEDBACK_EVENT : finalizes
    COMMENT ||--o{ AUDIT_EVENT : audited

    USER { uuid user_id PK }
    CONTENT_ITEM { uuid content_item_id PK string type datetime created_at }
    COMMENT { uuid comment_id PK uuid content_item_id FK uuid user_id FK string text_ref string language datetime created_at }
    MODEL { uuid model_id PK string name string task }
    MODEL_VERSION { uuid model_version_id PK uuid model_id FK string semantic_version string artifact_uri string checksum string status }
    PREDICTION { uuid prediction_id PK uuid comment_id FK uuid model_version_id FK json scores_json json explanation_json datetime inferred_at int latency_ms }
    MODERATION_DECISION { uuid decision_id PK uuid comment_id FK uuid prediction_id FK uuid parent_decision_id FK string actor_type string action string policy_version datetime created_at }
    MODERATOR { uuid moderator_id PK string team boolean active }
    APPEAL { uuid appeal_id PK uuid decision_id FK uuid user_id FK string status string reason_ref datetime created_at }
    LABEL { uuid label_id PK uuid comment_id FK string class float value string source uuid moderator_id FK }
    DATASET_VERSION { uuid dataset_version_id PK string version string manifest_uri string checksum datetime cutoff_at }
    DATASET_ITEM { uuid dataset_version_id PK_FK uuid label_id PK_FK string split datetime included_at }
    FEEDBACK_EVENT { uuid feedback_event_id PK uuid comment_id FK uuid prediction_id FK uuid decision_id FK string event_type datetime created_at }
    AUDIT_EVENT { uuid audit_event_id PK string aggregate_type uuid aggregate_id string actor_ref string action string payload_hash string prev_hash datetime occurred_at }
```

Операционная БД хранит workflow metadata, event bus — события, warehouse — обезличенные факты, object storage — raw по ссылке/датасеты/модели, audit store — append-only события. Текст не дублируется в Prediction, Decision или AuditEvent.
