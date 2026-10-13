
Декомпозируются контейнеры `Scoring Service` и `Policy & Queue Orchestrator`

```mermaid
flowchart LR
    events["Container<br/><b>Event Ingestion</b>"]
    fs["Container<br/><b>Feature Store</b>"]
    registry["Container<br/><b>Model Registry</b>"]
    queue["Container<br/><b>Review Queue</b>"]
    audit["Container<br/><b>Monitoring & Audit</b>"]

    subgraph runtime["Контейнеры Scoring Service + Policy Orchestrator"]
      consumer["Component<br/><b>Request Consumer</b><br/>Дедупликация и correlation ID"]
      schema["Component<br/><b>Schema Validator</b><br/>Версия и обязательные поля"]
      assembler["Component<br/><b>Feature Assembler</b><br/>Point-in-time join"]
      dq["Component<br/><b>Data Quality / OOD Gate</b><br/>missing, stale, cold start, OOD"]
      model["Component<br/><b>Model Adapter</b><br/>Обучаемый ranking/anomaly score"]
      calibrator["Component<br/><b>Calibration & Explanation</b><br/>score, bands, reason codes"]
      output["Component<br/><b>Output Contract Validator</b><br/>JSON Schema, ranges, source refs"]
      rules["Component<br/><b>Mandatory Rule Evaluator</b><br/>Обычный код; правила внутреннего контроля"]
      policy["Component<br/><b>Prioritization Policy</b><br/>top-K, abstain, conflict, quotas"]
      sampler["Component<br/><b>Control Sampler</b><br/>Вероятностная/стратифицированная выборка"]
      publisher["Component<br/><b>Queue Publisher</b><br/>Idempotent publish и retry"]
    end

    events --> consumer
    consumer --> schema
    schema --> assembler
    fs -. "TB-1: признаки" .-> assembler
    assembler --> dq
    dq --> model
    registry -. "TB-3: веса и schema" .-> model
    model --> calibrator
    calibrator --> output
    schema --> rules
    output --> policy
    rules --> policy
    dq --> policy
    sampler --> policy
    policy --> publisher
    publisher -. "TB-4: только валидированный кейс" .-> queue
    consumer --> sampler
    schema --> audit
    dq --> audit
    output --> audit
    policy --> audit
```

## Порядок выполнения

1. Обычный код валидирует событие, устраняет повторы и собирает признаки
   строго на момент операции
2. Gate проверяет полноту, freshness, область применимости и наличие истории.
3. Только обучаемое звено рассчитывает ranking/anomaly score
4. Детерминированный слой калибрует контракт, выполняет обязательные правила,
   обрабатывает расхождения/воздержание и резервирует контрольную выборку
5. В очередь публикуется рекомендация на ручную проверку; бизнес-решение
   принимает специалист в Case Management

Retrieval и агентские инструменты отсутствуют: для таблично-временного
скоринга они не улучшают требуемый контракт. Текст назначения преобразуется
версионированным локальным encoder/feature extractor без внешнего поиска и без
права выполнять действия
