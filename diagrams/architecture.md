# Архитектура

## Online inference и offline training

Сплошные стрелки — синхронные HTTPS/gRPC вызовы, пунктирные — асинхронные события/пакеты. Online не зависит от доступности training-контура.

```mermaid
flowchart LR
  U[Пользователь] -->|HTTPS| SN[Социальная сеть]
  SN -->|sync: publish| GW[API Gateway / Event Producer]
  GW -. CommentEvent .-> BUS[(Event Bus)]
  BUS -.-> PRE[Preprocessor]
  PRE -->|gRPC| INF[Inference Service]
  REG[(Model Registry)] -->|approved artifact| INF
  INF --> POL[Policy Engine]
  POL --> ROUTE[Moderation Router]
  ROUTE -->|ALLOW| SN
  ROUTE -->|REVIEW| Q[(Review Queue)]
  ROUTE -->|TEMP_HIDE| SN
  ROUTE -. priority task .-> Q
  Q --> UI[Moderator UI / API]
  UI --> ODB[(Operational DB)]
  UI --> ROUTE
  U -->|appeal| APP[Appeal Service]
  APP --> ODB
  APP --> UI
  ROUTE -. audit .-> AUD[(Immutable Audit Log)]
  UI -. audit and feedback .-> AUD
  UI -. FeedbackEvent .-> FB[Feedback Pipeline]
  BUS -. sanitized events .-> WH[(Analytics Warehouse)]
  FB --> OBJ[(Object Storage)]
  OBJ --> TRAIN[Training Pipeline]
  TRAIN --> EVAL[Quality / Bias / Security Gates]
  EVAL --> REG
  MON[Monitoring] -. metrics .- INF
  MON -. metrics .- BUS
  MON -. metrics .- UI
```

При timeout/error inference circuit breaker направляет событие в Policy Engine с `model_unavailable`; критические детерминированные правила сохраняются, прочие случаи — `REVIEW`. Подтверждённое событие не отбрасывается: retry, затем DLQ и алерт.

## Размещение и границы доверия

```mermaid
flowchart TB
  subgraph Public[Недоверенная внешняя зона]
    USER[Web / Mobile Client]
  end
  subgraph Edge[Edge trust boundary]
    WAF[WAF + API Gateway]
    API[Social Network API / Event Producer]
  end
  subgraph Online[Private online cluster, multi-AZ]
    BUS[(Managed Event Bus)]
    PP[Preprocessor replicas]
    IS[Inference autoscaling pool]
    PE[Policy + Router replicas]
    MR[Moderator API]
    DB[(Relational DB primary + replica)]
    CACHE[(Policy/model metadata cache)]
  end
  subgraph Data[Restricted data/ML zone]
    OBJ[(Versioned Object Storage)]
    WH[(Analytics Warehouse)]
    AL[(WORM Audit Log)]
    TP[Isolated Training Jobs]
    RG[Model Registry]
  end
  subgraph Ops[Operations]
    OBS[Metrics / Logs / Traces / Alerts]
    CICD[Signed CI/CD + Approval]
  end
  USER --> WAF --> API --> BUS
  BUS --> PP --> IS --> PE --> MR --> DB
  CACHE --> IS
  CACHE --> PE
  BUS -. sanitized .-> WH
  MR -. feedback .-> OBJ
  PE -. audit .-> AL
  OBJ --> TP --> RG
  RG --> CICD --> IS
  Online -. telemetry without raw text .-> OBS
  Data -. telemetry .-> OBS
```

Доступ между зонами — allow-list, workload identity и mTLS. Модератор использует MFA/RBAC; pipeline обучения читает только утверждённые snapshots. Конкретный облачный вендор намеренно не закреплён.
