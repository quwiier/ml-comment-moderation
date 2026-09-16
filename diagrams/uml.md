# UML-комплект

## Компонентная схема

Назначение: показать интерфейсы online-маршрута, проверки человеком, апелляции и ML lifecycle. Mermaid не реализует отдельный `componentDiagram`, поэтому используется эквивалентная компонентная схема.

```mermaid
flowchart LR
  Platform[SocialNetwork] -->|ICommentEvents| Bus[EventBus]
  Bus -->|IEventConsumer| Preprocessor
  Preprocessor -->|IInference| InferenceService
  ModelRegistry -->|IModelArtifact| InferenceService
  InferenceService -->|IPrediction| PolicyEngine
  PolicyEngine -->|IRoutingDecision| ModerationRouter
  ModerationRouter -->|IContentAction| Platform
  ModerationRouter -->|IReviewTask| ReviewService
  ModeratorUI -->|IReviewAPI| ReviewService
  AppealService -->|IReviewAPI| ReviewService
  ReviewService -->|IDecisionRepository| OperationalDB[(OperationalDB)]
  ModerationRouter -->|IAuditWriter| AuditLog[(AuditLog)]
  ReviewService -->|IFeedbackPublisher| FeedbackPipeline
  FeedbackPipeline --> ObjectStorage[(ObjectStorage)]
  ObjectStorage --> TrainingPipeline
  TrainingPipeline --> ModelRegistry
```

## Диаграмма классов

Назначение: зафиксировать ключевые доменные типы. История решений реализована связью `parentDecision`; status апелляции и action решения — перечисления.

```mermaid
classDiagram
  class User { +UUID userId }
  class ContentItem { +UUID contentItemId +ContentType type +Instant createdAt }
  class Comment { +UUID commentId +String textRef +String language +Instant createdAt +markDeleted() }
  class Model { +UUID modelId +String name +ModelTask task }
  class ModelVersion { +UUID modelVersionId +String semanticVersion +URI artifactUri +String checksum +ModelStatus status +approve() }
  class Prediction { +UUID predictionId +Map scores +Map explanation +Instant inferredAt +int latencyMs +maxRisk() float }
  class ModerationDecision { +UUID decisionId +ActorType actorType +DecisionAction action +String reasonCode +String policyVersion +Instant createdAt +supersede() }
  class Moderator { +UUID moderatorId +String team +review() ModerationDecision }
  class Appeal { +UUID appealId +AppealStatus status +String reasonRef +resolve() }
  class Label { +UUID labelId +String className +float value +LabelSource source }
  class DatasetVersion { +UUID datasetVersionId +String version +URI manifestUri +String checksum +seal() }
  class FeedbackEvent { +UUID feedbackEventId +FeedbackType eventType +Instant createdAt }
  class AuditEvent { +UUID auditEventId +String action +String payloadHash +String prevHash +Instant occurredAt }
  class DecisionAction { <<enumeration>> ALLOW REVIEW TEMP_HIDE REMOVE RESTORE }
  class AppealStatus { <<enumeration>> OPEN UPHELD OVERTURNED }
  class ModelStatus { <<enumeration>> CANDIDATE APPROVED RETIRED }

  User "1" --> "0..*" Comment : writes
  ContentItem "1" --> "0..*" Comment : contains
  Model "1" --> "1..*" ModelVersion : versions
  Comment "1" --> "0..*" Prediction
  ModelVersion "1" --> "0..*" Prediction
  Comment "1" --> "0..*" ModerationDecision
  Prediction "0..1" --> "0..*" ModerationDecision : informs
  Moderator "0..1" --> "0..*" ModerationDecision : makes
  ModerationDecision "0..1" --> "0..*" ModerationDecision : parentDecision
  ModerationDecision "1" --> "0..*" Appeal
  Comment "1" --> "0..*" Label
  DatasetVersion "0..*" --> "0..*" Label : includes
  Prediction "1" --> "0..*" FeedbackEvent
  ModerationDecision "1" --> "0..*" FeedbackEvent
  Comment "1" --> "0..*" AuditEvent
```

## Диаграмма последовательности

Назначение: обработка нового комментария, timeout модели, три маршрута, ручное решение, апелляция и feedback loop. Допущение: публикация подтверждается после durable enqueue; `TEMP_HIDE` обратим.

```mermaid
sequenceDiagram
  autonumber
  actor User
  participant SN as SocialNetwork
  participant Bus as EventBus
  participant Pre as Preprocessor
  participant Inf as InferenceService
  participant Pol as PolicyEngine
  participant Rou as ModerationRouter
  participant Rev as ReviewService
  actor Mod as Moderator
  participant App as AppealService
  participant Fbk as FeedbackPipeline
  participant Aud as AuditLog

  User->>SN: publish(comment)
  SN->>Bus: CommentCreated(eventId)
  Bus-->>SN: durable ack
  Bus->>Pre: deliver event
  Pre->>Inf: predict(normalized text, context)
  alt inference success
    Inf-->>Pre: scores + explanation + modelVersion
    Pre->>Pol: evaluate(prediction, policyVersion)
  else timeout or error
    Inf--xPre: timeout/error
    Pre->>Pol: evaluate(modelUnavailable, criticalRules)
  end
  Pol-->>Rou: route recommendation
  alt ALLOW
    Rou->>SN: keep visible
    opt random audit sample
      Rou->>Rev: create audit task
    end
  else REVIEW or low confidence
    Rou->>Rev: create normal task
  else TEMP_HIDE
    Rou->>SN: temporarily hide
    Rou->>Rev: create priority task
  end
  Rou-->>Aud: append automated decision
  opt manual task exists
    Rev->>Mod: show task + bounded explanation
    Mod-->>Rev: final decision + reason
    Rev->>SN: apply final action
    Rev-->>Aud: append human decision
    Rev-->>Fbk: FeedbackEvent(prediction, finalDecision)
  end
  opt user appeals
    User->>App: submit appeal
    App->>Rev: priority re-review
    Mod-->>Rev: uphold or overturn
    alt overturned
      Rev->>SN: restore content
    end
    Rev-->>Aud: append appeal resolution
    Rev-->>Fbk: corrected FeedbackEvent
  end
  Fbk->>Fbk: validate and build DatasetVersion
```

## Проверка согласованности

Имена `InferenceService`, `PolicyEngine`, `ModerationRouter`, `ReviewService`, `AppealService`, `FeedbackPipeline` совпадают с production-архитектурой. Доменные сущности совпадают с ER; дополнительные перечисления описывают статусы и не являются отдельными хранилищами.

